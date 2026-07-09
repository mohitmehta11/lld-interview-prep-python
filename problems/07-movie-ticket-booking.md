# Movie Ticket Booking System

This is the concurrency showcase problem in this knowledge base. The interesting part isn't the entity model (movies/shows/seats are straightforward) — it's that two users can click "book" on the same seat within milliseconds of each other, and the design must make double-booking *structurally impossible*, not just unlikely. See [../04-concurrency-essentials.md](../04-concurrency-essentials.md) for the underlying lock primitives; this file is where you apply them to a real race.

## Requirements

- "Single theater or multi-theater/multi-city?" → **You decide**: model one `Show` (movie + screen + start time) as the booking unit; multi-theater is a pure scaling follow-up (see below), not a core design change.
- "What happens between 'select seats' and 'payment confirmed'?" → **You decide**: seats enter a temporary `LOCKED` state for a bounded hold window (e.g., 5 minutes); payment success transitions to `BOOKED`, payment failure or timeout releases back to `AVAILABLE`. This hold mechanic is the crux of the problem.
- "Can a seat be locked by more than one user at once?" → **You decide**: no — lock acquisition is exclusive and atomic per seat; this is the concurrency requirement the whole design centers on.
- "Seat pricing?" → **You decide**: price varies by seat type (regular/premium/recliner), pluggable per show (a recliner in a weekday matinee vs a weekend premiere may price differently).
- In scope: browsing shows, selecting seats, locking with TTL, payment confirmation/failure, auto-release on timeout, seat-type pricing, notifying on booking confirmation.
- Out of scope: actual payment gateway integration (assume a `PaymentGateway` interface that returns success/failure), seat recommendation, discounts/coupons, refunds/cancellation flow.

## Core entities & relationships

```
Theater
  └─ has-a[*] Screen

Screen
  ├─ has-a[*] Seat
  └─ has-a[*] Show

Show
  ├─ has-a[1] Movie
  ├─ has-a[1] Screen
  └─ has-a[1] startTime

Seat
  ├─ has-a[1] SeatType (enum: REGULAR, PREMIUM, RECLINER)
  ├─ has-a[1] SeatState (enum: AVAILABLE, LOCKED, BOOKED)   -- explicit state machine, not booleans
  └─ has-a[1] Lock (threading.Lock)                          -- one lock per seat, guards state transitions
                                                               -- (or a version counter, for the optimistic
                                                               -- alternative covered below)

SeatLockManager
  ├─ locks[*] Seat
  └─ schedules[*] auto-release timers (one per lock, keyed by seat+bookingId)

PricingStrategy (interface)
  └─ implemented-by[*] one per SeatType (or per show)

Booking
  ├─ has-a[*] Seat
  ├─ has-a[1] User
  └─ has-a[1] BookingStatus (PENDING, CONFIRMED, FAILED)

BookingService
  ├─ uses-a[1] SeatLockManager
  ├─ uses-a[1] PaymentGateway (interface)
  └─ notifies[*] BookingObserver
```

`Seat` owns its own `Lock` rather than the system using one global lock — this is the deliberate concurrency design choice: booking seat A3 and booking seat C7 for two different users must not block each other, only contention on the *same* seat should serialize. State is modeled as an explicit `SeatState` enum with a guarded transition method, not a `bool is_locked` + `bool is_booked` pair — booleans allow invalid combinations (`is_locked=True, is_booked=True`) that an enum with one authoritative value cannot.

## Design patterns applied

- [State](../patterns/03-behavioral-patterns.md#state) — `Seat` transitions `AVAILABLE → LOCKED → BOOKED` (or `LOCKED → AVAILABLE` on timeout/failure); modeling this as an explicit state machine with guarded transitions, rather than boolean flags, is exactly what prevents the class of bug where a seat is simultaneously "locked" and "available" due to a missed flag reset.
- [Strategy](../patterns/03-behavioral-patterns.md#strategy) — `PricingStrategy` varies price by `SeatType` (and could vary further by show/time-of-day) independent of the booking flow; a new seat tier or dynamic/surge pricing rule is a new class, not a new `if` branch in `BookingService`.
- [Observer](../patterns/03-behavioral-patterns.md#observer) — `BookingObserver` implementations (email/SMS/push) get notified on booking confirmation without `BookingService` knowing or caring how notification happens; also usable to push live seat-map updates to other clients viewing the same show.

## Python implementation

This is the primary, fully-worked solution — the code below is what you'd actually write in an interview, not a translation of something written elsewhere first.

```python
from __future__ import annotations

import threading
import uuid
from abc import ABC, abstractmethod
from contextlib import contextmanager
from dataclasses import dataclass, field
from decimal import Decimal
from enum import Enum, auto
from typing import Protocol


class SeatType(Enum):
    REGULAR = auto()
    PREMIUM = auto()
    RECLINER = auto()


class SeatState(Enum):
    AVAILABLE = auto()
    LOCKED = auto()
    BOOKED = auto()


class BookingStatus(Enum):
    PENDING = auto()
    CONFIRMED = auto()
    FAILED = auto()


class SeatUnavailableError(RuntimeError):
    """Raised by SeatLockManager.held() when one or more requested seats can't be acquired."""


class Seat:
    """Owns its own threading.Lock — contention on seat A3 never blocks a booking for seat C7.

    `state` is modeled as a single SeatState, not a pair of booleans, so there is no reachable
    combination that means "locked and booked at once." `version` is bumped on every transition;
    it's unused by the pessimistic path below but is exactly what the optimistic-locking
    alternative (see further down) needs, so it's cheap to carry from the start.
    """

    def __init__(self, seat_id: str, seat_type: SeatType) -> None:
        self.id = seat_id
        self.type = seat_type
        self.state: SeatState = SeatState.AVAILABLE
        self.version: int = 0
        self._locked_by: str | None = None
        self._lock = threading.Lock()

    def __repr__(self) -> str:
        return f"Seat(id={self.id!r}, type={self.type.name}, state={self.state.name}, version={self.version})"

    def try_lock(self, booking_id: str) -> bool:
        """Atomic check-and-transition AVAILABLE -> LOCKED. Returns False if already taken."""
        with self._lock:
            if self.state is not SeatState.AVAILABLE:
                return False
            self.state = SeatState.LOCKED
            self._locked_by = booking_id
            self.version += 1
            return True

    def confirm(self, booking_id: str) -> bool:
        """Called on payment success. Only the booking that holds the lock may confirm it."""
        with self._lock:
            if self.state is not SeatState.LOCKED or self._locked_by != booking_id:
                return False
            self.state = SeatState.BOOKED
            self.version += 1
            return True

    def release(self, booking_id: str) -> None:
        """Called on payment failure or TTL timeout. No-op if already booked or re-locked by someone else."""
        with self._lock:
            if self.state is SeatState.LOCKED and self._locked_by == booking_id:
                self.state = SeatState.AVAILABLE
                self._locked_by = None
                self.version += 1


class SeatLockManager:
    """TTL-based auto-release via one threading.Timer per lock. There's no stdlib equivalent to
    Java's ScheduledExecutorService; threading.Timer is the direct single-process analog. For a
    multi-instance deployment this whole class gets replaced by a distributed lock — see the
    concurrency deep dive below and ../04-concurrency-essentials.md.
    """

    HOLD_SECONDS: float = 5 * 60

    def lock_seats(self, seats: list[Seat], booking_id: str) -> bool:
        """All-or-nothing: if any seat is unavailable, roll back the ones already acquired so a
        partial hold never blocks other users on seats this booking won't get."""
        acquired: list[Seat] = []
        for seat in seats:
            if seat.try_lock(booking_id):
                acquired.append(seat)
            else:
                for taken in acquired:
                    taken.release(booking_id)
                return False
        for seat in seats:
            timer = threading.Timer(self.HOLD_SECONDS, seat.release, args=(booking_id,))
            timer.daemon = True
            timer.start()
        return True

    def confirm_seats(self, seats: list[Seat], booking_id: str) -> None:
        for seat in seats:
            seat.confirm(booking_id)  # the pending release timer becomes a no-op post-confirm

    def release_seats(self, seats: list[Seat], booking_id: str) -> None:
        for seat in seats:
            seat.release(booking_id)

    @contextmanager
    def held(self, seats: list[Seat], booking_id: str):
        """Context-manager wrapper around lock_seats/release_seats: guarantees the hold is
        released if anything inside the `with` block raises, without every call site needing
        its own try/finally. BookingService.book() below still calls lock_seats/release_seats
        directly, since it needs to branch on CONFIRMED vs FAILED rather than propagate an
        exception — this is for call sites that just want "do this with the seats held, and
        clean up no matter what."
        """
        if not self.lock_seats(seats, booking_id):
            raise SeatUnavailableError(f"one or more seats unavailable for booking {booking_id}")
        try:
            yield seats
        except Exception:
            self.release_seats(seats, booking_id)
            raise
```

> **Pythonic idiom note:** `SeatLockManager.held()` uses `@contextmanager` instead of a hand-written class with `__enter__`/`__exit__`. Either works, but a generator-based context manager is the idiomatic choice when the "resource" is a plain acquire/release pair with no extra state to track between calls — it reads top-to-bottom like the code it's replacing, rather than splitting setup and teardown across two methods.

```python
class PricingStrategy(ABC):
    """ABC, not Protocol: this is an extension point we own and expect a bounded, known set of
    implementations to plug into (per-seat-type, per-show, surge pricing) — an internal
    hierarchy, not a boundary to an external system. See the Protocol note on PaymentGateway
    below for the contrasting case.
    """

    @abstractmethod
    def price_for(self, seat: Seat, show: "Show") -> Decimal: ...


class SeatTypePricingStrategy(PricingStrategy):
    def __init__(self, base_prices: dict[SeatType, Decimal]) -> None:
        self.base_prices = base_prices

    def price_for(self, seat: Seat, show: "Show") -> Decimal:
        return self.base_prices[seat.type]


class PaymentGateway(Protocol):
    """typing.Protocol, not ABC: this models an *external* dependency — a real payment
    processor's client object that BookingService only ever calls, never constructs or
    subclasses. Any object with a matching `charge(user_id, amount) -> bool` method satisfies
    this structurally, including a third-party SDK class we don't control and can't retroactively
    make inherit from an ABC we define. That's exactly the case Protocol exists for.
    """

    def charge(self, user_id: str, amount: Decimal) -> bool: ...


class BookingObserver(ABC):
    @abstractmethod
    def on_booking_confirmed(self, booking: "Booking") -> None: ...


@dataclass
class Show:
    id: str
    movie_title: str
    start_time: float


@dataclass
class Booking:
    id: str
    user_id: str
    seats: list[Seat]
    status: BookingStatus = BookingStatus.PENDING
    total_price: Decimal = field(default=Decimal("0"))


class BookingService:
    def __init__(self, lock_manager: SeatLockManager, pricing: PricingStrategy, gateway: PaymentGateway) -> None:
        self.lock_manager = lock_manager
        self.pricing = pricing
        self.gateway = gateway
        self._observers: list[BookingObserver] = []

    def add_observer(self, observer: BookingObserver) -> None:
        self._observers.append(observer)

    def book(self, user_id: str, show: Show, seats: list[Seat]) -> Booking:
        """The end-to-end flow: lock -> price -> charge -> confirm-or-release."""
        booking_id = str(uuid.uuid4())
        booking = Booking(id=booking_id, user_id=user_id, seats=seats)

        if not self.lock_manager.lock_seats(seats, booking_id):
            booking.status = BookingStatus.FAILED  # someone else holds >= 1 requested seat
            return booking

        booking.total_price = sum(
            (self.pricing.price_for(s, show) for s in seats), start=Decimal("0")
        )

        if self.gateway.charge(user_id, booking.total_price):
            self.lock_manager.confirm_seats(seats, booking_id)
            booking.status = BookingStatus.CONFIRMED
            for observer in self._observers:
                observer.on_booking_confirmed(booking)
        else:
            self.lock_manager.release_seats(seats, booking_id)  # payment failed -> release the hold immediately
            booking.status = BookingStatus.FAILED
        return booking
```

> **Pythonic idiom note:** `Show` and `Booking` are `@dataclass`, `Seat` is a plain class. The dataclasses get a free `__init__`, `__repr__`, and `__eq__` because they're value-shaped — two `Booking`s with the same fields really are equal. `Seat` isn't: it owns a `threading.Lock` (not comparable, and you never want two `Seat` objects considered equal just because their fields momentarily match), and its methods enforce invariants (`try_lock`, `confirm`, `release`) rather than just holding data — a dataclass would let a careless caller do `seat.state = SeatState.BOOKED` directly, bypassing the lock entirely. Reach for `@dataclass` for the record-like objects, and a real class the moment a lock, an invariant, or a base class enters the picture.

## Concurrency deep dive: pessimistic locking vs optimistic locking via version numbers

The pessimistic design above (`threading.Lock` per `Seat`) is the right default answer for this problem, but "what's the alternative, and when would you use it instead?" is a near-guaranteed follow-up. See [../04-concurrency-essentials.md](../04-concurrency-essentials.md) for the general primitives; here's how the two strategies map onto this specific problem.

```python
@dataclass
class VersionedSeat:
    """Optimistic-locking alternative to Seat's threading.Lock: no lock is held while the
    caller "thinks" (browses the seat map, fills in payment details) — only the compare-and-swap
    moment at the end is serialized. This maps directly onto a DB-backed design:
    `UPDATE seats SET state = %s, version = %s WHERE id = %s AND version = %s` — a 0-row result
    means someone else moved the version first, and the caller re-reads and retries.
    """

    id: str
    type: SeatType
    state: SeatState = SeatState.AVAILABLE
    version: int = 0


class OptimisticLockConflict(RuntimeError):
    """Raised when a seat's version moved between read and write — the caller lost the race."""


def try_lock_optimistic(
    store: dict[str, VersionedSeat], seat_id: str, expected_version: int, booking_id: str
) -> VersionedSeat:
    """Compare-and-swap against a shared store — the direct in-memory analog of
    `UPDATE seats SET state = %s, version = %s WHERE id = %s AND version = %s`. Raises
    OptimisticLockConflict if the row's version moved since the caller last read it. Note
    that the caller passes `expected_version` from *its own* earlier read, not from `store`
    directly — that gap between "what I read" and "what's committed now" is the race this
    function exists to catch.
    """
    current = store[seat_id]
    if current.version != expected_version or current.state is not SeatState.AVAILABLE:
        raise OptimisticLockConflict(f"seat {seat_id} moved past version {expected_version}")
    updated = VersionedSeat(id=current.id, type=current.type, state=SeatState.LOCKED, version=current.version + 1)
    store[seat_id] = updated
    return updated


def book_with_retry(
    store: dict[str, VersionedSeat], seat_id: str, booking_id: str, max_attempts: int = 3
) -> VersionedSeat:
    """The retry loop every optimistic-locking caller needs: read the current committed
    version fresh from `store`, attempt the compare-and-swap, and on conflict re-read and try
    again, up to a bound — unbounded retry under sustained contention is its own bug (a hot
    seat becomes a busy-retry loop that never terminates)."""
    for attempt in range(1, max_attempts + 1):
        current_version = store[seat_id].version
        try:
            return try_lock_optimistic(store, seat_id, current_version, booking_id)
        except OptimisticLockConflict:
            if attempt == max_attempts:
                raise
    raise AssertionError("unreachable")  # loop always returns or raises
```

**When each wins:**

- **Pessimistic (`threading.Lock`, the primary implementation above)** — right when contention is expected and correctness under contention matters more than throughput under contention: a front-row seat on a premiere night *will* have multiple simultaneous claimants, and you'd rather have the second thread block briefly (or fail fast) than do wasted work. It's simple to reason about and is what the rest of this file assumes. Its limit: an in-process `threading.Lock` only coordinates threads *inside one process* — it does nothing across app-server instances.
- **Optimistic (version number)** — right when contention is expected to be low (most seats, most of the time, aren't being fought over) and you want to avoid holding a lock across something slow, like a network round trip to a payment pre-auth check. Its natural home is a database row (`version` column), which makes it work correctly across *any number of app-server instances* for free, since the compare-and-swap happens in the DB, not in any one process's memory. Its cost: wasted work and a worse user experience on conflict (you retried, or the seat was already gone), and a real risk of a retry storm if many users converge on the exact same seat at once.
- In an interview, the strongest answer is: pessimistic per-seat locking for the primary single-process design (matches the requirements as scoped), explicitly naming the DB-row-with-`version`-column pattern as what you'd reach for the moment the design needs to span multiple instances — see the follow-up question below on exactly that prompt.

## Sample walkthrough

```python
seat_a1 = Seat("A1", SeatType.PREMIUM)
seat_a2 = Seat("A2", SeatType.PREMIUM)
show = Show(id="s1", movie_title="Dune: Part Three", start_time=1_800_000_000.0)


class AlwaysApprove:
    def charge(self, user_id: str, amount: Decimal) -> bool:
        return True


class ConsoleNotifier(BookingObserver):
    def on_booking_confirmed(self, booking: Booking) -> None:
        print(f"Booking {booking.id} confirmed for {booking.user_id} (${booking.total_price})")


service = BookingService(
    SeatLockManager(),
    SeatTypePricingStrategy({SeatType.PREMIUM: Decimal("15.00"), SeatType.REGULAR: Decimal("10.00")}),
    AlwaysApprove(),
)
service.add_observer(ConsoleNotifier())

booking = service.book(user_id="alice", show=show, seats=[seat_a1, seat_a2])
assert booking.status == BookingStatus.CONFIRMED
assert seat_a1.state is SeatState.BOOKED and seat_a2.state is SeatState.BOOKED

# A second user tries the same seats after alice already locked them -> fails atomically
second_booking = service.book(user_id="bob", show=show, seats=[seat_a1, seat_a2])
assert second_booking.status == BookingStatus.FAILED

# Optimistic path, standalone: two "reads" of the same row racing to book it. Both alice and
# bob read version 0 before either writes; alice's compare-and-swap lands first.
store: dict[str, VersionedSeat] = {"B7": VersionedSeat(id="B7", type=SeatType.REGULAR)}

winner = try_lock_optimistic(store, "B7", expected_version=0, booking_id="alice-booking")
assert winner.state is SeatState.LOCKED and winner.version == 1

try:
    try_lock_optimistic(store, "B7", expected_version=0, booking_id="bob-booking")  # bob's read is now stale
    raise AssertionError("expected a conflict")
except OptimisticLockConflict:
    pass  # bob must re-fetch from `store`, see version 1 / LOCKED, and give up or pick another seat
```

## Follow-up questions

- **"Two users click 'book' on the same seat at the exact same instant — walk through what actually happens."** Both threads call `seat.try_lock(booking_id)`; the `threading.Lock` inside `try_lock` serializes them so only one thread's check-`state==AVAILABLE`-then-set-`LOCKED` executes as one atomic unit — the second thread's `try_lock` runs *after* the first has already flipped `state` to `LOCKED`, sees `state is not AVAILABLE`, and returns `False`. The lock is what turns a check-then-act race into a single atomic operation; without it, both threads could pass the `state == AVAILABLE` check before either writes `LOCKED`.
- **"Payment fails after the seat was locked — what happens to the seat?"** `BookingService.book` catches the `False` return from `gateway.charge` and calls `lock_manager.release_seats`, which calls `seat.release(booking_id)` — guarded by `booking_id` match so a stale/late release call can't accidentally free a seat some *other* booking has since re-locked.
- **"User closes the tab / never completes payment — seat held forever?"** The TTL auto-release (`threading.Timer`) fires `release(booking_id)` after `HOLD_SECONDS` regardless of what the user does; `release` is a no-op if the seat was already confirmed, so a booking that completes just before the timer fires is unaffected.
- **"Extend to multiple screens/theaters, and to searching shows by city."** `Theater has-a[*] Screen has-a[*] Show` already scales horizontally — add a `TheaterRepository`/search index keyed by city + movie + date; no change to the locking or booking core, since locking is already scoped to individual `Seat` objects, not global state.
- **"What if the interviewer requires this to work across multiple app server instances (not just multi-threaded within one process)?"** In-process `threading.Lock` objects don't coordinate across machines — the fix is either a distributed lock (Redis `SETNX`/Redlock, or a DB row-level lock via `SELECT ... FOR UPDATE` on the seat row) with the same TTL-hold semantics, or the optimistic version-number approach from the deep dive above, since a `version` column on the seat row naturally extends the same compare-and-swap check across any number of instances without any distributed-lock infrastructure at all. Flag this explicitly as a scope question rather than silently assuming single-instance — it changes the locking primitive entirely.

## Common mistakes on this problem

- Modeling seat availability as a `bool is_booked` (or `is_booked` + `is_locked`) instead of an explicit `SeatState` enum — booleans permit invalid combined states and make the timeout-release transition error-prone to reason about.
- Checking `if seat.state == SeatState.AVAILABLE: seat.state = SeatState.LOCKED` as two separate statements instead of one atomic `try_lock` — this is the check-then-act race that causes the exact double-booking bug the problem is designed to test for; the check and the mutation must happen under the same lock acquisition.
- Using one global lock over the entire seat map for booking — technically prevents double-booking but serializes unrelated bookings for different seats/shows, killing throughput; the fix is per-seat (or per-show) lock granularity.
- Forgetting the auto-release path entirely — a design that locks seats on selection but has no timeout/failure release leaves seats permanently unavailable after an abandoned checkout, which is the first thing an interviewer will probe ("what if the user just closes the browser?").

## Continue

Next: [08-library-management-system.md](08-library-management-system.md)
