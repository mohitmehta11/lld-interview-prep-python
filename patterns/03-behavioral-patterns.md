# Behavioral Patterns

Behavioral patterns answer: **how do objects communicate and vary their behavior at runtime without hardcoding the exact algorithm/flow/receiver?** This is the largest pattern family and the one interviewers reach for most, because most LLD problems are really "model behavior that varies," not "model data that varies."

**Prioritize Strategy, Observer, State, Command, Chain of Responsibility if short on time — the rest are lower-frequency in practice.**

This file treats Python as the primary and only language. Wherever Python's idioms diverge materially from the generic-OOP textbook description of a pattern — first-class functions, closures, `functools`, structural typing via `Protocol`, or dynamic attribute/`__class__` tricks — there's a **Pythonic idiom note** calling it out explicitly, because knowing *when to skip the class hierarchy* is itself a senior-engineer signal in an interview.

---

## Strategy

**Intent:** Define a family of interchangeable algorithms/policies behind one interface, and let the client pick/inject the one it needs at runtime.

**When to reach for it in LLD:**
- Requirement language: "fee/pricing/discount/split **differs by** type," "support multiple payment/notification/sorting methods" — the canonical "varies by type" tell.
- You catch yourself about to write `if type == X: ... elif type == Y: ...` on behavior (not just data).

**Structure:**
```
PricingStrategy (interface)
  ├─ HourlyPricingStrategy
  └─ FlatRatePricingStrategy

ParkingLot
  └─ has-a[1] PricingStrategy   // injected, swappable without touching ParkingLot
```

**Python — the classic, class-based shape:**
```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from datetime import timedelta


class PricingStrategy(ABC):
    @abstractmethod
    def calculate_fee(self, parked_duration: timedelta) -> float: ...


class HourlyPricingStrategy(PricingStrategy):
    def __init__(self, rate_per_hour: float = 20.0) -> None:
        self._rate_per_hour = rate_per_hour

    def calculate_fee(self, parked_duration: timedelta) -> float:
        hours = parked_duration.total_seconds() / 3600
        return hours * self._rate_per_hour


class FlatRatePricingStrategy(PricingStrategy):
    def __init__(self, flat_fee: float = 50.0) -> None:
        self._flat_fee = flat_fee

    def calculate_fee(self, parked_duration: timedelta) -> float:
        return self._flat_fee  # ignores duration entirely — still substitutable


@dataclass
class ParkingLot:
    pricing_strategy: PricingStrategy  # injected — see DIP

    def bill(self, parked: timedelta) -> float:
        return self.pricing_strategy.calculate_fee(parked)


lot = ParkingLot(pricing_strategy=HourlyPricingStrategy(rate_per_hour=20.0))
```

**Pythonic idiom note:** Python rarely needs a one-method `ABC` to get Strategy's benefit, because functions are first-class objects. The "interface" can be a `Callable` type alias, and each "concrete strategy" a plain function or a `functools.partial`/closure capturing configuration — no class, no `self`:

```python
from functools import partial
from typing import Callable

PricingFn = Callable[[timedelta], float]


def hourly_pricing(parked_duration: timedelta, *, rate_per_hour: float) -> float:
    return parked_duration.total_seconds() / 3600 * rate_per_hour


def flat_rate_pricing(parked_duration: timedelta, *, flat_fee: float) -> float:
    return flat_fee


class ParkingLot:
    def __init__(self, pricing_fn: PricingFn) -> None:
        self._pricing_fn = pricing_fn

    def bill(self, parked: timedelta) -> float:
        return self._pricing_fn(parked)


lot = ParkingLot(pricing_fn=partial(hourly_pricing, rate_per_hour=20.0))
```

Reach for the class-based version when a strategy needs to carry rich internal state, expose multiple related methods, or participate in `isinstance` checks elsewhere in the design. Reach for the closure/`Callable` version when it's a single pure function of its inputs plus some captured config — which describes most pricing/discount/scoring strategies in interview problems. A `typing.Protocol` is the middle ground: it gives structural typing so any object with a matching method satisfies the "interface" without inheriting from an `ABC` — useful when third-party or otherwise-unrelated classes need to plug in:

```python
from typing import Protocol


class SupportsPricing(Protocol):
    def calculate_fee(self, parked_duration: timedelta) -> float: ...
```

Say out loud which of the three (ABC, `Protocol`, plain callable) you're choosing and why — it's a stronger signal than defaulting to the Java-shaped ABC hierarchy by reflex.

**Related principle:** Strategy *is* the mechanism for [OCP](../02-solid-principles.md#o--openclosed-principle) — see the direct callout in [../02-solid-principles.md](../02-solid-principles.md#o--openclosed-principle). Constructor injection of the strategy is [DIP](../02-solid-principles.md#d--dependency-inversion-principle) in action.

**Used in:** [../problems/01-parking-lot.md](../problems/01-parking-lot.md) (pricing by vehicle type/duration), [../problems/06-splitwise-expense-sharing.md](../problems/06-splitwise-expense-sharing.md) (equal/percentage/exact split strategies).

**Watch out for:** don't build a `Strategy` interface (or even a `Callable` seam) for behavior that never actually varies in the problem — if there's exactly one pricing rule and no signal a second is coming, a plain method is correct and a Strategy is premature abstraction.

---

## Observer

**Intent:** Define a one-to-many dependency so that when one object (the subject) changes state, all registered dependents are notified automatically, without the subject knowing their concrete types.

**When to reach for it in LLD:**
- Requirement language: "notify all subscribers when X happens," "update the display board when a spot's status changes," "multiple channels should react to one event" — a variable, growable set of listeners reacting to one source of truth.

**Structure:**
```
Subject (interface): subscribe(Observer), unsubscribe(Observer), notify_all()
  └─ ParkingFloor — has-a[*] Observer

Observer (interface): on_update(event)
  ├─ DisplayBoard
  └─ MobileAppNotifier
```

**Python — the classic, class-based shape:**
```python
from abc import ABC, abstractmethod


class Observer(ABC):
    @abstractmethod
    def on_update(self, event: str) -> None: ...


class ParkingFloor:
    def __init__(self) -> None:
        self._observers: list[Observer] = []

    def subscribe(self, observer: Observer) -> None:
        self._observers.append(observer)

    def unsubscribe(self, observer: Observer) -> None:
        self._observers.remove(observer)

    def on_spot_freed(self, spot_id: str) -> None:
        event = f"Spot {spot_id} is now free"
        for observer in self._observers:
            observer.on_update(event)


class DisplayBoard(Observer):
    def on_update(self, event: str) -> None:
        print(f"[Board] {event}")
```

**Pythonic idiom note:** when an observer is really just "one function that runs on an event," a `list[Observer]` of one-method objects is ceremony you can skip. Model subscribers as a list of plain callables instead — subscribing is `append`, and any function, bound method, or lambda satisfies the "interface" for free:

```python
from typing import Callable

EventHandler = Callable[[str], None]


class ParkingFloor:
    def __init__(self) -> None:
        self._handlers: list[EventHandler] = []

    def subscribe(self, handler: EventHandler) -> None:
        self._handlers.append(handler)

    def on_spot_freed(self, spot_id: str) -> None:
        event = f"Spot {spot_id} is now free"
        for handler in self._handlers:
            try:
                handler(event)
            except Exception as exc:                     # noqa: BLE001 — isolate one bad subscriber
                print(f"observer {handler!r} failed: {exc}")


def print_to_board(event: str) -> None:
    print(f"[Board] {event}")


floor = ParkingFloor()
floor.subscribe(print_to_board)
floor.subscribe(lambda event: send_push_notification(event))  # any callable works, no class needed
```

This scales down cleanly to `asyncio` too: swap the loop body for `await handler(event)` and the handlers for coroutine functions, and you have a minimal async pub-sub with no interface changes. Reach for full `Observer` objects instead of bare callables when a subscriber needs its own state, its own lifecycle (`connect()`/`disconnect()`), or must be inspected/compared as a first-class entity (e.g., "list all currently subscribed display boards by name").

**Related principle:** [OCP](../02-solid-principles.md#o--openclosed-principle) — adding a new observer type never touches `ParkingFloor`; also decouples subject from dependents, an [ISP](../02-solid-principles.md#i--interface-segregation-principle)-flavored win since observers only need to implement `on_update` (or, in the callable form, nothing at all).

**Used in:** [../problems/10-notification-and-observer-system.md](../problems/10-notification-and-observer-system.md) (the canonical pub-sub showcase), [../problems/01-parking-lot.md](../problems/01-parking-lot.md) (display boards reacting to spot-availability changes).

**Watch out for:** if there's exactly one listener and it's never going to grow, a direct method call is simpler than standing up subject/observer machinery — Observer earns its keep when the *set* of listeners is variable or unknown ahead of time. Also watch notification ordering/failure isolation: one throwing observer shouldn't be able to break delivery to the rest — wrap each call in its own `try`/`except`, don't let one bad listener take down the loop (shown above).

---

## State

**Intent:** Let an object alter its behavior when its internal state changes, by giving each state its own class and delegating state-dependent behavior to the current state object, instead of branching on a status field everywhere.

**When to reach for it in LLD:**
- Requirement language: the problem explicitly names states and legal transitions between them ("Idle → HasMoney → Dispensing → OutOfStock," "elevator is Idle/Moving/DoorsOpen") and different operations are valid/invalid depending on current state.

**Structure:**
```
VendingMachineState (interface): insert_coin(), select_item(), dispense()
  ├─ IdleState
  ├─ HasMoneyState
  ├─ DispensingState
  └─ OutOfStockState

VendingMachine
  └─ has-a[1] VendingMachineState (current)   // delegates every call to it, swaps it on transition
```

**Python:**
```python
from abc import ABC, abstractmethod


class VendingMachineState(ABC):
    @abstractmethod
    def insert_coin(self, machine: "VendingMachine") -> None: ...

    @abstractmethod
    def select_item(self, machine: "VendingMachine") -> None: ...

    @abstractmethod
    def dispense(self, machine: "VendingMachine") -> None: ...


class IdleState(VendingMachineState):
    def insert_coin(self, machine: "VendingMachine") -> None:
        machine.state = HasMoneyState()

    def select_item(self, machine: "VendingMachine") -> None:
        raise RuntimeError("Insert coin first")

    def dispense(self, machine: "VendingMachine") -> None:
        raise RuntimeError("Insert coin first")


class HasMoneyState(VendingMachineState):
    def insert_coin(self, machine: "VendingMachine") -> None:
        pass  # accept extra coin, stay in this state

    def select_item(self, machine: "VendingMachine") -> None:
        machine.state = DispensingState()

    def dispense(self, machine: "VendingMachine") -> None:
        raise RuntimeError("Select an item first")


class DispensingState(VendingMachineState):
    def insert_coin(self, machine: "VendingMachine") -> None:
        raise RuntimeError("Already dispensing")

    def select_item(self, machine: "VendingMachine") -> None:
        raise RuntimeError("Already dispensing")

    def dispense(self, machine: "VendingMachine") -> None:
        machine.state = IdleState()


class VendingMachine:
    def __init__(self) -> None:
        self.state: VendingMachineState = IdleState()  # current state, swapped on transition

    def insert_coin(self) -> None:
        self.state.insert_coin(self)

    def select_item(self) -> None:
        self.state.select_item(self)

    def dispense(self) -> None:
        self.state.dispense(self)
```

**Pythonic idiom note:** Python lets an object reassign its *own* `__class__` at runtime, which is a genuine, if unusual, way to implement State with zero delegation boilerplate — there's no `machine.state.insert_coin(machine)` indirection; the machine literally becomes a `HasMoneyState` instance:

```python
class VendingMachine:
    def insert_coin(self) -> None:
        raise RuntimeError(f"insert_coin invalid in {type(self).__name__}")

    def select_item(self) -> None:
        raise RuntimeError(f"select_item invalid in {type(self).__name__}")


class IdleVendingMachine(VendingMachine):
    def insert_coin(self) -> None:
        self.__class__ = HasMoneyVendingMachine


class HasMoneyVendingMachine(VendingMachine):
    def select_item(self) -> None:
        self.__class__ = DispensingVendingMachine


class DispensingVendingMachine(VendingMachine):
    def dispense(self) -> None:
        self.__class__ = IdleVendingMachine
```

This is a real technique (used in some Python state-machine libraries) but it's also easy to overuse: it only works because all state subclasses share the same instance attributes and constructor shape, it's opaque to static type checkers (`mypy` cannot narrow `self.__class__`), and `isinstance` checks on the object become misleading mid-lifetime. For an interview, mention it as a "Python can do this, here's the tradeoff" aside rather than defaulting to it — the delegation-based version above is the safer, more broadly understood answer to lead with. Also contrast with **enum-with-behavior**: if the states are few, fixed, and transitions are simple data lookups rather than branching logic, a behavior-carrying `Enum` (see [../03-python-oop-essentials.md](../03-python-oop-essentials.md), enums-with-behavior section) is less ceremony than a full class-per-state hierarchy. Reach for full State when transition logic is nontrivial or states need to hold per-state data (like the vending machine's `DispensingState` needing a timer, or the elevator's `DoorsOpenState` needing an opened-at timestamp).

**Related principle:** [OCP](../02-solid-principles.md#o--openclosed-principle) — a new state is a new class; [LSP](../02-solid-principles.md#l--liskov-substitution-principle) — every state must honor the full `VendingMachineState` contract, even if a transition raises (raising is a *documented* part of the contract here, not a silent surprise).

**Used in:** [../problems/03-vending-machine.md](../problems/03-vending-machine.md) (the cleanest State showcase in this set), [../problems/02-elevator-system.md](../problems/02-elevator-system.md) (Idle/Moving/DoorsOpen car states).

**Watch out for:** contrast with enum-with-behavior ([../03-python-oop-essentials.md](../03-python-oop-essentials.md)) — if the states are few, fixed, and transitions are simple, a behavior-carrying `Enum` can be less ceremony than a full class-per-state hierarchy. Reach for full State when transition logic is nontrivial or states need to hold per-state data.

---

## Command

**Intent:** Encapsulate a request as an object (receiver + action + params), so requests can be queued, logged, undone, or executed by something that doesn't know the request's concrete details.

**When to reach for it in LLD:**
- Requirement language: "queue requests and process them in order," "support undo," "log every operation for replay/audit" — anywhere "the action itself" needs to be a first-class value, not just an immediate method call.

**Structure:**
```
Command (interface): execute(), undo()
  ├─ InsertCoinCommand
  └─ DispenseItemCommand

ElevatorController
  └─ has-a[*] Command (request queue) — enqueues, dequeues and executes FIFO/by-priority
```

**Python — the classic, class-based shape:**
```python
from abc import ABC, abstractmethod
from collections import deque


class Command(ABC):
    @abstractmethod
    def execute(self) -> None: ...

    @abstractmethod
    def undo(self) -> None: ...


class FloorRequestCommand(Command):
    def __init__(self, elevator: "Elevator", target_floor: int) -> None:
        self._elevator = elevator
        self._target_floor = target_floor
        self._previous_floor: int | None = None

    def execute(self) -> None:
        self._previous_floor = self._elevator.current_floor
        self._elevator.move_to(self._target_floor)

    def undo(self) -> None:
        assert self._previous_floor is not None
        self._elevator.move_to(self._previous_floor)


class ElevatorController:
    def __init__(self) -> None:
        self._pending: deque[Command] = deque()
        self._history: list[Command] = []

    def submit(self, command: Command) -> None:
        self._pending.append(command)

    def process_next(self) -> None:
        if self._pending:
            command = self._pending.popleft()
            command.execute()
            self._history.append(command)

    def undo_last(self) -> None:
        if self._history:
            self._history.pop().undo()
```

**Pythonic idiom note:** when a command doesn't need `undo` — a pure fire-and-log-it action — `functools.partial` or a closure is a lighter-weight way to make "an action" a first-class, queueable value than a full `Command` class:

```python
from collections import deque
from functools import partial
from typing import Callable

Action = Callable[[], None]


def move_elevator(elevator: "Elevator", target_floor: int) -> None:
    elevator.move_to(target_floor)


queue: deque[Action] = deque()
queue.append(partial(move_elevator, elevator, 7))
queue.popleft()()  # dequeue and invoke
```

For undo, closures can still work — a command factory returns a pair of callables (or a small `NamedTuple`/`dataclass` of two callables) instead of an object with two methods:

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class UndoableAction:
    execute: Action
    undo: Action


def floor_request(elevator: "Elevator", target_floor: int) -> UndoableAction:
    previous_floor = elevator.current_floor

    def do() -> None:
        elevator.move_to(target_floor)

    def undo() -> None:
        elevator.move_to(previous_floor)

    return UndoableAction(execute=do, undo=undo)
```

Note the closure captures `previous_floor` *at construction time*, which is subtly different from the class version capturing it inside `execute()` — decide deliberately which timing you want and say so; it's an easy off-by-one-in-time bug otherwise. Reach for the full `Command` class when a command needs to be serialized (persisted to a log/audit trail, sent over the network), inspected (`isinstance` checks, a `command.name` for display), or subclassed with shared behavior — closures don't serialize and don't have a stable identity beyond the enclosing function.

**Related principle:** [SRP](../02-solid-principles.md#s--single-responsibility-principle) — the "what to do" (Command) is separated from "when/how it gets invoked" (the invoker/queue), each with its own reason to change.

**Used in:** [../problems/02-elevator-system.md](../problems/02-elevator-system.md) (floor requests queued and dispatched as commands), [../problems/09-logging-framework.md](../problems/09-logging-framework.md) (log-write-as-command for buffering/async flush).

**Watch out for:** don't wrap a single, immediate, un-undoable, un-queueable action in a `Command` object (or a closure) "for structure" — if nothing ever queues it, logs it, or undoes it, it's just a method call wearing a costume.

---

## Chain of Responsibility

**Intent:** Give more than one object a chance to handle a request by chaining handlers; each handler decides to handle it, pass it to the next, or both, without the sender knowing which handler will actually process it.

**When to reach for it in LLD:**
- Requirement language: "a log message is only emitted if it meets the configured level, and gets routed to potentially multiple appenders," "a request passes through validation steps, any of which can reject it," "escalate through support tiers."

**Structure:**
```
LogHandler (abstract): set_next(handler), handle(record)
  ├─ DebugHandler
  ├─ InfoHandler
  └─ ErrorHandler
        each: if can_handle: process(record); always pass to next (unless terminal)
```

**Python — the classic, class-based shape:**
```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from enum import IntEnum


class Level(IntEnum):
    DEBUG = 10
    INFO = 20
    ERROR = 40


@dataclass(frozen=True)
class LogRecord:
    level: Level
    message: str


class LogHandler(ABC):
    def __init__(self) -> None:
        self._next: "LogHandler | None" = None

    def set_next(self, next_handler: "LogHandler") -> "LogHandler":
        self._next = next_handler
        return next_handler

    def handle(self, record: LogRecord) -> None:
        if self._can_handle(record):
            self._process(record)
        if self._next is not None:
            self._next.handle(record)

    @abstractmethod
    def _can_handle(self, record: LogRecord) -> bool: ...

    @abstractmethod
    def _process(self, record: LogRecord) -> None: ...


class ConsoleHandler(LogHandler):
    def _can_handle(self, record: LogRecord) -> bool:
        return record.level >= Level.INFO

    def _process(self, record: LogRecord) -> None:
        print(record)


class FileHandler(LogHandler):
    def _can_handle(self, record: LogRecord) -> bool:
        return record.level >= Level.ERROR

    def _process(self, record: LogRecord) -> None:
        with open("app.log", "a") as f:
            f.write(f"{record.message}\n")


chain = ConsoleHandler()
chain.set_next(FileHandler())
chain.handle(LogRecord(Level.ERROR, "disk full"))  # both handlers get a look, each decides independently
```

**Pythonic idiom note:** if the chain always propagates to every handler (rather than stopping at the first that fires), it's not really a *chain* at all — it's a filtered pipeline, and a list comprehension/generator expresses that more directly than a linked list of objects with `set_next`:

```python
from typing import Callable

Handler = Callable[[LogRecord], None]


def make_level_handler(min_level: Level, sink: Callable[[LogRecord], None]) -> Handler:
    def handle(record: LogRecord) -> None:
        if record.level >= min_level:
            sink(record)
    return handle


handlers: list[Handler] = [
    make_level_handler(Level.INFO, print),
    make_level_handler(Level.ERROR, lambda r: open("app.log", "a").write(f"{r.message}\n")),
]


def dispatch(record: LogRecord) -> None:
    for handle in handlers:
        handle(record)          # every handler independently decides to fire — this is the pipeline shape
```

If instead you want classic "first handler that can handle it wins, stop there" semantics, `next()` over a generator expression captures that in one line without any class:

```python
handler_for = next((h for h in handlers if h.can_handle(record)), None)
if handler_for is not None:
    handler_for.process(record)
```

Reach for the linked-list-of-objects version when handlers need per-handler configuration state beyond a closure, need to be inserted/removed from the chain dynamically at runtime, or the chain's *order* itself is a first-class, user-configurable thing (e.g., middleware ordering) — a plain `list` you can `insert()`/`remove()` into usually covers even that case with less machinery than dedicated `set_next` plumbing.

**Related principle:** [OCP](../02-solid-principles.md#o--openclosed-principle) — new handler = new class (or new function) spliced into the chain, no edits to existing handlers; [SRP](../02-solid-principles.md#s--single-responsibility-principle) — each handler owns exactly one filter/action decision.

**Used in:** [../problems/09-logging-framework.md](../problems/09-logging-framework.md) (the canonical use case — level filtering + multi-appender routing), [../problems/06-splitwise-expense-sharing.md](../problems/06-splitwise-expense-sharing.md) (a validation chain: auth check → balance check → limit check before settling).

**Watch out for:** be explicit about whether your chain **stops at the first handler that fires** (classic "pass it on only if unhandled") or **always propagates to every handler** (like the logging example above, where multiple appenders may all legitimately fire) — conflating the two is a common in-interview bug. State which one you're building before writing code.

---

## Template Method

**Intent:** Define the skeleton of an algorithm in a base class method, deferring specific steps to subclasses — the *shape* of the algorithm is fixed and closed, individual steps are open.

**When to reach for it in LLD:** requirement language like "all games follow setup → turns-until-win/draw → announce winner, but win-checking differs per game" — a shared flow, varying steps.

**Python:**
```python
from abc import ABC, abstractmethod


class BoardGame(ABC):
    def play(self) -> None:                     # template method — fixed shape
        self._initialize()
        while not self._is_game_over():
            self._take_turn()
        self._announce_result()

    @abstractmethod
    def _initialize(self) -> None: ...

    @abstractmethod
    def _is_game_over(self) -> bool: ...

    @abstractmethod
    def _take_turn(self) -> None: ...

    def _announce_result(self) -> None:          # default step, overridable
        print("Game over")


class TicTacToe(BoardGame):
    def _initialize(self) -> None:
        self._board: list[list[str | None]] = [[None] * 3 for _ in range(3)]

    def _is_game_over(self) -> bool:
        return self._check_win() or self._board_full()

    def _take_turn(self) -> None:
        ...  # current player marks a cell

    def _check_win(self) -> bool:
        return False

    def _board_full(self) -> bool:
        return all(cell is not None for row in self._board for cell in row)
```

**Pythonic idiom note:** Python has no `final` keyword to lock the skeleton method against being overridden the way Java can — nothing stops a subclass from overriding `play()` itself and bypassing the template entirely. Convention (a leading underscore on the steps, a docstring on `play()` saying "do not override") is doing the enforcement work that the compiler does in Java; make that a spoken tradeoff if asked "how do you guarantee subclasses can't skip steps?" A common alternative when the "subclass" side is thin is to invert Template Method into Strategy-of-callables: pass the four steps in as constructor arguments (functions) to one concrete `BoardGame` class instead of creating a subclass per game. That trades inheritance for composition, and is usually the better call in Python once you have more than two or three concrete games, per the general "favor composition" guidance in [../02-solid-principles.md](../02-solid-principles.md).

**Related principle:** the flip side of [OCP](../02-solid-principles.md#o--openclosed-principle) — here the *algorithm structure* is closed while individual *steps* are the open extension points.

**Used in:** [../problems/04-tic-tac-toe-and-chess.md](../problems/04-tic-tac-toe-and-chess.md) (shared turn-based game loop, per-game win/move rules).

**Watch out for:** if subclasses end up overriding almost every step with wildly different logic, there's no real shared skeleton left — that's a sign you want Strategy (compose the varying part) instead of Template Method (inherit and override it).

---

## Iterator

**Intent:** Provide sequential access to elements of a collection without exposing its underlying representation (array, tree, linked structure).

**When to reach for it in LLD:** requirement language like "iterate over search results/catalog entries" where the backing structure might change (list today, paginated/lazy source later) and callers shouldn't care.

**Python** — implement the iterator protocol (`__iter__`/`__next__`), or simply write a generator function, which is more idiomatic than a hand-rolled iterator class for the common case:

```python
from collections.abc import Iterator


class BookShelf:
    def __init__(self) -> None:
        self._books: list["Book"] = []

    def add(self, book: "Book") -> None:
        self._books.append(book)

    def __iter__(self) -> Iterator["Book"]:
        yield from self._books   # generator — idiomatic Python Iterator, no separate Iterator class needed

    def __len__(self) -> int:
        return len(self._books)


for book in book_shelf:
    print(book)
```

For a traversal that's genuinely nontrivial (e.g., lazily paginating through a remote catalog, or walking a tree without materializing it), a generator function is still usually the cleanest route — it *is* an iterator, obtained by calling the function, with `__next__` handled for you by the `yield` machinery:

```python
def paginated_catalog(fetch_page: Callable[[int], list["Book"]]) -> Iterator["Book"]:
    page = 0
    while books := fetch_page(page):
        yield from books
        page += 1
```

Only hand-write a class with explicit `__iter__`/`__next__` when you need the iterator itself to expose extra state or methods beyond "give me the next element" (e.g., a `.peek()` or a `.reset()`) — a bare generator can't be rewound or peeked into without consuming it.

**Related principle:** [ISP](../02-solid-principles.md#i--interface-segregation-principle)-adjacent — callers depend only on "give me the next element," not on the collection's full interface.

**Used in:** [../problems/08-library-management-system.md](../problems/08-library-management-system.md) (traversing catalog/search results without exposing storage).

**Watch out for:** don't hand-write a full iterator class when a generator function already covers it — that's transliterating iterator ceremony from a language that lacks generators. Reserve the class form for the peek/reset/extra-state cases above.

---

## Visitor

**Intent:** Add new operations to a stable object structure (often a [Composite](02-structural-patterns.md#composite) tree) without modifying the classes in that structure — the operation "visits" each element via double dispatch.

**When to reach for it in LLD:** you have a fixed hierarchy (rarely changes) but need to keep adding *new, unrelated operations* over it (compute total value, render, export) and don't want every new operation to mean editing every element class.

**Python — the classic double-dispatch shape** (rarely needed, shown for completeness):
```python
from abc import ABC, abstractmethod


class CategoryVisitor(ABC):
    @abstractmethod
    def visit_book(self, book: "Book") -> None: ...

    @abstractmethod
    def visit_category(self, category: "Category") -> None: ...


class CatalogEntry(ABC):
    @abstractmethod
    def accept(self, visitor: CategoryVisitor) -> None: ...


class Book(CatalogEntry):
    def accept(self, visitor: CategoryVisitor) -> None:
        visitor.visit_book(self)


class TotalCountVisitor(CategoryVisitor):
    def __init__(self) -> None:
        self.count = 0

    def visit_book(self, book: "Book") -> None:
        self.count += 1

    def visit_category(self, category: "Category") -> None:
        for entry in category.entries:
            entry.accept(self)
```

**Pythonic idiom note:** Python doesn't need the `accept`/double-dispatch dance to add a function over a type union — `functools.singledispatch` (or, for a fixed closed set of types known up front, a `match` statement) gets the same "one operation, many types" result with plain functions instead of a parallel visitor-class hierarchy:

```python
from functools import singledispatch


@singledispatch
def describe(entry: "CatalogEntry") -> str:
    raise NotImplementedError(f"no describe() for {type(entry)}")


@describe.register
def _(entry: "Book") -> str:
    return f"Book: {entry.title}"


@describe.register
def _(entry: "Category") -> str:
    return f"Category: {entry.name} ({len(entry.entries)} items)"
```

Or, since Python 3.10, structural pattern matching against `isinstance` patterns is a readable single-function alternative when the type set is small and stable — no registry, no decorators:

```python
def describe(entry: "CatalogEntry") -> str:
    match entry:
        case Book(title=title):
            return f"Book: {title}"
        case Category(name=name, entries=entries):
            return f"Category: {name} ({len(entries)} items)"
        case _:
            raise NotImplementedError(f"no describe() for {type(entry)}")
```

`singledispatch` scales better than `match` when new element types are added by *other* modules later (each registers its own handler without editing a central `match`); `match` is more readable when you own the full type set and want everything in one place. Either way, this is the option to lead with in Python — reach for the full `Visitor` class hierarchy only if you specifically need double dispatch (the operation depends on *two* runtime types, not one) or the "visitor" itself needs to accumulate rich state across a traversal that a bag of standalone functions would make awkward to thread through.

**Related principle:** trades off against [OCP](../02-solid-principles.md#o--openclosed-principle) in the opposite direction from Strategy — Visitor makes adding **operations** easy at the cost of making adding **new element types** hard (every visitor needs a new case). Say this trade-off out loud if you name it.

**Used in:** [../problems/08-library-management-system.md](../problems/08-library-management-system.md) (operations like count/export over a stable category tree).

**Watch out for:** this is the least frequently *correct* pattern in LLD interviews — the element hierarchy has to be genuinely stable and the operations genuinely growing, which is a narrower fit than almost any other pattern here. In Python especially, reach for `singledispatch` or `match` before a full Visitor class hierarchy.

---

## Mediator

**Intent:** Centralize how a set of objects communicate through one mediator object, so they reference the mediator instead of each other directly, cutting many-to-many coupling down to many-to-one.

**When to reach for it in LLD:** multiple peer objects (elevator cars, chat participants) would otherwise need direct references to each other to coordinate — introduce one coordinator they all talk to instead.

**Python:**
```python
from dataclasses import dataclass, field


@dataclass
class ElevatorDispatcher:                       # mediator
    cars: list["Elevator"] = field(default_factory=list)

    def request_floor(self, floor: int, direction: "Direction") -> None:
        best = min(self.cars, key=lambda c: abs(c.current_floor - floor))
        best.move_to(floor)                     # cars never talk to each other directly
```

**Related principle:** [SRP](../02-solid-principles.md#s--single-responsibility-principle) — coordination logic (which car answers a request) lives in one place, not smeared across every `Elevator` instance.

**Used in:** [../problems/02-elevator-system.md](../problems/02-elevator-system.md) (dispatcher coordinating multiple cars/floor requests).

**Watch out for:** don't relabel an ordinary service/controller class as "Mediator" just to name a pattern — it's worth naming specifically when the alternative would be peer objects holding direct references to each other; if there was never going to be direct peer coupling, you just have a normal coordinating service.

## Continue

Next: [../problems/00-approach-framework.md](../problems/00-approach-framework.md)
