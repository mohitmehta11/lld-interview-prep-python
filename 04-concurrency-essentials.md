# Concurrency Essentials for LLD (Python)

LLD interviews rarely *require* a fully thread-safe design up front, but the follow-up
almost always arrives: "what if two requests hit this at the same time?" This file is the
Python concurrency toolbox for that follow-up; the application shows up concretely in
[problems/01-parking-lot.md](problems/01-parking-lot.md),
[problems/02-elevator-system.md](problems/02-elevator-system.md), and especially
[problems/07-movie-ticket-booking.md](problems/07-movie-ticket-booking.md) (the
seat-locking showcase).

## The two questions to answer before reaching for any primitive

1. **What is the shared mutable state?** (a `dict[SpotId, Spot]`, a seat's status field, a
   counter)
2. **What is the smallest critical section that protects it?** Lock the smallest region
   that preserves correctness — locking too broadly kills throughput, locking too
   narrowly reintroduces races.

## The GIL — say this out loud when asked about Python concurrency

CPython's Global Interpreter Lock means only one thread executes Python bytecode at a
time, even across multiple OS threads. This one fact drives almost every concurrency
choice below:

- **I/O-bound work** (waiting on a lock, a network call, a disk read, a DB query) —
  `threading` is still the right tool. Threads release the GIL while blocked on I/O, so
  you get real concurrency for the "waiting" parts even though only one thread runs Python
  bytecode at any instant.
- **CPU-bound work** (image processing, heavy numeric computation, parsing large payloads)
  — threads *don't* help; every thread still contends for the same GIL, so N threads
  doing CPU work run barely faster, sometimes slower, than one. Reach for `multiprocessing`
  instead, which sidesteps the GIL entirely by using separate processes, each with its own
  interpreter.
- LLD interview problems are almost always I/O/coordination-bound (locking a seat,
  reserving a spot, waiting for an elevator) rather than number-crunching, so
  `threading` + `Lock` (or `asyncio`, see below) is the right default answer to give —
  reach for `multiprocessing` only if the interviewer explicitly steers toward a CPU-bound
  variant (e.g. "now recompute pricing for a million bookings").
- You still need explicit locks even under the GIL, because the GIL only guarantees
  atomicity of individual bytecode instructions, not multi-step operations like "check
  status, then set status" — that's still a race, GIL or not:

```python
# NOT safe, even under the GIL — two bytecode-level steps, a thread switch can land between them
if seat.status == SeatStatus.AVAILABLE:   # step 1: read
    seat.status = SeatStatus.LOCKED       # step 2: write — another thread may have run between 1 and 2
```

## `threading` — the primary tool for I/O-bound LLD concurrency

### `threading.Lock` — the basic mutual-exclusion primitive

```python
import threading

class Seat:
    def __init__(self) -> None:
        self._lock = threading.Lock()
        self.status = SeatStatus.AVAILABLE

    def try_reserve(self) -> bool:
        if not self._lock.acquire(blocking=False):   # non-blocking attempt, no other thread waits on us
            return False
        try:
            if self.status != SeatStatus.AVAILABLE:
                return False
            self.status = SeatStatus.LOCKED
            return True
        finally:
            self._lock.release()

    def reserve_blocking(self) -> None:
        with self._lock:                              # idiomatic form when tryLock semantics aren't needed
            self.status = SeatStatus.LOCKED
```

One lock **per `Seat`**, not one lock for the whole theater — fine-grained locking is what
makes the seat-locking flow in
[problems/07-movie-ticket-booking.md](problems/07-movie-ticket-booking.md) scale; a single
global lock around the whole seating chart would serialize every booking in the system.

### `threading.RLock` — reentrant lock

A plain `Lock` deadlocks if the *same thread* tries to acquire it twice (e.g. a locked
method calls another method that acquires the same lock). `RLock` tracks the owning
thread and an acquisition count, so the owner can re-acquire it without blocking on
itself:

```python
class Account:
    def __init__(self, balance: float) -> None:
        self._lock = threading.RLock()
        self._balance = balance

    def withdraw(self, amount: float) -> None:
        with self._lock:
            self._validate(amount)     # also acquires self._lock — fine, same thread, RLock
            self._balance -= amount

    def _validate(self, amount: float) -> None:
        with self._lock:
            if amount > self._balance:
                raise ValueError("Insufficient funds")
```

### `threading.Condition` — wait/notify for producer-consumer style coordination

Use when a thread needs to block until some *condition* becomes true (not just until a
lock is free) — e.g. an elevator car waiting for a request, or a bounded buffer:

```python
class RequestBoard:
    def __init__(self) -> None:
        self._condition = threading.Condition()
        self._pending: list["ElevatorRequest"] = []

    def add_request(self, request: "ElevatorRequest") -> None:
        with self._condition:
            self._pending.append(request)
            self._condition.notify()          # wake one waiting consumer

    def next_request(self) -> "ElevatorRequest":
        with self._condition:
            while not self._pending:
                self._condition.wait()        # releases the lock while waiting, reacquires on wakeup
            return self._pending.pop(0)
```

`queue.Queue` (below) is built on exactly this pattern and is almost always preferable to
hand-rolling a `Condition` for simple producer-consumer cases — reach for `Condition`
directly only when the wake-up logic is more nuanced than "an item became available."

### `threading.Event` — a one-shot or resettable signal flag

Simpler than `Condition` when you just need "has X happened yet," with no associated data:

```python
class Elevator:
    def __init__(self) -> None:
        self._maintenance_mode = threading.Event()

    def enter_maintenance(self) -> None:
        self._maintenance_mode.set()

    def resume_service(self) -> None:
        self._maintenance_mode.clear()

    def wait_until_in_service(self, timeout: float | None = None) -> bool:
        return not self._maintenance_mode.wait(timeout)   # blocks until cleared, or timeout
```

### Thread-safety of common structures — know what's safe by default and what isn't

- Plain `dict` and `list` mutations (`d[k] = v`, `lst.append(x)`) are individually atomic
  in CPython because of the GIL — but a *sequence* of operations on them (check-then-set,
  read-modify-write) is not, and needs an explicit lock around the whole sequence.
- `collections.deque` has atomic `append`/`popleft` from either end, making it a
  reasonable lock-free choice for simple producer/consumer queues between exactly one
  producer and one consumer thread — but prefer `queue.Queue` once there are multiple
  producers/consumers or you need blocking semantics.
- Never assume a `dict` or `list` is safe for *compound* operations just because
  individual bytecode ops are atomic — `if key not in d: d[key] = []` is a classic
  two-step race.

### `queue.Queue` — thread-safe producer/consumer, e.g. elevator request queue

```python
import queue

requests: "queue.Queue[ElevatorRequest]" = queue.Queue()
requests.put(ElevatorRequest(floor=5, direction=Direction.UP))   # thread-safe; blocks if bounded & full
next_request = requests.get()                                     # blocks until an item is available
```

This is the natural backbone for the elevator dispatch loop in
[problems/02-elevator-system.md](problems/02-elevator-system.md): a dispatcher thread
`put()`s incoming floor requests, worker/elevator-car threads `get()` them off the queue.

### Timed auto-release — the movie-booking follow-up

A very common LLD follow-up: "what if a user locks a seat and never completes payment?"

```python
import threading

def release_seat_if_still_locked(seat: "Seat") -> None:
    with seat._lock:
        if seat.status == SeatStatus.LOCKED:
            seat.status = SeatStatus.AVAILABLE

timer = threading.Timer(600, release_seat_if_still_locked, args=[seat])  # fires once, after 10 minutes
timer.start()
# timer.cancel() if payment completes before it fires
```

For anything more elaborate than a single delayed callback (recurring cleanup, many
scheduled releases), prefer a scheduled task in a `ThreadPoolExecutor` or a dedicated
background thread over stacking many `Timer` objects.

## `multiprocessing` — when threading isn't enough

Reach for `multiprocessing` specifically when the workload is **CPU-bound** and you need
real parallelism across cores — the GIL makes `threading` ineffective for this case
because every thread still serializes on bytecode execution.

```python
from multiprocessing import Pool

def compute_dynamic_price(booking: "Booking") -> float:
    ...   # CPU-heavy pricing model

with Pool(processes=4) as pool:
    prices = pool.map(compute_dynamic_price, bookings)   # runs across 4 separate processes/interpreters
```

Key differences from threading to state explicitly if asked:
- **No shared memory by default** — each process has its own interpreter and memory
  space. Sharing state requires `multiprocessing.Queue`, `multiprocessing.Manager`, or
  `multiprocessing.shared_memory`, not a plain object reference.
- **Higher overhead** — process creation and inter-process communication (which pickles
  data across the process boundary) cost far more than a thread context switch; only
  worth it when the per-task CPU work dwarfs that overhead.
- Locks exist here too (`multiprocessing.Lock`), for the same reasons as `threading.Lock`,
  but protecting state shared via a `Manager` or `shared_memory` block rather than a plain
  in-process object.

For LLD interviews specifically: almost none of the standard problems in this repo
(parking lot, elevator, movie booking, LRU cache, rate limiter) are CPU-bound enough to
justify `multiprocessing` — mention it to show you know the tool exists and *why* it
wouldn't be your first choice here, rather than reaching for it by default.

## `concurrent.futures` — a uniform pool interface for both threads and processes

`concurrent.futures.ThreadPoolExecutor` and `ProcessPoolExecutor` share one API, so
swapping between thread- and process-based parallelism is a one-line change once code is
written against the interface:

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor, as_completed

with ThreadPoolExecutor(max_workers=4) as pool:
    future = pool.submit(process_booking, seat)
    result = future.result()             # blocks for this task's result; propagates exceptions raised inside

with ThreadPoolExecutor(max_workers=4) as pool:
    futures = [pool.submit(process_booking, seat) for seat in seats]
    for future in as_completed(futures):
        handle(future.result())

# identical call shape, but CPU-bound work now runs in separate processes:
with ProcessPoolExecutor(max_workers=4) as pool:
    prices = list(pool.map(compute_dynamic_price, bookings))
```

Default to `ThreadPoolExecutor` for I/O-bound LLD work (booking processing, notification
dispatch); swap in `ProcessPoolExecutor` only when the task body is genuinely CPU-bound.
`future.result()` re-raises any exception the task raised, which composes naturally with
the custom exception hierarchies covered in
[03-python-oop-essentials.md §14](03-python-oop-essentials.md#14-exceptions-no-checked-exceptions-and-the-idioms-that-replace-them).

## `asyncio` — cooperative concurrency for I/O-bound, high-fan-out workloads

`asyncio` runs many coroutines on a **single thread**, cooperatively switching between
them at each `await` point — no GIL contention between coroutines (there's only one
thread), and far lower per-task overhead than a thread per connection, which is why it's
the standard choice for things like "handle 10,000 concurrent booking-status
websocket connections."

```python
import asyncio

async def fetch_seat_status(seat_id: str) -> "SeatStatus":
    async with aiohttp_session.get(f"/seats/{seat_id}") as response:   # await yields control while waiting on I/O
        data = await response.json()
    return SeatStatus(data["status"])

async def fetch_all_statuses(seat_ids: list[str]) -> list["SeatStatus"]:
    return await asyncio.gather(*(fetch_seat_status(sid) for sid in seat_ids))

asyncio.run(fetch_all_statuses(["A1", "A2", "A3"]))
```

`asyncio` vs `threading` — the question to answer out loud when it comes up:

| | `threading` | `asyncio` |
|---|---|---|
| Concurrency model | preemptive, OS-scheduled threads | cooperative, single-threaded event loop |
| Best for | moderate numbers of blocking I/O calls (locks, blocking DB/network libraries) | very high fan-out I/O (thousands of concurrent connections/awaits) |
| Gotcha | classic races on shared mutable state, needs explicit locks | a single blocking (non-`await`) call anywhere stalls the *entire* event loop — every coroutine, not just one |
| Library requirement | works with any blocking library as-is | needs `async`-native libraries (`aiohttp`, `asyncpg`, ...) to get the benefit — wrapping a blocking call still blocks the loop unless off-loaded via `run_in_executor` |

For most LLD problems in this repo, the shared-state coordination story (locks around a
`Seat`, a `ParkingSpot`, a counter) is what interviewers are actually probing, and
`threading` + `Lock`/`Condition` is the more direct way to demonstrate that. Bring up
`asyncio` when the problem is framed as a *service* fielding many concurrent requests
(e.g. "design this as a web service handling booking requests") rather than as an
in-process object model — that framing is the cue that cooperative, high-fan-out
concurrency is the axis being tested, not mutual exclusion.

## Deadlock — the follow-up trap

If a single operation needs to lock **two** resources (a transfer between two accounts, a
swap of two parking spots, a seat-swap between two bookings), always acquire locks in a
**consistent global order** (e.g. by ID) to avoid the classic "A locks 1 and waits for 2,
B locks 2 and waits for 1" deadlock:

```python
def transfer(account_a: "Account", account_b: "Account", amount: float) -> None:
    first, second = sorted((account_a, account_b), key=lambda acc: acc.id)  # consistent order by id
    with first._lock:
        with second._lock:
            account_a.withdraw(amount)
            account_b.deposit(amount)
```

Mention this unprompted if a problem involves any two-resource operation (Splitwise
settlement, seat swap) — it's a strong senior signal.

## What to say if you're out of time to fully implement thread-safety

It's fine — and expected — to design single-threaded first and narrate the extension:
"I'll design the happy path assuming single-threaded access, and call out the two places
I'd add locking: [X] and [Y], using [primitive], because [shared state]." This demonstrates
the awareness (scored) without burning your entire time budget implementing it (see the
time-budget table in [00-evaluation-framework.md](00-evaluation-framework.md)).

## Continue

Next: [problems/00-approach-framework.md](problems/00-approach-framework.md) to start
practicing, or [05-common-mistakes.md](05-common-mistakes.md) if you're doing a final pass.
