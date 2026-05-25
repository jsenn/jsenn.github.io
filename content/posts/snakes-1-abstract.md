---
title: "Modeling Snakes and Ladders"
date: 2026-05-25
mathjax: true
url: "/snakes-1"
---

Recently I've been learning [Verus](https://github.com/verus-lang/verus), which is a system for proving the correctness of Rust programs. I'm particularly interested in defining abstract state machines in the style of [TLA+](https://lamport.azurewebsites.net/tla/tla.html) or [Event-B](https://en.wikipedia.org/wiki/B-Method#Event-B), and using Verus to prove properties about those machines, as well as code implementing them. Event-B's philosophy is to model your program with a simple abstract model first, and incrementally **refine** it until it can be implemented in its full complexity.

Along these lines I have been tinkering with [Event-V](https://github.com/jsenn/Event-V), which is a framework that implements the Event-B incremental refinement methodology in Verus. The code in this post is adapted from the [Event-V examples](https://github.com/jsenn/Event-V/tree/main/examples/snakes).

As an example of this way of working, consider [Snakes and Ladders](https://en.wikipedia.org/wiki/Snakes_and_ladders). At the highest level, this is a board game where players start at the first square and take turns moving around the board until one of them reaches the final square, at which point that player wins and the game is over. Yes, there are dice and snakes and ladders involved, but let's worry about that later.

## The Abstract Model

Our first model abstracts the board away completely: all we know is its size. What we *will* model are the positions of each player on the board, the game's turn-taking mechanic, and the win condition. To keep life simple, we will only model a 2-player game, though the model is easily extended to any number of players.

Before defining the model, we will create a **context** object. The context contains configuration that doesn't change over the lifetime of a machine. In our case, the only piece of configuration is the size of the board.
```rust
pub struct Context {
    pub board_size: nat,
}
```

Note that `nat` is a Verus type that represents a natural number. It only exists in specification code.

In order for our context object to be usable by Event-V, we have to implement the `MachineContext` trait. Its only requirement is a function `valid` so Event-V knows what counts as a valid context:

```rust
impl MachineContext for Context {
    open spec fn valid(&self) -> bool {
        // A board with only 1 square would be unplayable as everyone would
        // win immediately!
        self.board_size > 1
    }
}
```

The `open spec` qualifier tells Verus that this is a spec-mode function whose body should be made visible to the solver.

Finally, we define a couple of helper functions to use in our spec:
```rust
impl Context {
    pub open spec fn in_bounds(self, square: int) -> bool {
        // Note that Verus allows chaining inequality operators
        0 <= square < self.board_size
    }

    pub open spec fn is_winner(self, square: int) -> bool {
        square == self.board_size - 1
    }
}
```

Now that we have a context object, we can define our machine. A machine can be written using the framework's low-level trait machinery, but that can be quite tedious. Event-V also provides a convenient DSL for defining machines. Below is the definition of our abstract machine. We call it `BoardGame`, as at this level of abstraction there's nothing very specific to Snakes and Ladders in it.

```rust
// Some quick notes on Verus syntax:
// * The `&&&` and `|||` operators are equivalent to the familiar `&&` and
//   `||`, but they are used in a "bullet point" style. Since specifications
//   often boil down to a long list of ands and ors, this can be quite handy.
// * Verus also provides some spec-only types like `nat` and `int`, which
//   represent mathematical natural numbers and integers.

machine BoardGame {
    // The context object contains configuration that can't change over a
    // machine's lifetime.
    context: Context

    // The state contains variables that can be changed by events.
    state {
        // Each player's position on the board is represented by an integer.
        // By convention, the next player is always the first in the pair, so
        // each time play passes to the next player we swap them.
        players: (int, int),
    }

    // The initializer takes the context and produces the initial state of
    // the machine. By convention, the state object takes the name of the
    // machine.
    init: |context| BoardGame {
        players: (0, 0), // Each player starts off on the first square
    }

    // The invariant defines what a valid game state looks like.
    invariant: |context, state| {
        // All players on the board
        &&& context.in_bounds(state.players.0)
        &&& context.in_bounds(state.players.1)
        // At most one winner
        &&& !(context.is_winner(state.players.0) &&
              context.is_winner(state.players.1))
    }

    // The machine has a single event which represents a single turn. At this
    // level, we don't model dice rolls or following snakes and ladders.
    // Instead, we just accept some arbitrary square and teleport the player
    // there.
    event Turn(new_square: int) {
        // The guard determines when the event can fire.
        guard: |context, state| {
            // Game not over
            &&& !state.is_done(context)
            // Valid next square
            &&& context.in_bounds(new_square)
        }

        // The action defines how the machine's state changes in response to
        // the event.
        action: |context, state| BoardGame {
            // players.0 moves to new_square, and players swap so players.1
            // will take their turn next.
            players: (state.players.1, new_square),
        }
    }
}
```

Behind the scenes, Event-V generates several **proof obligations** so that Verus will complain if our machine has any bugs. For example, if we initialized the players to square `-1`, Verus would complain that the `in_bounds` requirement in the invariant was not satisfied. Similarly, it is not possible to define an `event` that produces an invalid state.

In future posts we will refine this model by fleshing out the board representation and adding dice rolls.
