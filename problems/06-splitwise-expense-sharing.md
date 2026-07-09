# Splitwise / Expense Sharing

## Requirements

- "Groups or just one-off expenses between individuals?" → **You decide**: support both — a group is just a named, persistent set of members that expenses can optionally reference; a one-off expense has no group, just an explicit participant list.
- "Split types?" → **You decide**: equal, exact amounts, and percentage — the three Splitwise actually ships, each validated differently (exact must sum to total; percentage must sum to 100).
- "Does the system need to compute a minimal set of settling transactions (min-cash-flow), or is a pairwise balance sheet enough?" → **You decide**: pairwise balance sheet (who-owes-whom, netted) is in scope and fully implemented; minimal-transaction settlement is a follow-up, discussed but not coded — it's a genuinely different algorithm (greedy max-heap matching), not a design-pattern question.
- "Multi-currency?" → **You decide**: out of scope for the core design — flagged as a follow-up since it changes the balance sheet's value type non-trivially.
- In scope: users, groups, expenses with pluggable split strategy, a running per-pair balance ledger, settling a debt (recording a payment that reduces balances), notifying involved users when an expense is added.
- Out of scope: multi-currency conversion, minimal-transaction optimization, recurring/scheduled expenses, real payment gateway integration.

## Core entities & relationships

```
User
  ├─ has-a[1] userId
  └─ observes[*] Expense            (implements ExpenseObserver)

Group
  └─ has-a[*] User                  (members)

Expense
  ├─ has-a[1] User                  (payer)
  ├─ has-a[*] User                  (participants)
  ├─ has-a[0..1] Group              (optional — null for one-off expenses)
  └─ uses-a[1] SplitStrategy (interface)

SplitStrategy (interface)
  ├─ implemented-by EqualSplit
  ├─ implemented-by ExactSplit
  └─ implemented-by PercentSplit

BalanceSheet
  └─ has-a[1] Map<User, Map<User, Money>>   (net amount owed, one direction only per pair)

ExpenseManager
  ├─ has-a[1] BalanceSheet
  └─ notifies[*] User (Observer)
```

`Group` doesn't own `Expense`s directly — `Expense` optionally references a `Group`. Modeling it the other way (`Group.expenses: list[Expense]`) would force every expense through a group, breaking the one-off case, and would make "move an expense between groups" a two-object update instead of one field. `BalanceSheet` stores only one direction per pair (`A owes B: $30`, never also `B owes A: $0`) — collapsing both directions into a single signed/netted entry at write-time is what makes "netting" (A owes B, B owes C ⇒ can simplify) tractable later without a separate reconciliation pass.

## Design patterns applied

- [Strategy](../patterns/03-behavioral-patterns.md#strategy) — `SplitStrategy` is the textbook case for this problem: equal/exact/percentage are three interchangeable algorithms for turning one total into per-participant shares, selected per-expense at creation time. This is the strongest [OCP](../02-solid-principles.md#o--openclosed-principle) example in the whole knowledge base — adding a fourth split type (e.g., "by shares/weights," like 2 shares for one roommate, 1 for another) is purely additive.
- [Observer](../patterns/03-behavioral-patterns.md#observer) — `ExpenseManager` notifies each involved `User` (or a notification service) when an expense is added, without `ExpenseManager` knowing whether that means push notification, email, or just an in-app feed update; decouples "an expense happened" from "what happens next," which varies per deployment and shouldn't require editing `ExpenseManager`.

## Implementation

`SplitStrategy` and `ExpenseObserver` are both modeled as `abc.ABC` rather than `typing.Protocol` here — deliberately, and worth saying why in an interview: both are meant to be *implemented by code you own and want to constrain* (every split type must validate its own inputs; every observer must handle the same notification event), and `ABC` gives you `@abstractmethod` enforcement at instantiation time (`TypeError` if a subclass forgets to implement it) plus a natural place to hang shared validation helpers later. `Protocol` would be the better call if you needed to accept pre-existing, unrelated classes (e.g., wiring in a third-party notification library's callback object) without forcing them through your inheritance tree — see [Python OOP Essentials §2](../03-python-oop-essentials.md#2-interfaces-typingprotocol-vs-abcabc--the-decision-interviewers-watch-for) for the general decision rule.

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from collections import defaultdict
from dataclasses import dataclass, field
from decimal import ROUND_DOWN, ROUND_HALF_UP, Decimal


@dataclass(frozen=True, slots=True)
class User:
    id: str
    name: str


@dataclass(slots=True)
class Group:
    id: str
    members: set[User] = field(default_factory=set)

    def add_member(self, user: User) -> None:
        self.members.add(user)


class SplitStrategy(ABC):
    """total -> per-participant shares. Validates its own inputs."""

    @abstractmethod
    def compute_shares(
        self, total: Decimal, participants: list[User], split_input: dict[User, Decimal]
    ) -> dict[User, Decimal]: ...


class EqualSplit(SplitStrategy):
    def compute_shares(self, total, participants, split_input):
        n = len(participants)
        share = (total / n).quantize(Decimal("0.01"), rounding=ROUND_DOWN)
        remainder = total - share * n
        shares: dict[User, Decimal] = {}
        for i, user in enumerate(participants):
            # dump the rounding remainder onto the first participant so shares sum exactly to total
            shares[user] = share + remainder if i == 0 else share
        return shares


class ExactSplit(SplitStrategy):
    def compute_shares(self, total, participants, split_input):
        total_given = sum(split_input.values(), Decimal(0))
        if total_given != total:
            raise ValueError(f"Exact amounts must sum to total: {total_given} != {total}")
        return dict(split_input)  # caller supplies User -> exact amount directly


class PercentSplit(SplitStrategy):
    def compute_shares(self, total, participants, split_input):
        pct_sum = sum(split_input.values(), Decimal(0))
        if pct_sum != Decimal(100):
            raise ValueError(f"Percentages must sum to 100, got {pct_sum}")
        return {
            user: (total * pct / Decimal(100)).quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)
            for user, pct in split_input.items()
        }


@dataclass
class Expense:
    id: str
    payer: User
    amount: Decimal
    participants: list[User]
    group: Group | None
    strategy: SplitStrategy
    split_input: dict[User, Decimal]
    shares: dict[User, Decimal] = field(init=False)  # computed in __post_init__, not passed by callers

    def __post_init__(self) -> None:
        self.shares = self.strategy.compute_shares(self.amount, self.participants, self.split_input)


class BalanceSheet:
    """balances[a][b] = amount `a` owes `b`, always >= 0; only one direction per pair is non-zero."""

    def __init__(self) -> None:
        self._balances: dict[User, dict[User, Decimal]] = defaultdict(lambda: defaultdict(lambda: Decimal(0)))

    def record_expense(self, expense: Expense) -> None:
        for debtor, share in expense.shares.items():
            if debtor != expense.payer:
                self.adjust(debtor, expense.payer, share)  # debtor owes payer this share

    def adjust(self, frm: User, to: User, amount: Decimal) -> None:
        """Records that `frm` owes `to` `amount` more, netting against any existing reverse debt."""
        reverse = self._balances[to][frm]
        if reverse >= amount:
            self._balances[to][frm] = reverse - amount  # reverse debt absorbs it entirely
        else:
            self._balances[to][frm] = Decimal(0)
            self._balances[frm][to] += amount - reverse

    def settle(self, payer: User, payee: User, amount: Decimal) -> None:
        """Records a payment settling (part of) a debt."""
        self.adjust(payee, payer, amount)  # payee "owing" payer decreases -> adjust in reverse

    def get(self, frm: User, to: User) -> Decimal:
        return self._balances[frm][to]


class ExpenseObserver(ABC):
    @abstractmethod
    def on_expense_added(self, expense: Expense) -> None: ...


class ExpenseManager:
    def __init__(self) -> None:
        self.balance_sheet = BalanceSheet()
        self._observers: list[ExpenseObserver] = []

    def add_observer(self, observer: ExpenseObserver) -> None:
        self._observers.append(observer)

    def add_expense(
        self,
        id: str,
        payer: User,
        amount: Decimal,
        participants: list[User],
        group: Group | None,
        strategy: SplitStrategy,
        split_input: dict[User, Decimal],
    ) -> Expense:
        expense = Expense(id, payer, amount, participants, group, strategy, split_input)
        self.balance_sheet.record_expense(expense)
        for observer in self._observers:
            observer.on_expense_added(expense)
        return expense
```

> **Pythonic idiom note:** `BalanceSheet` uses a nested `defaultdict(lambda: defaultdict(lambda: Decimal(0)))` so `get`/`adjust` never need an explicit "does this pair exist yet" branch — any never-seen `(from, to)` pair silently reads as `Decimal(0)`, which is exactly the ledger's zero-state semantics. The trade-off worth naming: `defaultdict` auto-vivifies entries on *read*, not just write (`self._balances[to][frm]` inside `get` creates a zero-value entry as a side effect of merely checking it), which can quietly grow the dict from read-only traffic. For this problem that's harmless (a zero entry is indistinguishable from a missing one), but it's the kind of thing worth flagging as a conscious choice rather than an accident if an interviewer asks about memory growth over a long-lived process. `User` as a `frozen=True` dataclass gets `__eq__`/`__hash__` for free (needed to use `User` as a dict key), replacing the boilerplate `equals()`/`hashCode()` override pair you'd otherwise hand-write.

## Sample walkthrough

```python
alice, bob, carol = User("u1", "Alice"), User("u2", "Bob"), User("u3", "Carol")
trip = Group("g1", members={alice, bob, carol})

manager = ExpenseManager()

# Alice pays $90 dinner, split equally three ways -> Bob and Carol each owe Alice $30
manager.add_expense("e1", payer=alice, amount=Decimal("90"),
                     participants=[alice, bob, carol], group=trip,
                     strategy=EqualSplit(), split_input={})

assert manager.balance_sheet.get(bob, alice) == Decimal("30.00")
assert manager.balance_sheet.get(carol, alice) == Decimal("30.00")

# Bob pays Carol back for something unrelated -> $30, one-off, no group
manager.add_expense("e2", payer=carol, amount=Decimal("30"),
                     participants=[bob], group=None,
                     strategy=ExactSplit(), split_input={bob: Decimal("30")})

# Bob now owes Alice 30 from e1, and now also owes Carol 30 from e2 -> two independent pairs,
# adjust() nets automatically within each pair; nothing here cancels across different counterparties.
assert manager.balance_sheet.get(bob, carol) == Decimal("30.00")

# Bob pays Alice back in full
manager.balance_sheet.settle(payer=bob, payee=alice, amount=Decimal("30"))
assert manager.balance_sheet.get(bob, alice) == Decimal("0")
```

## Follow-up questions

- **"Add a new split type — by shares/weights (e.g., 2:1:1 ratio)."** Add a `SharesSplit(SplitStrategy)` subclass that divides `total` proportional to `split_input` weights — zero changes to `Expense`, `BalanceSheet`, or `ExpenseManager`; this is the OCP payoff stated up front.
- **"Support group expenses vs one-off expenses differently — e.g., only group members can be participants in a group expense."** Validate `set(participants) <= group.members` inside `add_expense` when `group is not None`; the model already supports it since `Expense.group` is optional and `participants` is independent of it.
- **"Add multi-currency support."** `BalanceSheet` values become a `Money` type (amount + currency code) instead of a raw `Decimal`; `adjust`/`get` need a currency-conversion step (inject a `CurrencyConverter` strategy) before netting across currencies — this is a real design change, not free, and worth calling out as such rather than hand-waving it.
- **"Given the current pairwise balances, compute the minimum number of transactions to settle everyone."** This is the classic min-cash-flow problem: separate net creditors/debtors into two max-heaps (by net amount, summing each user's balances across all counterparties into one net figure first) and greedily match the largest creditor against the largest debtor. It's an algorithmic add-on that reads from `BalanceSheet`'s net-per-user view — doesn't require restructuring the ledger, just an additional `DebtSimplifier.simplify(balance_sheet) -> list[Transaction]` utility, and `heapq` (negated for a max-heap, since Python's `heapq` is min-heap only) is the natural tool.
- **"Notify a user's phone (push) vs email based on their preference."** `ExpenseObserver` implementations are swappable per user (or a `NotificationDispatcher` that picks a channel per user) — this is exactly the Observer decoupling already in place; no change to `ExpenseManager`.

## Common mistakes on this problem

- Storing balances as a plain `total_paid - total_owed` scalar per user instead of a pairwise ledger — loses "who owes whom" entirely, which is the actual point of Splitwise (a global net-zero total tells you nothing about who to pay).
- Putting split-calculation logic as `if split_type == "EQUAL": ... elif ...` inside `Expense` or `ExpenseManager` instead of behind `SplitStrategy` — the single most common OCP violation on this problem, and the first thing interviewers probe with "now add a new split type."
- Forgetting rounding: equal split of an odd total (e.g., $100 / 3) must not silently lose or gain a cent — shares must sum to exactly the input total.
- Conflating "settling a debt" with "adding an expense" as the same code path — a payment is not an expense with a split, it's a direct balance adjustment; merging them tends to produce awkward negative-share hacks.

## Continue

Next: [07-movie-ticket-booking.md](07-movie-ticket-booking.md)
