# SOLID Principles — the highest-leverage topic in LLD

Every pattern in [patterns/](patterns/00-overview.md) and every problem in [problems/](problems/00-approach-framework.md) is, underneath, an application of one or more of these five. Internalize these once, deeply, and everything downstream is just pattern-matching.

The recurring meta-point: SOLID is about **isolating the axis of change**. Every principle answers "if requirement X changes, how many existing files do I have to touch?" The target is always: touch 0 existing files, add 1 new file.

---

## S — Single Responsibility Principle

**A class should have one reason to change.** Not "one method" — one *axis of change*.

**Smell:** a `ParkingTicket` class that also computes pricing, formats a receipt string, and sends an email. Three unrelated reasons to change (fee rules change / receipt format changes / notification mechanism changes) are welded into one class.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from datetime import datetime


# BAD: one class owns data, pricing, and notification — three axes of change
class ParkingTicket:
    def __init__(self, vehicle: "Vehicle", entry_time: datetime) -> None:
        self.vehicle = vehicle
        self.entry_time = entry_time

    def calculate_fee(self, exit_time: datetime) -> float:
        ...  # pricing logic

    def send_receipt_email(self, email: str) -> None:
        ...  # SMTP logic


# GOOD: split by axis of change
@dataclass(frozen=True, slots=True)
class ParkingTicket:
    """Immutable value object — just data, no behavior that varies independently."""
    vehicle: "Vehicle"
    entry_time: datetime


class PricingStrategy(ABC):
    @abstractmethod
    def calculate_fee(self, ticket: ParkingTicket, exit_time: datetime) -> float: ...


class NotificationService(ABC):
    @abstractmethod
    def send(self, recipient: str, message: str) -> None: ...
```

> **Pythonic idiom note:** `@dataclass(frozen=True, slots=True)` gives you a correct `__init__`, `__eq__`, `__repr__`, and `__hash__` for free, and `frozen=True` makes the ticket genuinely immutable (assignment to a field raises `FrozenInstanceError`) — the equivalent of Java's `final` fields plus hand-rolled `equals`/`hashCode`/`toString`, in one line. `slots=True` (Python 3.10+) drops the per-instance `__dict__`, which matters if you're creating thousands of tickets. Writing this by hand (`__init__`, `__eq__`, `__repr__`) is the single most common "translating Java into Python" tell interviewers watch for — don't do it when a dataclass covers it.

---

## O — Open/Closed Principle

**Open for extension, closed for modification.** Adding a new variant should mean *adding a new class*, never editing a switch/if-else on a type.

**Smell:** `if vehicle.type == VehicleType.CAR: fee = ... elif vehicle.type == VehicleType.TRUCK: fee = ...` — every new vehicle type means editing this function again.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from datetime import timedelta
from enum import Enum, auto


# BAD
class VehicleType(Enum):
    CAR = auto()
    TRUCK = auto()

def calculate_fee(vehicle_type: VehicleType, parked: timedelta) -> float:
    if vehicle_type is VehicleType.CAR:
        return parked.seconds / 3600 * 20
    elif vehicle_type is VehicleType.TRUCK:
        return parked.seconds / 3600 * 50
    raise ValueError(f"Unhandled vehicle type: {vehicle_type}")


# GOOD — new vehicle type = new class, this abstraction never changes again
class FeeCalculator(ABC):
    @abstractmethod
    def calculate(self, parked_duration: timedelta) -> float: ...


@dataclass(frozen=True, slots=True)
class HourlyRateFeeCalculator(FeeCalculator):
    hourly_rate: float

    def calculate(self, parked_duration: timedelta) -> float:
        return parked_duration.total_seconds() / 3600 * self.hourly_rate


CAR_FEE_CALCULATOR = HourlyRateFeeCalculator(hourly_rate=20)
TRUCK_FEE_CALCULATOR = HourlyRateFeeCalculator(hourly_rate=50)
```

This is literally the [Strategy pattern](patterns/03-behavioral-patterns.md#strategy) — OCP is the principle, Strategy is the mechanism.

> **Pythonic idiom note:** notice the second `GOOD` version doesn't even need two subclasses — parameterizing a single `HourlyRateFeeCalculator` dataclass with `hourly_rate` is *more* idiomatic than Java-style one-subclass-per-variant when the only thing that varies is data, not behavior. Reserve a full subclass (or a distinct `FeeCalculator` implementation) for when the *algorithm* itself differs (e.g. a `FlatFeeCalculator`, a `SurgePricingFeeCalculator`). A second Python-native way to satisfy OCP without any class hierarchy at all is `functools.singledispatch`: register one function per vehicle *type* and dispatch on the runtime type of the first argument — new vehicle type means registering a new function, zero edits to existing code, and no ABC required:
> ```python
> from functools import singledispatch
>
> @singledispatch
> def calculate_fee(vehicle: "Vehicle", parked: timedelta) -> float:
>     raise NotImplementedError(f"No fee rule for {type(vehicle)}")
>
> @calculate_fee.register
> def _(vehicle: "Car", parked: timedelta) -> float:
>     return parked.total_seconds() / 3600 * 20
>
> @calculate_fee.register
> def _(vehicle: "Truck", parked: timedelta) -> float:
>     return parked.total_seconds() / 3600 * 50
> ```
> Say this out loud as an alternative if the interviewer asks "is there another way to do this in Python?" — it shows you're not just transliterating the Strategy pattern from Java, you know Python has a native OCP mechanism for type-based dispatch.

---

## L — Liskov Substitution Principle

**Subtypes must be substitutable for their base type without breaking caller expectations.** A subclass shouldn't strengthen preconditions, weaken postconditions, or throw new exceptions the base type's contract didn't promise.

**Classic violation:** `Square(Rectangle)` where `set_width`/`set_height` are overridden to keep width==height — this breaks any code that does `rect.set_width(5); rect.set_height(10); assert rect.area() == 50`.

**LLD-relevant violation:** a `ReadOnlyAccount(Account)` that overrides `withdraw()` to raise `NotImplementedError`. Any code that polymorphically calls `withdraw()` on an `Account` now has a landmine.

```python
from abc import ABC, abstractmethod


# BAD — violates LSP: caller can't treat every Bird as substitutable
class Bird(ABC):
    @abstractmethod
    def fly(self) -> None: ...

class Sparrow(Bird):
    def fly(self) -> None:
        print("flying")

class Penguin(Bird):
    def fly(self) -> None:
        raise NotImplementedError("penguins can't fly")  # surprise!


# GOOD — model the actual capability, don't force a false is-a
class Bird(ABC):
    """Base capability shared by every bird (walk, eat, ...) — no fly() promise here."""

class FlyingBird(Bird, ABC):
    @abstractmethod
    def fly(self) -> None: ...

class Sparrow(FlyingBird):
    def fly(self) -> None:
        print("flying")

class Penguin(Bird):
    pass  # correctly has no fly() to break — no lie in the type
```

**Interview tell:** if you catch yourself overriding a method just to make it a no-op or raise, that's an LSP red flag — say so out loud and restructure the hierarchy. Interviewers plant this on purpose (e.g., "what if we add a `Motorcycle` that doesn't need a `ParkingSpot` with EV charging" or "what if some `Account` types can't be overdrawn").

> **Pythonic idiom note:** Python enforces LSP even less than Java does — there's no compiler checking that an override's signature is contravariant/covariant-compatible, and duck typing means *any* object with a matching method name will be accepted at the call site regardless of hierarchy. That makes LSP purely a runtime and code-review discipline in Python: a type checker like `mypy` will flag some structural mismatches (wrong parameter/return types under `Liskov`-aware variance rules), but it will happily let `Penguin.fly()` raise at runtime with no warning. Say this out loud if asked "how would tooling catch this" — the honest answer is "static analysis catches signature-shape violations, not behavioral ones; this is what tests and interface design discipline are for."

---

## I — Interface Segregation Principle

**Don't force a class to implement methods it doesn't need.** Prefer several small, focused interfaces over one fat interface.

**Smell:** one `Worker` interface with `code()`, `attend_standup()`, `deploy()` forced onto every implementer including a `ContractDesigner` who doesn't deploy.

```python
from typing import Protocol


# BAD
class Worker(Protocol):
    def code(self) -> None: ...
    def deploy(self) -> None: ...
    def design_ui(self) -> None: ...


# GOOD — segregate by role, as small structural Protocols
class Coder(Protocol):
    def code(self) -> None: ...

class Deployer(Protocol):
    def deploy(self) -> None: ...

class Designer(Protocol):
    def design_ui(self) -> None: ...


class BackendEngineer:
    """Implements Coder and Deployer structurally — no inheritance, no declaration needed."""
    def code(self) -> None: ...
    def deploy(self) -> None: ...

class UIDesigner:
    """Implements only Designer."""
    def design_ui(self) -> None: ...


def ship(deployer: Deployer) -> None:
    deployer.deploy()

ship(BackendEngineer())  # type-checks: BackendEngineer satisfies the Deployer protocol
```

**LLD-relevant example:** a `Printable` + `Scannable` + `Faxable` split for a multi-function-printer problem, instead of one `MultiFunctionDevice` interface that a plain `Printer` is forced to stub out.

> **Pythonic idiom note:** this is the principle where Python's idiom diverges most from Java's. In Java, ISP is a discipline you have to *impose* — the language's nominal typing means every `implements` is an explicit, binding declaration, so a fat interface forces every implementer to stub out methods it doesn't need (or throw `UnsupportedOperationException`, an LSP violation on top of the ISP one). In Python, duck typing makes ISP close to *automatic* by default: a function that calls `.deploy()` on its argument only cares that the argument has a `deploy` method — it never required a class to declare `implements Deployer` in the first place, so there was never a fat-interface tax to pay. `typing.Protocol` (PEP 544) is what lets you keep that structural flexibility while still getting static type checking and a documented contract: `BackendEngineer` never inherits from `Deployer`, it just happens to match its shape, and `mypy`/`pyright` verify the match at every call site. Reach for `Protocol` (not `ABC`) specifically when segregating interfaces by *role* rather than by *hierarchy* — it's the more idiomatic Python answer to "how do you do ISP here," and naming it explicitly is a strong interview signal.

---

## D — Dependency Inversion Principle

**High-level modules should depend on abstractions, not concretions. Concretions depend on abstractions too.** In practice: constructor-inject a dependency, never instantiate a concrete class inside business logic.

**Smell:** `ParkingLot` directly instantiates `CreditCardPayment` inside `process_payment()` — now `ParkingLot` (high-level policy) is coupled to a low-level payment detail, and you can't swap/mock/extend it.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass


# BAD — high-level ParkingLot depends on a concrete low-level class
class ParkingLot:
    def __init__(self) -> None:
        self.payment_method = CreditCardPayment()  # hard dependency, can't swap or mock


# GOOD — depend on the abstraction, inject the concretion
class PaymentMethod(ABC):
    @abstractmethod
    def pay(self, amount: float) -> bool: ...


@dataclass(slots=True)
class ParkingLot:
    payment_method: PaymentMethod  # injected, not constructed internally


@dataclass(frozen=True, slots=True)
class CreditCardPayment(PaymentMethod):
    card_number: str

    def pay(self, amount: float) -> bool:
        return True  # charge the card


# wiring happens at the "top" (main / composition root)
lot = ParkingLot(payment_method=CreditCardPayment(card_number="4111-...-1111"))
```

**Why interviewers love probing this:** DIP is what makes your design *testable* and is exactly what enables the "now add X" follow-up (swap in `UPIPayment`, a `FakePaymentMethod` for tests) without touching `ParkingLot` at all. Always constructor-inject collaborators; never hardcode a concrete class instantiation inside a class that represents business logic.

> **Pythonic idiom note:** DIP and duck typing are closely related but not the same thing, and interviewers sometimes probe the difference. DIP says *depend on an abstraction*; it doesn't mandate that the abstraction be a named `ABC` — in Python you can satisfy DIP with nothing more than an unenforced structural contract: `ParkingLot.__init__(self, payment_method)` never checks the type at all, and *any* object with a `.pay(amount: float) -> bool` method works, satisfying DIP purely through duck typing with zero base classes. The tradeoff: without an `ABC` or `Protocol`, the contract is implicit and undocumented — a future reader has to infer it from usage. The idiomatic middle ground for interview-grade code is to declare the contract as a `Protocol` (documents the shape, gets static-checker support, costs nothing at runtime) unless you specifically need `ABC`'s runtime enforcement (`TypeError` at instantiation if a subclass forgets a method) or shared default behavior across implementations. Either way, the wiring — `CreditCardPayment(...)` being constructed — must live at the composition root (`main()`, a factory, or a DI container), never inside `ParkingLot` itself; that placement, not the choice of `ABC` vs `Protocol` vs nothing, is what DIP actually mandates.

---

## The one-sentence version of each (say these out loud in interviews)

- **S**: one class, one axis of change.
- **O**: add a class (or register a new dispatch case), don't edit an if/elif chain.
- **L**: a subtype must honor every promise the base type made.
- **I**: many small role-interfaces beat one fat interface — in Python, often via `Protocol`.
- **D**: depend on abstractions, inject concretions at the top.

## Continue

Next: [03-python-oop-essentials.md](03-python-oop-essentials.md)
