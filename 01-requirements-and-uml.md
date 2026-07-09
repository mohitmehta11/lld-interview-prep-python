# Requirements Gathering & Just-Enough UML

## The clarifying-questions script

Run through these categories fast (2-3 questions each, not an interrogation). This is the same script for every problem in [problems/](problems/00-approach-framework.md).

1. **Actors** — Who/what uses this system? (Customer, Admin, System-clock/scheduler?)
2. **Core use cases** — What are the 3-5 things an actor *does*? Say them back as a numbered list and get a nod.
3. **Scale/concurrency** — Single process or distributed? Multiple threads/users hitting it at once? (You can say "I'll design for single-process, multi-threaded, and call out where I'd change things for distributed" — that's a senior move. In Python specifically, be ready to say whether you mean OS threads under the GIL, `asyncio` concurrency, or `multiprocessing` — they are not interchangeable answers, see [04-concurrency-essentials.md](04-concurrency-essentials.md).)
4. **Persistence** — In-memory only, or does state need to survive restarts? (Usually in-memory is fine to assume — say so explicitly.)
5. **Explicit non-goals** — "I will *not* handle payment gateway integration / auth / UI" — state this out loud so the interviewer can correct scope creep early.

Write these as a short bullet list at the top of your workspace before touching a class. This single habit is worth more than any UML polish.

## UML you actually need (verbal or scratch-diagram level)

You will **not** be asked to produce a formal UML diagram with exact arrow notation in most interviews — but you should think in these relationship types and be able to sketch boxes+arrows on a whiteboard/doc in under 2 minutes.

| Relationship | Meaning | Code shape | Example |
|---|---|---|---|
| **Inheritance ("is-a")** | Subtype specializes supertype | `class Car(Vehicle):` | `Car is-a Vehicle` |
| **Interface implementation** | Contract, no shared state | `class Car(Parkable, ABC):` (nominal) or a class that merely *matches* a `Parkable` `Protocol` (structural, no inheritance needed) | `Car implements Payable` |
| **Composition ("has-a", owns, can't outlive")** | Strong ownership; part dies with whole | field created in `__init__`, owned exclusively | `Car has-a Engine` (engine has no meaning outside this car) |
| **Aggregation ("has-a", shared/independent lifetime)** | Weak ownership; part outlives whole | field, but reference to an externally-owned object | `ParkingLot has-a list[ParkingSpot]` but spots could conceptually be reused/queried elsewhere |
| **Association ("uses-a")** | One class calls/references another without ownership | method parameter, or a plain reference field | `PaymentProcessor uses-a Logger` |
| **Multiplicity** | 1-1, 1-many, many-many | collection type in the field (`list[...]`, `dict[...]`) | `ParkingLot 1 -- * ParkingSpot` |

**Pythonic idiom note:** the "interface implementation" row has two legitimate Python shapes where Java only really has one (`implements`). Use `abc.ABC` + `@abstractmethod` when you want a *nominal*, enforced contract (subclasses that forget to implement a method fail at instantiation time) — this is the closer analogue to a Java interface. Use `typing.Protocol` when you want *structural* typing: any object with the right method signatures satisfies the contract, with no inheritance or explicit declaration required. In interviews, reach for `Protocol` when you're describing a capability a type happens to have (duck typing, e.g. anything with `.read()`), and `ABC` when you're deliberately building a polymorphic hierarchy the caller will `isinstance`-check or that needs shared default behavior. This distinction is explored further in [02-solid-principles.md](02-solid-principles.md) under ISP and DIP.

### Text-diagram convention used throughout this knowledge base

Since we're not drawing in a GUI, every problem file uses this ASCII shorthand — learn to read it once:

```
Vehicle (abstract)
  ├─ Car
  ├─ Motorcycle
  └─ Truck

ParkingLot
  ├─ has-a[*] ParkingFloor
  └─ has-a[1] PricingStrategy (interface)

ParkingFloor
  └─ has-a[*] ParkingSpot

ParkingSpot
  └─ uses-a Vehicle (when occupied)
```

`[*]` = one-to-many, `[1]` = exactly one, `(interface)` flags a Strategy/polymorphic seam (an `ABC` or `Protocol`).

## Identifying entities: nouns and verbs pass

A fast, reliable technique once you've gathered requirements:

1. **Underline every noun** in the use cases → candidate classes (`Vehicle`, `ParkingSpot`, `Ticket`, `Payment`).
2. **Underline every verb** → candidate methods (`park()`, `unpark()`, `calculate_fee()`, `pay()`).
3. **Group verbs under the noun they most naturally belong to** — this is your first-pass class responsibility split. If one noun ends up owning 15 verbs, it's a God Object smell — split it (this is [SRP](02-solid-principles.md#s--single-responsibility-principle)).
4. **Look for "varies by type" language** ("fee differs by vehicle type," "different payment methods," "different notification channels") — every one of these phrases is a [Strategy pattern](patterns/03-behavioral-patterns.md#strategy) seam, and should become an abstraction (`ABC` or `Protocol`), not an `if/elif` chain on a type field.

## Continue

Next: [02-solid-principles.md](02-solid-principles.md)
