# Tic-Tac-Toe and Chess

Both problems live in one file because they teach the same lesson from opposite ends: Tic-Tac-Toe is "how do you generalize a board so win-checking isn't hardcoded to 3×3," and Chess is "how do you model per-piece behavior without a Strategy explosion when polymorphism already does the job." Read them together.

## Requirements

### Tic-Tac-Toe

- "Fixed 3×3, or should the design generalize?" → **You decide**: generalize to N×N with a "get K in a row to win" rule — this is a near-guaranteed follow-up, so design for it up front rather than retrofitting.
- "How many players?" → 2, alternating turns. Out of scope: >2 players, AI opponent (mention Minimax as a follow-up talking point, don't implement it).
- "Draw detection?" → Yes — board full with no winner.

**In scope:** N×N board, alternating turns, win detection in O(1) per move (not a full-board rescan), draw detection.
**Out of scope:** AI opponent, network multiplayer, undo/redo (contrast with Chess, where undo is a natural follow-up).

### Chess

- "Full chess rules?" → **No — explicitly out of scope.** Castling, en passant, pawn promotion, threefold repetition, and full checkmate/stalemate detection are called out as follow-up talking points, not implemented. Say this out loud before writing code so the interviewer scopes with you instead of expecting FIDE-complete rules in 20 minutes.
- "What *is* in scope?" → Board representation, a `Piece` hierarchy with correct per-piece pseudo-legal move generation (Rook, Knight, Pawn, King shown; Bishop/Queen are the same sliding-piece pattern as Rook), move execution, and basic check detection (is a king's square currently attacked).
- "Move history / undo?" → Not core scope, but the design should visibly support it — see [Command](../patterns/03-behavioral-patterns.md#command) in Design patterns below.

**In scope:** 8×8 board, piece movement rules (pseudo-legal + "does this move leave your own king in check" filtering), turn alternation, check detection.
**Out of scope:** castling, en passant, promotion, checkmate/stalemate detection, draw-by-repetition, time controls, full legal-move enumeration for an engine.

## Core entities & relationships

### Tic-Tac-Toe

```
TicTacToeGame
  ├─ has-a[1] Board
  └─ has-a[*] Player

Board
  └─ has-a[*] Cell (Mark enum: EMPTY, X, O)
```

Board size lives as a constructor parameter (`size`, `win_length`), not a hardcoded `3`. `Mark` is a plain `Enum` with no behavior beyond a `__str__` for pretty-printing — X/O don't do anything differently from each other, so there's no polymorphism to reach for here at all, unlike Chess's pieces.

### Chess

```
ChessGame
  ├─ has-a[1] Board
  └─ has-a[*] Player (has-a[1] Color)

Board
  └─ has-a[*] Piece (abstract)

Piece (abstract, shared state: Color, Position)
  ├─ Rook
  ├─ Knight
  ├─ Pawn
  └─ King
```

`Piece` is an `abc.ABC`, not a `typing.Protocol` — every piece shares real state (`color`, `position`) and shared default behavior (bounds-checking, "is this square occupied by an enemy" helpers) that concrete subclasses inherit rather than reimplement. That's exactly the decision rule in [Python OOP Essentials §2](../03-python-oop-essentials.md#2-interfaces-typingprotocol-vs-abcabc--the-decision-interviewers-watch-for): reach for `ABC` when subtypes share state and some default logic; reach for `Protocol` when you're only promising a call signature and want structural typing with no shared implementation and no inheritance requirement (e.g., "anything with a `get_valid_moves(board) -> list[Position]` method," useful if you wanted third-party piece classes to plug in without inheriting from your `Piece`). A `PieceType` enum would fail here the way it succeeded for the parking lot's `VehicleType` — piece *behavior* (how a Rook vs a Knight generates moves) genuinely differs, it isn't just a data lookup.

## Design patterns applied

- **Polymorphism, not Strategy, for `get_valid_moves()`** — worth stating this distinction explicitly in an interview: each `Piece` subclass overriding `get_valid_moves()` is ordinary polymorphism (the behavior is intrinsic to *being* a Rook, never swapped at runtime), whereas [Strategy](../patterns/03-behavioral-patterns.md#strategy) implies an *injectable, interchangeable* algorithm independent of the object's identity (like `PricingStrategy` in the parking lot problem, which any vehicle could in principle be charged under). Calling this "Strategy" in an interview is a common imprecision — say "this is polymorphism; I'd reach for Strategy if move rules needed to be swappable independent of piece identity, e.g. a chess-variant mode with custom piece rules," which shows you know the difference.
- [Command](../patterns/03-behavioral-patterns.md#command) — *not implemented in the core design, mentioned as the right extension point*: wrapping each executed move as a `MoveCommand` (with `execute()`/`undo()` and a reference to any captured piece) is how you'd add move-history/undo without polluting `Board` or `Piece` with history-tracking concerns. See Follow-ups.
- Composite is deliberately **not** used here — a chess board is a flat grid of independent pieces, not a part-whole tree where operations recurse (there's no "move this piece and everything nested inside it"); forcing Composite onto a board is a common overuse mistake worth naming and rejecting out loud.

## Implementation

### Tic-Tac-Toe

```python
from __future__ import annotations

from collections import deque
from dataclasses import dataclass
from enum import Enum


class Mark(Enum):
    EMPTY = 0
    X = 1
    O = 2

    def __str__(self) -> str:  # dunder used for board pretty-printing, no lookup table needed at call sites
        return {Mark.EMPTY: ".", Mark.X: "X", Mark.O: "O"}[self]


@dataclass(frozen=True, slots=True)
class Player:
    name: str
    mark: Mark


class Board:
    """N x N grid of Marks. Knows nothing about turns or win conditions — single responsibility."""

    def __init__(self, size: int) -> None:
        self.size = size
        self._grid: list[list[Mark]] = [[Mark.EMPTY] * size for _ in range(size)]

    def place_mark(self, row: int, col: int, mark: Mark) -> bool:
        if self._grid[row][col] != Mark.EMPTY:
            return False
        self._grid[row][col] = mark
        return True

    def at(self, row: int, col: int) -> Mark:
        return self._grid[row][col]

    def __str__(self) -> str:
        return "\n".join(" ".join(str(cell) for cell in row) for row in self._grid)


class TicTacToeGame:
    """O(1)-per-move win detection via running signed sums per row/col/diagonal —
    generalizes to any (size, win_length) instead of hardcoding a 3x3 scan."""

    def __init__(self, size: int, win_length: int, players: list[Player]) -> None:
        self.board = Board(size)
        self.win_length = win_length
        self._turn_order: deque[Player] = deque(players)
        self._row_count = [0] * size
        self._col_count = [0] * size
        self._diag_count = 0
        self._anti_diag_count = 0
        self._moves_played = 0

    @property
    def current_player(self) -> Player:
        return self._turn_order[0]

    def play(self, row: int, col: int) -> Player | None:
        current = self.current_player
        if not self.board.place_mark(row, col, current.mark):
            raise ValueError(f"Cell ({row}, {col}) is already occupied")

        delta = 1 if current.mark == Mark.X else -1  # signed running sum: |count| == win_length means a sweep
        size = self.board.size
        self._row_count[row] += delta
        self._col_count[col] += delta
        if row == col:
            self._diag_count += delta
        if row + col == size - 1:
            self._anti_diag_count += delta
        self._moves_played += 1

        won = (
            abs(self._row_count[row]) == self.win_length
            or abs(self._col_count[col]) == self.win_length
            or abs(self._diag_count) == self.win_length
            or abs(self._anti_diag_count) == self.win_length
        )
        if won:
            return current
        self._turn_order.rotate(-1)  # O(1) turn rotation; no manual pop-then-append bookkeeping
        return None

    def is_draw(self) -> bool:
        return self._moves_played == self.board.size ** 2
```

> **Pythonic idiom note:** `deque.rotate(-1)` replaces the classic "dequeue current player, enqueue at the back" two-step with one O(1) call — `collections.deque` is designed exactly for this rotate-the-queue access pattern, whereas a `list` would force an O(n) `pop(0)`. The generalized win-check (running sums keyed only by final `(row, col)`, not by *how* a mark landed there) is the real payoff and is language-agnostic — it's what makes the Connect-4 follow-up below a non-event.

The running-sum trick generalizes to any `size`/`win_length` (e.g. Connect-4-style "4 in a row on a bigger board") without touching the win-check logic — it's the concrete payoff of not hardcoding a 3×3 scan.

### Chess

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass
from enum import Enum


class Color(Enum):
    WHITE = 1
    BLACK = 2

    @property
    def opponent(self) -> "Color":
        return Color.BLACK if self is Color.WHITE else Color.WHITE


@dataclass(frozen=True, slots=True)
class Position:
    row: int
    col: int

    def in_bounds(self) -> bool:
        return 0 <= self.row < 8 and 0 <= self.col < 8

    def __str__(self) -> str:  # algebraic notation (e.g. "e4") for logging/board printing
        return f"{chr(ord('a') + self.col)}{self.row + 1}"


class Piece(ABC):
    """abc.ABC, not typing.Protocol: every piece shares real state (color, position) and a default
    method (`can_land_on`) inherited rather than reimplemented per subclass. Protocol would be the
    right call only if pieces had no shared state/behavior at all — just a promised
    `get_valid_moves` signature implemented independently by otherwise-unrelated classes."""

    symbol: str = "?"  # overridden per subclass; used only for board printing, not game logic

    def __init__(self, color: Color, position: Position) -> None:
        self.color = color
        self.position = position

    @abstractmethod
    def get_valid_moves(self, board: "Board") -> list[Position]:
        """Pseudo-legal moves only — ChessGame filters out moves that leave your own king in check."""

    def can_land_on(self, board: "Board", position: Position) -> bool:
        if not position.in_bounds():
            return False
        occupant = board.piece_at(position)
        return occupant is None or occupant.color != self.color

    def __repr__(self) -> str:
        return f"{self.color.name[0]}{self.symbol}@{self.position}"


class Rook(Piece):
    symbol = "R"
    _DIRECTIONS = ((1, 0), (-1, 0), (0, 1), (0, -1))

    def get_valid_moves(self, board: "Board") -> list[Position]:
        moves: list[Position] = []
        for dr, dc in self._DIRECTIONS:
            r, c = self.position.row + dr, self.position.col + dc
            candidate = Position(r, c)
            while self.can_land_on(board, candidate):
                moves.append(candidate)
                if board.piece_at(candidate) is not None:
                    break  # capture square is the last reachable step in this direction
                r += dr
                c += dc
                candidate = Position(r, c)
        return moves


class Knight(Piece):
    symbol = "N"
    _DELTAS = ((1, 2), (2, 1), (-1, 2), (-2, 1), (1, -2), (2, -1), (-1, -2), (-2, -1))

    def get_valid_moves(self, board: "Board") -> list[Position]:
        candidates = (Position(self.position.row + dr, self.position.col + dc) for dr, dc in self._DELTAS)
        return [p for p in candidates if self.can_land_on(board, p)]


class Pawn(Piece):
    symbol = "P"

    def get_valid_moves(self, board: "Board") -> list[Position]:
        moves: list[Position] = []
        direction = 1 if self.color == Color.WHITE else -1  # no double-step/en-passant/promotion — see Out of scope
        forward = Position(self.position.row + direction, self.position.col)
        if forward.in_bounds() and board.piece_at(forward) is None:
            moves.append(forward)
        for dc in (-1, 1):
            diagonal = Position(self.position.row + direction, self.position.col + dc)
            occupant = board.piece_at(diagonal) if diagonal.in_bounds() else None
            if occupant is not None and occupant.color != self.color:
                moves.append(diagonal)
        return moves


class King(Piece):
    symbol = "K"

    def get_valid_moves(self, board: "Board") -> list[Position]:  # no castling
        moves: list[Position] = []
        for dr in (-1, 0, 1):
            for dc in (-1, 0, 1):
                if dr == 0 and dc == 0:
                    continue
                candidate = Position(self.position.row + dr, self.position.col + dc)
                if self.can_land_on(board, candidate):
                    moves.append(candidate)
        return moves


class Board:
    def __init__(self) -> None:
        self._occupied: dict[Position, Piece] = {}

    def place(self, piece: Piece) -> None:
        self._occupied[piece.position] = piece

    def piece_at(self, position: Position) -> Piece | None:
        return self._occupied.get(position)

    def move(self, piece: Piece, to: Position) -> None:
        del self._occupied[piece.position]
        self._occupied[to] = piece
        piece.position = to

    def remove(self, position: Position) -> None:
        self._occupied.pop(position, None)

    def pieces_of(self, color: Color) -> list[Piece]:
        return [p for p in self._occupied.values() if p.color == color]

    def __str__(self) -> str:  # 8x8 ASCII rendering, rank 8 at top like a real board
        rows = []
        for row in range(7, -1, -1):
            cells = [
                "." if (piece := self.piece_at(Position(row, col))) is None else f"{piece.color.name[0]}{piece.symbol}"
                for col in range(8)
            ]
            rows.append(" ".join(cells))
        return "\n".join(rows)


class ChessGame:
    def __init__(self) -> None:
        self.board = Board()
        self.turn = Color.WHITE

    def is_king_in_check(self, king_color: Color) -> bool:
        king_position = next(p.position for p in self.board.pieces_of(king_color) if isinstance(p, King))
        enemy_pieces = self.board.pieces_of(king_color.opponent)
        return any(king_position in p.get_valid_moves(self.board) for p in enemy_pieces)

    def make_move(self, piece: Piece, to: Position) -> bool:
        if piece.color != self.turn:
            raise RuntimeError("Not your turn")
        if to not in piece.get_valid_moves(self.board):
            return False

        captured = self.board.piece_at(to)
        origin = piece.position
        self.board.move(piece, to)  # tentative

        if self.is_king_in_check(self.turn):  # illegal: exposes own king — revert
            self.board.move(piece, origin)
            if captured is not None:
                self.board.place(captured)
            return False

        self.turn = self.turn.opponent
        return True
```

> **Pythonic idiom note:** `Color.opponent` as an `Enum` property replaces the ternary `WHITE if color == BLACK else BLACK` that would otherwise be duplicated at every flip site (`ChessGame.make_move`, `is_king_in_check`, and any future check/checkmate code) — putting the flip on the enum itself is a small but real teaching point about giving enums behavior (`Enum` members are first-class objects; nothing stops them from having `@property`s). `Piece.__repr__` and `Board.__str__` are dunder methods worth calling out by name in an interview: implement `__repr__` for unambiguous debugging output (`WN@e4`) and `__str__` for a human-facing board rendering — Python distinguishes the two by convention (`__repr__` for developers/debuggers, `__str__` for end users and `print()`), so reach for both rather than collapsing everything into one method.

## Sample walkthrough

Shown as two independent scripts, not one — both sections define a class named `Board` (each module's own board representation), so pasting both implementations into one file and one script is a real namespace collision to watch for, not just a doc-formatting nicety: run each subsection's implementation in its own module/file.

```python
# Tic-Tac-Toe, 3x3, standard rules
p1, p2 = Player("A", Mark.X), Player("B", Mark.O)
game = TicTacToeGame(size=3, win_length=3, players=[p1, p2])
game.play(0, 0)  # X
game.play(1, 1)  # O
winner = game.play(0, 1)  # X, no win yet -> None
winner = game.play(2, 2)  # O
winner = game.play(0, 2)  # X completes row 0 -> row_count[0] hits +3 -> returns p1
print(game.board)
```

```python
# Chess: White knight opens, Black responds
chess = ChessGame()
wk = King(Color.WHITE, Position(0, 4)); chess.board.place(wk)
wn = Knight(Color.WHITE, Position(0, 1)); chess.board.place(wn)
bk = King(Color.BLACK, Position(7, 4)); chess.board.place(bk)
chess.make_move(wn, Position(2, 2))   # legal knight hop, turn flips to BLACK
print(chess.board)
```

## Follow-up questions

- **"Generalize Tic-Tac-Toe to Connect-4 (gravity, drop into a column)."** `Board.place_mark` changes from "place at (row, col)" to "drop into column, land on lowest empty row" — the win-check running-sum machinery in `TicTacToeGame.play` is untouched since it only cares about the final `(row, col)` a mark landed on, not how it got there.
- **"Add an undo button to the chess game."** Wrap `ChessGame.make_move` in a `MoveCommand` (holding piece, from, to, captured-piece) with an `undo()` that reverses `board.move`/`board.place` — per [Command](../patterns/03-behavioral-patterns.md#command); maintain a `deque[MoveCommand]` history in `ChessGame`. This is additive: `Piece`/`Board` don't need to know history is being tracked.
- **"Detect checkmate, not just check."** Extend `is_king_in_check` into an `is_checkmate` method: king is in check *and* no legal move by any of that color's pieces resolves it — i.e., simulate `make_move` for every `(piece, destination)` pair for the side to move and see if any leaves `is_king_in_check` `False`. Expensive but a direct extension of the existing self-check-filtering logic in `make_move`, not a new subsystem.
- **"Support castling / en passant / pawn promotion."** Each needs board-state beyond "current piece positions" — castling needs "has this king/rook ever moved," en passant needs "did the enemy pawn's *last* move double-step," promotion needs a user choice on reaching the back rank. Flag that this means `Board` needs a small amount of move-history-aware state (or `ChessGame` tracks it), and `Pawn`/`King` gain a few extra conditional branches — real work, correctly scoped out up front rather than discovered mid-interview.
- **"Two Tic-Tac-Toe / Chess clients over a network — where does state live?"** `TicTacToeGame`/`ChessGame` stay authoritative on a server; clients send `(row, col)` or `(piece, destination)` intents and receive the resulting state — the class boundary you already have (game owns board, exposes one `play`/`make_move` entry point) is exactly the seam a network layer wraps, not a redesign.

## Common mistakes on this problem

- Hardcoding Tic-Tac-Toe's win check to `size == 3` (eight explicit line checks) instead of the generalized running-sum approach — works for the stated problem, collapses the instant "what about 4×4" follow-up lands.
- Modeling chess pieces with a single `Piece` class and a `PieceType` enum plus a giant `if/elif` chain (or `match` statement) in `get_valid_moves()` — this is the *wrong* lesson to take from the parking lot's `VehicleType`; piece move generation is genuinely different logic per type (sliding vs jumping vs single-step), which is precisely when polymorphism earns its keep over an enum.
- Forcing [Composite](../patterns/02-structural-patterns.md#composite) onto the chess board ("pieces and squares and the board are all `BoardComponent`") because "board games feel tree-shaped" — there's no part-whole recursion here; resist naming a pattern without a concrete use, per the overuse trap called out in [patterns/00-overview.md](../patterns/00-overview.md).
- Skipping the "does this move leave my own king in check" filter and calling pseudo-legal moves "valid moves" — a `Rook` correctly refusing to jump over pieces is necessary but not sufficient; a move that's otherwise legal but exposes your own king must still be rejected, and it's a common gap in rushed chess implementations.

## Continue

Next: [05-lru-cache-and-rate-limiter.md](05-lru-cache-and-rate-limiter.md)
