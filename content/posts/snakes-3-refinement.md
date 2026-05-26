---
title: "Modeling Snakes and Ladders: Refinement"
date: 2026-05-24
mathjax: true
url: "/snakes-3"
---

In the third post in this series, we will refine the abstract model we created in [part 1](/snakes-1) using the board representation we defined in [part 2](/snakes-2).

Recall the model `BoardGame` we established in the first post. There's nothing specific to Snakes and Ladders in it; the same machine could be used as the base for most board games. To turn this into a Snakes and Ladders game, we need to add dice rolls and rules about navigating the board.

We could do this by creating an entirely new machine, but that would require repeating a lot of the properties we have already stated and proved in `BoardGame`. Instead, we use **refinement**. A refinement of an abstract machine is another machine whose context and state can be **lifted** into their abstract representations. The concrete machine may add more detail to its data structures, and may even have events that don't exist in the abstract machine, but there must be a way to zoom out from that detail to the abstract level.

Our refinement doesn't need to add any detail to the state---the players still just have integer positions---but it does need to fill out the context with our [board representation](/snakes-2). The main API the concrete machine needs is `Board::roll()`, which takes a player's current square and their roll and moves them along any snakes or ladders to their final square. Another function we use is `Board::is_at_rest()`, which indicates whether a given square is "neutral", meaning it does not sit at the base of a ladder or the head of a snake.

```rust
machine Snakes refines BoardGame {
    // The Board reprsentation is factored out
    context: Board

    state {
        players: (int, int),
    }

    // The lift_context function tells Event-V how to map the concrete board
    // into an abstract board. The abstract "board" is just as size, so we
    // take the length of the board.
    lift_context: |context| abs::Context {
        board_size: context.len(),
    }

    // The state lift isn't very interesting for this machine because the
    // concrete state is identical to the abstract.
    lift: |state| BoardGame {
        players: state.players,
    }

    init: |_| Snakes {
        players: (0, 0),
    }

    invariant: |board, state| {
        // Players can't stay at the top of a snake or the bottom of a ladder
        &&& board.is_at_rest(state.players.0)
        &&& board.is_at_rest(state.players.1)
    }

    // Instead of taking a square to teleport to, the concrete Turn takes a
    // dice roll. `DiceRoll` is an enum with six values for each face of a
    // six-sided die.
    refined event Turn(roll: DiceRoll) {
        lift_in: |board, state| {
            board.roll(state.players.0, roll)
        }

        guard: |board, state| {
            // Game not over
            !state.lift().is_done(Snakes::lift_context(board))
        }

        action: |board, state| {
            let next_square = board.roll(state.players.0, roll)
            Snakes {
                players: (state.players.1, next_square),
            }
        }
    }
}
```

Other than the refinement machinery, there are 3 main differences from the abstract machine:
1. The context object, which contains the board representation that we discuss below;
2. The invariant, which has the Snakes and Ladders specific rule that players must always follow the snakes and the ladders; and
3. The `Turn` event now takes a dice roll instead of an arbitrary square to teleport to.

Note that we didn't need to restate invariants already established in the abstract model---for example, that only one player can win the game. The refined model only needs to be explicit about the new details it is adding about dice rolls and the board. Behind the scenes, Event-V generates a number of proof obligations which ensure that our `Snakes` machine really is a refinement of the abstract machine.

For example, Event-V generates a `proof`-mode function called `proof_lift_safe`, which ensures that the mapping from concrete to abstract produces a valid abstract state. More interesting is the relationship between the concrete and abstract events. Suppose we tried to define the concrete `Turn` event as follows:

```rust
refined event Turn(roll: DiceRoll) {
    lift_in: |board, state| {
        board.roll(state.players.0, roll)
    }

    guard: |board, state| {
        // Game not over
        !state.lift().is_done(Snakes::lift_context(board))
    }

    action: |board, state| {
        let next_square = board.roll(state.players.0, roll);
        Snakes {
            // OOPS: forgot to swap player positions
            players: (next_square, state.players.1),
        }
    }
}
```

Here we forgot to swap the players' positions, so that the current player would not pass play to the next player. If we try to verify this code, Verus will complain that a postcondition does not hold in a function called [`proof_simulation`](https://github.com/jsenn/Event-V/blob/683300d87db5b9ddbb5b9431be8a24f9752575a6/src/machine.rs#L338). This **simulation proof** is generated for every refined event by Event-V. It states that applying the concrete event then lifting the result to the abstract representation produces the same abstract state and output as you would get from applying the *abstract* event to a lifted concrete state and input. 

In our broken code above, the simulation property does not hold, as the result of applying the concrete event then lifting to an abstract state produces a different value than the abstract event did: the players are swapped.

Similarly, if we forgot to add the `is_done` condition in the event's guard, Verus would complain that a postcondition on [`proof_strengthening`](https://github.com/jsenn/Event-V/blob/683300d87db5b9ddbb5b9431be8a24f9752575a6/src/machine.rs#L328) was violated. **Guard strengthening** is another fundamental property of refinements that Event-V forces us to satisfy. It says that the concrete guard may not be enabled in any situation in which the abstract event's guard is not.

These guardrails allow us to refine our model one step at a time without introducing errors. The [Event-V source](https://github.com/jsenn/Event-V/blob/main/src/machine.rs) contains much more detail about how refinement works.

In the [next post](/snakes-4), we will state and prove a theorem about the winnability of Snakes and Ladders.
