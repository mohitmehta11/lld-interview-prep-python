# Creational Patterns

Creational patterns answer one question: **who decides which concrete class gets instantiated, and how much construction-time logic does that decision need?** If the answer is "no logic, just construct it directly," you don't need a pattern — say so and move on.

Python changes the calculus more here than anywhere else: the language has first-class functions, module-level state, `__new__`, metaclasses, and `dataclasses` doing a lot of what a class hierarchy does elsewhere. Each section below leads with the idiomatic Python approach and calls out explicitly when it replaces the classic class-hierarchy version outright versus when the class-hierarchy version is still what an interviewer will expect to see.

---

## Singleton

**Intent:** Guarantee a class has exactly one instance and provide a single global access point to it.

**When to reach for it in LLD:**
- The problem domain itself has exactly one of something *by definition* — one `ParkingLot`, one `ElevatorControlSystem`, one config registry — not "there happens to be one right now."
- You need a single shared coordinator that mediates access to a scarce resource (a connection pool, an ID generator).

**Structure:**
```
Singleton
  ├─ private static field: instance
  ├─ private constructor (blocks external instantiation)
  └─ public static get_instance() -> Singleton
```

**Pythonic idiom note:** in Python, "there's only one of these" is most often expressed by *not writing a class at all* — a module's globals are already a singleton, because CPython caches imported modules, so importing it twice never re-runs the module body. Reach for a class-based Singleton only when the interviewer explicitly wants OOP mechanics shown, or when the single instance must satisfy an interface/be dependency-injected.

```python
from __future__ import annotations
import threading

# Pythonic: the module itself is the singleton — import caching guarantees exactly
# one instance per process, with none of the getInstance()/lazy-init ceremony.
# id_generator.py
_lock = threading.Lock()
_counter = 0

def next_id() -> int:
    global _counter
    with _lock:                     # only needed if called from multiple threads
        _counter += 1
        return _counter

# usage elsewhere: from id_generator import next_id; next_id()


# Class-based, for when the interviewer wants explicit OOP or an injectable object:
class ParkingLot:
    _instance: "ParkingLot | None" = None
    _lock = threading.Lock()

    def __new__(cls) -> "ParkingLot":
        if cls._instance is None:
            with cls._lock:                       # Python's version of double-checked locking —
                if cls._instance is None:          # double-checked: avoids taking the lock on every call
                    cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self) -> None:
        if getattr(self, "_initialized", False):
            return
        self._initialized = True
        self.floors: list["Floor"] = []            # init floors, spots


# Metaclass version — write the locking/caching logic once, reuse across every
# Singleton in a codebase instead of repeating __new__ boilerplate per class:
class SingletonMeta(type):
    _instances: dict[type, object] = {}
    _lock = threading.Lock()

    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            with cls._lock:
                if cls not in cls._instances:
                    cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class ElevatorControlSystem(metaclass=SingletonMeta):
    def __init__(self) -> None:
        self.cars: list["ElevatorCar"] = []

# ElevatorControlSystem() always returns the same instance, from any call site
```
The naive `if instance is None: instance = ParkingLot()` is a race under concurrent first access — if you lazy-init a class-based singleton without a lock, say the word "race" out loud. See [../04-concurrency-essentials.md](../04-concurrency-essentials.md) for why the lock-guarded `__new__`/metaclass forms sidestep it.

**Related principle:** tension with [DIP](../02-solid-principles.md#d--dependency-inversion-principle) — a global accessor (a class-based singleton or a bare module import used deep in business logic) hides a dependency instead of injecting it explicitly, which is exactly what DIP tells you to avoid. Prefer constructing the single instance once at the composition root and passing it in.

**Used in:** [../problems/01-parking-lot.md](../problems/01-parking-lot.md) (one `ParkingLot` coordinating floors/spots), [../problems/02-elevator-system.md](../problems/02-elevator-system.md) (one dispatcher coordinating multiple cars).

**Watch out for:** this is the single most over-named pattern in LLD interviews. Most "there's only one of these" requirements mean "the composition root (`main`) constructs one instance and passes it around," **not** "the class must enforce its own singleton-ness." Enforcing it via `__new__`/metaclass tricks makes the class harder to test (can't inject a fresh one per test without extra teardown) and hides a dependency that DIP says should be explicit. Default to "construct once, inject everywhere"; only reach for a real Singleton when the interviewer's problem would break with two instances (e.g. two `IdGenerator`s handing out colliding IDs) and say that reasoning out loud.

---

## Factory Method

**Intent:** Define one creation method that subclasses (or a simple internal branch) override/vary to produce **one product type**, without the caller knowing which concrete class it got.

**When to reach for it in LLD:**
- Requirement language: "create a notification/vehicle/document of a given type" where the caller only cares about the resulting interface, not the concrete class.
- You want to remove direct constructor calls scattered through client code and centralize them behind one seam ([OCP](../02-solid-principles.md#o--openclosed-principle): adding a new type = one new branch or subclass, not edits everywhere a constructor was called).

**Structure:**
```
NotificationFactory
  └─ create(type) -> Notification (interface)
                        ├─ EmailNotification
                        ├─ SmsNotification
                        └─ PushNotification
```

**Pythonic idiom note:** an `if`/`elif` chain (or even a class with a `create()` classmethod that branches internally) is rarely the most idiomatic form here. A **dict of callables** is a first-class factory with no class required, and a **registration decorator** takes it further still — adding a new type touches zero existing lines, not even the factory's dict literal. `@classmethod` alternate constructors (`Notification.from_config(...)`, mirroring `dict.fromkeys` / `datetime.fromtimestamp`) are the other common Pythonic "Factory Method" — a single class offering several named ways to build itself.

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from enum import Enum
from typing import Callable

class Notification(ABC):
    @abstractmethod
    def send(self, message: str) -> None: ...

    @classmethod
    def from_config(cls, config: dict) -> "Notification":
        """Alternate-constructor idiom: a classmethod that builds `cls` from some
        other shape of input, the same idea as dict.fromkeys/datetime.fromtimestamp."""
        return cls(**config)

class EmailNotification(Notification):
    def __init__(self, address: str = "") -> None:
        self._address = address
    def send(self, message: str) -> None:
        print(f"Email to {self._address}: {message}")

class SmsNotification(Notification):
    def __init__(self, phone: str = "") -> None:
        self._phone = phone
    def send(self, message: str) -> None:
        print(f"SMS to {self._phone}: {message}")

class PushNotification(Notification):
    def send(self, message: str) -> None:
        print(f"Push: {message}")

class NotificationType(Enum):
    EMAIL = "email"
    SMS = "sms"
    PUSH = "push"

# Dict-of-callables: this *is* the factory. No class needed.
_FACTORIES: dict[NotificationType, Callable[[], Notification]] = {
    NotificationType.EMAIL: EmailNotification,
    NotificationType.SMS: SmsNotification,
    NotificationType.PUSH: PushNotification,
}

def create_notification(kind: NotificationType) -> Notification:
    return _FACTORIES[kind]()

n = create_notification(NotificationType.EMAIL)
n.send("Your order shipped")


# Self-registering variant — new subclasses register themselves via a decorator,
# so "adding a type" never touches the factory function or an existing dict literal
# at all (OCP taken further than a switch statement ever allows):
_REGISTRY: dict[NotificationType, type[Notification]] = {}

def register(kind: NotificationType) -> Callable[[type[Notification]], type[Notification]]:
    def decorator(cls: type[Notification]) -> type[Notification]:
        _REGISTRY[kind] = cls
        return cls
    return decorator

@register(NotificationType.EMAIL)
class RegisteredEmailNotification(Notification):
    def send(self, message: str) -> None:
        print(f"Email: {message}")

def create(kind: NotificationType) -> Notification:
    return _REGISTRY[kind]()
```

**Related principle:** direct application of [OCP](../02-solid-principles.md#o--openclosed-principle) — adding `PushNotification` means adding a class and a registry entry, never editing existing call sites.

**Used in:** [../problems/10-notification-and-observer-system.md](../problems/10-notification-and-observer-system.md) (channel creation by type), [../problems/01-parking-lot.md](../problems/01-parking-lot.md) (vehicle/spot-size lookup by type).

**Watch out for:** don't build a `Factory` class (or even a dict registry) for a single concrete type with no variation on the horizon — that's ceremony with no payoff. Also don't confuse this with Abstract Factory (below): Factory Method produces **one** product family member per call; if the interviewer's problem needs *several related objects that must be consistent with each other*, that's Abstract Factory, not a bigger Factory Method.

---

## Abstract Factory

**Intent:** Provide an interface for creating **families of related objects** without specifying their concrete classes — and guarantee the objects returned together are mutually compatible.

**When to reach for it in LLD:**
- Requirement language implies a *pairing/bundle* that must stay consistent: "each payment gateway needs its own processor **and** its own refund handler, and they must match" — mixing a Stripe processor with a PayPal refund handler is a bug, not a valid config.
- You're switching an entire "platform"/"provider" at once (all objects from that provider), not picking one object type independently.

**Structure:**
```
PaymentGatewayFactory (interface)
  ├─ create_processor() -> PaymentProcessor (interface)
  └─ create_refund_handler() -> RefundHandler (interface)

StripeGatewayFactory implements PaymentGatewayFactory
  ├─ create_processor() -> StripeProcessor
  └─ create_refund_handler() -> StripeRefundHandler

PaypalGatewayFactory implements PaymentGatewayFactory
  ├─ create_processor() -> PaypalProcessor
  └─ create_refund_handler() -> PaypalRefundHandler
```

**Pythonic idiom note:** `ABC` is still the right call for the *factory* itself — it's a real is-a contract with two required methods and no attractive structural-typing shortcut for the "must implement both creation methods" guarantee. But the **products** returned rarely need a formal base class at all: `typing.Protocol` lets `StripeProcessor`/`PaypalProcessor` satisfy `PaymentProcessor` structurally, and the whole factory hierarchy can be replaced by a `dict` mapping a provider name to a `@dataclass` bundling the matched pair — no inheritance anywhere.

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Callable, Protocol

class PaymentProcessor(Protocol):
    def charge(self, amount: float) -> bool: ...

class RefundHandler(Protocol):
    def refund(self, transaction_id: str) -> bool: ...

class StripeProcessor:
    def charge(self, amount: float) -> bool:
        return True   # call out to Stripe's SDK

class StripeRefundHandler:
    def refund(self, transaction_id: str) -> bool:
        return True

class PaypalProcessor:
    def charge(self, amount: float) -> bool:
        return True

class PaypalRefundHandler:
    def refund(self, transaction_id: str) -> bool:
        return True

# Class-hierarchy version — what an interviewer expects to see, guarantees the pairing
# via the type system:
class PaymentGatewayFactory(ABC):
    @abstractmethod
    def create_processor(self) -> PaymentProcessor: ...
    @abstractmethod
    def create_refund_handler(self) -> RefundHandler: ...

class StripeGatewayFactory(PaymentGatewayFactory):
    def create_processor(self) -> PaymentProcessor:
        return StripeProcessor()
    def create_refund_handler(self) -> RefundHandler:
        return StripeRefundHandler()

class PaypalGatewayFactory(PaymentGatewayFactory):
    def create_processor(self) -> PaymentProcessor:
        return PaypalProcessor()
    def create_refund_handler(self) -> RefundHandler:
        return PaypalRefundHandler()

class CheckoutService:
    def __init__(self, factory: PaymentGatewayFactory) -> None:
        self.processor = factory.create_processor()
        self.refund_handler = factory.create_refund_handler()   # guaranteed to match processor's provider


# Pythonic idiom: skip the factory class hierarchy entirely. Bundle the matched pair in
# a frozen dataclass and look the bundle up by provider name — the pairing guarantee
# comes from "one lambda builds both objects," not from a type hierarchy.
@dataclass(frozen=True)
class GatewayBundle:
    processor: PaymentProcessor
    refund_handler: RefundHandler

_GATEWAYS: dict[str, Callable[[], GatewayBundle]] = {
    "stripe": lambda: GatewayBundle(StripeProcessor(), StripeRefundHandler()),
    "paypal": lambda: GatewayBundle(PaypalProcessor(), PaypalRefundHandler()),
}

def get_gateway(provider: str) -> GatewayBundle:
    return _GATEWAYS[provider]()

bundle = get_gateway("stripe")
bundle.processor.charge(19.99)
```

**Related principle:** [DIP](../02-solid-principles.md#d--dependency-inversion-principle) at a family granularity — `CheckoutService` depends only on the two abstractions and the factory abstraction, never on `Stripe*`/`Paypal*` concretes.

**Used in:** [../problems/03-vending-machine.md](../problems/03-vending-machine.md) (if modeling per-brand product/dispenser families), [../problems/06-splitwise-expense-sharing.md](../problems/06-splitwise-expense-sharing.md) (a settlement-provider family: transfer executor + receipt generator per payment rail).

**Watch out for:** if the "family" only ever has one member (you never actually construct a second related object alongside the first), you don't have an Abstract Factory — you have a Factory Method with an unnecessary wrapper. Only escalate to Abstract Factory when the interviewer's problem has a real *pairing* requirement.

---

## Builder

**Intent:** Separate the construction of a complex object from its representation, so the same construction process can produce different configurations, and callers avoid a telescoping constructor or fragile setter sequence.

**When to reach for it in LLD:**
- Requirement language: an object with many fields, several of which are optional, and combinations matter (a `Notification` with recipient/channel/message/priority/retry-policy; a `SearchQuery` with many optional filters).
- You want the constructed object to be immutable but still ergonomic to build.

**Structure:**
```
Notification (immutable result)
  └─ nested Builder
        ├─ recipient(...) -> Builder
        ├─ channel(...)  -> Builder
        ├─ message(...)  -> Builder
        └─ build()        -> Notification
```

**Pythonic idiom note:** a `@dataclass` with keyword-only arguments and field defaults covers the overwhelming majority of "many optional params" requirements outright — Python's named-argument calling convention already solves the telescoping-constructor problem that Builder was invented to solve. Validation goes in `__post_init__`. Reach for an explicit fluent `Builder` class only when construction genuinely needs multi-step/ordered validation, or when the same target must be assembled via visibly different construction paths.

```python
from __future__ import annotations
from dataclasses import dataclass, field
from enum import Enum

class Channel(Enum):
    EMAIL = "email"
    SMS = "sms"

class Priority(Enum):
    LOW = "low"
    NORMAL = "normal"
    HIGH = "high"

# Usually enough — no Builder needed. kw_only=True forbids positional calls, so field
# order can never be a foot-gun, and __post_init__ gives you validation for free.
@dataclass(frozen=True, kw_only=True)
class Notification:
    recipient: str
    channel: Channel
    message: str
    priority: Priority = Priority.NORMAL
    retry_count: int = field(default=0)

    def __post_init__(self) -> None:
        if not self.recipient:
            raise ValueError("recipient is required")
        if self.channel is Channel.EMAIL and "@" not in self.recipient:
            raise ValueError("EMAIL channel requires a valid address")

n = Notification(recipient="user@x.com", channel=Channel.EMAIL, message="Your spot is confirmed")


# Reach for an explicit fluent Builder only when steps need ordering/validation beyond
# what __post_init__ can express cleanly, or when partial construction across several
# call sites is a real requirement:
class NotificationBuilder:
    def __init__(self) -> None:
        self._recipient: str | None = None
        self._channel: Channel | None = None
        self._message: str = ""
        self._priority: Priority = Priority.NORMAL

    def recipient(self, recipient: str) -> "NotificationBuilder":
        self._recipient = recipient
        return self

    def channel(self, channel: Channel) -> "NotificationBuilder":
        self._channel = channel
        return self

    def message(self, message: str) -> "NotificationBuilder":
        self._message = message
        return self

    def priority(self, priority: Priority) -> "NotificationBuilder":
        self._priority = priority
        return self

    def build(self) -> Notification:
        if self._recipient is None or self._channel is None:
            raise ValueError("recipient and channel are required")
        return Notification(
            recipient=self._recipient,
            channel=self._channel,
            message=self._message,
            priority=self._priority,
        )

n2 = (
    NotificationBuilder()
    .recipient("user@x.com")
    .channel(Channel.EMAIL)
    .message("Your spot is confirmed")
    .priority(Priority.HIGH)
    .build()
)
```

**Related principle:** supports [SRP](../02-solid-principles.md#s--single-responsibility-principle) — construction/validation logic lives in the Builder (or in `__post_init__`), not bloating the value object's own class with setter-sequencing rules.

**Used in:** [../problems/09-logging-framework.md](../problems/09-logging-framework.md) (`LogRecord` built with many optional fields), [../problems/10-notification-and-observer-system.md](../problems/10-notification-and-observer-system.md) (multi-channel notification construction).

**Watch out for:** in Python especially, don't build a fluent `Builder` class for an object with 3 required fields and no optional combinatorics — a `@dataclass` constructor call already does that job. Reach for Builder when there's real construction complexity (validation ordering, optional-field combinatorics), not just "this class has more than one field."

---

## Prototype

**Intent:** Create new objects by cloning an existing, fully-configured instance instead of constructing from scratch, when construction is expensive or the "template" configuration is easier to copy than to rebuild.

**When to reach for it in LLD:**
- Requirement language: "simulate a move without affecting the real game state," "duplicate a configured template per new instance" (e.g. a seat-layout template reused per showtime).
- Object graphs that are expensive/awkward to reconstruct field-by-field but trivial to deep-copy.

**Structure:**
```
Board (prototype interface)
  └─ clone() -> Board

ChessBoard implements Board
  └─ clone() -> deep copy of piece grid, for AI move simulation
```

**Pythonic idiom note:** `copy.deepcopy` is the direct, built-in equivalent of a hand-written `clone()` — Python rarely needs a formal `Prototype` interface at all, since every object already supports `copy.copy`/`copy.deepcopy` (customizable via `__copy__`/`__deepcopy__`). For immutable-ish value/config objects, `dataclasses.replace()` goes one step further and *is* the Prototype pattern: clone-with-overrides in a single call, no `clone()` method to write.

```python
from __future__ import annotations
import copy
from dataclasses import dataclass, field, replace

@dataclass
class ChessBoard:
    grid: list[list["Piece | None"]]

    def clone(self) -> "ChessBoard":
        return copy.deepcopy(self)   # idiomatic: no hand-written field-by-field copy needed

    def __deepcopy__(self, memo: dict) -> "ChessBoard":
        # override when the default recursive deepcopy is too slow/broad — e.g. Piece
        # objects are immutable, so a shallow copy per row is enough and much cheaper:
        new_grid = [row.copy() for row in self.grid]
        return ChessBoard(grid=new_grid)

    def apply_move(self, move: "Move") -> None: ...

simulated = current_board.clone()
simulated.apply_move(candidate_move)   # mutate the clone freely, real board untouched


# Pythonic idiom: for immutable config/value objects, dataclasses.replace() IS the
# Prototype pattern — clone-with-overrides in one call.
@dataclass(frozen=True)
class SeatLayoutTemplate:
    rows: int
    seats_per_row: int
    aisle_after: int = field(default=0)

standard = SeatLayoutTemplate(rows=10, seats_per_row=12, aisle_after=6)
imax_showtime_layout = replace(standard, rows=8)   # clone the template, override just one field
```

**Related principle:** loosely supports [OCP](../02-solid-principles.md#o--openclosed-principle) — new "kinds" of board/template can override `__deepcopy__`/`clone()` without touching client simulation code, but the bigger win here is just avoiding expensive/error-prone reconstruction.

**Used in:** [../problems/04-tic-tac-toe-and-chess.md](../problems/04-tic-tac-toe-and-chess.md) (clone board to simulate candidate moves before committing), [../problems/07-movie-ticket-booking.md](../problems/07-movie-ticket-booking.md) (clone a seat-layout template per new showtime).

**Watch out for:** this is the lowest-frequency creational pattern in LLD interviews — don't force it in. If plain construction from a config is just as cheap and clear as cloning, skip Prototype; only reach for it when reconstruction is genuinely expensive or when a "same-shape template, independent copies" requirement is explicit.

## Continue

Next: [02-structural-patterns.md](02-structural-patterns.md)
