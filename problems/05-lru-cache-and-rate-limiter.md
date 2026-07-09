# LRU Cache & Rate Limiter

Two warm-up problems in one file on purpose: both are graded on axis 2 (right data structure) more than axis 4 (patterns). The recurring trap is building a class hierarchy where the interviewer wanted you to reach for the right built-in data structure (`OrderedDict`, `deque`, `heapq`) in one line instead of hand-rolling a class hierarchy, then layer patterns only where real variability exists (rate-limiting algorithm choice).

## Requirements

### LRU Cache

- "What's the capacity model — fixed at construction, or resizable?" → **You decide**: fixed capacity passed to the constructor; resizing is an out-of-scope follow-up.
- "Do we need thread safety?" → **You decide**: assume single-threaded core; mention the lock wrapper as a follow-up (see Concurrency note below).
- "Should `get` count as a "use" for recency, or only `put`?" → **You decide**: both `get` and `put` refresh recency — that's the definition of LRU (vs LFU, which tracks frequency, not recency).
- In scope: `get(key) -> value`, `put(key, value)`, O(1) average time for both, eviction of the least-recently-used entry on overflow.
- Out of scope: LFU variant, TTL-based expiry (that's the rate limiter's concern below), persistence, distributed cache coherence.

### Rate Limiter

- "Per-user or global limiter?" → **You decide**: per-key (e.g., per user ID or API key) — a single global limiter is a degenerate case of the same design with one key.
- "Which algorithm?" → **You decide**: support token bucket and sliding-window log behind one interface; the interviewer usually wants to see you *swap* algorithms, not commit to one.
- "What happens on rejection — block, queue, or drop?" → **You decide**: drop (return `False`/deny) — queuing turns this into a different problem (a scheduler).
- In scope: `allow_request(key) -> bool`, pluggable algorithm, per-key isolation.
- Out of scope: distributed rate limiting (Redis-backed counters), leaky bucket and fixed-window full implementations (discussed, not coded), backpressure/queueing.

## Core entities & relationships

```
LRUCache
  ├─ has-a[1] Map<Key, Node>        (hash index for O(1) lookup)
  └─ has-a[1] doubly-linked list    (recency order; head=MRU, tail=LRU)

RateLimiter (interface)
  ├─ implemented-by TokenBucketRateLimiter
  └─ implemented-by SlidingWindowRateLimiter

RateLimiterRegistry
  └─ has-a[*] RateLimiter            (one instance per key, e.g. per user)
```

The LRU cache is deliberately *not* modeled as a class hierarchy — there's no polymorphic seam here, just a data-structure choice (hash map for O(1) lookup + linked list for O(1) reorder/evict). Forcing a Strategy or Template Method onto this is the single most common overuse mistake on this problem (see `../patterns/00-overview.md` overuse trap). The rate limiter is the opposite case: the algorithm *is* the variability, so it gets a real interface.

## Design patterns applied

- [Strategy](../patterns/03-behavioral-patterns.md#strategy) — `RateLimiter` is the interface; `TokenBucketRateLimiter` and `SlidingWindowRateLimiter` are interchangeable policies selected at construction, satisfying [OCP](../02-solid-principles.md#o--openclosed-principle): a new algorithm (leaky bucket) means a new class, zero edits to callers.
- No pattern on the LRU cache itself — noted explicitly because *not* applying one is the correct call here; the variability (eviction policy) is a single fixed rule, not something that swaps at runtime. If an interviewer asks for a pluggable eviction policy (LRU vs LFU vs FIFO), *then* Strategy becomes justified — see Follow-up questions.
- `RateLimiter` as `abc.ABC` rather than `typing.Protocol` is a judgment call, not a hard rule, worth naming out loud: neither implementation shares state or default logic with the other, so a `Protocol` (`class RateLimiter(Protocol): def allow_request(self, now: float) -> bool: ...`) would work equally well and lets a test stub satisfy the interface with a bare function/lambda, no inheritance required. `ABC` is used here because the classic Strategy write-up expects an explicit, documented contract that reads as "subclass me" — reach for `Protocol` instead when you want structural typing (e.g., accepting anything duck-typed, including objects defined in code you don't own).

## Implementation

### LRU Cache

> **Pythonic idiom note:** `collections.OrderedDict` already maintains insertion/access order and supports O(1) `move_to_end` and `popitem(last=False)`. This *is* the hash-map-plus-doubly-linked-list structure the problem is testing for — CPython's `OrderedDict` is a dict combined with an internal doubly linked list. Reach for this when the interviewer says "use the language," and say so out loud: "I'm using `OrderedDict.move_to_end`, which is exactly an LRU list + hash index under the hood." **Some interviewers explicitly forbid stdlib shortcuts** to test whether you understand the underlying mechanism — know both versions below, lead with whichever the interviewer signals they want, and mention that the other exists (that's the bonus point: showing you know it's a shortcut, not that you don't know what's underneath).

```python
from collections import OrderedDict
from typing import Any


class LRUCache:
    """Idiomatic version: OrderedDict.move_to_end + popitem(last=False)."""

    def __init__(self, capacity: int) -> None:
        self.capacity = capacity
        self._store: OrderedDict[Any, Any] = OrderedDict()

    def get(self, key: Any) -> Any | None:
        if key not in self._store:
            return None
        self._store.move_to_end(key)
        return self._store[key]

    def put(self, key: Any, value: Any) -> None:
        if key in self._store:
            self._store.move_to_end(key)
        self._store[key] = value
        if len(self._store) > self.capacity:
            self._store.popitem(last=False)  # evict LRU (front of the dict)

    def __len__(self) -> int:
        return len(self._store)

    def __contains__(self, key: Any) -> bool:
        return key in self._store
```

**From scratch (when `OrderedDict` is disallowed):** a `dict` for O(1) lookup plus a hand-rolled doubly-linked list (sentinel head/tail nodes to avoid null-checks at the boundaries) for O(1) move-to-front/evict. This is the version to lead with if the interviewer says "implement the mechanism yourself" — narrate the sentinel-node trick, since it's what eliminates edge-case branching for "the list is empty" or "removing the only node."

```python
class _Node:
    __slots__ = ("key", "value", "prev", "next")

    def __init__(self, key: Any = None, value: Any = None) -> None:
        self.key = key
        self.value = value
        self.prev: "_Node | None" = None
        self.next: "_Node | None" = None


class LRUCacheManual:
    """From-scratch version: dict for O(1) lookup + hand-rolled doubly-linked list."""

    def __init__(self, capacity: int) -> None:
        self.capacity = capacity
        self._index: dict[Any, _Node] = {}
        self._head, self._tail = _Node(), _Node()  # sentinels — no None-checks at list boundaries
        self._head.next, self._tail.prev = self._tail, self._head

    def get(self, key: Any) -> Any | None:
        node = self._index.get(key)
        if node is None:
            return None
        self._move_to_front(node)
        return node.value

    def put(self, key: Any, value: Any) -> None:
        node = self._index.get(key)
        if node is not None:
            node.value = value
            self._move_to_front(node)
            return
        if len(self._index) == self.capacity:
            lru = self._tail.prev
            self._remove(lru)
            del self._index[lru.key]
        node = _Node(key, value)
        self._index[key] = node
        self._insert_after_head(node)

    def _move_to_front(self, node: _Node) -> None:
        self._remove(node)
        self._insert_after_head(node)

    def _remove(self, node: _Node) -> None:
        node.prev.next, node.next.prev = node.next, node.prev

    def _insert_after_head(self, node: _Node) -> None:
        node.next, node.prev = self._head.next, self._head
        self._head.next.prev, self._head.next = node, node

    def __len__(self) -> int:
        return len(self._index)
```

`_Node` uses `__slots__` — a small but real interview signal: it says "I know Python objects carry a `__dict__` by default, and for a node type instantiated potentially millions of times, that per-instance dict overhead is worth avoiding."

### Rate Limiter — Strategy

```python
from abc import ABC, abstractmethod
from collections import deque
from typing import Callable
import threading


class RateLimiter(ABC):
    @abstractmethod
    def allow_request(self, now: float) -> bool: ...


class TokenBucketRateLimiter(RateLimiter):
    """Continuous refill; allows bursts up to bucket capacity."""

    def __init__(self, capacity: float, refill_per_second: float, now: float) -> None:
        self.capacity = capacity
        self.refill_per_second = refill_per_second
        self.tokens = capacity
        self.last_refill = now
        self._lock = threading.Lock()

    def allow_request(self, now: float) -> bool:
        with self._lock:
            elapsed = now - self.last_refill
            if elapsed > 0:
                self.tokens = min(self.capacity, self.tokens + elapsed * self.refill_per_second)
                self.last_refill = now
            if self.tokens >= 1:
                self.tokens -= 1
                return True
            return False


class SlidingWindowRateLimiter(RateLimiter):
    """Exact rolling-window count; no burst tolerance."""

    def __init__(self, max_requests: int, window_seconds: float) -> None:
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self.timestamps: deque[float] = deque()
        self._lock = threading.Lock()

    def allow_request(self, now: float) -> bool:
        with self._lock:
            while self.timestamps and now - self.timestamps[0] >= self.window_seconds:
                self.timestamps.popleft()  # drop entries that aged out of the window
            if len(self.timestamps) < self.max_requests:
                self.timestamps.append(now)
                return True
            return False


class RateLimiterRegistry:
    """Per-key isolation; one limiter instance per key, lazily created."""

    def __init__(self, factory: Callable[[], RateLimiter]) -> None:
        self._factory = factory
        self._limiters: dict[str, RateLimiter] = {}
        self._lock = threading.Lock()

    def allow(self, key: str, now: float) -> bool:
        with self._lock:
            limiter = self._limiters.setdefault(key, self._factory())
        return limiter.allow_request(now)
```

Fixed-window counters (reset a counter every N seconds — simple but allows 2x burst at window boundaries) and leaky bucket (fixed-rate outflow, smooths bursts more aggressively than token bucket) are the other two textbook algorithms; both slot in as one more `RateLimiter` subclass each, no other code changes — that's the OCP payoff of the Strategy seam.

> **Pythonic idiom note:** `RateLimiterRegistry.allow` uses `dict.setdefault` for "get or lazily create and insert" in one call instead of the `if key not in dict: dict[key] = ...` two-step — idiomatic, but note it always evaluates `self._factory()` eagerly (constructing a throwaway `RateLimiter` on every call even when the key already exists) unless you guard it; `collections.defaultdict(factory)` has the same eager-construction caveat. For a genuinely expensive factory, prefer the explicit `if key not in self._limiters: self._limiters[key] = self._factory()` form instead — worth flagging in an interview as a "idiomatic but has a subtle cost" trade-off rather than reaching for `setdefault` unthinkingly.

## Sample walkthrough

```python
cache = LRUCache(capacity=2)
cache.put("a", 1)
cache.put("b", 2)
cache.get("a")          # "a" now most-recently-used; order is b, a
cache.put("c", 3)       # capacity exceeded -> evicts "b" (the LRU)
assert cache.get("b") is None
assert cache.get("a") == 1
assert cache.get("c") == 3

import time
registry = RateLimiterRegistry(lambda: TokenBucketRateLimiter(capacity=5, refill_per_second=1, now=time.time()))
now = time.time()
for _ in range(5):
    assert registry.allow("user-42", now) is True   # burst of 5 consumes the full bucket
assert registry.allow("user-42", now) is False       # 6th request in the same instant is denied
assert registry.allow("user-99", now) is True        # different key, isolated bucket
```

## Follow-up questions

- **"Make the eviction policy pluggable (LRU vs LFU vs FIFO)."** This is where Strategy *does* become justified on the cache: extract an `EvictionPolicy` `ABC`/`Protocol` with `on_access(key)`/`on_insert(key)`/`eviction_candidate()`, and have `LRUCache` delegate to it instead of hardcoding recency order.
- **"Support TTL expiry per entry."** Store `(value, expires_at)` in the node/value slot; check-and-evict lazily on `get`, plus an optional background sweep — doesn't change the core structure.
- **"Make the cache thread-safe under concurrent readers/writers."** Wrap the whole structure in one `threading.Lock` (coarse-grained is fine here — LRU reordering on every `get` makes fine-grained per-node locking correctness-hazardous); see [../04-concurrency-essentials.md](../04-concurrency-essentials.md) for the primitives and trade-offs.
- **"Rate limiter needs to work across multiple app servers, not just in-process."** Swap the in-memory `dict` in `RateLimiterRegistry` for a Redis-backed counter (`INCR` + `EXPIRE` for fixed-window, a Lua script for atomic token-bucket) — the `RateLimiter` interface doesn't change, only the implementation's storage.
- **"What if two algorithms need to run together (e.g., both a burst limit and a sustained-rate limit)?"** Compose two `RateLimiter` instances and require both to return `True` — a `CompositeRateLimiter` implementing the same interface, no client-code change.

## Common mistakes on this problem

- Building an `EvictionStrategy` interface with only one implementation for the cache — this is the overuse trap (see `../patterns/00-overview.md`): no interviewer-stated variability means no pattern yet.
- Using a plain `list` for the LRU order and doing O(n) linear scans to find/move an entry — defeats the entire point of the problem; the interviewer is specifically listening for "hash map + doubly-linked list."
- Forgetting that `get` must also update recency (common bug: only `put` updates order, silently turning it into FIFO-with-a-cache-shaped-API).
- Hardcoding one rate-limiting algorithm inline in the endpoint/controller instead of behind an interface, then having no answer when asked "how would you A/B test two algorithms."

## Continue

Next: [06-splitwise-expense-sharing.md](06-splitwise-expense-sharing.md)
