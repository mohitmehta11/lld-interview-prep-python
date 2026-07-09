# Elevator System

The state-machine problem. If you model elevator behavior as booleans (`is_moving`, `doors_open`) you will contradict yourself within 10 minutes of follow-ups ("can doors be open while moving?" — your booleans don't forbid it, but physics does). An explicit state machine forbids illegal combinations by construction.

## Requirements

- "How many elevators, how many floors?" → **You decide**: N elevators, M floors, both configured at construction; not relevant to the class design beyond sizing collections.
- "Dispatch algorithm — nearest elevator, or something smarter?" → Support nearest-idle-elevator as the default, but make it pluggable — an interviewer will ask "what if two requests come from floors on the way — should a moving elevator pick them up?" (SCAN-style) as a near-certain follow-up.
- "Can a request specify direction (hall button) vs a destination (car button)?" → **You decide**: model both — a `Request` carries a target floor and, for hall calls, a desired direction; car-panel requests only need a target floor which implies direction from current position.
- "Do doors auto-close after a timeout?" → Yes, model this as a real state (`DoorsOpen`) with a timeout transition back to `Moving`/`Idle`, not a fire-and-forget sleep.
- "Overload/weight sensors, fire alarm override, maintenance mode?" → Out of scope for the core design; call out that `OutOfService` would be one more `ElevatorState` implementation if asked.
- "Multiple elevator banks / zones (low-rise vs high-rise)?" → Out of scope; the `DispatchStrategy` seam is exactly where zone-aware routing would plug in later.

**In scope:** multi-elevator dispatch, explicit per-elevator state machine, thread-safe request intake while an elevator is mid-move, floor display panels reacting to position/door changes.

**Out of scope:** weight/capacity limits, fire/emergency override modes, multi-bank zoning, destination-dispatch UI (grouping passengers by destination before boarding).

## Core entities & relationships

```
ElevatorSystem
  ├─ has-a[*] Elevator
  ├─ has-a[1] DispatchStrategy (interface)
  └─ has-a[*] ElevatorObserver (interface)

Elevator
  ├─ has-a[1] ElevatorState (interface — Idle/Moving/DoorsOpen)
  ├─ has-a[1] Direction (enum)
  └─ has-a[*] Request (thread-safe queue)

Request
  └─ has-a[1] target floor (+ optional desired direction, for hall calls)
```

`ElevatorState` is a real class hierarchy behind an interface, not an enum-with-behavior like the parking lot's `VehicleType`. The difference: each elevator state carries genuinely different *transition logic and side effects* — `Idle` just waits for a request, `Moving` advances a floor and decides whether it has arrived, `DoorsOpen` starts a timer and decides where to go next — not just a different data lookup. That's the GoF-State litmus test: reach for it when the *behavior* of "handle a tick" or "handle a new request" differs by state, not merely a constant differs.

`Direction` (`UP`/`DOWN`/`IDLE`) stays a plain enum — it's an attribute the state machine sets, not itself a thing with behavior worth polymorphism over.

## Design patterns applied

- [State](../patterns/03-behavioral-patterns.md#state) — the core lesson of this problem: `Idle` → `Moving` → `DoorsOpen` → (`Moving` or `Idle`) as explicit classes means "doors open while moving" is structurally impossible rather than a bug you hope not to introduce with a stray boolean flip.
- [Strategy](../patterns/03-behavioral-patterns.md#strategy) — `DispatchStrategy` isolates "which elevator answers this call" from the elevators themselves; swapping nearest-idle for a SCAN-style algorithm that also considers already-moving elevators touches only `ElevatorSystem`'s constructor argument.
- [Observer](../patterns/03-behavioral-patterns.md#observer) — floor display panels subscribe to `ElevatorObserver` and react to position/door-state changes pushed by the elevator, instead of polling every elevator every tick.

## Python implementation

This is the full, primary implementation, organized here as one file for readability; the comment banners mark where you'd split it into separate modules (`requests.py`, `states.py`, `elevator.py`, `dispatch.py`, `system.py`) in a real codebase.

```python
# ── requests.py: value types ────────────────────────────────────────────────
from __future__ import annotations

import heapq
import time
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from enum import Enum, auto
from threading import Lock, Timer
from typing import Protocol


class Direction(Enum):
    UP = auto()
    DOWN = auto()
    IDLE = auto()


@dataclass(frozen=True, slots=True)
class Request:
    target_floor: int
    desired_direction: Direction | None = None  # None for car-panel requests


# ── observers.py: display panels react to elevator events ──────────────────

class ElevatorObserver(ABC):
    """Kept as a small ABC rather than a bag of callables: the two callbacks
    are related facets of "one subscriber" (a physical display panel), which
    is exactly the case in ../patterns/03-behavioral-patterns.md#observer
    where a class earns its keep over a plain `Callable` list."""

    @abstractmethod
    def on_position_changed(self, elevator_id: str, floor: int, direction: Direction) -> None: ...

    @abstractmethod
    def on_doors_state_changed(self, elevator_id: str, is_open: bool) -> None: ...


class FloorDisplayPanel(ElevatorObserver):
    def on_position_changed(self, elevator_id: str, floor: int, direction: Direction) -> None:
        print(f"Elevator {elevator_id} at floor {floor}, heading {direction.name}")

    def on_doors_state_changed(self, elevator_id: str, is_open: bool) -> None:
        print(f"Elevator {elevator_id} doors {'OPEN' if is_open else 'CLOSED'}")


# ── states.py: the State pattern payoff ─────────────────────────────────────

class ElevatorState(ABC):
    @abstractmethod
    def add_request(self, elevator: "Elevator", request: Request) -> None: ...

    @abstractmethod
    def step(self, elevator: "Elevator") -> None: ...


class IdleState(ElevatorState):
    def add_request(self, elevator: "Elevator", request: Request) -> None:
        elevator.enqueue(request)
        elevator.state = MovingState()

    def step(self, elevator: "Elevator") -> None:
        pass  # nothing queued, nothing to do


class MovingState(ElevatorState):
    def add_request(self, elevator: "Elevator", request: Request) -> None:
        elevator.enqueue(request)

    def step(self, elevator: "Elevator") -> None:
        target = elevator.peek_next_floor()
        if target is None:
            elevator.state = IdleState()
            elevator.direction = Direction.IDLE
            return
        if target > elevator.current_floor:
            elevator.move_to(elevator.current_floor + 1)
            elevator.direction = Direction.UP
        elif target < elevator.current_floor:
            elevator.move_to(elevator.current_floor - 1)
            elevator.direction = Direction.DOWN
        if target == elevator.current_floor:
            elevator.pop_next_floor()
            elevator.state = DoorsOpenState()


class DoorsOpenState(ElevatorState):
    DOOR_OPEN_SECONDS = 3.0

    def __init__(self) -> None:
        self._opened_at = time.monotonic()

    def add_request(self, elevator: "Elevator", request: Request) -> None:
        elevator.enqueue(request)  # accepted, acted on after close

    def step(self, elevator: "Elevator") -> None:
        if time.monotonic() - self._opened_at < self.DOOR_OPEN_SECONDS:
            return  # still within dwell time
        elevator.notify_doors(is_open=False)
        elevator.state = MovingState() if elevator.has_pending_requests() else IdleState()


class OutOfServiceState(ElevatorState):
    """Named in the follow-up questions below: one more class, no existing
    transition logic touched — this is exactly why State beats booleans."""

    def add_request(self, elevator: "Elevator", request: Request) -> None:
        raise RuntimeError(f"Elevator {elevator.id} is out of service")

    def step(self, elevator: "Elevator") -> None:
        pass


# ── elevator.py: the car itself ─────────────────────────────────────────────

class Elevator:
    def __init__(self, elevator_id: str, start_floor: int, observers: list[ElevatorObserver]) -> None:
        self.id = elevator_id
        self.current_floor = start_floor
        self.direction = Direction.IDLE
        self.state: ElevatorState = IdleState()
        self._destinations: list[int] = []  # min-heap; real SCAN keeps separate up/down heaps
        self._observers = observers
        # Guards `state` and `_destinations` together — a request-submission
        # thread and the tick thread both touch both fields, and they must
        # transition atomically relative to each other. See the concurrency
        # follow-up below for the queue.Queue-based alternative.
        self._lock = Lock()

    def add_request(self, request: Request) -> None:
        with self._lock:
            self.state.add_request(self, request)

    def step(self) -> None:
        with self._lock:
            self.state.step(self)

    def enqueue(self, request: Request) -> None:
        heapq.heappush(self._destinations, request.target_floor)

    def peek_next_floor(self) -> int | None:
        return self._destinations[0] if self._destinations else None

    def pop_next_floor(self) -> None:
        heapq.heappop(self._destinations)

    def has_pending_requests(self) -> bool:
        return bool(self._destinations)

    def move_to(self, floor: int) -> None:
        self.current_floor = floor
        for observer in self._observers:
            observer.on_position_changed(self.id, floor, self.direction)

    def notify_doors(self, is_open: bool) -> None:
        for observer in self._observers:
            observer.on_doors_state_changed(self.id, is_open)

    def distance_to(self, floor: int) -> int:
        return abs(self.current_floor - floor)

    def __repr__(self) -> str:
        return f"Elevator({self.id!r}, floor={self.current_floor}, {type(self.state).__name__})"


# ── dispatch.py: which car answers a call ───────────────────────────────────

class DispatchStrategy(Protocol):
    """`Protocol` rather than `ABC` here: dispatch strategies are pure
    functions of `(elevators, request) -> Elevator` with no shared state or
    default behavior worth inheriting — a good fit for structural typing,
    per the Strategy discussion in ../patterns/03-behavioral-patterns.md#strategy.
    A plain function assigned to `ElevatorSystem.dispatch_strategy` would work
    identically; the class form is shown because `ScanAwareDispatch` needs to
    fall back to `NearestElevatorDispatch`'s logic, which reads more clearly
    as composed objects than as nested closures."""

    def select(self, elevators: list[Elevator], request: Request) -> Elevator: ...


class NearestElevatorDispatch:
    def select(self, elevators: list[Elevator], request: Request) -> Elevator:
        return min(elevators, key=lambda e: e.distance_to(request.target_floor))


class ScanAwareDispatch:
    """Prefers an elevator already heading the requested direction (or idle)
    over the raw-nearest choice."""

    def __init__(self) -> None:
        self._fallback = NearestElevatorDispatch()

    def select(self, elevators: list[Elevator], request: Request) -> Elevator:
        candidates = [
            e for e in elevators
            if e.direction in (request.desired_direction, Direction.IDLE)
        ]
        pool = candidates or elevators
        if not candidates:
            return self._fallback.select(elevators, request)
        return min(pool, key=lambda e: e.distance_to(request.target_floor))


# ── system.py: coordinates elevators, dispatch, and the tick loop ──────────

class ElevatorSystem:
    """Doubles as a context manager so the background tick thread's
    start/stop is acquire/release-shaped instead of leaking a running Timer
    if the caller forgets to stop it — `with ElevatorSystem(...) as system:`
    guarantees the ticking stops on exit, including on exception."""

    def __init__(
        self,
        elevators: list[Elevator],
        dispatch_strategy: DispatchStrategy,
        tick_seconds: float = 0.5,
    ) -> None:
        self.elevators = elevators
        self.dispatch_strategy = dispatch_strategy
        self._tick_seconds = tick_seconds
        self._timer: Timer | None = None
        self._running = False

    def submit_request(self, floor: int, desired_direction: Direction) -> None:  # hall call
        request = Request(floor, desired_direction)
        chosen = self.dispatch_strategy.select(self.elevators, request)
        chosen.add_request(request)

    def start(self) -> "ElevatorSystem":
        self._running = True
        self._schedule_tick()
        return self

    def stop(self) -> None:
        self._running = False
        if self._timer is not None:
            self._timer.cancel()

    def _schedule_tick(self) -> None:
        if not self._running:
            return
        for elevator in self.elevators:
            elevator.step()
        self._timer = Timer(self._tick_seconds, self._schedule_tick)
        self._timer.daemon = True
        self._timer.start()

    def __enter__(self) -> "ElevatorSystem":
        return self.start()

    def __exit__(self, exc_type: object, exc: object, tb: object) -> None:
        self.stop()
```

## Sample walkthrough

```python
panel = FloorDisplayPanel()
e1 = Elevator("E1", start_floor=0, observers=[panel])
e2 = Elevator("E2", start_floor=5, observers=[panel])

with ElevatorSystem([e1, e2], NearestElevatorDispatch(), tick_seconds=1000) as system:
    system.submit_request(floor=3, desired_direction=Direction.UP)
    # distance(e1, 3) == 3, distance(e2, 3) == 2 -> E2 is dispatched
    for _ in range(10):
        e1.step()
        e2.step()
# `system.stop()` runs automatically on exit, even if the loop above raised.
```

## Follow-up questions

- **"Multiple floors send requests while an elevator is mid-move — does a request get dropped?"** No — `add_request` and `step` both acquire the same per-elevator `threading.Lock` before touching `state` and the destination heap, so concurrent calls serialize rather than race. For a real deployment, swap the ad-hoc lock for a `queue.Queue` of incoming requests per elevator and a single consumer thread draining it — see [../04-concurrency-essentials.md](../04-concurrency-essentials.md) for the `queue.Queue`-based producer/consumer pattern applied to exactly this case.
- **"What if dispatch should consider passenger load / not send an already-full elevator?"** Add a `capacity`/`current_load` field to `Elevator` and have `DispatchStrategy.select` filter on it — no change to `ElevatorState` or the state machine, since load is a dispatch-time concern, not a per-tick behavior concern.
- **"How do you avoid starving a floor that a SCAN-style algorithm keeps skipping?"** Track request age (timestamp each `Request` at creation via a `field(default_factory=time.monotonic)`) and add a "boost" rule to `ScanAwareDispatch` (or a new `FairnessAwareDispatch`) that forces pickup once wait time exceeds a threshold — swap the strategy, `Elevator`/`ElevatorSystem` untouched.
- **"Add an emergency/out-of-service mode."** `OutOfServiceState`, shown above, rejects `add_request` and ignores `step`; transitioning an elevator into it is just `elevator.state = OutOfServiceState()` from whatever admin path triggers it — this is exactly why State beats booleans: one more class, no existing transition logic touched.
- **"Undo/replay of elevator moves for debugging?"** Wrap each `Request` submission as a [Command](../patterns/03-behavioral-patterns.md#command) object (or an `UndoableAction` pair of closures, per the Pythonic idiom note in that section) with an `execute()`/log entry; the state machine doesn't need to know moves are being recorded — see the chess undo discussion in [problems/04](04-tic-tac-toe-and-chess.md) for the same pattern applied to move history.

## Common mistakes on this problem

- Modeling elevator status as `is_moving: bool` + `doors_open: bool` + `direction: int` (`0`/`1`/`-1`) instead of an explicit `ElevatorState` hierarchy — this permits illegal states (doors open *and* moving) and every new mode (out-of-service, emergency) means auditing every boolean check across the codebase.
- Coupling dispatch logic directly into `Elevator` (`elevator.should_i_answer(request)`) instead of extracting `DispatchStrategy` — makes it impossible to swap algorithms without touching every elevator instance.
- Using an unbounded, unsynchronized `list[Request]` as the request queue and reading/writing it from both the request-submission thread and the tick thread — a textbook lost-update/race-condition bug that interviewers specifically probe for on this problem.
- Treating `Direction` as just UI decoration instead of using it to drive dispatch decisions (an elevator moving up should preferentially pick up other up-calls on its way) — misses the SCAN-algorithm follow-up entirely.

## Continue

Next: [03-vending-machine.md](03-vending-machine.md)
