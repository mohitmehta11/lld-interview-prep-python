# Python OOP Essentials for LLD

This file replaces the old `03-java-oop-essentials.md` / `04-python-oop-essentials.md` /
`05-java-vs-python-cheatsheet.md` trio from the Java+Python version of this repo. There is
no Java track here, and this is not a "refresher" bolted onto a Java-first curriculum — this
*is* the curriculum. You already know general-purpose OOP; what follows is the complete,
interview-grade tour of how Python actually expresses every one of those concepts —
interfaces, inheritance, encapsulation, operator overloading, generics, exceptions,
pattern matching — using idiomatic modern Python (3.11+), not a transliteration of Java
syntax into Python punctuation. Read it top to bottom once, then keep it as a reference:
each section builds on the ones before it, from bare classes up to the tools (`Protocol`,
`dataclass`, `match`/`case`) that actually differentiate a senior Python answer from a
merely-correct one in an LLD interview.

## 1. Classes, instances, `__init__`, `self`

A class is a blueprint; `__init__` is not a constructor in the Java/C++ sense — Python
already allocated the (empty) instance via `__new__` before `__init__` runs. `__init__`'s
job is purely to *initialize* that instance. `self` is an ordinary parameter (not a
keyword) — it's the instance, passed explicitly because Python favors "explicit over
implicit" (`import this`).

```python
class ParkingSpot:
    def __init__(self, spot_id: str, size: "VehicleSize") -> None:
        self.spot_id = spot_id
        self.size = size
        self.parked_vehicle: "Vehicle | None" = None   # None == "free"

    def is_free(self) -> bool:
        return self.parked_vehicle is None

    def park(self, vehicle: "Vehicle") -> None:
        if not self.is_free():
            raise ValueError(f"Spot {self.spot_id} already occupied")
        self.parked_vehicle = vehicle
```

**Why this matters in an LLD interview:** interviewers watch whether you reach for plain
attributes and simple methods first, rather than immediately reaching for getters/setters
or over-engineering — Python rewards starting minimal and adding ceremony (`@property`,
`ABC`, `dataclass`) only when a concrete requirement calls for it.

## 2. Interfaces: `typing.Protocol` vs `abc.ABC` — the decision interviewers watch for

Python has no `interface` keyword. It has two different, non-interchangeable answers to
"I want a contract," and **choosing between them, out loud, is one of the highest-signal
moments in a Python LLD interview** — it's the direct analog of the Java
abstract-class-vs-interface decision, but the axis that matters is different: *nominal
(is-a, declared) vs structural (shaped-like, duck-typed)*.

### `abc.ABC` + `@abstractmethod` — nominal typing, enforced at instantiation

Use when you want a real hierarchy: subclasses must explicitly opt in (`class Foo(Base)`),
and Python will refuse — at instantiation time, with a `TypeError` — to construct a
subclass that hasn't implemented every abstract method. Reach for this for Strategy-style
seams that are part of your own class hierarchy (payment methods, pricing strategies,
notification channels) where you also want to share some concrete implementation.

```python
from abc import ABC, abstractmethod

class PaymentMethod(ABC):
    @abstractmethod
    def pay(self, amount: float) -> bool: ...

    def receipt(self, amount: float) -> str:      # concrete, shared, inherited "for free"
        return f"Charged ${amount:.2f} via {type(self).__name__}"

class CreditCardPayment(PaymentMethod):
    def pay(self, amount: float) -> bool:
        print(f"Charging ${amount:.2f} to credit card")
        return True

# PaymentMethod()  -> TypeError: Can't instantiate abstract class PaymentMethod
#                     with abstract method pay
```

### `typing.Protocol` — structural typing, no inheritance required

Use when you only care that an object *has the right shape* — you don't own the class, or
you don't want to force unrelated classes to share a base just to satisfy a type checker.
This is Python's most natural expression of the
[Interface Segregation Principle](02-solid-principles.md#i--interface-segregation-principle):
define the smallest possible protocol a caller actually needs.

```python
from typing import Protocol

class Notifiable(Protocol):
    def notify(self, message: str) -> None: ...

def alert_all(subscribers: list[Notifiable], message: str) -> None:
    for s in subscribers:
        s.notify(message)      # any object with a matching .notify() satisfies this —
                                # no `class X(Notifiable)` required, no import coupling
```

A class never has to declare `class EmailUser(Notifiable)` — a static type checker
(mypy/pyright) verifies the shape matches, and at runtime nothing is enforced unless you
mark the protocol `@runtime_checkable` (which only checks method *names* exist, not
signatures — don't rely on it for real validation).

**When to reach for which:**

| Use `Protocol` when... | Use `ABC` when... |
|---|---|
| You just need "anything with a `.pay()` method," not a real is-a relationship | Subclasses share state or concrete logic you want inherited |
| The implementing classes are third-party / you don't control them | You control the whole hierarchy and want `TypeError` enforcement at construction time |
| You want to type-hint duck typing without runtime cost | You want a `NotImplementedError`-can't-happen guarantee, not just a type-checker hint |

**Why this matters in an LLD interview:** saying "I'm using a `Protocol` here because I
only need structural compatibility, not a real is-a relationship" is the single sentence
that most reliably signals you understand Python's type system rather than just
translating Java habits — interviewers ask "why not ABC?" specifically to check for this.

## 3. Abstract classes and abstract methods, in depth

`ABC` classes can mix abstract methods, concrete methods, and even fields — this is
Python's version of Java's Template Method pattern (shared skeleton, subclasses fill gaps):

```python
from abc import ABC, abstractmethod

class Vehicle(ABC):
    def __init__(self, license_plate: str, size: "VehicleSize") -> None:
        self.license_plate = license_plate
        self.size = size

    def describe(self) -> str:                 # concrete — shared by every subclass
        return f"{self.vehicle_type} [{self.license_plate}]"

    @property
    @abstractmethod
    def vehicle_type(self) -> str: ...          # abstract property — forces subclasses to fill this in

class Car(Vehicle):
    def __init__(self, plate: str) -> None:
        super().__init__(plate, VehicleSize.MEDIUM)

    @property
    def vehicle_type(self) -> str:
        return "Car"
```

Stacking `@property` under `@abstractmethod` (order matters — `@property` outermost)
forces subclasses to provide a *property*, not just any method, which is idiomatic when
the abstract member is conceptually a piece of data (`vehicle_type`) rather than a
behavior (`pay()`).

**Why this matters in an LLD interview:** a base class with `pass`-bodied methods and a
comment ("subclasses must override this") is the "Java-abstract-class idiom used wrong in
Python" — it compiles and instantiates without complaint, silently defeating the whole
point. Always reach for `ABC` + `@abstractmethod` when you want the enforcement, not a
manual convention.

## 4. Inheritance, `super()`, MRO, multiple inheritance, and mixins

`super()` (zero-argument form, Python 3+) walks the **Method Resolution Order (MRO)** —
computed once per class via C3 linearization — rather than literally "the parent class."
This matters the moment multiple inheritance enters the picture, which Python allows for
classes (Java only allows it for interfaces).

```python
class Vehicle:
    def __init__(self, license_plate: str) -> None:
        self.license_plate = license_plate

    def honk(self) -> str:
        return "generic honk"

class Car(Vehicle):
    def __init__(self, plate: str) -> None:
        super().__init__(plate)         # delegates up the MRO, not "to the parent" literally

    def honk(self) -> str:              # overriding — same signature, resolved at call time
        return "beep beep"

vehicle: Vehicle = Car("KA01")
print(vehicle.honk())                   # "beep beep" — dynamic dispatch, same as Java's @Override
```

### Multiple inheritance and mixins

A **mixin** is a small class meant to add one capability, never instantiated on its own —
Python's cleanest idiom for what Java would need default methods on an interface for:

```python
from datetime import datetime

class TimestampMixin:
    def touch(self) -> None:
        self.updated_at = datetime.now()

class LoggingMixin:
    def log(self, msg: str) -> None:
        print(f"[{type(self).__name__}] {msg}")

class Ticket(TimestampMixin, LoggingMixin):
    ...

print(Ticket.__mro__)
# (<class 'Ticket'>, <class 'TimestampMixin'>, <class 'LoggingMixin'>, <class 'object'>)
```

Method lookup walks `__mro__` left to right; the first class in that order defining the
name wins. Inspect `ClassName.__mro__` (or `ClassName.mro()`) whenever diamond-shaped
inheritance makes resolution non-obvious.

**Why this matters in an LLD interview:** prefer composition or a shallow mixin over deep
multiple-inheritance chains — being able to explain *why* (MRO complexity grows fast,
fragile base class problem) is more valuable than proving you can write `class X(A, B, C,
D)`. If you catch yourself with more than two mixins, that's usually a sign the design
wants composition instead (see §12).

## 5. Method overriding vs Java-style overloading

Overriding — same signature, subclass redefines behavior, resolved at runtime — works
exactly as shown above. Python has **no method overloading in the Java sense**: you cannot
define two methods with the same name and different parameter lists in one class; the
second definition just replaces the first. Python's idiomatic answers to "I want different
behavior per argument type/shape":

### Default and keyword arguments — the everyday substitute

Covers the vast majority of cases Java would use overloading for:

```python
class ParkingTicket:
    def __init__(
        self,
        ticket_id: str,
        vehicle: "Vehicle",
        entry_time: "datetime | None" = None,   # substitutes for a second Java constructor overload
    ) -> None:
        self.ticket_id = ticket_id
        self.vehicle = vehicle
        self.entry_time = entry_time or datetime.now()
```

### `functools.singledispatch` — dispatch on the type of the first argument

The Pythonic answer when behavior genuinely varies *by type*, e.g. a visitor-like fee
calculator over a small closed set of vehicle types:

```python
from functools import singledispatch

@singledispatch
def calculate_fee(vehicle: object, hours: int) -> float:
    raise NotImplementedError(f"No fee rule for {type(vehicle).__name__}")

@calculate_fee.register
def _(vehicle: "Motorcycle", hours: int) -> float:
    return hours * 1.0

@calculate_fee.register
def _(vehicle: "Car", hours: int) -> float:
    return hours * 2.5

@calculate_fee.register
def _(vehicle: "Truck", hours: int) -> float:
    return hours * 5.0
```

`calculate_fee(car, 3)` dispatches on `type(car)` at call time — this is a free-function
alternative to a polymorphic `.fee()` method on each `Vehicle` subclass, useful when the
fee logic doesn't conceptually belong to `Vehicle` itself (e.g. it's a pricing policy
that changes independently of the vehicle hierarchy).

### `typing.overload` — type-checker-only signature documentation

Purely a static-typing aid; only the final, undecorated implementation runs:

```python
from typing import overload

class Cache:
    @overload
    def get(self, key: str) -> str: ...
    @overload
    def get(self, key: str, default: str) -> str: ...

    def get(self, key: str, default: "str | None" = None) -> "str | None":
        return self._store.get(key, default)
```

**Why this matters in an LLD interview:** if you find yourself wanting Java-style
overloading, that's the cue to say out loud "Python doesn't overload methods — I'd use a
default argument here / `singledispatch` if the branching is really by type" rather than
faking it with `*args`/`isinstance` checks, which reads as translated-Java, not Python.

## 6. Encapsulation: name mangling, `@property`, and why there's no true "private"

Python has no access modifiers. Convention carries the whole weight:

| Convention | Meaning |
|---|---|
| `name` | public — part of the API |
| `_name` | "internal use," not enforced, just a signal to callers/tools |
| `__name` | name-mangled to `_ClassName__name` — avoids accidental collisions in subclasses, still accessible if you really want to |

```python
class BankAccount:
    def __init__(self, balance: float) -> None:
        self.__balance = balance    # stored as self._BankAccount__balance

    def deposit(self, amount: float) -> None:
        self.__balance += amount

acct = BankAccount(100)
acct._BankAccount__balance          # works — "private" is a strong convention, not a wall
```

Name mangling exists to prevent a subclass from *accidentally* shadowing a base class's
internal attribute, not to lock external callers out. Python's philosophy ("we're all
consenting adults here") trades enforced privacy for trusting callers to respect
underscores.

### `@property` / `@x.setter` — validated or computed attributes, not manual getters/setters

Writing `get_x()`/`set_x()` everywhere is the "Java in Python syntax" anti-pattern —
start with a plain public attribute; reach for `@property` only once you need
validation or a computed value, so external code never has to change its access syntax:

```python
class ParkingSpot:
    def __init__(self, spot_id: str, size: "VehicleSize") -> None:
        self._spot_id = spot_id
        self.size = size
        self._parked_vehicle: "Vehicle | None" = None

    @property
    def is_free(self) -> bool:                     # read as `spot.is_free`, no parens
        return self._parked_vehicle is None

    @property
    def parked_vehicle(self) -> "Vehicle | None":
        return self._parked_vehicle

    @parked_vehicle.setter
    def parked_vehicle(self, vehicle: "Vehicle") -> None:
        if self._parked_vehicle is not None:
            raise ValueError("Spot already occupied")   # validation lives on assignment
        self._parked_vehicle = vehicle
```

**Why this matters in an LLD interview:** an interviewer seeing `get_foo()`/`set_foo()`
pairs in your Python code will usually ask "why not a property?" — have the answer ready
("I only reach for `@property` when there's real validation or computation; plain
attributes otherwise") rather than defaulting to getters/setters out of Java habit.

## 7. `@staticmethod` vs `@classmethod` vs instance methods

| Kind | First param | Knows about | Typical LLD use |
|---|---|---|---|
| instance method | `self` | the specific instance | normal behavior (`park()`, `pay()`) |
| `@classmethod` | `cls` | the class (and, via `cls`, the *actual* subclass at call time) | alternate constructors, subclass-aware factories |
| `@staticmethod` | neither | nothing — just namespaced in the class | a pure helper that logically belongs with the class but needs no instance or class state |

```python
class ParkingLot:
    _total_lots_created: int = 0

    def __init__(self, lot_id: str) -> None:
        self.lot_id = lot_id
        ParkingLot._total_lots_created += 1

    @classmethod
    def create_default(cls) -> "ParkingLot":
        # uses `cls`, not `ParkingLot` — a subclass calling this gets an instance
        # of the SUBCLASS, which a @staticmethod factory could not do
        return cls(lot_id=f"LOT-{cls._total_lots_created + 1}")

    @staticmethod
    def is_valid_lot_id(lot_id: str) -> bool:
        return lot_id.startswith("LOT-")   # doesn't touch self or cls at all
```

`@classmethod` is the idiomatic Python home for the Factory pattern
([details here](patterns/01-creational-patterns.md#factory-method)) precisely because
`cls(...)` respects subclassing — a plain `@staticmethod` factory would hardcode the base
class name and break for subclasses.

**Why this matters in an LLD interview:** using `@classmethod` for `from_json`,
`create_default`, or other alternate-constructor factories (rather than overloading
`__init__` with sentinel default arguments) is a small but real signal of Python fluency.

## 8. `@dataclass` vs hand-written `__init__`/`__eq__`/`__repr__`

`@dataclass` is Python's idiomatic value-object generator — reach for it the moment a
class is "mostly data," and hand-write only when you need custom construction logic,
inheritance nuances dataclasses don't handle well, or `__slots__` tuning beyond what
`@dataclass(slots=True)` gives you (Python 3.10+).

```python
from dataclasses import dataclass, field
from datetime import datetime

@dataclass(frozen=True)
class Money:
    cents: int

    def __add__(self, other: "Money") -> "Money":
        return Money(self.cents + other.cents)   # new instance — respects frozen immutability

@dataclass
class ParkingTicket:
    vehicle: "Vehicle"
    entry_time: datetime
    exit_time: "datetime | None" = None
    events: list[str] = field(default_factory=list)   # mutable default MUST use default_factory
```

`@dataclass` auto-generates, from the field list alone:
- `__init__` (in field-declaration order, with defaults respected)
- `__repr__` (readable, e.g. `ParkingTicket(vehicle=..., entry_time=...)`)
- `__eq__` (field-by-field tuple comparison)
- `__hash__` **only if you ask for it** — by default a mutable dataclass (`eq=True`,
  the default) gets `__hash__ = None` (unhashable, correctly, since mutable objects
  shouldn't be dict keys/set members); `frozen=True` restores a generated `__hash__`
  automatically since the fields can't change under you.

```python
Money(100) == Money(100)     # True — field-by-field equality, for free
{Money(100), Money(200)}     # works — frozen dataclass is hashable
```

**Interview trap:** `events: list = []` as a default is the classic Python bug — one list
object shared across every instance ever created, because default argument values are
evaluated once at class-definition time, not per call. Always use
`field(default_factory=list)`. Naming this unprompted is a strong signal.

**Why this matters in an LLD interview:** hand-writing `__init__`/`__eq__`/`__repr__` for
a plain data holder is boilerplate an interviewer will read as either unfamiliarity with
`dataclass` or (if you explain why) a deliberate choice — e.g. you need a custom `__eq__`
that ignores a timestamp field, or the class has real behavior beyond data + trivial
methods and you want that to read clearly as "not just a dataclass."

## 9. Dunder methods — operator overloading and protocol hooks

Dunder ("double underscore") methods are how a class opts into Python's built-in
operators, syntax, and protocols. This is Python's direct, and more powerful, answer to
Java's `equals`/`hashCode`/`toString`/`Comparable`/operator restrictions (Java only
overloads operators for a handful of built-in types; Python lets *any* class do it).

```python
from functools import total_ordering

@total_ordering
class ParkingTicket:
    def __init__(self, ticket_id: str, entry_time: datetime, fee: float) -> None:
        self.ticket_id = ticket_id
        self.entry_time = entry_time
        self.fee = fee

    def __repr__(self) -> str:                     # unambiguous, dev-facing: repr(ticket), REPL echo
        return f"ParkingTicket(id={self.ticket_id!r}, fee={self.fee})"

    def __str__(self) -> str:                       # readable, user-facing: print(ticket), str(ticket)
        return f"Ticket {self.ticket_id}: ${self.fee:.2f}"

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, ParkingTicket):
            return NotImplemented
        return self.entry_time == other.entry_time

    def __lt__(self, other: "ParkingTicket") -> bool:
        return self.entry_time < other.entry_time   # total_ordering derives <=, >, >= from this + __eq__

    def __add__(self, other: "ParkingTicket") -> float:
        return self.fee + other.fee                 # arbitrary meaning, use sparingly/only when intuitive
```

`__repr__` vs `__str__`: if you only implement one, implement `__repr__` — it's the
fallback for `__str__`, `print()`, and container display (`[ticket1, ticket2]` calls
`repr` on each element), and should ideally be unambiguous enough to reconstruct the
object.

Container and resource-management protocols, the ones that come up constantly in LLD:

```python
class Garage:
    def __init__(self, spots: list["ParkingSpot"]) -> None:
        self._spots = spots

    def __len__(self) -> int:
        return len(self._spots)

    def __getitem__(self, index: int) -> "ParkingSpot":
        return self._spots[index]

    def __iter__(self):
        return iter(self._spots)          # lets `for spot in garage:` and `list(garage)` just work

class SeatLock:
    def __init__(self, seat: "Seat") -> None:
        self.seat = seat

    def __enter__(self) -> "Seat":
        self.seat.lock()
        return self.seat

    def __exit__(self, exc_type, exc_value, traceback) -> None:
        self.seat.unlock()                # guaranteed to run even if the with-block raises

with SeatLock(seat) as locked_seat:
    process_payment(locked_seat)
```

**Why this matters in an LLD interview:** implementing `__lt__`/`__eq__` on a domain
object so `tickets.sort()` and `heapq` just work — instead of writing a manual comparator
class — is exactly the kind of idiom-fluency interviewers are listening for. Likewise,
reaching for `__enter__`/`__exit__` on anything acquire/release-shaped (locks, seats,
connections) rather than a manual try/finally is the Pythonic answer to Java's
try-with-resources.

## 10. `__slots__` — when it matters, and when it doesn't

By default, every instance carries a `__dict__` for arbitrary attribute storage — flexible,
but with real per-instance memory overhead and zero protection against attribute typos
(`self.statuss = "x"` just silently creates a new attribute). `__slots__` declares a fixed
attribute set, trading flexibility for memory and safety:

```python
class ParkingSpot:
    __slots__ = ("spot_id", "size", "parked_vehicle")   # no per-instance __dict__ at all

    def __init__(self, spot_id: str, size: "VehicleSize") -> None:
        self.spot_id = spot_id
        self.size = size
        self.parked_vehicle: "Vehicle | None" = None

spot = ParkingSpot("A1", VehicleSize.CAR)
spot.statuss = "x"   # AttributeError: 'ParkingSpot' object has no attribute 'statuss'
                      # — the typo is caught immediately instead of silently succeeding
```

Or, with `@dataclass(slots=True)` (3.10+), get both generated dunders and slots together:

```python
@dataclass(slots=True)
class Point:
    x: int
    y: int
```

Tradeoffs to state explicitly: `__slots__` classes cannot get new attributes added ad hoc,
multiple inheritance with slots is finicky (only one base in the hierarchy may define
non-empty slots, generally), and you lose the ability to `weakref` an instance unless you
add `__weakref__` to the slots explicitly. In practice: default to *not* using
`__slots__`; reach for it when you're modeling many thousands of small, fixed-shape
objects (grid cells, seats, spots) where the `__dict__` overhead is measurable, or when
attribute-typo safety on a widely-shared class is worth the rigidity.

**Why this matters in an LLD interview:** bringing up `__slots__` unprompted for a
high-cardinality object like a parking spot or a board cell (thousands of instances)
signals memory-consciousness; knowing *not* to reach for it reflexively (most classes
don't need it) signals you're not cargo-culting.

## 11. Enums — `Enum`, `IntEnum`, `Flag`, `auto()`

Python enums, like Java's, are full classes: they can carry data, methods, and properties
per member. This is Python's answer to "enum with behavior," and is frequently the
*correct minimal* choice when the variant set is small, fixed, and known upfront —
contrast with Strategy (a `Protocol`/`ABC` + one class per variant) when behavior needs
to be injected or the variant set is expected to grow.

```python
from enum import Enum, IntEnum, Flag, auto

class VehicleSize(Enum):
    MOTORCYCLE = 1
    CAR = 2
    TRUCK = 4

    @property
    def spots_required(self) -> int:
        return self.value

class Direction(Enum):
    UP = auto()
    DOWN = auto()

    def opposite(self) -> "Direction":
        return Direction.DOWN if self is Direction.UP else Direction.UP

class Priority(IntEnum):          # IntEnum: also compares/sorts as an int, useful for a heapq priority
    LOW = 1
    MEDIUM = 2
    HIGH = 3

class Permission(Flag):           # Flag: members combine with bitwise operators
    READ = auto()
    WRITE = auto()
    EXECUTE = auto()

editor_access = Permission.READ | Permission.WRITE
Permission.WRITE in editor_access   # True
```

Use plain `Enum` by default; `IntEnum` only when you specifically need the members to
behave as ints (e.g. sorted in a `heapq`, or serialized as small integers); `Flag` when
members represent combinable bit-flags (permissions, feature toggles).

**Why this matters in an LLD interview:** using an `Enum` with a computed `@property`
(like `spots_required` above) instead of a bare `int`/`str` constant, or instead of a full
class hierarchy, shows you can tell when the variant set is closed and small enough that
polymorphism would be overkill.

## 12. Composition over inheritance — and why duck typing makes it easier in Python

Prefer "has-a" (a field holding a collaborator) over "is-a" (inheritance) whenever the
relationship is really about delegating a capability rather than being a specialized kind
of something. Python makes composition more natural than Java does, because a composed
collaborator doesn't need to implement any particular interface at all — duck typing means
"anything with the right method" already satisfies the caller, with `Protocol` available
purely as an optional, checkable type hint on top:

```python
class ReceiptPrinter:
    def render(self, ticket: "ParkingTicket") -> str:
        return f"Receipt for {ticket.ticket_id}: ${ticket.fee:.2f}"

class ParkingLot:
    def __init__(self, receipt_printer: "ReceiptPrinter") -> None:
        self._receipt_printer = receipt_printer     # composition: ParkingLot HAS-A printer

    def checkout(self, ticket: "ParkingTicket") -> str:
        return self._receipt_printer.render(ticket)

# swap in any object with a compatible .render(ticket) — no shared base class required
lot = ParkingLot(receipt_printer=ReceiptPrinter())
```

If `ParkingLot` instead *inherited* from `ReceiptPrinter`, every `ParkingLot` would be
forced to carry printer internals it doesn't need, and you'd be unable to swap printers at
runtime — a classic inheritance-overreach smell.

**Why this matters in an LLD interview:** when an interviewer probes "why composition
here, not inheritance?", the answer is the is-a/has-a test plus runtime flexibility
(swappable collaborators, no forced coupling to a base class's full surface area) —
stating this explicitly is worth more than the code itself.

## 13. Generics and typing: `TypeVar`, `Generic`, PEP 604, and modern container hints

Modern Python (3.9+ for generic builtins, 3.10+ for `X | Y`) lets you type-hint containers
directly without importing from `typing`:

```python
# modern (3.9+ / 3.10+) — prefer this
vehicles: list["Vehicle"] = []
spots_by_id: dict[str, "ParkingSpot"] = {}
maybe_spot: "ParkingSpot | None" = None            # PEP 604 union syntax

# legacy (still valid, seen in older codebases — recognize it, don't write it)
from typing import List, Dict, Optional
vehicles_legacy: List["Vehicle"] = []
maybe_spot_legacy: Optional["ParkingSpot"] = None
```

Writing your own generic container class — rarer in LLD than *using* `list[Vehicle]`, but
worth knowing cold when a problem calls for a generic cache, box, or pair:

```python
from typing import Generic, TypeVar

T = TypeVar("T")

class Box(Generic[T]):
    def __init__(self, item: T) -> None:
        self._item = item

    def get(self) -> T:
        return self._item

int_box: Box[int] = Box(42)

# bounded type variable — T must support comparison, mirrors Java's <T extends Comparable<T>>
from typing import Protocol

class Comparable(Protocol):
    def __lt__(self, other: object) -> bool: ...

C = TypeVar("C", bound=Comparable)

class SortedBox(Generic[C]):
    def __init__(self) -> None:
        self._items: list[C] = []

    def add(self, item: C) -> None:
        self._items.append(item)
        self._items.sort()
```

Python 3.12 introduced a terser generic syntax (`class Box[T]:`) as an alternative to
`TypeVar` + `Generic` — know it exists, but `TypeVar`/`Generic` is still the form you'll
see in most production code and is safe to default to.

**Why this matters in an LLD interview:** type-hinting public method signatures
throughout your design (`list[Vehicle]`, `ParkingSpot | None`, not bare `list`/no hints)
is the single easiest way to read as rigorous — it's the closest Python gets to a
compile-time contract, and interviewers explicitly look for it in a "senior Python" answer.

## 14. Exceptions: no checked exceptions, and the idioms that replace them

Every Python exception is, in Java terms, "unchecked" — there is no `throws` declaration,
and the compiler never forces a caller to handle anything. This is a deliberate design
choice, not a gap, and it comes with its own idioms rather than a workaround:

### Custom exception hierarchies

Define a small domain-specific hierarchy — it reads as senior and gives callers a single
place to catch related failures at whatever granularity they need:

```python
class ParkingError(Exception):
    """Base for all domain errors in this system."""

class SpotNotAvailableError(ParkingError):
    def __init__(self, vehicle: "Vehicle") -> None:
        super().__init__(f"No available spot for {vehicle.vehicle_type}")
        self.vehicle = vehicle

class InvalidTicketError(ParkingError):
    pass

def park(vehicle: "Vehicle") -> "ParkingSpot":
    spot = find_available_spot(vehicle)
    if spot is None:
        raise SpotNotAvailableError(vehicle)
    spot.park(vehicle)
    return spot

try:
    park(motorcycle)
except ParkingError:              # catches SpotNotAvailableError, InvalidTicketError, etc.
    notify_operator()
```

### `raise ... from ...` — preserving causality across abstraction layers

When you catch a low-level exception and re-raise a domain-specific one, chain them so the
original traceback isn't lost — critical for debugging, and a clear senior signal:

```python
def load_ticket(ticket_id: str) -> "ParkingTicket":
    try:
        data = database.fetch(ticket_id)
    except ConnectionError as exc:
        raise InvalidTicketError(f"Could not load ticket {ticket_id}") from exc
    return ParkingTicket(**data)
```

### Context managers instead of forced try/finally

Because there's no checked-exception mechanism forcing a caller to clean up, Python instead
makes cleanup automatic via `__enter__`/`__exit__` (see §9) or `contextlib`:

```python
from contextlib import contextmanager

@contextmanager
def reserved_seat(seat: "Seat"):
    seat.lock()
    try:
        yield seat
    finally:
        seat.unlock()          # guaranteed release, regardless of what happens inside the `with`

with reserved_seat(seat) as s:
    process_payment(s)          # if this raises, unlock() still runs
```

**Why this matters in an LLD interview:** Python's absence of checked exceptions means
*you* have to proactively demonstrate the discipline the compiler would otherwise force in
Java — a clean exception hierarchy, `raise ... from ...` at layer boundaries, and
context managers for anything acquire/release-shaped. Not doing this, and just letting
bare `Exception`s or unrelated stack traces surface, reads as junior.

## 15. `match`/`case` — structural pattern matching for LLD-style dispatch

`match`/`case` (3.10+) goes well beyond a C-style `switch`: it matches on **type and
shape** simultaneously, which makes it a genuinely useful alternative to an `isinstance`
chain for visitor-like dispatch over a small closed set of types — exactly the kind of
`if isinstance(x, Car): ... elif isinstance(x, Truck): ...` chain that's usually a
polymorphism smell, but where a dataclass-shaped, closed set of event/command types makes
`match` more readable than either a chain or a full Visitor class hierarchy.

```python
from dataclasses import dataclass

@dataclass
class ParkRequest:
    vehicle: "Vehicle"

@dataclass
class ExitRequest:
    ticket_id: str

@dataclass
class ExtendStayRequest:
    ticket_id: str
    extra_hours: int

Event = ParkRequest | ExitRequest | ExtendStayRequest

def handle(event: Event) -> str:
    match event:
        case ParkRequest(vehicle=vehicle) if vehicle.size == VehicleSize.TRUCK:
            return "routing to oversized bay"        # type + guard condition
        case ParkRequest(vehicle=vehicle):
            return f"parking {vehicle.vehicle_type}"
        case ExitRequest(ticket_id=ticket_id):
            return f"closing out {ticket_id}"
        case ExtendStayRequest(ticket_id=ticket_id, extra_hours=hours) if hours > 24:
            return "extension too long, needs manual approval"
        case ExtendStayRequest(ticket_id=ticket_id):
            return f"extending {ticket_id}"
        case _:
            raise ValueError(f"Unhandled event: {event!r}")
```

This destructures each dataclass by field name, matches on the concrete type, and supports
`if` guards — one construct doing what would otherwise take an `isinstance` chain plus
separate attribute access, or a full Visitor pattern with a `.accept()` method on every
event type. For a genuinely open-ended, extensible set of event types (new event classes
added by other teams later), prefer real polymorphism (a `handle()` method per class, or
the [Visitor pattern](patterns/03-behavioral-patterns.md)) instead — `match`/`case` is at
its best over a small, closed, locally-defined set of shapes, which is exactly what most
LLD command/event dispatch looks like.

**Why this matters in an LLD interview:** using `match`/`case` on a closed set of
`@dataclass` event types, with guards, is a crisp way to show modern Python fluency —
but knowing when to prefer real polymorphism instead (open-ended, externally-extended
variant sets) is the more important half of the answer.

## Continue

Next: [04-concurrency-essentials.md](04-concurrency-essentials.md) for how these idioms
extend to thread-safety and async, then [05-common-mistakes.md](05-common-mistakes.md) for
a final pass before [06-final-checklist.md](06-final-checklist.md).
