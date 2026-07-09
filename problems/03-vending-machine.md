# Vending Machine

The cleanest possible showcase for [State](../patterns/03-behavioral-patterns.md#state): four states, each with obviously different behavior for the same two triggering actions (select item, insert money), and a transition graph you can draw in 30 seconds. If you can't explain State cleanly on this problem, revisit it before the interview — every other State-pattern problem is a variation on this shape.

## Requirements

- "What can go wrong that the machine must handle gracefully?" → Invalid selection, insufficient funds, item out of stock, and the machine being unable to make exact change — **you decide** to model all four explicitly rather than hand-wave "assume happy path," since this is precisely what the State machine is for.
- "Cash only, or cards too?" → Cash (coins + notes) for the core design; **you decide** to model payment as accepted denominations for now and flag card payment as a `PaymentMethod` Strategy extension in follow-ups, not core scope.
- "Multiple items per slot, or one-of-each?" → Each slot holds a quantity of one `Item`; multiple slots can hold the same item code (real machines do this) — out of scope to optimize slot selection, just decrement the first slot with stock.
- "Does the machine ever fully run out and stop accepting money?" → Yes — machine-wide `OutOfStock` (all slots empty) is a state, not just a per-item error, per the problem's own naming.
- "Refunds — full transaction cancel only, or partial?" → **You decide**: full refund of inserted money only, no partial dispense-and-partial-refund logic.
- "Restocking — who does it and when?" → An admin-only `restock()` operation that isn't part of the customer-facing state transitions (modeled as a direct call on `VendingMachine`, not a `VendingMachineState` method) — out of scope to model an admin role/auth.

**In scope:** item selection, money insertion (cumulative, can under-pay across multiple inserts), exact-change dispensing via a pluggable algorithm, out-of-stock detection at both item and machine level, refund.

**Out of scope:** card/UPI payment integration, multi-currency, partial dispense, admin authentication, inventory replenishment scheduling.

## Core entities & relationships

```
VendingMachine
  ├─ has-a[1] VendingMachineState (interface)
  ├─ has-a[*] Slot
  └─ has-a[1] ChangeCalculator (interface)

Slot
  └─ has-a[1] Item + quantity
```

`VendingMachineState` is the textbook GoF State: `Idle`, `HasMoney`, `Dispensing`, `OutOfStock` each react *differently* to the same two public triggers (`select_item`, `insert_money`) — `Idle` requires a selection before accepting money, `HasMoney` accumulates and auto-transitions once sufficient, `Dispensing` is a transient state that rejects new input mid-transaction, `OutOfStock` rejects everything until an out-of-band `restock()`. Contrast with the parking lot's `VehicleType`, which is an enum precisely *because* it has no transition logic — here the transition logic is the entire point.

`Slot` is composition (a `VendingMachine` without slots is meaningless and slots don't outlive the machine); `Item` is a plain immutable value object (code/name/price) referenced by possibly-multiple slots, so it's aggregation from the slot's side.

## Design patterns applied

- [State](../patterns/03-behavioral-patterns.md#state) — the primary lesson here: `Idle → HasMoney → Dispensing → (Idle | OutOfStock)` as explicit classes means "dispense while no money inserted" or "accept money while dispensing" are structurally unreachable, not bugs you're hoping your `if` chain prevents.
- [Strategy](../patterns/03-behavioral-patterns.md#strategy) — `ChangeCalculator` isolates the change-making algorithm (greedy largest-denomination-first today) from the state machine; swapping in an algorithm that accounts for scarce denominations (see follow-ups) touches one class.

## Implementation

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass
from enum import IntEnum


class Denomination(IntEnum):
    """IntEnum so `.value` participates directly in cents arithmetic without a `.value` lookup everywhere."""
    PENNY = 1
    NICKEL = 5
    DIME = 10
    QUARTER = 25
    ONE_DOLLAR = 100
    FIVE_DOLLAR = 500


@dataclass(frozen=True, slots=True)
class Item:
    code: str
    name: str
    price_cents: int


class Slot:
    def __init__(self, item: Item, quantity: int) -> None:
        self.item = item
        self.quantity = quantity

    def in_stock(self) -> bool:
        return self.quantity > 0

    def decrement(self) -> None:
        self.quantity -= 1

    def restock(self, count: int) -> None:
        self.quantity += count


class ChangeCalculator(ABC):
    @abstractmethod
    def make_change(
        self, amount_cents: int, available: dict[Denomination, int]
    ) -> dict[Denomination, int]: ...


class GreedyChangeCalculator(ChangeCalculator):
    """Largest-denomination-first. Fails loudly (rather than shorting the customer) when exact
    change isn't achievable with current cash inventory — see Follow-up questions."""

    def make_change(
        self, amount_cents: int, available: dict[Denomination, int]
    ) -> dict[Denomination, int]:
        result: dict[Denomination, int] = {}
        remaining = amount_cents
        for d in sorted(Denomination, key=lambda x: -x.value):
            have = available.get(d, 0)
            use = min(have, remaining // d.value)
            if use:
                result[d] = use
                remaining -= use * d.value
        if remaining != 0:
            raise RuntimeError("Cannot make exact change")
        return result


class VendingMachineState(ABC):
    @abstractmethod
    def select_item(self, machine: "VendingMachine", code: str) -> None: ...

    @abstractmethod
    def insert_money(self, machine: "VendingMachine", denom: Denomination | None) -> None: ...

    @abstractmethod
    def refund(self, machine: "VendingMachine") -> None: ...

    def on_enter(self, machine: "VendingMachine") -> None:
        """Hook run immediately after transitioning into this state. Default no-op; states that
        need to act the instant they're entered (Dispensing) override it. See the design-wart
        note below `IdleState.insert_money` for why every state having this hook is the clean fix."""


class IdleState(VendingMachineState):
    def select_item(self, machine: "VendingMachine", code: str) -> None:
        slot = machine.find_slot(code)
        if not slot or not slot.in_stock():
            raise ValueError(f"Unavailable: {code}")
        machine.selected_slot = slot

    def insert_money(self, machine: "VendingMachine", denom: Denomination | None) -> None:
        if machine.selected_slot is None:
            raise RuntimeError("Select an item first")
        assert denom is not None
        machine.add_balance(denom)
        machine.transition_to(HasMoneyState())

    def refund(self, machine: "VendingMachine") -> None:
        pass  # nothing inserted yet


class HasMoneyState(VendingMachineState):
    def select_item(self, machine: "VendingMachine", code: str) -> None:
        raise RuntimeError("Finish or cancel current transaction first")

    def insert_money(self, machine: "VendingMachine", denom: Denomination | None) -> None:
        if denom is not None:
            machine.add_balance(denom)
        self._check_sufficiency(machine)

    def on_enter(self, machine: "VendingMachine") -> None:
        self._check_sufficiency(machine)

    def _check_sufficiency(self, machine: "VendingMachine") -> None:
        assert machine.selected_slot is not None
        if machine.balance_cents >= machine.selected_slot.item.price_cents:
            machine.transition_to(DispensingState())

    def refund(self, machine: "VendingMachine") -> None:
        machine.refund_balance()
        machine.reset()
        machine.transition_to(IdleState())


class DispensingState(VendingMachineState):
    def on_enter(self, machine: "VendingMachine") -> None:
        slot = machine.selected_slot
        assert slot is not None
        slot.decrement()
        change_due = machine.balance_cents - slot.item.price_cents
        change_given = machine.change_calculator.make_change(change_due, machine.cash_inventory)
        machine.dispense_item(slot.item, change_given)
        machine.reset()
        machine.transition_to(IdleState() if machine.has_any_stock() else OutOfStockState())

    def select_item(self, machine: "VendingMachine", code: str) -> None:
        raise RuntimeError("Dispensing in progress")

    def insert_money(self, machine: "VendingMachine", denom: Denomination | None) -> None:
        raise RuntimeError("Dispensing in progress")

    def refund(self, machine: "VendingMachine") -> None:
        raise RuntimeError("Dispensing in progress")


class OutOfStockState(VendingMachineState):
    def select_item(self, machine: "VendingMachine", code: str) -> None:
        raise RuntimeError("Machine out of stock")

    def insert_money(self, machine: "VendingMachine", denom: Denomination | None) -> None:
        raise RuntimeError("Machine out of stock")

    def refund(self, machine: "VendingMachine") -> None:
        pass  # no balance can exist in this state


class VendingMachine:
    def __init__(self, initial_slots: list[Slot], change_calculator: ChangeCalculator) -> None:
        self.slots: dict[str, Slot] = {s.item.code: s for s in initial_slots}
        self.cash_inventory: dict[Denomination, int] = {}
        self.change_calculator = change_calculator
        self.state: VendingMachineState = IdleState()
        self.selected_slot: Slot | None = None
        self.balance_cents = 0

    def select_item(self, code: str) -> None:
        self.state.select_item(self, code)

    def insert_money(self, denom: Denomination) -> None:
        self.state.insert_money(self, denom)

    def refund(self) -> None:
        self.state.refund(self)

    def restock(self, code: str, count: int) -> None:
        self.slots[code].restock(count)
        if isinstance(self.state, OutOfStockState):
            self.transition_to(IdleState())

    def transition_to(self, new_state: VendingMachineState) -> None:
        self.state = new_state
        new_state.on_enter(self)

    def find_slot(self, code: str) -> Slot | None:
        return self.slots.get(code)

    def add_balance(self, denom: Denomination) -> None:
        self.balance_cents += denom.value
        self.cash_inventory[denom] = self.cash_inventory.get(denom, 0) + 1

    def refund_balance(self) -> None:
        pass  # symmetric to dispense_item's cash-drawer bookkeeping — omitted for brevity

    def reset(self) -> None:
        self.selected_slot = None
        self.balance_cents = 0

    def has_any_stock(self) -> bool:
        return any(s.in_stock() for s in self.slots.values())

    def dispense_item(self, item: Item, change: dict[Denomination, int]) -> None:
        print(f"Dispensing {item.name}, change: {change}")
```

**Design wart worth naming out loud:** `IdleState.insert_money` transitions to `HasMoneyState` and relies on `VendingMachine.transition_to` calling the new state's `on_enter` hook to immediately re-check sufficiency — so a single sufficient payment doesn't need a second call. This is *why* every state gets an `on_enter(machine)` hook (not just `DispensingState`, which obviously needs one): centralizing "re-check after I just changed state" in one place (`transition_to`) instead of special-casing it inside `insert_money` is the fix for what would otherwise be an awkward manual re-dispatch. Naming this design decision out loud — "I gave every state an `on_enter` hook so transition-time side effects live in one place" — is exactly the kind of thing that separates a clean State implementation from a merely-working one in an interview.

**Pythonic idiom note:** `Denomination` is an `IntEnum` rather than a plain `Enum` specifically so `denom.value` participates in arithmetic (`remaining // d.value`) without friction — a plain `Enum` would require `.value` everywhere something numeric is expected, while `IntEnum` members *are* ints for all practical purposes (comparable, hashable, usable as dict keys and in arithmetic) while still being a named, type-checked enumeration. This is the standard-library answer to "give me a named constant that also behaves like its underlying primitive."

## Sample walkthrough

```python
cola = Item("A1", "Cola", 125)
machine = VendingMachine([Slot(cola, quantity=2)], GreedyChangeCalculator())

machine.select_item("A1")                       # Idle.select_item -> selected_slot = cola slot
machine.insert_money(Denomination.ONE_DOLLAR)   # Idle -> HasMoney, balance = 100, not enough (125 needed)
machine.insert_money(Denomination.QUARTER)      # HasMoney: balance = 125 == price -> Dispensing.on_enter fires
                                                 # -> dispenses Cola, change = {} (exact), state -> Idle (1 left in stock)
```

## Follow-up questions

- **"What if the machine can't make exact change?"** `GreedyChangeCalculator.make_change` already raises when `remaining != 0`; catch that in `DispensingState.on_enter` and route to a refund-and-abort transition instead of a dispense (back to `HasMoneyState` with the money still held, or straight to `IdleState` after auto-refund) — the fix is a `try/except` at the one call site, the state machine shape doesn't change.
- **"Add card payment alongside cash."** Introduce a `PaymentMethod` interface (cash today, card tomorrow) that `HasMoneyState` delegates to for "is this transaction fully paid" instead of comparing `balance_cents` directly — this is the same DIP move as the parking lot's `PaymentMethod`, and doesn't touch the State hierarchy at all since payment method is orthogonal to transaction state.
- **"Two customers use the machine at the same 'moment' (multi-machine fleet with shared inventory backend)?"** A single physical machine is inherently single-transaction — the real question is usually about a shared backend tracking inventory across many machines, which is a different service boundary; within one `VendingMachine`, guard `state`/`selected_slot`/`balance_cents` mutations with a `threading.Lock` per [../04-concurrency-essentials.md](../04-concurrency-essentials.md) if the same machine is somehow driven by two threads (e.g., a touchscreen thread and a coin-sensor interrupt thread).
- **"Support combo deals (buy 2 get 1 free) or discounts?"** Add a `PricingPolicy`/discount Strategy consulted in `HasMoneyState._check_sufficiency` before comparing balance to price — again additive, doesn't touch `IdleState`/`DispensingState`/`OutOfStockState`.
- **"What if a single slot runs out mid-selection but others still have stock?"** Already handled: `OutOfStockState` is machine-wide (`has_any_stock()` across *all* slots), while a single empty slot is just rejected at `select_item` time (`slot.in_stock()` check in `IdleState`) — the two failure modes are deliberately different states/checks, worth narrating why.

## Common mistakes on this problem

- Modeling machine status as a `status: str` field with values like `"idle"`/`"has_money"` checked via `if status == "idle":` chains — loses the compiler/type-checker's ability to catch a typo'd status string, and is the exact anti-pattern State replaces.
- Letting `DispensingState` be a public, externally-triggerable state (e.g., a public `dispense()` method callers invoke directly) instead of an internal transition — callers should never need to know dispensing is a distinct state; it's an implementation detail of the transaction.
- Conflating "this item's slot is empty" with "the machine is out of stock" — collapsing both into one `OutOfStock` state produces wrong behavior (rejecting selection of an item that's still available in another slot).
- Doing change calculation inline inside the state class instead of extracting `ChangeCalculator` — works fine until the interviewer asks "what if we run out of quarters," and now you're refactoring under time pressure instead of swapping a strategy.

## Continue

Next: [04-tic-tac-toe-and-chess.md](04-tic-tac-toe-and-chess.md)
