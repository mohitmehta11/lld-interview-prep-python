# The Repeatable Approach Framework

Run this exact 6-step script on every problem below (and on any problem you get in a real interview that isn't in this list — the script generalizes). The goal is to make "starting" automatic so your working memory is free for the actual design decisions.

## The 6 steps

1. **Clarify requirements** (2-3 min). Run the actor/use-case/scale/persistence/non-goals script from [../01-requirements-and-uml.md](../01-requirements-and-uml.md). Say your assumptions out loud even if the interviewer doesn't push back.

2. **Nouns-and-verbs pass** (2-3 min). List the use cases as sentences, underline nouns → candidate classes, underline verbs → candidate methods, group verbs under their noun. Flag every "varies by type" phrase — that's a [Strategy](../patterns/03-behavioral-patterns.md#strategy) seam, not an `if`/`elif` chain.

3. **Sketch entities + relationships** (3-5 min). ASCII box diagram using the `has-a[*]`/`has-a[1]`/`(interface)` convention from [../01-requirements-and-uml.md](../01-requirements-and-uml.md). Decide is-a vs has-a vs uses-a for every pair. This is where you catch a God Object before writing a line of code.

4. **Name your patterns before coding** (1-2 min). For each polymorphic seam you found in step 2, say which pattern fits and why — out loud, one sentence, e.g. "pricing varies by vehicle type and needs to be swappable, so `PricingStrategy` as an injected interface (or callable)." Check [../patterns/00-overview.md](../patterns/00-overview.md) if you're unsure. Resist naming a pattern you don't have a concrete use for (see the overuse trap in that file).

5. **Write the code** (15-20 min). Abstract base classes/`Protocol`s first, then 2-3 concrete implementations, then the coordinating class that wires them together (constructor-injected, per [DIP](../02-solid-principles.md#d--dependency-inversion-principle)). Narrate SOLID/pattern choices as you type — see the narration habit in [../00-evaluation-framework.md](../00-evaluation-framework.md).

6. **Walk a use case end-to-end, then invite the follow-up** (5-10 min). Trace one real call through your classes out loud. Then proactively say "the obvious next ask here is probably X — here's how I'd extend it" before the interviewer asks. This is the single highest-leverage sentence you can say in the whole interview.

## The self-check before you call it done

- Can I add a new variant (vehicle type / payment method / notification channel / split type) by **adding one class (or one function)**, touching **zero** existing classes?
- Is there any class with more than ~5-6 unrelated responsibilities? (SRP smell)
- Any `isinstance` chain or `if type ==` branch in business logic? (OCP/polymorphism smell — see [../03-python-oop-essentials.md](../03-python-oop-essentials.md) for when a `match` statement or `functools.singledispatch` is a legitimate alternative to it, versus when it's a sign you're missing a Strategy/polymorphic seam.)
- Any subclass that overrides a method to raise/no-op? (LSP red flag)
- Any concrete class instantiated directly *inside* another business-logic class, instead of injected? (DIP smell)
- Did I say out loud which pattern I used and why, for at least 2 design decisions?

## The problem set, in the order this knowledge base covers them

1. [01-parking-lot.md](01-parking-lot.md) — Strategy, Factory, Singleton nuance, Observer, concurrency intro
2. [02-elevator-system.md](02-elevator-system.md) — State machine, Strategy (dispatch), Observer, concurrency
3. [03-vending-machine.md](03-vending-machine.md) — State pattern showcase
4. [04-tic-tac-toe-and-chess.md](04-tic-tac-toe-and-chess.md) — polymorphism vs Strategy, scoping discipline
5. [05-lru-cache-and-rate-limiter.md](05-lru-cache-and-rate-limiter.md) — data-structure-heavy LLD
6. [06-splitwise-expense-sharing.md](06-splitwise-expense-sharing.md) — Strategy/OCP, ledger modeling
7. [07-movie-ticket-booking.md](07-movie-ticket-booking.md) — the concurrency showcase (seat locking)
8. [08-library-management-system.md](08-library-management-system.md) — entity-relationship depth
9. [09-logging-framework.md](09-logging-framework.md) — Chain of Responsibility showcase, Builder
10. [10-notification-and-observer-system.md](10-notification-and-observer-system.md) — Observer showcase

## Continue

Next: [01-parking-lot.md](01-parking-lot.md)
