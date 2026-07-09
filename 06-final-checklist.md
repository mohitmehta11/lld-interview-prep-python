# Final Checklist — 10 Minutes Before the Interview

A pure drill, no new concepts. If you can rattle through this without opening any other file, you're ready.

## Say these 4 sentences out loud, filling in any example

1. "I'll spend the first few minutes clarifying actors, use cases, scale, persistence, and non-goals before touching code." → [01-requirements-and-uml.md](01-requirements-and-uml.md)
2. "One class, one axis of change; add a class (or register a new handler), don't edit a branch statement; a subtype must honor the base type's promises; many small interfaces beat one fat one; depend on interfaces, inject concretions at the top." → [02-solid-principles.md](02-solid-principles.md)
3. "When behavior varies by type, that's Strategy. When construction is complex, that's Builder. When I need to notify many dependents, that's Observer. When a class's behavior genuinely changes with its internal state, that's State, not booleans." → [patterns/00-overview.md](patterns/00-overview.md)
4. "In Python I use `abc.ABC` for real enforced contracts, `typing.Protocol` for structural typing, `@dataclass(frozen=True)` for value objects, `Enum` instead of string/int constants, and I never leave a mutable default argument." → [03-python-oop-essentials.md](03-python-oop-essentials.md)

## The 60-second entity-modeling drill

Pick any random noun-phrase problem (e.g. "design a hotel booking system") and, without writing anything, mentally run:
1. Actors? Use cases (3-5)?
2. Nouns → classes, verbs → methods, group by noun.
3. Any "varies by type" phrase → name the Strategy interface (or ask whether a plain callable/`functools.singledispatch` is enough — you don't always need a class hierarchy).
4. Any is-a that might be a lie (LSP check)? Would `Protocol` (structural) fit better here than inheritance (nominal)?
5. Say the one sentence: "I'd inject X into Y rather than construct it inside, so I can swap it later."

If you can do this in under a minute for an unfamiliar problem, the framework has stuck.

## The extensibility reflex

Before the interviewer asks "now add X," you say it first. Pick one problem you've studied and say out loud: "the obvious next ask here is [X], and my design absorbs it by [one sentence]." Do this for at least one problem from each of [01-parking-lot.md](problems/01-parking-lot.md), [02-elevator-system.md](problems/02-elevator-system.md), and [07-movie-ticket-booking.md](problems/07-movie-ticket-booking.md) — these three cover the extensibility + concurrency follow-ups most likely to recur across any problem you actually get asked.

## Concurrency one-liner, ready to deploy

"I'll design single-threaded first, and here are the two places I'd add locking if asked: [shared mutable state], using [`threading.Lock`/`Condition`/an atomic-compare-and-swap version field], acquired in a consistent order to avoid deadlock. If the bottleneck were I/O rather than shared state, I'd reach for `asyncio` instead of threads." → [04-concurrency-essentials.md](04-concurrency-essentials.md)

## The idiom reflex

Before you write a class hierarchy, ask yourself the two questions the whole essentials file boils down to: "does this need shared state/behavior (→ `ABC`) or just a matching shape (→ `Protocol`)?" and "is this actually a value object (→ `@dataclass`) before I hand-write `__init__`/`__eq__`/`__repr__`?" Getting these two right by reflex, without having to stop and think, is what separates "knows Python" from "knows OOP and happens to be typing Python syntax."

## The absolute last thing to remember

Narrate. A correct silent design scores lower than a good design you talk through. Every sentence above is designed to be said out loud, not just known.

## If you have 20 more minutes, not 10

Skim [problems/](problems/00-approach-framework.md) in order — even 2 minutes per file on the ones you haven't gotten to ([04-tic-tac-toe-and-chess.md](problems/04-tic-tac-toe-and-chess.md), [08-library-management-system.md](problems/08-library-management-system.md), [09-logging-framework.md](problems/09-logging-framework.md), [10-notification-and-observer-system.md](problems/10-notification-and-observer-system.md)) is enough to recognize the shape if one of them comes up.
