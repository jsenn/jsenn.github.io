---
title: "Modeling Snakes and Ladders: The Board"
date: 2026-05-23
mathjax: true
url: "/snakes-2"
---

In the [last post](/snakes-1), we introduced [Verus](https://github.com/verus-lang/verus) and used it to create an abstract formal model of Snakes and Ladders. The model knows that the board has some number of squares, that players take turns moving around the board, and if someone lands on the final square the game is over, but the board itself was abstracted away completely. Here, we model the board.

Recall that a Snakes and Ladders board is a sequence of squares. On some squares there is the base of a ladder. Players on that square can climb the ladder to go further along the board. Other squares have the head of a snake on them; when a player lands on one of those, they slide down the snake's back to an earlier square.

{{< figure
    src="/assets/images/snakes-board.jpg"
    alt="Snakes and Ladders board"
    caption="A Snakes and Ladders board"
    attr="muffinn from Worcester, UK, CC BY 2.0 <https://creativecommons.org/licenses/by/2.0>, via Wikimedia Commons"
    attrlink="https://upload.wikimedia.org/wikipedia/commons/a/a4/Berrington_Hall_-_snakes_and_ladders_%2813826426425%29.jpg"
>}}

We model the board as a sequence of integers, where each integer represents the number of squares forward or backward a player must move after landing on that square:

```rust
pub struct Board {
    // `Seq` is a Verus built-in type that represents an abstract sequence
    pub squares: Seq<int>,
}
```

A square that sits at the base of a ladder will have a positive value; one that sits at the head of a snake will have a negative value. Zero-valued squares are neutral.

For example, the board `[0, 0, 0]` has no snakes or ladders. The board `[0, 1, 0]` has a ladder in the second square, so if you land on it you immediately move forward to the final square and win the game. Conversely, the board `[0, -1, 0]` has a snake in the second square; if you land on it you get sent back one square to the start.

A few helpers will make properties easier to state. Note that `DiceRoll` is an enum with a case for each face on a six-sided die.

```rust
impl Board {
    /// Get the total number of squares on the board.
    pub open spec fn len(self) -> nat {
        self.squares.len()
    }

    /// Determine if the given square is on the board.
    pub open spec fn in_bounds(self, square: int) -> bool {
        0 <= square < self.len()
    }

    /// Determine if the given square is the winning square.
    pub open spec fn is_winner(self, square: int) -> bool {
        square == self.len() - 1
    }

    /// Compute the resting square of a player currently at the given square
    /// after they roll the given value from the die.
    pub open spec fn roll(self, square: int, roll: DiceRoll) -> int {
        self.follow(square + roll.value())
    }

    /// Compute the resting square of a player who landed on the given square
    /// after following any snakes or ladders on that square.
    pub open spec fn follow(self, roll_square: int) -> int {
        if roll_square >= self.len() {
            self.len() - 1
        } else {
            roll_square + self.squares[roll_square]
        }
    }

    /// Determine whether a player can come to rest at the given square.
    /// Returns false if the given square is at the head of a snake or the
    /// bottom of a ladder.
    pub open spec fn is_at_rest(self, square: int) -> bool {
        self.squares[square] == 0
    }

    /// Determine whether there is some dice roll that makes forward progress
    /// from a given square.
    pub open spec fn has_forward_progress(self, square: int) -> bool {
        ||| self.roll(square, DiceRoll::One) > square
        ||| self.roll(square, DiceRoll::Two) > square
        ||| self.roll(square, DiceRoll::Three) > square
        ||| self.roll(square, DiceRoll::Four) > square
        ||| self.roll(square, DiceRoll::Five) > square
        ||| self.roll(square, DiceRoll::Six) > square
    }
}
```

## Board Invariants
As in the abstract model, a board must have at least 2 squares, otherwise everyone would win immediately:
> **Invariant:** a board must have at least 2 squares

This condition can be expressed on our `Board` type as:

```rust
self.len() > 1
```

The board `[0, 2, 0]` is invalid: the second square asks you to move forward 2 spaces, but there is only one square beyond it on the board. This suggests an invariant:

> **Invariant:** snakes and ladders cannot send a player off the board
> ```rust
> forall |square: int|
>     self.in_bounds(square) ==> self.in_bounds(self.follow(square))
> ```

In words, for every position on the board, adding the value of the square at that position can't send you before the first square or after the last. Note that `forall` is a spec-mode operator in Verus that represents the [universal quantifier](https://en.wikipedia.org/wiki/Universal_quantification) \(\forall\).

Another invalid board is `[2, 0, 0]`, in which the starting square has a ladder to the final square. This board is unplayable: everyone would win before the game even began! Similarly, the board `[0, 0, -1]` is unplayable, because every time you land on the final square you are immediately kicked back, so the game cannot be won. We need another invariant to rule these cases out:

> **Invariant:** a board cannot have a snake or ladder on the first or last square
> ```rust
> self.squares[0] == 0 && self.squares[self.len() - 1] == 0
> ```

Now consider the board `[0, -1, -2, -3, -4, -5, -6, 0]`. This board has 6 snakes in a row. No matter what you roll on your first turn, you are sent back to the starting square. There is therefore no traversable path from start to finish, and the game is again unwinnable. We can define traversability recursively, and we will do so below. But Verus is not able to prove traversability automatically for a realistic Snakes and Ladders board, and writing the proof manually is very tedious: you would have to spell out the exact sequence of moves from start to finish.

Instead, we require a stronger property:

> **Invariant:** at every square on the board other than the final one, there must be some dice roll that will land the player further ahead than they started
> ```rust
> forall |square: int|
>     self.in_bounds(square) && !self.is_winner(square) ==>
>         self.has_forward_progress(square)
> ```

This immediately rules out the unwinnable board above. However, it also rules out boards that are winnable. For example, the board `[0, 8, 0, -3, -4, -5, -6, -7, -8, 0]` is ruled out, as there is no way to make forward progress with a single dice roll from the third square. But it is traversable: if you roll a 1 after the snakes send you back to the start, you win immediately.

What we lose in generality we gain in convenience: this invariant is easy for Z3 to prove automatically, which means that we can instantiate Snakes and Ladders boards without having to construct a long and tedious proof of traversability for each one.

We define a final convenience property for similar reasons:

> **Invariant:** Snakes and Ladders cannot chain together
> ```rust
> forall |square: int|
>     self.in_bounds(square) && !self.is_at_rest(square) ==>
>         self.is_at_rest(self.follow(square))
> ```

This property rules out boards like `[0, 1, -2, 0]`, where the top of a ladder lands you at the head of a snake. Supporting boards like this would require recursive turn-taking logic, which again is hard for Verus to deal with.

## Validating a Concrete Instance
Just for fun, we can encode the board shown in the image above and make sure that Verus can prove it is valid:

```rust
proof fn proof_board_valid() {
    let board = Board {
        squares: seq![
            0, 0, 0, 0, 0, 0, 0, 18, 0, 0,
            0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
            61, 0, 0, 0, 0, 0, 0, 0, 0, 0,
            0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
            0, 0, 34, -22, 0, -41, 0, -39, 0, 41,
            0, -41, 0, 39, -48, 0, 0, 0, -42, 0,
            0, 34, 0, -28, 0, 21, 0, 0, -60, 0,
            0, 0, -72, 0, 0, 0, 0, 0, 0, 20,
            0, 0, -64, 0, 0, 0, 0, 0, 0, 0,
            0, -41, 0, 0, -71, 0, 0, -70, 0, 0

        ],
    };

    assert(board.valid());
}
```

Note that a `proof`-mode function is used here so we can use Verus's `assert` statement.

If we remove the forward-progress invariant, and instead rely on the recursive winnability property, Verus is unable to prove that this board is valid by itself.

In the [next post](/snakes-3), we will use our `Board` type to refine the abstract `BoardGame` model from part one into a full model of Snakes and Ladders.
