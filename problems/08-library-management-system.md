# Library Management System

## Requirements

- "How many copies can a title have?" — Multiple physical copies per title, each independently trackable (barcode, condition, current status). **You decide**: this is the central modeling lesson of this problem — split `Book` (catalog metadata) from `BookItem` (physical copy).
- "Do member types have different privileges?" — Yes: `Student` (5 books, 14-day loan), `Faculty` (10 books, 30-day loan), `Guest` (2 books, 7-day loan, no reservations). **You decide** if not specified: assume at least two tiers to justify polymorphism.
- "How are overdue fines computed?" — Assume a flat per-day rate that can vary by member tier or book category (reference books cost more per day late). Model as pluggable, not hardcoded.
- "What happens when a reserved book becomes available?" — Notify the first member in the reservation queue (FIFO); they get a hold window (e.g. 48 hours) before it goes to the next person. **You decide**: single library branch for the core design, multi-branch is a follow-up.
- "Is this single-threaded?" — Assume single-threaded core logic for the interview; note that checkout/reservation of the *same* `BookItem` needs a lock if concurrent (see [../04-concurrency-essentials.md](../04-concurrency-essentials.md)) — the Python implementation below actually wires this in rather than leaving it purely hypothetical.

**In scope**: catalog (Book/BookItem), member management with tiers, checkout, return, reservation (hold queue), fine calculation, librarian actions (add/remove book, block member).

**Out of scope**: payments for fines (assume an external `PaymentGateway` stub), multi-branch inventory transfer, digital/e-book lending, search/indexing internals.

## Core entities & relationships

```
Library
  ├─ has-a[*] Book                     (catalog entry: title/author/ISBN — metadata only)
  │     └─ has-a[*] BookItem           (physical copy: barcode, condition, status)
  ├─ has-a[*] Member (interface: LibraryAccount)
  │     ├─ Student   (implements LibraryAccount)
  │     ├─ Faculty    (implements LibraryAccount)
  │     └─ Guest       (implements LibraryAccount)
  ├─ has-a[*] Librarian
  └─ has-a[1] FineCalculationStrategy (interface)

BookItem
  └─ has-a[1] BookItemStatus enum      (AVAILABLE | LOANED | RESERVED | LOST)

Loan
  ├─ has-a[1] BookItem
  ├─ has-a[1] Member
  └─ has-a[1] due date, [0..1] returned date

ReservationQueue (per Book, or per BookItem pool)
  └─ has-a[*] Member                   (FIFO hold queue)
```

**Why `Book` vs `BookItem` is a has-a[*], not a single class**: a `Book` is an idea ("Clean Code, 1st ed, ISBN X") — it never changes state. A `BookItem` is a physical, stateful object — it gets loaned, damaged, lost, and returned. Collapsing them forces you to duplicate title/author on every copy row and makes "how many copies of X are available" a filter query instead of a first-class relationship. This is the #1 thing interviewers watch for on this specific problem.

**Why `Member` is an abstract type with tiers as subtypes, not a `type` enum field**: loan limit and loan duration differ by tier, but *no method's behavior changes in a surprising way* — `checkout_limit` and `loan_duration_days` are just different return values, not different control flow. This is a clean [LSP](../02-solid-principles.md#l--liskov-substitution-principle) case: a `Faculty` is fully substitutable anywhere a `Member`/`LibraryAccount` is expected — nothing overridden throws or contradicts the base contract, so subtyping is safe here (contrast with the `ReadOnlyAccount extends Account` anti-example in the SOLID doc). If a tier needed genuinely different *behavior* (e.g. Guests can't reserve at all), that's still fine as an overridden property returning `False`/no-op, not a thrown exception — the contract stays honest.

**Why reservation is a queue keyed per title, not per copy**: a member reserving "Clean Code" doesn't care which physical copy they get, only that *a* copy becomes available — so the hold queue lives on `Book` and hands off to whichever `BookItem` becomes `AVAILABLE` first.

## Design patterns applied

- [Strategy](../patterns/03-behavioral-patterns.md#strategy) — `FineCalculationStrategy` varies fine-per-day by member tier or book category; new fine schemes (e.g. capped fine, grace period) are new classes, not edits to a `calculate_fine` if-chain.
- [Observer](../patterns/03-behavioral-patterns.md#observer) — when a `BookItem` transitions to `AVAILABLE`, the `Book`'s reservation queue is notified so the head-of-queue member gets a hold notice; decouples inventory state changes from the notification mechanism.
- [Factory Method](../patterns/01-creational-patterns.md#factory-method) — `create_member(MemberType, ...)` centralizes which concrete `Member` subtype (and its loan-limit policy) to instantiate, so account-creation logic isn't duplicated at every call site that signs up a member.
- [Template Method](../patterns/03-behavioral-patterns.md#template-method) *(optional, if probed)* — a shared `checkout()` skeleton (validate limit → create loan → mark item loaned → log) with tier-specific hook steps, if tiers ever need extra checkout-time validation.

## Python implementation

This is the primary, fully-worked solution.

```python
from __future__ import annotations

import threading
from abc import ABC, abstractmethod
from collections import deque
from dataclasses import dataclass, field
from datetime import date, timedelta
from enum import Enum, auto
from typing import Callable, Optional, Protocol


class BookItemStatus(Enum):
    AVAILABLE = auto()
    LOANED = auto()
    RESERVED = auto()
    LOST = auto()


@dataclass
class Book:
    """A catalog entry — metadata only, and stateless: a Book never transitions between
    'available' and 'loaned,' its BookItems do. `_checkout_lock` guards the read-then-mutate
    sequence in checkout_first_available() below (see the concurrency note there); it's an
    underscored dataclass field rather than a method-local variable because the lock has to
    outlive any single call and be shared by every caller racing on this title.
    """

    isbn: str
    title: str
    author: str
    copies: list["BookItem"] = field(default_factory=list)
    _reservation_queue: deque["Member"] = field(default_factory=deque, repr=False)
    _checkout_lock: threading.Lock = field(default_factory=threading.Lock, repr=False)

    def add_copy(self, item: "BookItem") -> None:
        self.copies.append(item)

    def first_available_copy(self) -> Optional["BookItem"]:
        return next((c for c in self.copies if c.status is BookItemStatus.AVAILABLE), None)

    def checkout_first_available(self) -> Optional["BookItem"]:
        """Atomic read-then-mutate: finds an available copy and marks it loaned under the same
        lock acquisition, closing the race two members checking out the last copy simultaneously
        would otherwise hit (see Follow-up questions and ../04-concurrency-essentials.md).
        `first_available_copy()` above stays as the plain, lock-free read for callers that only
        want to *look*, e.g. rendering "3 of 5 copies available" in a catalog UI.
        """
        with self._checkout_lock:
            item = self.first_available_copy()
            if item is not None:
                item.mark_loaned()
            return item

    def enqueue_reservation(self, member: "Member") -> None:
        self._reservation_queue.append(member)

    def next_in_queue(self) -> Optional["Member"]:
        return self._reservation_queue.popleft() if self._reservation_queue else None
```

> **Pythonic idiom note:** `checkout_first_available()` folds "find the available copy" and "mark it loaned" into one locked method, rather than exposing `first_available_copy()` and `mark_loaned()` as two calls a `Library` composes — the same check-then-act trap as the seat-booking problem (see [07-movie-ticket-booking.md](07-movie-ticket-booking.md)) would otherwise reappear here. Any time you find yourself writing "check X, then based on the result, mutate X" across two public calls with a gap in between, that gap is exactly where a second thread gets in.

```python
AvailabilityListener = Callable[["Book", "BookItem"], None]


class BookItem:
    def __init__(self, barcode: str, book: Book) -> None:
        self.barcode = barcode
        self.book = book
        self.status = BookItemStatus.AVAILABLE
        self._listeners: list[AvailabilityListener] = []

    def add_availability_listener(self, listener: AvailabilityListener) -> None:
        self._listeners.append(listener)

    def mark_loaned(self) -> None:
        self.status = BookItemStatus.LOANED

    def mark_returned(self) -> None:
        self.status = BookItemStatus.AVAILABLE
        for listener in self._listeners:
            listener(self.book, self)

    def mark_lost(self) -> None:
        self.status = BookItemStatus.LOST


class FineCalculationStrategy(ABC):
    @abstractmethod
    def calculate_fine(self, loan: "Loan", return_date: date) -> float: ...


class FlatDailyFineStrategy(FineCalculationStrategy):
    def __init__(self, per_day_rate: float) -> None:
        self.per_day_rate = per_day_rate

    def calculate_fine(self, loan: "Loan", return_date: date) -> float:
        overdue_days = max(0, (return_date - loan.due_date).days)
        return overdue_days * self.per_day_rate


class CappedFineStrategy(FineCalculationStrategy):
    """Decorator over a base strategy — a fine cap is additive behavior, not a reason to fork
    FlatDailyFineStrategy into a near-duplicate class. Follow-up-question fodder made concrete."""

    def __init__(self, base: FineCalculationStrategy, cap: float) -> None:
        self.base = base
        self.cap = cap

    def calculate_fine(self, loan: "Loan", return_date: date) -> float:
        return min(self.base.calculate_fine(loan, return_date), self.cap)


class Member(ABC):
    """ABC, not Protocol: Member has real shared state (`active_loans`) and shared behavior
    (`can_checkout_more`) that every subclass inherits — this is exactly the "I own the
    hierarchy and want shared implementation plus an enforced contract" case ABC is for.
    Contrast with `LendingPolicy` further down (a follow-up extension point with no shared
    state), which is a better Protocol candidate.
    """

    def __init__(self, member_id: str) -> None:
        self.member_id = member_id
        self.active_loans: list["Loan"] = []

    @property
    @abstractmethod
    def checkout_limit(self) -> int: ...

    @property
    @abstractmethod
    def loan_duration_days(self) -> int: ...

    @property
    @abstractmethod
    def can_reserve(self) -> bool: ...

    def can_checkout_more(self) -> bool:
        return len(self.active_loans) < self.checkout_limit


class Student(Member):
    checkout_limit = 5
    loan_duration_days = 14
    can_reserve = True


class Faculty(Member):
    checkout_limit = 10
    loan_duration_days = 30
    can_reserve = True


class Guest(Member):
    checkout_limit = 2
    loan_duration_days = 7
    can_reserve = False  # opts out of the hold queue entirely — see Library.reserve()'s PermissionError


class MemberType(Enum):
    STUDENT = "student"
    FACULTY = "faculty"
    GUEST = "guest"


def create_member(member_type: MemberType, member_id: str) -> Member:
    return {
        MemberType.STUDENT: Student,
        MemberType.FACULTY: Faculty,
        MemberType.GUEST: Guest,
    }[member_type](member_id)


@dataclass(order=True)
class Loan:
    """`order=True` gives Loan a free `<`/`<=`/`>`/`>=` ordering by field order (item, member,
    due_date, ...) — enough to `sorted(loans)` by due date without hand-writing comparison
    dunders or reaching for functools.total_ordering. If you only need ordering by one field
    (due_date) rather than the whole tuple of fields, `sorted(loans, key=lambda l: l.due_date)`
    is more precise and doesn't require BookItem/Member to be orderable at all — prefer that
    unless you genuinely want dataclass-field-order semantics.
    """

    item: "BookItem"
    member: Member
    due_date: date
    returned_date: Optional[date] = None

    @property
    def is_active(self) -> bool:
        return self.returned_date is None


class Library:
    def __init__(self, fine_strategy: FineCalculationStrategy) -> None:
        self.fine_strategy = fine_strategy
        self._catalog: dict[str, Book] = {}

    def add_book(self, book: Book) -> None:
        self._catalog[book.isbn] = book

    def checkout(self, member: Member, isbn: str, today: date) -> Loan:
        if not member.can_checkout_more():
            raise RuntimeError(f"Checkout limit reached for {member.member_id}")

        book = self._catalog[isbn]
        item = book.checkout_first_available()
        if item is None:
            raise LookupError(f"No available copy of {isbn}")

        loan = Loan(item, member, today + timedelta(days=member.loan_duration_days))
        member.active_loans.append(loan)

        def on_available(b: Book, freed_item: BookItem) -> None:
            nxt = b.next_in_queue()
            if nxt:
                print(f"Notify {nxt.member_id}: '{b.title}' is available for pickup")

        item.add_availability_listener(on_available)
        return loan

    def return_book(self, loan: Loan, today: date) -> float:
        loan.returned_date = today
        loan.member.active_loans.remove(loan)
        loan.item.mark_returned()  # fires listener -> reservation hand-off
        return self.fine_strategy.calculate_fine(loan, today)

    def reserve(self, member: Member, isbn: str) -> None:
        if not member.can_reserve:
            raise PermissionError(f"{member.member_id} tier cannot reserve")
        self._catalog[isbn].enqueue_reservation(member)
```

> **Pythonic idiom note:** `Student`/`Faculty`/`Guest` assign `checkout_limit`, `loan_duration_days`, and `can_reserve` as plain class attributes rather than overriding `@property` methods with a `return` statement each — Python lets a class attribute satisfy an abstract `@property` declared on the base class as long as it resolves to the right value on instance access, so the subtypes read as pure data (`checkout_limit = 5`) instead of boilerplate one-line getters. This is a small but real case of Python's data model doing OOP-textbook work without OOP-textbook ceremony.

## Sample walkthrough

```python
library = Library(CappedFineStrategy(FlatDailyFineStrategy(per_day_rate=0.50), cap=20.0))

book = Book(isbn="978-0132350884", title="Clean Code", author="Robert C. Martin")
book.add_copy(BookItem("CC-001", book))
library.add_book(book)

alice = create_member(MemberType.STUDENT, "alice")
bob = create_member(MemberType.STUDENT, "bob")

loan = library.checkout(alice, "978-0132350884", today=date(2026, 6, 1))
library.reserve(bob, "978-0132350884")           # only copy is out, bob queues

fine = library.return_book(loan, today=date(2026, 6, 20))
# -> prints "Notify bob: 'Clean Code' is available for pickup"
print(f"Fine owed: ${fine:.2f}")                 # 5 days overdue * $0.50 = $2.50, well under the $20 cap
```

## Follow-up questions

- **"Now support multiple branches, each with its own copies."** Add a `Branch` entity that owns a subset of `BookItem`s for a `Book`; `first_available_copy()`/`checkout_first_available()` become branch-scoped (or branch-preferred with fallback). No change to `Member`, `Loan`, or the fine strategy — the change is additive at the `Book`↔`BookItem` join.
- **"What if two members try to check out the last copy at the same instant?"** This is exactly what `checkout_first_available()`'s `threading.Lock` closes: without it, both members' calls could evaluate `first_available_copy()` before either calls `mark_loaned()`, and both would walk away thinking they got the last copy. Guarding the find-and-mark sequence with one lock per `Book` (not one global lock) keeps unrelated titles from serializing against each other. See [../04-concurrency-essentials.md](../04-concurrency-essentials.md) for `Lock` vs `RLock` vs an optimistic version-number approach (the same trade-off discussed in depth in [07-movie-ticket-booking.md](07-movie-ticket-booking.md)).
- **"Add a hold expiration — if the notified member doesn't pick up within 48 hours, offer it to the next person."** Store a hold-expiry timestamp on the head of the queue when notified; a scheduled sweep (or lazy check on next queue access) pops expired holds and re-notifies. Purely additive to `Book`'s queue handling.
- **"Support e-books with no physical inventory / infinite copies."** Introduce a `LendingPolicy` Protocol: `checkout(book) -> BookItem | None`, satisfied by a `PhysicalLendingPolicy` (finite copies, uses `BookItem`) or a `DigitalLendingPolicy` (concurrent-license-capped or infinite). Protocol, not ABC, is the right call here — the two policies share no implementation, only a shape, and `Book` only ever needs to call the method, never `isinstance`-check the type. `Book` delegates availability checks to its policy instead of assuming `BookItem` always applies.
- **"Different fine caps per member tier (e.g. Faculty fines cap at $20)."** Already shown above: `CappedFineStrategy` wraps any base strategy; give each tier its own capped instance (or make the cap a constructor parameter looked up by tier) — no change to `Library.return_book`.

## Common mistakes on this problem

- Collapsing `Book` and `BookItem` into one class with a `copies_available: int` counter — this loses per-copy state (which specific copy is damaged/lost/loaned to whom) and is the single most common critique on this problem.
- Modeling member tiers as an `int discount_level` field or `type` enum checked via `if`/`elif` in `checkout()`, instead of letting `checkout_limit`/`loan_duration_days` be polymorphic — reintroduces the OCP violation the tier hierarchy exists to avoid.
- Putting the reservation queue on `BookItem` (the copy) instead of `Book` (the title) — a member reserving a title shouldn't care which physical barcode eventually satisfies it; queueing per-copy causes members to wait behind a specific copy while other copies of the same title sit available.
- Forgetting fines are computed at *return* time against the loan's due date, and instead trying to track "days late" incrementally while the loan is active — needless state; it's a pure function of `(due_date, return_date)`.

## Continue

Next: [09-logging-framework.md](09-logging-framework.md)
