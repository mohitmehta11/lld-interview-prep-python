# Logging Framework

## Requirements

- "What log levels do we need, and do they nest?" — Assume the standard totally-ordered set `DEBUG < INFO < WARN < ERROR`; a logger configured at `INFO` suppresses `DEBUG` but emits `INFO` and above. **You decide** if not specified.
- "How many output destinations, and can a log line go to more than one?" — Assume yes: a single log call can fan out to console + file + network appenders simultaneously, each with its own independent level filter (e.g. file appender logs `DEBUG+`, network appender only `ERROR+`).
- "Sync or async?" — Design synchronous first (simpler, correct), call out async (queue + worker thread/pool) as a follow-up so it's clear you know the failure modes (backpressure, dropped logs, ordering) rather than defaulting to it and hand-waving.
- "Is this a library other teams consume, or an app-internal component?" — Treat it as a library: the public surface is a `Logger` built via a `LoggerBuilder`, obtained through a `LoggerFactory`, not a class apps subclass. This is a systems/infra design problem, not a UI-adjacent one — frame it that way in the interview (think Python's own `logging` module, or `log4j`/`slf4j`, not a chat app's message list).
- "Do we need log rotation / structured (JSON) output?" — Out of scope for the core design; flagged as follow-ups since they're additive to `Appender` and don't change the core pipeline.

**In scope**: `Logger` with a configured minimum level, multiple pluggable `Appender`s (console/file/network stub), a level-based dispatch pipeline (Chain of Responsibility), fluent configuration (Builder), a process-wide registry (Singleton, justified), decorators for cross-cutting appender behavior (timestamp formatting, async flush).

**Out of scope**: distributed log aggregation (ELK/Splunk shipping), log rotation policy implementation detail (mentioned, not built), structured/JSON serialization format, sampling.

## Core entities & relationships

```
LoggerFactory (Singleton)
  └─ has-a[*] Logger                    (keyed by name, e.g. per-class/module)

Logger
  ├─ has-a[1] LogLevel                  (minimum level this logger accepts)
  ├─ has-a[*] Appender (interface)      (fan-out destinations)
  └─ built by[1] LoggerBuilder          (fluent construction)

Appender (interface)
  ├─ ConsoleAppender
  ├─ FileAppender
  ├─ NetworkAppender
  └─ decorated by[*] AppenderDecorator (interface, extends Appender)
        ├─ TimestampingAppender
        └─ AsyncAppender

LogLevelHandler (interface)             — Chain of Responsibility link
  DEBUG -> INFO -> WARN -> ERROR        (each decides "mine to handle, and/or pass down")
```

**Why `Logger` holds a *list* of `Appender`s rather than exactly one**: the requirement is explicit fan-out (console + file + network from one log call), and each `Appender` needs its *own* level filter independent of the `Logger`'s own level — a `Logger` set to `DEBUG` might still have a `NetworkAppender` that only forwards `ERROR+` to avoid flooding a remote sink. Two independent level checks (logger-level, then per-appender-level) is the key nuance graders look for.

**Why level dispatch is Chain of Responsibility and not a single `if level >= threshold` check**: that single check *is* a valid minimal implementation and is fine to state as the baseline — CoR earns its keep the moment you need level-specific side effects beyond filtering (e.g. `ERROR` additionally pages on-call, `WARN` additionally increments a metrics counter). Framing it as a chain from the start means adding that later is a new handler, not new branches in existing code. This file walks the chain explicitly since it's this knowledge base's canonical CoR example.

## Design patterns applied

- [Chain of Responsibility](../patterns/03-behavioral-patterns.md#chain-of-responsibility) — each `LogLevelHandler` (DEBUG→INFO→WARN→ERROR) decides whether it's responsible for a message and whether to pass it further down; a new level or level-specific side effect (e.g. ERROR also alerts) is a new link, not an edit to existing handlers. This is the textbook use case for the pattern in this knowledge base.
- [Builder](../patterns/01-creational-patterns.md#builder) — `LoggerBuilder` configures level + appenders + format, all optional/combinable knobs with sane defaults; a telescoping constructor (`Logger(level, appenders, format, is_async, ...)`) would be unreadable at 4+ knobs.
- [Singleton](../patterns/01-creational-patterns.md#singleton) — `LoggerFactory` is a genuinely justified Singleton: there is exactly one process-wide logger registry, and multiple instances would mean multiple independent configurations for "the same" named logger, which is a real bug, not just convenient sharing. Contrast with the overuse trap noted in [../patterns/01-creational-patterns.md](../patterns/01-creational-patterns.md#singleton) — this is the case where the smell ("need exactly one shared coordinator") is actually present in the requirements.
- [Decorator](../patterns/02-structural-patterns.md#decorator) — `TimestampingAppender`/`AsyncAppender` wrap a base `Appender` to add genuinely additive behavior (prefix a timestamp, offload to a background thread) without subclassing `FileAppender`, `ConsoleAppender`, and `NetworkAppender` three separate times each for "with timestamp" and "with async" — avoids the `N × M` subclass explosion.

## Python implementation

This is the primary, fully-worked solution.

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from concurrent.futures import ThreadPoolExecutor
from dataclasses import dataclass, field
from datetime import datetime, timezone
from enum import IntEnum, auto
from typing import Optional, Protocol


class LogLevel(IntEnum):
    """IntEnum, not Enum: level comparisons (`level >= self.min_level`) need ordering, and
    IntEnum gives that for free while still being a real enum (named, iterable, type-safe)
    rather than falling back to bare ints scattered through the codebase."""

    DEBUG = 10
    INFO = 20
    WARN = 30
    ERROR = 40


@dataclass(frozen=True, slots=True)
class LogRecord:
    """`frozen=True` because a record represents a fact about something that already
    happened — nothing downstream (an Appender, a decorator) should be able to mutate a
    record another Appender already processed. `slots=True` skips the per-instance `__dict__`,
    worth it here since a busy logger can allocate one LogRecord per call."""

    level: LogLevel
    message: str
    timestamp: datetime = field(default_factory=lambda: datetime.now(timezone.utc))


class Appender(Protocol):
    """typing.Protocol: an Appender is purely "has an `append(record)` method" — there's no
    shared state or default behavior any implementation benefits from inheriting, so a nominal
    ABC hierarchy would add ceremony (explicit `class ConsoleAppender(Appender)`) without buying
    anything a structural check doesn't already give. Contrast with `LogLevelHandler` below,
    which *is* an ABC because it has real shared state (`next`) and a concrete `handle()` method
    every subclass reuses.
    """

    def append(self, record: LogRecord) -> None: ...


class ConsoleAppender:
    def append(self, record: LogRecord) -> None:
        print(f"[{record.level.name}] {record.message}")


class FileAppender:
    def __init__(self, path: str) -> None:
        self.path = path

    def append(self, record: LogRecord) -> None:
        print(f"(file:{self.path}) [{record.level.name}] {record.message}")  # real impl: buffered write + rotation hook


class NetworkAppender:
    def __init__(self, endpoint: str) -> None:
        self.endpoint = endpoint

    def append(self, record: LogRecord) -> None:
        print(f"(POST {self.endpoint}) [{record.level.name}] {record.message}")


class LevelFilteringAppender:
    """Decorator giving one Appender its own minimum level, independent of the owning Logger's
    level — the nuance called out in Core entities above (e.g. console shows DEBUG+, a
    NetworkAppender only forwards ERROR+)."""

    def __init__(self, delegate: Appender, min_level: LogLevel) -> None:
        self.delegate = delegate
        self.min_level = min_level

    def append(self, record: LogRecord) -> None:
        if record.level >= self.min_level:
            self.delegate.append(record)


class TimestampingAppender:
    """Additive behavior, base appenders untouched: mutates presentation only, the delegate
    still owns the actual sink write."""

    def __init__(self, delegate: Appender) -> None:
        self.delegate = delegate

    def append(self, record: LogRecord) -> None:
        stamped = LogRecord(record.level, f"[{record.timestamp.isoformat()}] {record.message}")
        self.delegate.append(stamped)


class AsyncAppender:
    """Offloads the write to a single-worker pool — preserves order, non-blocking for the
    caller. Unbounded task submission can back up under sustained overload; see follow-ups.
    Implements the context-manager protocol so callers can guarantee the worker thread drains
    and shuts down cleanly (`with AsyncAppender(...) as appender: ...`) instead of relying on
    the pool's finalizer, which gives no delivery guarantee for in-flight records.
    """

    def __init__(self, delegate: Appender, max_workers: int = 1) -> None:
        self.delegate = delegate
        self._pool = ThreadPoolExecutor(max_workers=max_workers)

    def append(self, record: LogRecord) -> None:
        self._pool.submit(self.delegate.append, record)

    def __enter__(self) -> "AsyncAppender":
        return self

    def __exit__(self, exc_type, exc_val, exc_tb) -> None:
        self._pool.shutdown(wait=True)  # drains queued appends before returning
```

> **Pythonic idiom note:** `Appender` is a `Protocol`, so `ConsoleAppender`/`FileAppender`/`NetworkAppender` don't inherit from it at all — they just happen to have a matching `append` method. This is the natural Python shape for a role with no shared implementation: it reads closer to "any object that knows how to append a record" than a Java-style `implements Appender` declaration would, and it means a third-party logging sink class could satisfy this interface without ever knowing this codebase exists.

```python
class LogLevelHandler(ABC):
    """ABC, not Protocol: every handler shares `handles`, `next`, and the concrete `handle()`
    dispatch logic — only `do_handle` varies per level. That shared state + shared method is
    exactly the case for inheritance over structural typing."""

    def __init__(self, handles: LogLevel) -> None:
        self.handles = handles
        self.next: Optional["LogLevelHandler"] = None

    def set_next(self, nxt: "LogLevelHandler") -> "LogLevelHandler":
        self.next = nxt
        return nxt

    def handle(self, record: LogRecord, appenders: list[Appender]) -> None:
        if record.level == self.handles:
            self.do_handle(record, appenders)
        if self.next:
            self.next.handle(record, appenders)

    @abstractmethod
    def do_handle(self, record: LogRecord, appenders: list[Appender]) -> None: ...


class DebugHandler(LogLevelHandler):
    def __init__(self) -> None:
        super().__init__(LogLevel.DEBUG)

    def do_handle(self, record: LogRecord, appenders: list[Appender]) -> None:
        for appender in appenders:
            appender.append(record)


class ErrorHandler(LogLevelHandler):
    def __init__(self) -> None:
        super().__init__(LogLevel.ERROR)

    def do_handle(self, record: LogRecord, appenders: list[Appender]) -> None:
        for appender in appenders:
            appender.append(record)
        print(f"ALERT: on-call paged for ERROR: {record.message}")  # side effect lives in the handler, not in Logger.log
# INFO/WARN handlers omitted — identical shape to DebugHandler, no extra side effect (yet).


class Logger:
    def __init__(self, name: str, min_level: LogLevel, appenders: list[Appender]) -> None:
        self.name = name
        self.min_level = min_level
        self.appenders = appenders

    def log(self, level: LogLevel, message: str) -> None:
        if level < self.min_level:
            return  # logger-level filter; per-appender filtering is composed via LevelFilteringAppender
        record = LogRecord(level, message)
        for appender in self.appenders:
            try:
                appender.append(record)
            except Exception as exc:  # one failing sink must not take down the others or the caller
                print(f"appender {appender!r} failed: {exc}")

    def debug(self, msg: str) -> None: self.log(LogLevel.DEBUG, msg)
    def info(self, msg: str) -> None: self.log(LogLevel.INFO, msg)
    def warn(self, msg: str) -> None: self.log(LogLevel.WARN, msg)
    def error(self, msg: str) -> None: self.log(LogLevel.ERROR, msg)


class LoggerBuilder:
    def __init__(self) -> None:
        self._name = "root"
        self._level = LogLevel.INFO
        self._appenders: list[Appender] = []

    def named(self, name: str) -> "LoggerBuilder":
        self._name = name
        return self

    def with_min_level(self, level: LogLevel) -> "LoggerBuilder":
        self._level = level
        return self

    def add_appender(self, appender: Appender) -> "LoggerBuilder":
        self._appenders.append(appender)
        return self

    def build(self) -> Logger:
        appenders = self._appenders or [ConsoleAppender()]
        return Logger(self._name, self._level, appenders)


class LoggerFactory:
    """Justified module-level Singleton: one process-wide named-logger registry.
    Python idiom: `__new__` returning a cached instance IS the Singleton — see
    ../03-python-oop-essentials.md for why this needs no private-constructor workaround
    the way a language without module-level state or first-class `__new__` override would."""

    _instance: Optional["LoggerFactory"] = None
    _registry: dict[str, Logger]

    def __new__(cls) -> "LoggerFactory":
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._registry = {}
        return cls._instance

    def get_or_create(self, name: str, factory) -> Logger:
        if name not in self._registry:
            self._registry[name] = factory()
        return self._registry[name]
```

> **Pythonic idiom note:** `LoggerFactory` could just as easily be "a module with a `_registry` dict and a `get_or_create` function" — a bare module *is* a singleton in Python, since it's only ever imported once and cached in `sys.modules`. The class-based `__new__` version above is kept because it matches the class-based `LoggerFactory.get_instance()` shape most interviewers expect to see named explicitly as "the Singleton," but it's worth saying out loud in an interview that Python doesn't strictly need the class ceremony to get the same guarantee.

## Sample walkthrough

```python
factory = LoggerFactory()

app_logger = factory.get_or_create(
    "app",
    lambda: (
        LoggerBuilder()
        .named("app")
        .with_min_level(LogLevel.DEBUG)
        .add_appender(TimestampingAppender(ConsoleAppender()))
        .add_appender(LevelFilteringAppender(NetworkAppender("https://logs.example.com"), min_level=LogLevel.ERROR))
        .build()
    ),
)

with AsyncAppender(FileAppender("/var/log/app.log")) as async_file:
    app_logger.appenders.append(async_file)
    app_logger.debug("cache miss for key=user:42")   # console (timestamped) + file (async); network stays silent, below its ERROR floor
    app_logger.error("payment gateway timeout")        # console + file + network all fire; a CoR-wired pipeline would also page on-call
# `with` block exit drains the async file appender's queue before moving on
```

## Follow-up questions

- **"Logging is blocking request latency under load — make it async end-to-end."** `AsyncAppender` already offloads a single appender; for the whole pipeline, put a bounded `queue.Queue(maxsize=...)` between `Logger.log()` and dispatch, with a fixed-size worker pool draining it — reference [../04-concurrency-essentials.md](../04-concurrency-essentials.md) for sizing the queue and pool and for the backpressure policy trade-off (drop-oldest vs block-caller vs drop-newest).
- **"Add log rotation (size- or time-based) to `FileAppender`."** Purely internal to `FileAppender` — swap the raw file handle for a rotating-file-writer strategy (Python's stdlib `logging.handlers.RotatingFileHandler` is the reference shape); no other class in the design changes, which is the point of keeping rotation an `Appender` concern rather than a `Logger` concern.
- **"Different appenders need different minimum levels (e.g. console shows DEBUG+, network only ERROR+)."** Already shown above: `LevelFilteringAppender` wraps any `Appender` with its own threshold, composed the same way as `TimestampingAppender` — the `Decorator` choice pays for itself again here instead of adding a level field to every appender class.
- **"What if an appender throws (e.g. file handle closed, network down)?"** `Logger.log()` above already wraps each `append()` call in a per-appender `try`/`except` so one failing sink doesn't take down the others or the caller; for transient network failures specifically, add a `RetryingAppender` decorator with bounded retries + backoff (see the retry discussion in [10-notification-and-observer-system.md](10-notification-and-observer-system.md)).
- **"We now need structured (JSON) log output for a log-aggregation pipeline."** Introduce a `LogFormatter` Protocol (`format(record: LogRecord) -> str`), inject it into `Appender` implementations instead of hardcoding string concatenation — additive, doesn't touch `Logger` or the CoR chain.

## Common mistakes on this problem

- Implementing level filtering as a single `if level >= threshold:` and stopping there when the interviewer explicitly asks for level-specific behavior (paging, metrics) — that's the signal to reach for Chain of Responsibility, not more `if` branches.
- Making `Logger` itself responsible for formatting, writing to disk, *and* deciding retry/async policy — a God Object that should instead delegate formatting to a `LogFormatter` and I/O + resilience to `Appender`/its decorators.
- Treating `LoggerFactory` as just "a Singleton because loggers are global," without articulating *why* multiple instances would be wrong (split registries → the same logger name resolves to different configs in different parts of the app) — say the justification out loud, don't just name the pattern.
- Building the async path with an unbounded queue/thread-per-log-call — this is the classic way a "make it async" follow-up turns into an OOM or thread-exhaustion incident; always reason about bounded queues and a fixed worker pool.

## Continue

Next: [10-notification-and-observer-system.md](10-notification-and-observer-system.md)
