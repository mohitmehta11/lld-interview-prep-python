# Structural Patterns

Structural patterns answer: **how do existing objects compose into larger structures without new code depending on their concrete shapes?** Adapter/Facade/Proxy all wrap something; Decorator/Composite build recursive structures; Flyweight/Bridge are about the *cost* and *axes* of that structure. Full depth on the first five (Adapter, Decorator, Composite, Facade, Proxy) — Flyweight and Bridge get a short treatment since they show up far less often.

Python's duck typing, `__getattr__`/dunder delegation, closures, and stdlib caching decorators (`functools.lru_cache`, `cached_property`) make several of these patterns *thinner* than their classic form — each section calls out where that happens.

---

## Adapter

**Intent:** Convert the interface of an existing class into another interface clients expect, without touching the existing class.

**When to reach for it in LLD:**
- Requirement language: "integrate a third-party SDK/payment gateway/legacy system" whose method names/signatures don't match your domain interface.
- You want your domain code to depend on *your* abstraction (per [DIP](../02-solid-principles.md#d--dependency-inversion-principle)), not the vendor's, so swapping vendors later doesn't ripple through business logic.

**Structure:**
```
PaymentMethod (interface, your domain)
  └─ StripeAdapter implements PaymentMethod
        └─ has-a[1] StripeSdkClient (third-party, incompatible interface)
```

**Pythonic idiom note:** duck typing means Python often needs *no* Adapter class at all if the shapes already line up — and even when they don't, the "interface" side can be a `typing.Protocol` instead of an `ABC`, so the adapter satisfies it structurally with zero inheritance. Write an explicit adapter class only when the third-party surface genuinely differs; when the adapter needs to pass most calls straight through and intercept only one or two, `__getattr__` delegation avoids writing a forwarding method for every attribute.

```python
from __future__ import annotations
from typing import Protocol

class PaymentMethod(Protocol):
    def pay(self, amount: float) -> bool: ...

class StripeChargeResult:
    successful: bool = True

class StripeSdkClient:                                   # third-party, different shape
    def submit_charge(self, amount_cents: int, currency: str) -> StripeChargeResult:
        return StripeChargeResult()

class StripeAdapter:
    """Satisfies PaymentMethod structurally — PaymentMethod is a Protocol, so no
    explicit `class StripeAdapter(PaymentMethod)` inheritance is required."""

    def __init__(self, client: StripeSdkClient) -> None:
        self._client = client

    def pay(self, amount: float) -> bool:
        result = self._client.submit_charge(int(amount * 100), "USD")
        return result.successful

def checkout(method: PaymentMethod, amount: float) -> bool:
    return method.pay(amount)          # duck-typed: works for StripeAdapter with zero inheritance

checkout(StripeAdapter(StripeSdkClient()), 19.99)


# Pythonic idiom: when most of the wrapped object's surface should pass straight
# through unchanged and only one or two methods need adapting/intercepting,
# __getattr__ delegation avoids a pass-through method per attribute:
class LoggingPaymentAdapter:
    def __init__(self, wrapped: PaymentMethod) -> None:
        self._wrapped = wrapped

    def pay(self, amount: float) -> bool:               # the one method we actually adapt
        print(f"[pay] amount={amount}")
        return self._wrapped.pay(amount)

    def __getattr__(self, name: str):                    # everything else forwards untouched
        return getattr(self._wrapped, name)
```

**Related principle:** [DIP](../02-solid-principles.md#d--dependency-inversion-principle) — the domain depends on `PaymentMethod`, the abstraction it defines; the adapter, not the business logic, absorbs the vendor's concretion.

**Used in:** [../problems/07-movie-ticket-booking.md](../problems/07-movie-ticket-booking.md) (wrapping an external payment gateway SDK), [../problems/06-splitwise-expense-sharing.md](../problems/06-splitwise-expense-sharing.md) (adapting a bank-transfer API to an internal `SettlementMethod`).

**Watch out for:** don't build an Adapter around an interface *you already control* — if you own both sides, just change one side to match. Adapter earns its keep only when one side is external/unowned/legacy.

---

## Decorator

**Intent:** Attach additional responsibilities to an object dynamically by wrapping it, giving a flexible alternative to subclassing for combinable behavior.

**When to reach for it in LLD:**
- Requirement language: "notifications can optionally be logged/retried/encrypted," "add a surcharge for weekend/EV-charging on top of the base fee" — behaviors that **stack in combination**, where subclassing would need one class per combination (`LoggedRetryingEmailNotifier`, `RetryingEmailNotifier`, ... — combinatorial explosion).
- Each added responsibility should be independently addable/removable at construction time.

**Structure:**
```
Notifier (interface)
  ├─ EmailNotifier (concrete base)
  └─ NotifierDecorator (abstract) implements Notifier, has-a[1] Notifier
        ├─ LoggingDecorator
        └─ RetryDecorator
```

**Pythonic idiom note:** this is the pattern Python most fully absorbs into the language itself. When the thing being decorated is a multi-method *object* (as here, an object implementing a `Notifier` interface), the class-wrapping-a-class version below is still the right shape. But when the thing being decorated is really just a *callable* — a single function — the entire wrapper-class hierarchy collapses into a closure-based `@decorator`, using `functools.wraps` to preserve the wrapped function's metadata. This is the far more common Python idiom for "stack behavior around a call," and it's worth naming explicitly as the Pythonic alternative even when you show the class-based version for an object-shaped requirement.

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from functools import wraps
from typing import Callable, TypeVar

class Notifier(ABC):
    @abstractmethod
    def send(self, message: str) -> None: ...

class EmailNotifier(Notifier):
    def send(self, message: str) -> None:
        print(f"Email: {message}")

class NotifierDecorator(Notifier, ABC):
    def __init__(self, wrapped: Notifier) -> None:
        self._wrapped = wrapped

class LoggingDecorator(NotifierDecorator):
    def send(self, message: str) -> None:
        print(f"[LOG] sending: {message}")
        self._wrapped.send(message)

class RetryDecorator(NotifierDecorator):
    def __init__(self, wrapped: Notifier, max_attempts: int = 3) -> None:
        super().__init__(wrapped)
        self._max_attempts = max_attempts

    def send(self, message: str) -> None:
        for attempt in range(self._max_attempts):
            try:
                self._wrapped.send(message)
                return
            except RuntimeError:
                if attempt == self._max_attempts - 1:
                    raise

# stack behaviors by composition, chosen at construction time — no new subclass needed
notifier = LoggingDecorator(RetryDecorator(EmailNotifier(), max_attempts=3))
notifier.send("Your order shipped")


# Pythonic idiom: when the component is a callable rather than a multi-method
# interface, replace the whole wrapper-class hierarchy with closures:
F = TypeVar("F", bound=Callable[..., None])

def with_logging(func: F) -> F:
    @wraps(func)
    def wrapper(*args, **kwargs):
        print(f"[LOG] calling {func.__name__} with {args}, {kwargs}")
        return func(*args, **kwargs)
    return wrapper  # type: ignore[return-value]

def with_retry(max_attempts: int = 3) -> Callable[[F], F]:
    def decorator(func: F) -> F:
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except RuntimeError:
                    if attempt == max_attempts - 1:
                        raise
        return wrapper  # type: ignore[return-value]
    return decorator

@with_logging
@with_retry(max_attempts=3)
def send_email(message: str) -> None:
    print(f"Email: {message}")

send_email("Your order shipped")   # decorators apply bottom-up, same stacking idea as the objects above
```
Say the distinction out loud if asked: GoF Decorator wraps objects that implement a shared multi-method interface; Python's `@decorator` syntax wraps single callables. They express the same intent — combinable, independently-addable behavior — at different granularities. Don't conflate them, but do point out that most "decorator" requirements in a Python codebase end up as the function form, not the class form.

**Related principle:** [OCP](../02-solid-principles.md#o--openclosed-principle) — new combinable behavior is a new decorator (class or function), existing notifiers/decorators are untouched.

**Used in:** [../problems/10-notification-and-observer-system.md](../problems/10-notification-and-observer-system.md) (stackable logging/retry/priority behavior on channels), [../problems/01-parking-lot.md](../problems/01-parking-lot.md) (fee surcharges layered on a base `PricingStrategy`).

**Watch out for:** if there's only ever one or two fixed combinations and they never mix independently, plain composition or a couple of concrete classes/functions is simpler than a Decorator hierarchy — don't reach for it just because "behavior gets added" once.

---

## Composite

**Intent:** Compose objects into tree structures and let clients treat an individual object (leaf) and a group of objects (composite) through the same interface.

**When to reach for it in LLD:**
- Requirement language: "a category can contain books or sub-categories," "a folder contains files or other folders," "a notification group can contain individual recipients or nested groups" — recursive part-whole structures where client code shouldn't branch on "is this a leaf or a group?"

**Structure:**
```
FileSystemEntry (interface)
  ├─ File (leaf) — size(): own size
  └─ Directory (composite) — size(): sum of children's size(), has-a[*] FileSystemEntry
```

**Pythonic idiom note:** the shape of the pattern doesn't change — you still need a shared abstract interface and a container that delegates recursively — but implementing `__iter__` (as a generator, `yield`-ing leaves and recursing into sub-containers) and `__len__` on the composite lets client code use `for entry in root:` and `len(root)` as if the tree were a built-in container, instead of exposing a separate `get_children()`/`walk()` method.

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from typing import Iterator

class FileSystemEntry(ABC):
    @abstractmethod
    def size(self) -> int: ...

class File(FileSystemEntry):
    def __init__(self, name: str, size_bytes: int) -> None:
        self.name = name
        self._size_bytes = size_bytes

    def size(self) -> int:
        return self._size_bytes

    def __repr__(self) -> str:
        return f"File({self.name!r}, {self._size_bytes}B)"

class Directory(FileSystemEntry):
    def __init__(self, name: str) -> None:
        self.name = name
        self._children: list[FileSystemEntry] = []

    def add(self, entry: FileSystemEntry) -> "Directory":
        self._children.append(entry)
        return self                      # chaining: root.add(a).add(b)

    def size(self) -> int:
        return sum(child.size() for child in self._children)   # uniform recursion, no isinstance() branching

    def __iter__(self) -> Iterator[FileSystemEntry]:
        """Dunder support: lets clients do `for entry in root:` without knowing the
        tree shape, and without exposing a separate walk()/get_children() method."""
        for child in self._children:
            yield child
            if isinstance(child, Directory):
                yield from child

    def __len__(self) -> int:
        return sum(1 for _ in self)

root = Directory("root")
root.add(File("a.txt", 100))
sub = Directory("sub")
sub.add(File("b.txt", 50))
root.add(sub)

root.size()   # 150 — caller never checks File vs Directory
len(root)     # 2 — dunder support makes the tree feel like a built-in container
list(root)    # [File('a.txt', 100B), File('b.txt', 50B)] via __iter__
```

**Related principle:** [LSP](../02-solid-principles.md#l--liskov-substitution-principle) — `Directory` and `File` must both honor the `FileSystemEntry` contract fully; the moment `Directory.size()` needs a special case that `File` doesn't, the abstraction is leaking.

**Used in:** [../problems/08-library-management-system.md](../problems/08-library-management-system.md) (category tree containing books or sub-categories), [../problems/06-splitwise-expense-sharing.md](../problems/06-splitwise-expense-sharing.md) (an expense group containing individual expenses or nested sub-groups).

**Watch out for:** don't force Composite onto a structure that's really just one flat collection with no recursive containment — if groups never nest inside groups, you don't need the pattern, a plain `list[Leaf]` is enough.

---

## Facade

**Intent:** Provide a single, simplified interface to a set of interfaces in a subsystem, so clients don't need to know or orchestrate the subsystem's internals.

**When to reach for it in LLD:**
- Requirement language: a use case that spans several already-well-factored subsystems (seat locking + payment + notification; spot search + ticket issuance + payment) and you want one clean call for the "happy path" orchestration without collapsing those subsystems into one God Object.

**Structure:**
```
BookingFacade
  ├─ uses-a SeatLockService
  ├─ uses-a PaymentMethod
  └─ uses-a NotificationService

Client → BookingFacade.book_seat(...)   // one call, three subsystems orchestrated behind it
```

**Pythonic idiom note:** the Facade itself doesn't change shape much — it's still "one class (or, if there's no per-call state to hold, just a module of top-level functions) that orchestrates several subsystems." The idiomatic upgrade is in *how* it manages resource acquisition/release: the lock-then-unlock-in-`finally` boilerplate a Facade tends to accumulate is exactly what `contextlib.contextmanager` (or making the Facade a context manager itself) is for — express "acquire, do the work, always release" as a `with` block instead of manual `try`/`finally`.

```python
from __future__ import annotations
from contextlib import contextmanager
from typing import Iterator

class BookingFacade:
    def __init__(
        self,
        seat_lock_service: "SeatLockService",
        payment_method: "PaymentMethod",
        notification_service: "NotificationService",
    ) -> None:
        self._seat_lock_service = seat_lock_service
        self._payment_method = payment_method
        self._notification_service = notification_service

    @contextmanager
    def _locked_seat(self, seat_id: str) -> Iterator[bool]:
        """Acquire/release as a context manager instead of manual try/finally —
        the direct Python replacement for lock-then-unlock-in-finally boilerplate."""
        acquired = self._seat_lock_service.lock(seat_id)
        try:
            yield acquired
        finally:
            if acquired:
                self._seat_lock_service.unlock(seat_id)

    def book_seat(self, seat_id: str, price: float, user_email: str) -> bool:
        with self._locked_seat(seat_id) as locked:
            if not locked:
                return False
            if not self._payment_method.pay(price):
                return False
            self._notification_service.send(user_email, f"Seat {seat_id} booked!")
            return True

# client's view: one call, no need to know locking/payment/notification exist separately
BookingFacade(seat_lock_service, payment_method, notification_service).book_seat("A1", 15.0, "u@x.com")
```

**Related principle:** [SRP](../02-solid-principles.md#s--single-responsibility-principle) at the orchestration layer — the Facade's one reason to change is "the booking *workflow* changed," while each subsystem still changes only for its own reason.

**Used in:** [../problems/07-movie-ticket-booking.md](../problems/07-movie-ticket-booking.md) (booking workflow across locking/payment/notification), [../problems/01-parking-lot.md](../problems/01-parking-lot.md) (`ParkingLot` facade over spot search + ticketing + pricing).

**Watch out for:** a Facade that just forwards one call to one subsystem method is a pointless indirection — it earns its place only when it's genuinely hiding multi-step orchestration across 2+ subsystems.

---

## Proxy

**Intent:** Provide a surrogate/placeholder for another object to control access to it — lazy instantiation, access control, caching, or logging, transparently to the caller.

**When to reach for it in LLD:**
- Requirement language: "expensive to construct, load on demand" (virtual proxy), "only certain roles can call this" (protection proxy), "cache repeated lookups" (caching proxy), "log/audit every call" (logging proxy).
- The caller should keep using the *same interface* it already had — the proxy is a drop-in.

**Structure:**
```
BookCatalog (interface)
  ├─ RealBookCatalog — hits the DB, slow
  └─ CachingBookCatalogProxy implements BookCatalog, has-a[1] RealBookCatalog
        └─ search(query): check cache first, else delegate + populate cache
```

**Pythonic idiom note:** a full caching-proxy class is frequently unnecessary — `functools.lru_cache` (method-level memoization) or `functools.cached_property` (for a proxy caching a single derived attribute) gives you the same caching behavior as a one-line decorator, no wrapper class or manual cache dict required. For a generic protection/logging proxy that should forward most calls untouched, `__getattr__` delegation (as in Adapter) avoids writing a pass-through method per attribute.

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from functools import lru_cache

class Book: ...

def run_expensive_db_query(query: str) -> list[Book]: ...

class BookCatalog(ABC):
    @abstractmethod
    def search(self, query: str) -> list[Book]: ...

class RealBookCatalog(BookCatalog):
    def search(self, query: str) -> list[Book]:
        return run_expensive_db_query(query)   # simulate an expensive DB/full-text search

class CachingBookCatalogProxy(BookCatalog):
    def __init__(self, real: RealBookCatalog) -> None:
        self._real = real
        self._cache: dict[str, list[Book]] = {}

    def search(self, query: str) -> list[Book]:
        if query not in self._cache:
            self._cache[query] = self._real.search(query)
        return self._cache[query]           # caller can't tell caching happened

catalog: BookCatalog = CachingBookCatalogProxy(RealBookCatalog())
catalog.search("dune")   # first call: hits DB
catalog.search("dune")   # second call: cache hit, same interface either way


# Pythonic idiom: functools.lru_cache is a ready-made caching-proxy decorator — no
# wrapper class or manual cache dict needed (return type must be hashable):
class BookCatalogWithBuiltinCache:
    @lru_cache(maxsize=256)
    def search(self, query: str) -> tuple[Book, ...]:
        return tuple(run_expensive_db_query(query))


# A generic pass-through/logging proxy via __getattr__ — most calls forward untouched,
# useful for a protection or audit-logging proxy that shouldn't need to know every
# method on the wrapped interface:
class LoggingProxy:
    def __init__(self, target: BookCatalog) -> None:
        self._target = target

    def __getattr__(self, name: str):
        attr = getattr(self._target, name)
        if callable(attr):
            def wrapper(*args, **kwargs):
                print(f"[CALL] {name}(args={args}, kwargs={kwargs})")
                return attr(*args, **kwargs)
            return wrapper
        return attr
```

**Related principle:** [OCP](../02-solid-principles.md#o--openclosed-principle)/[DIP](../02-solid-principles.md#d--dependency-inversion-principle) — callers depend on `BookCatalog`, so swapping the real implementation for a caching/protection wrapper requires zero changes at call sites.

**Used in:** [../problems/08-library-management-system.md](../problems/08-library-management-system.md) (caching proxy over catalog search), [../problems/05-lru-cache-and-rate-limiter.md](../problems/05-lru-cache-and-rate-limiter.md) (a rate-limiting proxy in front of an API handler).

**Watch out for:** don't confuse Proxy with Decorator — they look identical in code (both wrap-and-delegate). The distinction is *intent*: Proxy controls **access** to the same conceptual operation (cache/lazy-load/guard); Decorator **adds new behavior/responsibility** on top. If you're stacking multiple independent behaviors, say Decorator; if you're gatekeeping one thing, say Proxy — and don't feel obligated to name either if `functools.lru_cache` or a plain method call suffices.

---

## Flyweight

**Intent:** Share immutable, fine-grained state across many logical objects to cut memory, instead of duplicating it per instance.

**When to reach for it in LLD:** many objects share identical, expensive-ish immutable data — e.g. every white pawn on a chess board shares the same movement rules/icon metadata; only position is per-instance.

**Pythonic idiom note:** `functools.lru_cache` is a ready-made Flyweight factory — decorate a function that builds the shared object and repeated calls with the same arguments return the same cached instance, with no hand-rolled `__new__`/cache-dict needed. (For plain strings specifically, `sys.intern` does the same job at the interpreter level.)

```python
from __future__ import annotations
from dataclasses import dataclass
from functools import lru_cache

@dataclass(frozen=True)
class PieceType:                      # shared, immutable — the flyweight
    name: str
    movement_rule: str

@lru_cache(maxsize=None)
def get_piece_type(name: str, movement_rule: str) -> PieceType:
    return PieceType(name, movement_rule)   # cached: identical args return the identical object

@dataclass
class Piece:
    type: PieceType                   # shared reference, not a copy
    position: tuple[int, int]         # per-instance, extrinsic state

white_pawn_a = Piece(type=get_piece_type("pawn", "forward-1-or-2"), position=(0, 1))
white_pawn_b = Piece(type=get_piece_type("pawn", "forward-1-or-2"), position=(1, 1))
assert white_pawn_a.type is white_pawn_b.type   # same object, not duplicated
```

**Related principle:** doesn't map to one SOLID letter directly — it's a memory-sharing optimization layered on top of whatever hierarchy already exists (often Strategy for the movement rule itself).

**Used in:** [../problems/04-tic-tac-toe-and-chess.md](../problems/04-tic-tac-toe-and-chess.md) (shared piece-type metadata across many piece instances).

**Watch out for:** almost never worth naming unless the interviewer explicitly probes memory footprint at scale — leading with Flyweight in a 45-minute interview reads as trivia, not judgment.

---

## Bridge

**Intent:** Decouple an abstraction from its implementation so the two can vary independently, avoiding a class explosion when two orthogonal dimensions both grow.

**When to reach for it in LLD:** you have two independent hierarchies that would otherwise multiply (`Alert`/`Report` × `Email`/`SMS` → 4 classes today, N×M as either side grows) — split into an abstraction hierarchy that *holds* an implementor hierarchy instead of inheriting from it.

**Pythonic idiom note:** if the implementor side genuinely needs only one method (as in the `DeliveryChannel` example below), a plain `Callable[[str], None]` — a function — replaces the entire implementor class hierarchy. First-class functions *are* the implementor; you only need a real class/`ABC` on that side once it needs more than one method or its own internal state.

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from typing import Callable

class DeliveryChannel(ABC):                              # implementor hierarchy
    @abstractmethod
    def deliver(self, content: str) -> None: ...

class EmailChannel(DeliveryChannel):
    def deliver(self, content: str) -> None:
        print(f"Email: {content}")

class SmsChannel(DeliveryChannel):
    def deliver(self, content: str) -> None:
        print(f"SMS: {content}")

class Notification(ABC):                                 # abstraction hierarchy
    def __init__(self, channel: DeliveryChannel) -> None:
        self._channel = channel        # bridge: composed, so Notification-kind and channel-kind vary independently

    @abstractmethod
    def send(self) -> None: ...

class AlertNotification(Notification):
    def send(self) -> None:
        self._channel.deliver("ALERT: system down")

class ReportNotification(Notification):
    def __init__(self, channel: DeliveryChannel, report_name: str) -> None:
        super().__init__(channel)
        self._report_name = report_name

    def send(self) -> None:
        self._channel.deliver(f"REPORT ready: {self._report_name}")

AlertNotification(EmailChannel()).send()
ReportNotification(SmsChannel(), "Q2 Revenue").send()


# Pythonic idiom: the implementor side here only needs one method, so a plain
# Callable[[str], None] replaces the DeliveryChannel class hierarchy entirely —
# first-class functions are the implementor:
def email_channel(content: str) -> None:
    print(f"Email: {content}")

def sms_channel(content: str) -> None:
    print(f"SMS: {content}")

class FunctionalNotification(ABC):
    def __init__(self, channel: Callable[[str], None]) -> None:
        self._channel = channel

    @abstractmethod
    def send(self) -> None: ...

class FunctionalAlertNotification(FunctionalNotification):
    def send(self) -> None:
        self._channel("ALERT: system down")

FunctionalAlertNotification(email_channel).send()
```

**Related principle:** [OCP](../02-solid-principles.md#o--openclosed-principle) on two axes at once — new notification kinds and new channels are both additive.

**Used in:** [../problems/10-notification-and-observer-system.md](../problems/10-notification-and-observer-system.md) (notification kind × delivery channel, kept independent).

**Watch out for:** if only one of the two dimensions actually varies in the problem, you have a plain Strategy, not a Bridge — don't reach for the heavier pattern name for a one-axis problem.

## Continue

Next: [03-behavioral-patterns.md](03-behavioral-patterns.md)
