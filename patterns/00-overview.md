# Design Patterns — Decision Map

Don't study patterns as a list to memorize — study them as answers to smells you'll hear in requirement language. This file is the lookup table; the next three files are the depth. When you catch yourself about to say "I'll use X pattern here," check the smell first — if the requirement doesn't actually say this, you're about to lose points, not gain them (see "The overuse trap" below).

## Symptom → Pattern

| Symptom / smell in requirements | Pattern | Why |
|---|---|---|
| "behavior varies by type" / "different X types behave differently" | [Strategy](03-behavioral-patterns.md#strategy) | Swap an algorithm/policy at runtime without an `if/else` on a type field — the direct mechanism for [OCP](../02-solid-principles.md#o--openclosed-principle) |
| "complex object construction with many optional params" | [Builder](01-creational-patterns.md#builder) | Replaces a telescoping-constructor or setter-soup with a fluent, validated, immutable-result construction path |
| "need exactly one instance / shared coordinator" | [Singleton](01-creational-patterns.md#singleton) | One shared access point — but see the overuse trap below; also a thread-safety hazard, see [../04-concurrency-essentials.md](../04-concurrency-essentials.md) |
| "family of related objects must be created together" | [Abstract Factory](01-creational-patterns.md#abstract-factory) | Guarantees the objects you get back are mutually compatible (e.g. a processor + its matching refund handler) |
| "adapt an incompatible interface" | [Adapter](02-structural-patterns.md#adapter) | Wrap a third-party/legacy interface so your code depends on the interface it actually wants |
| "add responsibilities dynamically without subclass explosion" | [Decorator](02-structural-patterns.md#decorator) | Stack behavior at runtime (e.g. notification channels, coffee add-ons) instead of `N × M` subclasses |
| "treat individual objects and groups uniformly" | [Composite](02-structural-patterns.md#composite) | Recursive tree where leaf and container share one interface — client code doesn't branch on "is this a group?" |
| "simplify a complex subsystem" | [Facade](02-structural-patterns.md#facade) | One clean entry point hides orchestration of several subsystems the client shouldn't need to know about |
| "control/defer access to an expensive or sensitive object" | [Proxy](02-structural-patterns.md#proxy) | Interpose lazy-loading, access control, caching, or logging without the client noticing |
| "notify many dependents of a state change" | [Observer](03-behavioral-patterns.md#observer) | Decouples the subject from a variable, growable set of listeners — pub/sub in-process |
| "behavior changes with internal state, and transitions are part of the domain" | [State](03-behavioral-patterns.md#state) | Encodes a finite state machine as classes instead of a `status` enum plus scattered `if` checks |
| "encapsulate a request as an object (undo/redo/queueing)" | [Command](03-behavioral-patterns.md#command) | Turns "do a thing" into a first-class object you can queue, log, undo, or retry |
| "pass a request along a chain of possible handlers" | [Chain of Responsibility](03-behavioral-patterns.md#chain-of-responsibility) | Each handler decides "mine or pass it on" — adding a new handler doesn't touch existing ones |
| "define a skeleton algorithm, let subclasses fill steps" | [Template Method](03-behavioral-patterns.md#template-method) | Fix the algorithm's shape once in a base class, vary only the steps that differ |
| "traverse a collection without exposing its structure" | [Iterator](03-behavioral-patterns.md#iterator) | Client walks elements one at a time without knowing if the backing store is an array, tree, or linked list |

## The overuse trap

Naming a pattern you didn't need is graded down **exactly as hard** as missing a pattern you did need — this is axis 4 of the [evaluation framework](../00-evaluation-framework.md), and it's the single most common way strong engineers lose points in LLD rounds. Announcing "I'll use Singleton for the `Logger`" when nobody asked about shared coordination, or wrapping a `PaymentMethod` interface in a `Factory` when there's exactly one implementation and no sign more are coming, reads as cargo-culting, not rigor. Before naming a pattern out loud, state the smell first ("the problem says fee calculation differs by vehicle type, so...") — if you can't point to the smell in the requirements, don't reach for the pattern yet. The minimal design that solves the actual variability in front of you beats a decorated one every time.

This also applies with extra force in Python: the language already gives you first-class functions, closures, `functools` decorators, and duck typing, so several GoF patterns that need a class hierarchy in a more rigid language collapse into a function, a dict, or a dataclass here. Naming the class-hierarchy version of a pattern when a five-line function would do is its own flavor of overuse — the next two files call out, pattern by pattern, when that's the case.

## Continue

Next: [01-creational-patterns.md](01-creational-patterns.md)
