---
title: "Modeling Snakes and Ladders: Winnability"
date: 2026-05-25
mathjax: true
url: "/snakes-4"
---

In our [board representation](/snakes-2) we took some pragmatic shortcuts, constraining allowed boards more than was strictly necessary. However, we would still like to state winnability in the most general and abstract way we can. Essentially we would like to say that at any point in a Snakes and Ladders game, it is possible for someone to win the game.

Before we can state this theorem, we need two new helpers on `Board`:

```rust
impl Board {
    // ...

    /// Determine if the winning square is reachable from the given square.
    pub open spec fn can_win_from(self, square: int) -> bool {
        exists |rolls: nat| self.can_win_from_within(square, rolls)
    }

    /// Determine if the winning square is reachable from the given square
    /// within the given number of dice rolls.
    pub open spec fn can_win_from_within(self, square: int, rolls: nat) -> bool
        decreases rolls
    {
        // Base case: `square` is the winning square
        ||| self.is_winner(square)
        // Induction: some dice roll lands on a square we can win from
        ||| (rolls > 0 && exists |roll: DiceRoll|
            self.can_win_from_within(self.roll(square, roll), (rolls - 1) as nat))
    }
}
```

Note that `exists` is Verus's equivalent of the [existential quantifier](https://en.wikipedia.org/wiki/Existential_quantification) \(\exists\). Our `can_win_from` function states that there exists some number of rolls such that it is possible to win within that number of rolls from the given square.

The `can_win_form_within` function is defined recursively. The `decreases` clause gives Verus a hint as to how to prove termination. On every recursive call, we decrease the value of `rolls`. Since `rolls` is a natural number, we can only do this a finite number of times before hitting the smallest possible value of 0, hence the recursion will terminate eventually.

With these helpers, we can state our winnability theorem:
```rust
pub proof fn proof_winnable(board: Board, square: int)
    requires
        board.valid(),
        board.in_bounds(square),
    ensures
        board.can_win_from(square),
{
    let max_rolls = (board.len() - square - 1) as nat;
    lemma_valid_implies_winnable_within(board, square, max_rolls);
}
```

The theorem is stated as a `proof`-mode function. Proof mode functions can reference spec-mode functions but not vice versa. The `requires` clause states the conditions under which our theorem holds---namely, when the board and square are valid. The `ensures` is the statement we are proving, which is that it is possible to win from the given square.

Note that we have to invoke a hand-written lemma to prove this theorem. This is the only place in our Snakes and Ladders model in which Verus/Z3 was unable to automatically supply a proof. The reason is that `can_win_from` is defined recursively, and Z3 often struggles to automatically prove facts about recursive functions.

Let's think about how we would prove this on paper. From our `Board` model we have the forward progress invariant: every square on a valid board has some dice roll that will make forward progress. We can use this to prove winnability by induction. The base case is if we are already on the winning square. Otherwise, we assume that squares in front of us are winnable, and prove that there is some dice roll that moves us forward to those winnable squares.

Recursive `proof` functions are how Verus expresses proofs by induction. In this case, the helper lemma says that we can win from the current square *within a certain number of rolls*. Specifically, we need at most as many rolls as there are squares on the board between the current square and the winning square:

```rust
proof fn lemma_valid_implies_winnable_within(board: Board, square: int, rolls: nat)
    requires
        board.valid(),
        board.in_bounds(square),
        rolls + square >= board.len() - 1,
    ensures
        board.can_win_from_within(square, rolls),
    decreases board.len() - square,
{
    if !board.is_winner(square) {
        // Make sure Verus remembers that there is always some dice roll that makes progress
        assert(board.has_forward_progress(square));
        // Choose a specific dice roll with forward progress
        let roll = choose |roll: DiceRoll| board.roll(square, roll) > square;
        // Recurse using that dice roll
        lemma_valid_implies_winnable_within(board, board.roll(square, roll), (rolls - 1) as nat);
    }
}
```

The base case (the square is already the winning square) is easy; Verus handles it automatically. The proof body helps Verus to prove the induction case. It says if we are not already on the winning square, recall that every square can make forward progress for some dice roll. Choose that forward-moving roll, and recurse to the square it lands us on. By construction, it must be ahead of us, and by the induction hypothesis squares ahead of us are winnable. Therefore, the current square is winnable.

With this, our abstract model of Snakes and Ladders is complete. We defined an abstract [`BoardGame`](/snakes-1) machine that modeled turn-taking. The [`Board` type](/snakes-2) encoded invariants about Snakes and Ladders boards. The [`Snakes`](/snakes-3) machine refined `BoardGame` to model dice rolls and interactions with the board. And now we have stated and proved a theorem that every Snakes and Ladders game can be won.
