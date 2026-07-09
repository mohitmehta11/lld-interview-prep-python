# Common Mistakes That Lose LLD Interview Points

Read this right before any interview. Every item here maps to a scoring axis in [00-evaluation-framework.md](00-evaluation-framework.md) — these are the specific, recurring ways candidates lose points on each axis, not generic advice.

## Process mistakes (lose points before you write a line of code)

- **Diving into code with zero clarifying questions.** Even a strong design gets marked down if the interviewer can't see you scope the problem. Always run the script in [01-requirements-and-uml.md](01-requirements-and-uml.md), even briefly.
- **Over-spending the talk phase.** 25 minutes of requirements + diagram, 10 minutes of rushed code is worse than a slightly-less-polished diagram and complete, clean code. Respect the time budget table in [00-evaluation-framework.md](00-evaluation-framework.md).
- **Silent design.** Working through the problem without narrating WHY. The interviewer can only score what they can observe — say the SOLID/pattern reasoning out loud as you go (see the narration habit).
- **Not stating assumptions.** "I'll assume single-threaded for now and mention how I'd extend it" is a strong sentence. Silently assuming and never mentioning it looks like you didn't consider it at all.

## Modeling mistakes

- **God Object.** One class (`ParkingLot`, `Game`, `System`) that owns pricing, persistence, notification, and validation. Fix: nouns-and-verbs pass, split by axis of change ([SRP](02-solid-principles.md#s--single-responsibility-principle)).
- **`if/elif` (or `match`) on a type field, in business logic, that will need a new branch for every new variant.** This is the single most commonly flagged issue. Fix: polymorphism / [Strategy](patterns/03-behavioral-patterns.md#strategy), or `functools.singledispatch` when the branching is purely type-based ([OCP](02-solid-principles.md#o--openclosed-principle)).
- **Modeling booleans instead of an explicit state enum/machine** (`is_moving`, `doors_open`, `is_dispensing` as separate flags that can go out of sync) when the problem is inherently state-machine-shaped (elevator, vending machine, seat, order). Fix: [State pattern](patterns/03-behavioral-patterns.md#state).
- **Forcing an is-a relationship that doesn't hold** just to reuse code, then overriding methods to `raise NotImplementedError`. This is an [LSP](02-solid-principles.md#l--liskov-substitution-principle) violation — restructure the hierarchy (extract a capability interface, often a `Protocol`) instead.
- **Fat interfaces** that force unrelated implementers to stub out methods they don't need. Fix: [ISP](02-solid-principles.md#i--interface-segregation-principle) — split into role interfaces, and reach for `typing.Protocol` when the split doesn't need shared implementation.
- **Confusing composition and aggregation**, e.g. modeling `ParkingSpot`s as something that could exist and be shared across multiple `ParkingLot`s when the problem clearly implies ownership. Minor, but a sharp interviewer notices.

## Pattern mistakes

- **Pattern-name-dropping without fit.** Saying "I'll use a Factory here" for a plain `SomeClass()` call with no variability, or wrapping everything in a Singleton "just in case." Overuse is graded down as hard as underuse (axis 4 in the evaluation framework) — only reach for a pattern when you can name the specific variability it's absorbing.
- **Singleton for state that doesn't need global enforcement.** Most "there's only one of these" requirements just mean "the composition root creates one instance" — they don't require the *class itself* to prevent multiple instantiation, and in Python the idiomatic form is often just a module (import caches it) rather than a `getInstance()`-style class. Default to plain instantiation + dependency injection; only reach for real Singleton enforcement when the problem explicitly requires it (see the nuance in [patterns/01-creational-patterns.md](patterns/01-creational-patterns.md#singleton)).
- **Hardcoding a concrete dependency inside a business-logic class** (`CreditCardPayment()` instantiated inside `ParkingLot`) instead of constructor-injecting an interface. This is a [DIP](02-solid-principles.md#d--dependency-inversion-principle) violation and it's exactly what breaks the "now swap in X" follow-up.
- **Reaching for Composite/Visitor/Flyweight when the problem doesn't call for them.** These are lower-frequency patterns — don't force them in to look sophisticated.

## Python-idiom mistakes

These are the specific tells that make a design read as "Java transliterated into Python syntax" rather than actual Python — interviewers who've seen a lot of candidates notice this fast, and it's graded (axis 4 in the evaluation framework covers idiom fluency, not just correctness):

- **Manual `get_x()`/`set_x()` methods everywhere** instead of `@property`/`@x.setter`, or hand-rolled `__init__` + boilerplate `__eq__`/`__repr__` where `@dataclass` would do it in one line. See [03-python-oop-essentials.md §6](03-python-oop-essentials.md#6-encapsulation-name-mangling-property-and-why-theres-no-true-private) and [§8](03-python-oop-essentials.md#8-dataclass-vs-hand-written-__init____eq____repr__).
- **A base class with `pass` bodies and a "must override this" comment** instead of `abc.ABC` + `@abstractmethod` — doesn't actually enforce the contract; a forgetful subclass just silently does nothing instead of failing at instantiation time.
- **Reaching for `abc.ABC` by reflex when `typing.Protocol` fits better** (or vice versa) — see the explicit decision rule in [03-python-oop-essentials.md §2](03-python-oop-essentials.md#2-interfaces-typingprotocol-vs-abcabc--the-decision-interviewers-watch-for). Defaulting to ABC because "that's how Java interfaces work" without considering structural typing is a specific, commonly-flagged tell.
- **Mutable default arguments** (`def __init__(self, items=[])`) — a classic, very visible bug. Always `field(default_factory=list)` on a dataclass, or `items: list | None = None` + `self.items = items if items is not None else []`.
- **Using plain string/int constants for a small fixed set of variants** instead of `enum.Enum` (or `IntEnum`/`Flag` where it fits) — see [03-python-oop-essentials.md §11](03-python-oop-essentials.md#11-enums--enum-intenum-flag-auto). Loses type-checker and IDE support for free.
- **No type hints on public method signatures.** Python doesn't require them, but omitting them on class/method boundaries reads as less rigorous in an interview context, and it's the thing that lets a `mypy`/`pyright` check catch the class of bugs static typing exists to catch.
- **Overriding `__eq__` without `__hash__`** (or vice versa) — silently breaks using the object as a dict key or in a set, same failure mode as Java's `equals`/`hashCode` contract, just less visible since Python won't warn you.
- **Faking method overloading with a pile of `isinstance` checks and `*args`** instead of reaching for `functools.singledispatch` or plain default/keyword arguments — see [03-python-oop-essentials.md §5](03-python-oop-essentials.md#5-method-overriding-vs-java-style-overloading).
- **Ignoring the GIL/threading question entirely, or claiming Python has no concurrency concerns because of it.** The GIL prevents *simultaneous bytecode execution*, not race conditions — a thread switch can still land between two bytecode-level steps of what looks like one atomic operation (`x += 1` is not atomic). See [04-concurrency-essentials.md](04-concurrency-essentials.md) for the precise version of this to say out loud.
- **Not knowing when to reach for a standard-library structure** (`collections.OrderedDict`/`deque`, `heapq`, `functools.lru_cache`) instead of hand-rolling the equivalent data structure from scratch — sometimes the interviewer wants to see you know the primitive underneath (a hand-rolled doubly-linked list for an LRU cache, say), but not knowing the shortcut exists at all is a gap, not a strength.

## The meta-mistake

**Treating this as a coding-only round.** LLD is a *design communication* round that happens to end in code. A merely-correct class hierarchy delivered silently scores lower than a slightly simpler design where you clearly narrated trade-offs, invited the follow-up, and showed you know exactly why each choice was made. Optimize for legible reasoning, not maximal cleverness.

## Continue

Next: [06-final-checklist.md](06-final-checklist.md) — the pre-interview drill.
