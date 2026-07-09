# LLD Interview Knowledge Base — Python

A self-contained, linked knowledge base for cracking Low-Level Design (LLD) interviews in **Python**. Forked from a Java+Python combined version of this repo — that version deliberately weighted Java heavier because its author was strong in Python already but weak in Java idiom. This fork drops Java entirely and goes the other direction: every concept, pattern, and problem gets the full, primary, idiomatic-Python treatment — not a "refresher" bolted onto a Java-first curriculum. If a Pythonic approach differs materially from the generic-OOP shape (a module instead of a Singleton class, a plain callable instead of a Strategy hierarchy, `Protocol` instead of `ABC`), that's called out explicitly as a **Pythonic idiom note** so you can say it out loud in an interview.

Everything is local markdown. No web browsing needed — just follow the links.

## How this is organized

- **Core spine (root files)** — language-agnostic LLD skills first: how you're evaluated, requirements/UML, SOLID, then one comprehensive Python OOP-idiom file, then Python concurrency, then mistakes, then a checklist.
- **`patterns/`** — the design patterns that actually show up in LLD interviews, with idiomatic Python for each — including, per pattern, whether the "textbook" class-hierarchy shape or a more Pythonic alternative (closures, `functools`, a module-level instance) is the better default.
- **`problems/`** — the canonical LLD problem set (11 problems). Each file: requirements → entities/class diagram → patterns used → full Python implementation → follow-ups an interviewer will throw at you.

Every problem/pattern file cross-links back to the SOLID/pattern concept it exercises, so once the spine is loaded in your head, the problems reinforce it instead of teaching from scratch.

## The 5-hour critical path (packed)

This is the **priority-ordered** route through the material — it is *not* everything in this directory, it's the highest-signal subset. Everything else in `problems/` and `patterns/` is bonus depth (see "Beyond the 5 hours" below) — read it after, or skim it in spare minutes; it's here so you have a full week's worth of reference material on tap, not just a 5-hour script.

Timings assume you read at your own fast pace and *type out the code samples once* rather than just eyeballing them — typing is what makes an idiom stick, especially the ones (`Protocol` vs `ABC`, `@dataclass`, `functools.singledispatch`) that don't have a muscle-memory equivalent from other languages.

### Hour 1 — Foundations & mental model (0:00–1:00)
| Time | File | Why first |
|---|---|---|
| 10 min | [00-evaluation-framework.md](00-evaluation-framework.md) | Know the rubric before you study to it |
| 15 min | [01-requirements-and-uml.md](01-requirements-and-uml.md) | The first 5 min of every real interview |
| 35 min | [02-solid-principles.md](02-solid-principles.md) | The single highest-leverage topic — everything else is an application of this |

### Hour 2 — The Python idiom deep dive (1:00–2:00)
| Time | File | Why |
|---|---|---|
| 50 min | [03-python-oop-essentials.md](03-python-oop-essentials.md) | This is the flagship file — 15 sections covering everything from `__init__` to `match`/`case`. Longer than a "refresher" on purpose: this is the primary treatment, and it's exactly what interviewers probe when they want to know if you actually know Python as a design language, not just as syntax. |
| 10 min | [patterns/00-overview.md](patterns/00-overview.md) | Map of which pattern solves which smell — short enough to fit here and sets up Hour 3 |

### Hour 3 — Design patterns deep dive (2:00–3:00)
| Time | File | Why |
|---|---|---|
| 20 min | [patterns/01-creational-patterns.md](patterns/01-creational-patterns.md) | Factory/Builder/Singleton show up constantly — and Singleton/Factory both have a more-Pythonic alternative worth knowing |
| 20 min | [patterns/02-structural-patterns.md](patterns/02-structural-patterns.md) | Decorator/Adapter/Composite/Facade/Proxy — Decorator in particular often collapses to a plain `@decorator` in Python |
| 20 min | [patterns/03-behavioral-patterns.md](patterns/03-behavioral-patterns.md) | Prioritize Strategy, Observer, State, Command, Chain of Responsibility — skim the rest |

### Hour 4 — Practice problems, batch 1 — the "must-know" four (3:00–4:00)
| Time | File | Why this problem |
|---|---|---|
| 5 min | [problems/00-approach-framework.md](problems/00-approach-framework.md) | The repeatable 6-step script you run in *every* problem below |
| 15 min | [problems/01-parking-lot.md](problems/01-parking-lot.md) | THE canonical LLD problem — Strategy + Factory |
| 15 min | [problems/02-elevator-system.md](problems/02-elevator-system.md) | State machine + concurrency angle |
| 10 min | [problems/05-lru-cache-and-rate-limiter.md](problems/05-lru-cache-and-rate-limiter.md) | Data-structure-heavy LLD, very common as a warm-up problem — know both the `OrderedDict` shortcut and the hand-rolled doubly-linked-list version |
| 15 min | [problems/06-splitwise-expense-sharing.md](problems/06-splitwise-expense-sharing.md) | Graph/ledger modeling, Strategy for split types |

### Hour 5 — Concurrency, batch 2, and close-out (4:00–5:00)
| Time | File | Why |
|---|---|---|
| 15 min | [04-concurrency-essentials.md](04-concurrency-essentials.md) | Elevator/parking-lot/booking follow-ups always probe thread-safety — GIL, `threading`, `multiprocessing`, `asyncio`, all covered here |
| 15 min | [problems/07-movie-ticket-booking.md](problems/07-movie-ticket-booking.md) | Concurrency-under-contention (seat locking, pessimistic vs optimistic) + Observer |
| 10 min | [problems/03-vending-machine.md](problems/03-vending-machine.md) | Cleanest possible State pattern showcase, fast to internalize |
| 10 min | [05-common-mistakes.md](05-common-mistakes.md) | What actually loses points — read this right before any interview |
| 10 min | [06-final-checklist.md](06-final-checklist.md) | Pre-interview 10-minute drill |

**End of 5 hours.** At this point you can competently run a full LLD interview in Python, including the two most commonly tested cross-cutting concerns (concurrency, extensibility) and the idiom fluency (`Protocol` vs `ABC`, `@dataclass`, `singledispatch`) that separates "knows OOP" from "knows Python."

## Beyond the 5 hours (the rest of the week's material)

These are fully written, same depth/format as above — read opportunistically, or the night before if you have a specific problem you're worried about:

- [problems/04-tic-tac-toe-and-chess.md](problems/04-tic-tac-toe-and-chess.md) — board-game modeling, good Strategy/Composite practice, `ABC` vs `Protocol` decision worked through explicitly
- [problems/08-library-management-system.md](problems/08-library-management-system.md) — classic "entity relationship" heavy problem
- [problems/09-logging-framework.md](problems/09-logging-framework.md) — Chain of Responsibility + Builder in a systems-y context
- [problems/10-notification-and-observer-system.md](problems/10-notification-and-observer-system.md) — pure Observer/Strategy combo, pub-sub framing

## Quick navigation by concern

- **"I only have 20 minutes before the interview"** → [06-final-checklist.md](06-final-checklist.md) + [05-common-mistakes.md](05-common-mistakes.md)
- **"I keep second-guessing `ABC` vs `Protocol`"** → [03-python-oop-essentials.md §2](03-python-oop-essentials.md#2-interfaces-typingprotocol-vs-abcabc--the-decision-interviewers-watch-for)
- **"Interviewer asked about thread safety"** → [04-concurrency-essentials.md](04-concurrency-essentials.md)
- **"I don't know which pattern to reach for"** → [patterns/00-overview.md](patterns/00-overview.md)
- **"How do I even start a problem"** → [problems/00-approach-framework.md](problems/00-approach-framework.md)
- **"Is there a Pythonic shortcut for this pattern, or do I need the full class hierarchy?"** → every pattern in `patterns/` has an explicit **Pythonic idiom note** answering exactly this
