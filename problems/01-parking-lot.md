# Parking Lot System

THE canonical LLD problem — mainly because it has just enough variability (vehicle types, pricing schemes, multi-floor layout, concurrency at the last-spot edge case) to justify 3-4 patterns without forcing any of them.

## Requirements

- "How many floors / spots per floor?" → **You decide**: assume a fixed number of floors known at construction time, each with a fixed spot layout. Dynamic floor add/remove is out of scope but the design shouldn't actively block it (floors live in a `list`, not baked into constants).
- "Which vehicle types?" → Motorcycle, Car, Truck, Electric (any vehicle, gets a charging-capable spot).
- "One spot size fits all, or does size matter?" → Spots are Small/Medium/Large; Motorcycle needs Small+, Car needs Medium+, Truck needs Large only, Electric needs Medium+ **and** a charger.
- "Pricing — flat hourly or something fancier?" → Support both flat-hourly and tiered (first N hours at rate X, beyond that rate Y) from day one, via a pluggable strategy — this is the headline extensibility axis for this problem.
- "Multiple entry/exit gates?" → **You decide**: yes, multiple gates issue tickets and settle payment independently; this is what makes the "two vehicles race for the last spot" concurrency follow-up real rather than academic.
- "Payment — cash, card, wallet?" → Out of scope for the core design; model a `PaymentMethod` interface and assume `pay()` succeeds/fails synchronously. Don't build a payment gateway integration.
- "Reservations ahead of time?" → Out of scope — first-come-first-served allocation only.

**In scope:** multi-floor spot inventory, vehicle-to-spot-size matching, ticket issue/close, pluggable pricing, a display board reacting to floor-full events, thread-safe spot allocation.

**Out of scope:** reservations, payment gateway internals, dynamic re-numbering of floors/spots, distributed multi-lot coordination (see the concurrency follow-up for where that line is).

## Core entities & relationships

```
ParkingLot
  ├─ has-a[*] ParkingFloor
  ├─ has-a[1] PricingStrategy (interface)
  └─ has-a[*] DisplayBoard (interface, observer)

ParkingFloor
  └─ has-a[*] ParkingSpot

ParkingSpot
  ├─ has-a[1] SpotSize (enum)
  └─ has-a[0..1] Vehicle

Vehicle
  └─ has-a[1] VehicleType (enum)

ParkingTicket
  ├─ has-a[1] Vehicle
  └─ has-a[1] ParkingSpot

EntryGate / ExitGate
  └─ uses-a[1] ParkingLot
```

`Vehicle` is one concrete class with a `VehicleType` enum field, **not** a `Motorcycle`/`Car`/`Truck` subclass hierarchy. There's no behavior that differs by type — only two data lookups (required spot size, whether it needs a charger) — and an enum-with-behavior (see [../03-python-oop-essentials.md](../03-python-oop-essentials.md)) captures that without the ceremony of four near-empty subclasses. Reach for subclassing only when a type needs genuinely different *logic*, not just different *data* — that's the elevator problem's `Piece` hierarchy in [problems/04](04-tic-tac-toe-and-chess.md), by contrast.

`ParkingSpot` holds a `Vehicle` by composition-of-reference (0..1), not composition-of-lifetime — the spot doesn't own the vehicle's lifecycle, it just tracks current occupancy. `ParkingTicket` is the join entity that survives after the vehicle leaves (for receipts/audit), so it holds copies of the relevant IDs rather than live references.

## Design patterns applied

- [Strategy](../patterns/03-behavioral-patterns.md#strategy) — `PricingStrategy` isolates "how do we bill this stay" from everything else; swapping flat-hourly for tiered-hourly (or adding weekend rates later) touches zero other classes, which is exactly the variability the interviewer is probing when they ask "what if pricing changes."
- [Factory Method](../patterns/01-creational-patterns.md#factory-method) (loosely) — `VehicleFactory.create(type, plate)` centralizes the type→construction mapping so `EntryGate` never has an `if`/`elif` chain on vehicle type. Being honest about the nuance: this is a **static/simple factory**, not the GoF Factory Method (no subclassed creators overriding a factory method) — there's no varying *creation algorithm* here, just one construction path per type, so the heavier pattern would be over-engineering. Worth saying this distinction out loud in an interview.
- [Singleton](../patterns/01-creational-patterns.md#singleton) — **judgment call: do not enforce it.** The natural instinct is a `ParkingLot.instance()` class method with a guarded constructor. Don't do it: a hard-enforced Singleton bakes global mutable state into the class, makes unit tests (two lots, one per test) impossible without ceremony, and violates [DIP](../02-solid-principles.md#d--dependency-inversion-principle) since every collaborator now depends on a concrete global accessor instead of an injected instance. Instead, construct exactly one `ParkingLot` at the composition root (your `main()`/wiring code) and hand it to every gate via constructor injection — "only one instance *exists*" is a deployment fact, not something the class itself needs to police. If you truly need cross-process single-instance guarantees (e.g., two `ParkingLot` processes racing to manage the same physical lot), that's a distributed-coordination problem (leader election / DB row lock), and a language-level Singleton can't solve it anyway — say this if the interviewer pushes on it.
- [Observer](../patterns/03-behavioral-patterns.md#observer) — `ParkingFloor` notifies subscribed `DisplayBoard`s on occupancy changes (spot taken/freed, floor full/available) instead of every caller polling `floor.is_full()`; new subscriber types (a mobile app push, a gate barrier controller) attach without touching `ParkingFloor`.

## Python implementation

This is the full, primary implementation — organized here as one file for readability, but the comment banners below mark where you'd split it into separate modules (`models.py`, `spots.py`, `pricing.py`, `lot.py`) in a real codebase.

```python
# ── models.py: vehicle-side value types ────────────────────────────────────
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from enum import Enum
from threading import Lock
from typing import Protocol
import math
import uuid


class SpotSize(Enum):
    SMALL = 1
    MEDIUM = 2
    LARGE = 3


class VehicleType(Enum):
    """Enum-with-behavior: each member carries the data that used to require
    a `Motorcycle`/`Car`/`Truck` subclass hierarchy, with none of the ceremony."""

    MOTORCYCLE = (SpotSize.SMALL, False)
    CAR = (SpotSize.MEDIUM, False)
    ELECTRIC = (SpotSize.MEDIUM, True)
    TRUCK = (SpotSize.LARGE, False)

    def __init__(self, required_size: SpotSize, needs_charger: bool) -> None:
        self.required_size = required_size
        self.needs_charger = needs_charger


@dataclass(frozen=True, slots=True)
class Vehicle:
    """Immutable value object — a license plate + type never mutates in place;
    a "new" vehicle is always a new `Vehicle`. `frozen=True` also gives a
    correct, hashable `__eq__`/`__hash__` for free, so `Vehicle` can be used
    as a dict key (see `ParkingLot._active_tickets` below)."""

    license_plate: str
    type: VehicleType


class VehicleFactory:
    """Static/simple factory (not GoF Factory Method — see the patterns
    discussion above): centralizes the one construction path per type."""

    @staticmethod
    def create(vehicle_type: VehicleType, license_plate: str) -> Vehicle:
        return Vehicle(license_plate=license_plate, type=vehicle_type)


# ── spots.py: physical spot + floor inventory ──────────────────────────────

class ParkingSpot:
    def __init__(self, spot_id: str, size: SpotSize, has_charger: bool) -> None:
        self.id = spot_id
        self.size = size
        self.has_charger = has_charger
        self._occupant: Vehicle | None = None
        # Per-spot lock: the second line of defense behind ParkingFloor's
        # floor-level lock — see ../04-concurrency-essentials.md.
        self._lock = Lock()

    def fits(self, vehicle: Vehicle) -> bool:
        size_ok = list(SpotSize).index(self.size) >= list(SpotSize).index(
            vehicle.type.required_size
        )
        return size_ok and (not vehicle.type.needs_charger or self.has_charger)

    def try_occupy(self, vehicle: Vehicle) -> bool:
        with self._lock:
            if self._occupant is not None:
                return False
            self._occupant = vehicle
            return True

    def vacate(self) -> None:
        with self._lock:
            self._occupant = None

    @property
    def is_free(self) -> bool:
        return self._occupant is None

    def __repr__(self) -> str:
        state = "free" if self.is_free else f"occupied by {self._occupant.license_plate}"
        return f"ParkingSpot(id={self.id!r}, size={self.size.name}, {state})"


class FloorObserver(ABC):
    @abstractmethod
    def on_floor_full(self, floor_id: str) -> None: ...

    @abstractmethod
    def on_floor_available(self, floor_id: str) -> None: ...


class ParkingFloor:
    def __init__(self, floor_id: str, spots: list[ParkingSpot]) -> None:
        self.id = floor_id
        self.spots = spots
        self._observers: list[FloorObserver] = []
        # Serializes the scan-then-claim sequence at floor granularity — see
        # the concurrency follow-up below for why per-spot locking alone
        # isn't sufficient.
        self._lock = Lock()

    def subscribe(self, observer: FloorObserver) -> None:
        self._observers.append(observer)

    def unsubscribe(self, observer: FloorObserver) -> None:
        self._observers.remove(observer)

    def allocate(self, vehicle: Vehicle) -> ParkingSpot | None:
        with self._lock:
            for spot in self.spots:
                if spot.is_free and spot.fits(vehicle) and spot.try_occupy(vehicle):
                    if all(not s.is_free for s in self.spots):
                        for observer in self._observers:
                            observer.on_floor_full(self.id)
                    return spot
            return None

    def release(self, spot: ParkingSpot) -> None:
        with self._lock:
            was_full = all(not s.is_free for s in self.spots)
            spot.vacate()
            if was_full:
                for observer in self._observers:
                    observer.on_floor_available(self.id)

    def maintenance_mode(self) -> "_FloorMaintenance":
        """Context manager: temporarily stop allocating on this floor (e.g.
        for cleaning) and guarantee it reopens even if the maintenance body
        raises. Demonstrates the acquire/release shape context managers exist
        for, beyond just locks."""
        return _FloorMaintenance(self)


class _FloorMaintenance:
    """Small private context-manager helper backing `ParkingFloor.maintenance_mode()`.
    Could equally be written as a `@contextmanager`-decorated generator function;
    the class form is shown here because it needs to hold `floor` as state."""

    def __init__(self, floor: ParkingFloor) -> None:
        self._floor = floor
        self._closed_spots: list[ParkingSpot] = []

    def __enter__(self) -> ParkingFloor:
        with self._floor._lock:
            self._closed_spots = [s for s in self._floor.spots if s.is_free]
            for spot in self._closed_spots:
                spot.try_occupy(Vehicle(license_plate="__MAINTENANCE__", type=VehicleType.CAR))
        return self._floor

    def __exit__(self, exc_type: object, exc: object, tb: object) -> None:
        for spot in self._closed_spots:
            spot.vacate()


class DisplayBoard(FloorObserver):
    def on_floor_full(self, floor_id: str) -> None:
        print(f"Floor {floor_id}: FULL")

    def on_floor_available(self, floor_id: str) -> None:
        print(f"Floor {floor_id}: spot available")


# ── pricing.py: pluggable billing ──────────────────────────────────────────

class PricingStrategy(ABC):
    @abstractmethod
    def calculate_fee(self, vehicle_type: VehicleType, parked_duration: timedelta) -> float: ...


class HourlyFlatPricing(PricingStrategy):
    def __init__(self, rate_per_hour: dict[VehicleType, float]) -> None:
        self.rate_per_hour = rate_per_hour

    def calculate_fee(self, vehicle_type: VehicleType, parked_duration: timedelta) -> float:
        hours = math.ceil(parked_duration.total_seconds() / 3600)
        return hours * self.rate_per_hour.get(vehicle_type, 0.0)


class TieredHourlyPricing(PricingStrategy):
    def __init__(self, base_hours: float, base_rate: float, overage_rate: float) -> None:
        self.base_hours = base_hours
        self.base_rate = base_rate
        self.overage_rate = overage_rate

    def calculate_fee(self, vehicle_type: VehicleType, parked_duration: timedelta) -> float:
        hours = math.ceil(parked_duration.total_seconds() / 3600)
        if hours <= self.base_hours:
            return hours * self.base_rate
        return self.base_hours * self.base_rate + (hours - self.base_hours) * self.overage_rate


# `PaymentMethod` as a `Protocol` instead of an `ABC`: nothing here needs to
# be part of an inheritance hierarchy — any object with a matching `pay`
# method (a real gateway client, a test double, a lambda-wrapping adapter)
# satisfies the type without inheriting from anything. Contrast with
# `PricingStrategy` above, which stays an `ABC` because its concrete
# implementations are owned by this module and share no useful default
# behavior worth mixing in — either choice is defensible per strategy; the
# point is to choose deliberately and be able to say why.
class PaymentMethod(Protocol):
    def pay(self, amount: float) -> bool: ...


class CashPayment:
    def pay(self, amount: float) -> bool:
        print(f"Collected ${amount:.2f} cash")
        return True


# ── lot.py: tickets + the coordinating ParkingLot ──────────────────────────

@dataclass
class ParkingTicket:
    vehicle: Vehicle
    spot: ParkingSpot
    entry_time: datetime = field(default_factory=datetime.now)
    exit_time: datetime | None = None
    id: str = field(default_factory=lambda: str(uuid.uuid4()))

    def close(self) -> None:
        self.exit_time = datetime.now()

    def duration(self) -> timedelta:
        return (self.exit_time or datetime.now()) - self.entry_time

    def __str__(self) -> str:
        status = "closed" if self.exit_time else "open"
        return f"Ticket({self.id[:8]}, {self.vehicle.license_plate}, {status})"


class ParkingLot:
    """Instantiated once at the composition root — no enforced Singleton,
    see the patterns discussion above."""

    def __init__(self, floors: list[ParkingFloor], pricing_strategy: PricingStrategy) -> None:
        self.floors = floors
        self.pricing_strategy = pricing_strategy
        self._active_tickets: dict[str, ParkingTicket] = {}

    def issue_ticket(self, vehicle: Vehicle) -> ParkingTicket:
        for floor in self.floors:
            spot = floor.allocate(vehicle)
            if spot is not None:
                ticket = ParkingTicket(vehicle=vehicle, spot=spot)
                self._active_tickets[vehicle.license_plate] = ticket
                return ticket
        raise RuntimeError(f"Lot full for {vehicle.type.name}")

    def settle(self, ticket: ParkingTicket, payment: PaymentMethod) -> float:
        ticket.close()
        fee = self.pricing_strategy.calculate_fee(ticket.vehicle.type, ticket.duration())
        if not payment.pay(fee):
            raise RuntimeError("Payment failed")
        for floor in self.floors:
            if ticket.spot in floor.spots:
                floor.release(ticket.spot)
                break
        del self._active_tickets[ticket.vehicle.license_plate]
        return fee
```

## Sample walkthrough

```python
floor1 = ParkingFloor(
    "F1",
    [
        ParkingSpot("F1-S1", SpotSize.SMALL, has_charger=False),
        ParkingSpot("F1-M1", SpotSize.MEDIUM, has_charger=False),
        ParkingSpot("F1-M2", SpotSize.MEDIUM, has_charger=True),
    ],
)
floor1.subscribe(DisplayBoard())

lot = ParkingLot(
    floors=[floor1],
    pricing_strategy=HourlyFlatPricing({VehicleType.CAR: 20.0, VehicleType.ELECTRIC: 25.0}),
)

car = VehicleFactory.create(VehicleType.CAR, "KA-01-1234")
ticket = lot.issue_ticket(car)            # allocated to F1-M1 (first fitting free spot)
# ... two hours later ...
fee = lot.settle(ticket, CashPayment())   # HourlyFlatPricing.calculate_fee -> 2 * 20.0 = 40.0
assert fee == 40.0
```

## Follow-up questions

- **"Two vehicles arrive simultaneously for the last spot — what happens?"** `ParkingFloor.allocate` holds a floor-level `threading.Lock` across the scan-then-claim sequence, so the second thread's scan simply sees the spot as taken. Per-spot `try_occupy` is a belt-and-suspenders second check. See [../04-concurrency-essentials.md](../04-concurrency-essentials.md) for the non-blocking `Lock.acquire(blocking=False)` variant if you want allocation to fail fast instead of waiting on contention.
- **"What if we need subscription/monthly pricing instead of pay-per-visit?"** Add a `SubscriptionPricing(PricingStrategy)` that returns `0` for subscribers and short-circuits at the gate — zero changes to `ParkingLot`, `ParkingFloor`, or any other `PricingStrategy`. This is the OCP payoff of making pricing a Strategy from the start.
- **"How would you support a reservation/waitlist so a specific spot is held for a car arriving in 10 minutes?"** Add a `reserve(spot, vehicle, expiry)` path on `ParkingFloor` that marks a spot `RESERVED` (a third occupancy state, not just free/occupied) and a background sweep that reverts to `FREE` on expiry — this is additive to the existing free/occupied check in `allocate`, not a rewrite.
- **"Multiple lots across a city, need to route a car to whichever lot has space?"** That's a new `ParkingLotLocator` service one layer up, holding references to multiple `ParkingLot` instances and querying availability — confirms the earlier call to *not* Singleton-enforce `ParkingLot`, since this follow-up needs more than one instance to coexist in the same process.
- **"What if a Truck needs 2 adjacent Large spots?"** `ParkingSpot.fits` and `ParkingFloor.allocate` would need to reason about contiguous spot ranges instead of a single spot — flag this as a real model change (spots become allocatable in groups) rather than something the current Strategy/Factory seams absorb for free; worth naming as a scope question up front.

## Common mistakes on this problem

- Hardcoding vehicle→spot-size logic as an `if vehicle.type == VehicleType.CAR: ... elif ...` chain instead of either a Strategy or (as here) enum-with-behavior — instant OCP red flag.
- Building a full `Motorcycle`/`Car`/`Truck`/`Electric` class hierarchy when nothing actually overrides behavior — over-engineering that costs interview time for zero payoff; justify subclassing only when behavior, not data, differs.
- Enforcing Singleton on `ParkingLot` via a module-level global or a guarded constructor "because parking lot problems always use Singleton" — cargo-culting the pattern name without checking whether the constraint (single instance) actually needs language-level enforcement vs. composition-root discipline.
- Forgetting to synchronize spot allocation at all, then hand-waving "assume single-threaded" when the interviewer explicitly asked about concurrent entry gates — if concurrency is in scope, show the lock, don't just promise one.

## Continue

Next: [02-elevator-system.md](02-elevator-system.md)
