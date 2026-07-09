# Notification and Observer System

## Requirements

- "What's a concrete example of a subject?" — Pick one to ground the design and state it: an `Order` whose status changes (`PLACED → SHIPPED → DELIVERED`); subscribers are users who want updates on that specific order. **You decide** if the interviewer wants something more abstract — the mechanism is identical for a stock-price ticker or a topic/channel subscription.
- "Push or pull model?" — Assume push: the subject sends the new state directly in the notification payload. **You decide** to mention pull (subject only sends "something changed," observer calls back to fetch details) as the alternative — useful when the payload is expensive to compute and not every observer needs it.
- "One channel or many, and does the user pick?" — Assume a user can subscribe with a preferred channel (email/SMS/push) per subscription; the *channel send mechanism* is a Strategy behind a common interface, decoupled from the *subscription/notify* mechanism, which is Observer. This is the crux of the problem: two different patterns solving two different variabilities.
- "Sync or async dispatch?" — Design synchronous first for correctness and clarity; call out async dispatch (thread pool fan-out) as the answer to "what if there are 10,000 observers," referencing [../04-concurrency-essentials.md](../04-concurrency-essentials.md).
- "Is this the same thing as an event bus / message queue?" — No, and this distinction is worth stating explicitly to the interviewer: **Observer** = the subject holds direct references to its observers and calls them synchronously/in-process; **event bus / pub-sub broker** = a decoupled intermediary (topic, message queue) sits between publishers and subscribers, neither knows about the other, and delivery is typically async/out-of-process. This file implements Observer; an event bus would replace the `Subject`'s direct observer list with "publish to broker" + "broker fans out to registered subscribers," and is the natural next step if you're asked to decouple further.

**In scope**: `Subject`/`Observer` interfaces (own implementation), a concrete `Order` subject, per-subscription channel preference via `NotificationChannel` strategy (Email/SMS/Push), subscribe/unsubscribe including the memory-leak discussion, sync dispatch with a note on async.

**Out of scope**: message persistence/durability, exactly-once delivery guarantees, an actual broker/queue implementation (mentioned as the follow-up direction, not built).

## Core entities & relationships

```
Subject (interface)
  └─ has-a[*] Observer (interface)      — subject holds DIRECT references; this is what distinguishes Observer from an event bus

Order (implements Subject)
  └─ has-a[1] OrderStatus enum

Subscription (implements Observer)
  ├─ has-a[1] User
  └─ has-a[1] NotificationChannel (interface)   — Strategy: how this observer wants to be reached
        ├─ EmailChannel
        ├─ SmsChannel
        └─ PushChannel
              └─ decorated by[*] ChannelDecorator (interface, extends NotificationChannel)
                    └─ RetryingChannel / ThrottlingChannel  — resilience/batching, optional
```

**Why `Subscription` (a User + a channel choice) is the `Observer`, not `User` directly**: a single user may want order-status updates via push but security-alert updates via email — the observer identity that a `Subject` notifies needs to carry *which channel this particular subscription uses*, which a bare `User` reference can't express without the subject reaching into user preferences itself (a DIP violation — the subject shouldn't need to know how preferences are stored). Wrapping `User` + `NotificationChannel` in a `Subscription` keeps `Order` fully decoupled from that policy.

**Why channel selection is Strategy, layered *underneath* Observer, not folded into it**: "how do I notify one observer" (Strategy: pick email vs SMS vs push) and "who gets notified and when" (Observer: subject's list, triggered on state change) are two independent axes of change. A new channel (WhatsApp) shouldn't touch `Order` or the notify loop; a new subject (add a `StockTicker`) shouldn't touch channel logic. Keeping them as two separate interfaces is what makes both axes independently extensible.

## Design patterns applied

- [Observer](../patterns/03-behavioral-patterns.md#observer) — the centerpiece of this file: `Order` (subject) maintains a growable, runtime-mutable collection of `Subscription` (observer) instances and pushes state changes to all of them without knowing what they are or how many there are. There's no stdlib base class to extend for this in Python (unlike some ecosystems' deprecated built-in Observer types) — rolling a small generic `Subject`/`Observer` pair, as done below, is the normal and expected approach; libraries like `blinker` exist for larger applications but a hand-rolled pair is exactly what this problem is testing.
- [Strategy](../patterns/03-behavioral-patterns.md#strategy) — `NotificationChannel` (Email/SMS/Push) varies *how* a single notification is delivered, completely orthogonal to *who* gets notified; new channels are new classes, zero changes to `Order` or the subscription list.
- [Decorator](../patterns/02-structural-patterns.md#decorator) *(optional)* — `RetryingChannel`/`ThrottlingChannel` wrap a base channel to add resilience or batch/rate-limit sends (e.g. collapse 10 stock-price ticks into 1 email per minute) without the base `EmailChannel` needing to know about retries or batching — additive behavior, not a new channel type.

## Python implementation

This is the primary, fully-worked solution.

```python
from __future__ import annotations

import time
import weakref
from abc import ABC, abstractmethod
from concurrent.futures import ThreadPoolExecutor
from dataclasses import dataclass
from enum import Enum, auto
from typing import Generic, Protocol, TypeVar

T = TypeVar("T")


class OrderStatus(Enum):
    PLACED = auto()
    SHIPPED = auto()
    DELIVERED = auto()
    CANCELLED = auto()


class Observer(ABC, Generic[T]):
    """ABC, not Protocol, even though it has only one method: Subject holds these in a
    collection and treats them uniformly by *type* (`Observer[OrderStatusChanged]`), and we
    want `isinstance`/registration-style enforcement for anything meant to plug into this
    specific notification pipeline. Compare with `NotificationChannel` below, which — despite
    looking structurally identical (one method) — is a better Protocol fit because channels
    are more of an interchangeable "any callable-shaped sender," including ones this codebase
    doesn't define.
    """

    @abstractmethod
    def on_update(self, event: T) -> None: ...


class Subject(ABC, Generic[T]):
    @abstractmethod
    def subscribe(self, observer: Observer[T]) -> None: ...
    @abstractmethod
    def unsubscribe(self, observer: Observer[T]) -> None: ...
    @abstractmethod
    def notify_observers(self, event: T) -> None: ...


@dataclass(frozen=True, slots=True)
class OrderStatusChanged:
    """`slots=True` on top of `frozen=True`: at the scale this problem's follow-ups probe
    (thousands of observers, one event object per notification), skipping the per-instance
    `__dict__` is a real memory win, and frozen immutability means the same event object can
    safely be handed to every observer without any of them being able to corrupt it for the
    others."""

    order_id: str
    new_status: OrderStatus


class Order(Subject[OrderStatusChanged]):
    def __init__(self, order_id: str) -> None:
        self.order_id = order_id
        self.status = OrderStatus.PLACED
        # WeakSet avoids the classic Observer leak: an observer that's otherwise garbage
        # can be collected without an explicit unsubscribe(); explicit unsubscribe is still
        # the correct primary mechanism — this is a safety net, not a substitute.
        self._observers: "weakref.WeakSet[Observer[OrderStatusChanged]]" = weakref.WeakSet()

    def subscribe(self, observer: Observer[OrderStatusChanged]) -> None:
        self._observers.add(observer)

    def unsubscribe(self, observer: Observer[OrderStatusChanged]) -> None:
        self._observers.discard(observer)

    def notify_observers(self, event: OrderStatusChanged) -> None:
        for observer in list(self._observers):  # snapshot: safe against mutation during iteration
            observer.on_update(event)

    def update_status(self, new_status: OrderStatus) -> None:
        self.status = new_status
        self.notify_observers(OrderStatusChanged(self.order_id, new_status))
```

> **Pythonic idiom note:** `self._observers = weakref.WeakSet()` must be assigned inside `__init__`, not as a class-level attribute (`_observers = weakref.WeakSet()` written directly under `class Order:`). A class attribute would be a single collection shared by *every* `Order` instance — every order would silently notify every other order's subscribers. This is a general Python trap, not specific to weak references: any mutable default (a list, dict, or set) belongs on `self` inside `__init__`, never as a bare class-body attribute, unless you deliberately want it shared. See the Common mistakes section below — this is common enough to be the first bullet.

```python
class NotificationChannel(Protocol):
    """Protocol: any object with a matching `send(recipient, message)` method works, including
    a real email/SMS provider's SDK client that this codebase doesn't own and can't make
    inherit from an ABC. Strategy implementations are the textbook Protocol case — see the
    contrast with `Observer` above."""

    def send(self, recipient: str, message: str) -> None: ...


class EmailChannel:
    def send(self, recipient: str, message: str) -> None:
        print(f"EMAIL to {recipient}: {message}")


class SmsChannel:
    def send(self, recipient: str, message: str) -> None:
        print(f"SMS to {recipient}: {message}")


class PushChannel:
    def send(self, recipient: str, message: str) -> None:
        print(f"PUSH to {recipient}: {message}")


class RetryingChannel:
    """Decorator: bounded retries with linear backoff around any NotificationChannel, so a
    transient SMS-gateway blip doesn't have to be handled inside every concrete channel class.
    Answers the first Follow-up question below with actual code rather than just narrative."""

    def __init__(self, delegate: NotificationChannel, max_attempts: int = 3, backoff_seconds: float = 0.5) -> None:
        self.delegate = delegate
        self.max_attempts = max_attempts
        self.backoff_seconds = backoff_seconds

    def send(self, recipient: str, message: str) -> None:
        last_error: Exception | None = None
        for attempt in range(1, self.max_attempts + 1):
            try:
                self.delegate.send(recipient, message)
                return
            except Exception as exc:  # network blip, provider 5xx, etc.
                last_error = exc
                if attempt < self.max_attempts:
                    time.sleep(self.backoff_seconds * attempt)
        raise RuntimeError(f"delivery to {recipient} failed after {self.max_attempts} attempts") from last_error


class Subscription(Observer[OrderStatusChanged]):
    def __init__(self, user_contact: str, channel: NotificationChannel) -> None:
        self.user_contact = user_contact
        self.channel = channel

    def on_update(self, event: OrderStatusChanged) -> None:
        message = f"Order {event.order_id} is now {event.new_status.name}"
        try:
            self.channel.send(self.user_contact, message)
        except Exception as send_failure:  # one bad channel must not break the fan-out loop
            print(f"Delivery failed for {self.user_contact}: {send_failure}")


class AsyncOrder(Subject[OrderStatusChanged]):
    """Async dispatch for the "thousands of observers" follow-up: fans out onto a thread pool
    instead of the calling thread, so one slow observer can't stall the rest. Note the trade-off
    this makes explicit — `notify_observers` (and therefore `update_status`) now returns before
    delivery is confirmed; see the Follow-up questions section."""

    def __init__(self, delegate: Order, pool: ThreadPoolExecutor) -> None:
        self.delegate = delegate
        self.pool = pool

    def subscribe(self, observer: Observer[OrderStatusChanged]) -> None:
        self.delegate.subscribe(observer)

    def unsubscribe(self, observer: Observer[OrderStatusChanged]) -> None:
        self.delegate.unsubscribe(observer)

    def notify_observers(self, event: OrderStatusChanged) -> None:
        self.pool.submit(self.delegate.notify_observers, event)
```

## Sample walkthrough

```python
order = Order("ORD-1001")

alice_sub = Subscription("alice@example.com", EmailChannel())
bob_sub = Subscription("+1-555-0100", RetryingChannel(SmsChannel(), max_attempts=3))

order.subscribe(alice_sub)
order.subscribe(bob_sub)

order.update_status(OrderStatus.SHIPPED)
# -> EMAIL to alice@example.com: Order ORD-1001 is now SHIPPED
# -> SMS to +1-555-0100: Order ORD-1001 is now SHIPPED

order.unsubscribe(bob_sub)          # bob no longer wants updates — explicit teardown, not just weakref hope
order.update_status(OrderStatus.DELIVERED)
# -> EMAIL to alice@example.com: Order ORD-1001 is now DELIVERED   (bob correctly silent)
```

## Follow-up questions

- **"A channel send fails intermittently (network blip on SMS) — what's the retry story?"** Shown above: `RetryingChannel` wraps any `NotificationChannel` with bounded retries + backoff, or push failed sends onto a dead-letter queue for later replay; `Subscription.on_update` already isolates one failure from breaking the fan-out loop, so retry logic slots in at the channel layer without touching `Order`.
- **"There are 50,000 observers on one hot subject — synchronous notify is too slow."** Switch to `AsyncOrder`/the pool-based dispatch shown above, referencing [../04-concurrency-essentials.md](../04-concurrency-essentials.md) for sizing the executor and deciding fire-and-forget vs waiting on `Future`s for delivery confirmation; note the trade-off that async means the caller (`update_status`) returns before delivery is confirmed.
- **"Users want to configure channel preference per event type, not per subscription."** Extend `Subscription` to hold a `dict[EventType, NotificationChannel]` (or one `Subscription` per event type, which is simpler and matches the existing model) — the `Order`/`Subject` side is untouched either way since it only ever calls `on_update`.
- **"Isn't this the same as an event bus — why not just use one?"** No: here `Order` holds direct references to its observers (tight but simple, in-process, synchronous-by-default). An event bus decouples further — publishers and subscribers never reference each other, a broker in between handles routing/delivery, usually async and often out-of-process (Kafka/SQS-style). Reach for an event bus when you have *many unrelated subject types* publishing to *many unrelated subscriber types* and don't want N×M direct wiring; Observer is the right minimal tool when one subject's state change matters to a bounded, directly-known set of listeners.
- **"How do we avoid the memory leak where forgotten subscriptions pile up?"** Discipline first: every `subscribe` should have a matching `unsubscribe` on teardown (e.g. when a user closes the order-tracking screen). As a safety net (not a substitute), the implementation above uses `weakref.WeakSet` so a `Subscription` with no other live references is collectible even if `unsubscribe` was missed — call this trade-off out explicitly if asked, and note that `WeakSet` only helps because `Subscription` objects are otherwise unreferenced; if a caller also keeps its own strong-referenced list of subscriptions "for bookkeeping," the weak reference in `Order` buys nothing.

## Common mistakes on this problem

- Declaring the observer collection as a class-level attribute (`_observers = weakref.WeakSet()` directly under `class Order:`) instead of assigning it inside `__init__`. This is a general Python trap — a mutable class attribute is shared across *every instance* of the class — but it's especially easy to trip on here because it silently produces a design that looks correct in a single-order test and then cross-notifies unrelated orders the moment a second `Order` instance exists.
- Conflating Observer with pub-sub/event-bus when asked to compare them — the direct-reference-vs-broker distinction (see follow-ups above) is a common probe, and hand-waving "they're basically the same" reads as a shallow pattern vocabulary rather than understood trade-offs.
- Baking the channel choice (email vs SMS) directly into the `Observer.on_update` implementation as an `if`/`elif` on a `channel_type` field, instead of injecting a `NotificationChannel` strategy — collapses two independent axes of variability into one class and makes adding WhatsApp support require editing existing subscription code.
- Never addressing the unsubscribe path at all — a design that only shows `subscribe()` and a working notify loop but no teardown story is an easy "what happens after 10,000 users unsubscribe... did they, though?" follow-up that exposes an unbounded-growth bug.

## Continue

This is the last file in `problems/`. For closing out interview prep, read [../05-common-mistakes.md](../05-common-mistakes.md) (what actually loses points across all of these problems) followed by [../06-final-checklist.md](../06-final-checklist.md) (the pre-interview 10-minute drill).
