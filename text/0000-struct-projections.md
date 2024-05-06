- Feature Name: Structure projections
- Start Date: 2024-04-28
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

This RFC proposes the notion of a "structure projection" which is another aspect
of a structure's public API. It is a generalization of partial moves and borrowing.

# Motivation
[motivation]: #motivation

It's very useful to be able to bundle several pieces of semi-independent data
together in a single structure. For example, you might have a `Player` structure
representing the player character in a game, which holds the players position,
velocity, inventory, health and other parameters. Generally inspecting the
inventory is not affected by the player's position, so you should be able to do
it even if the position is currently borrowed exclusively.

Similarly in embedded systems, it's common to represent all the available
hardware devices as a single structure; the individual devices are often
completely independent however. Different parts of the program use different
devices, so they use the separate parts of the structure independently.

Rust currently supports such operations via "partial borrows" and "partial
moves". It can track access to the fields of a structure independently, so a
borrow or move of one field does not prevent another field from being mutated.

However, this only works when directly accessing the fields. Once you try to do
this via a method, the reciever (`&self`/`&mut self`/`self`) will attempt to
borrow/move the entire structure, and fail if this is not possible.

This severely limits the usefulness of partial borrows/moves.

This RFC proposes a mechanism to generalize partial borrows/moves to allow
struct impls to act on logical subsets of the state encoded in a structure. It
does this by introducing the notion of a "projection" which is a way to name a
subset of the state. Methods are labelled with which projection they operate on,
and are only allowed to access fields from the corresponding projection. 

Projections are part of the API signatures for the type, but don't dictate a
specific implementation. The goal is that projections reflect semantically
meaningful aspects of the API rather than be a direct reflection of the
implementation. For example, `Player` may have a `velocity` projection, but that
doesn't mean it necessarily has a `velocity` field - it could be synthesizing it
from `speed` and `direction`.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

(Preliminary syntax for explanation only: 
`#[projection(...)]` and `@projection`)


```rust
#[projection(position)]
#[projection(direction)]
#[projection(speed)]
#[projection(velocity: speed + direction)]
struct Player {
    #[projection(position)]
    position: (f32, f32),
    #[projection(direction)]
    direction: f32,
    #[projection(speed)]
    speed: f32,
    #[projection(velocity)]
    _velocity: (), // TODO: need to tie all projections to actual fields?
}

impl Player {
    #[projection(speed)]
    pub fn speed(&self) -> &@speed f32 { ... }

    #[projection(direction)]
    pub fn direction(&self) -> & @direction f32 { ... }

    #[projection(velocity)]
    pub fn velocity(&self) -> (f32, f32) { ... }

    #[projection(position, velocity)]
    pub fn update_pos(&mut @position self) { ... }
}
```

A type's API interface consists of a number of different elements, such as its
visible fields, intrinsic and trait implementations, associated constants and so
on. In addition to these it may also have a set of "projections". These are a
description of pieces of independent state a structure may contain, which can be
treated almost as if they were actual fields with partial borrowing or moving.

For example, this structure defines independent position and velocity projections:
```rust
#[projection(position)]
#[projection(velocity)]
struct Player {
  #[projection(position)]
  position: (f32, f32),
  #[projection(velocity)]
  velocity: (f32, f32)
}
```

The initial `projection` attributes on the `struct` define `position` and `velocity`
projections as parts of the `Player` structure's API. They tell the user of this
type that they exist as independent state, but without committing to any
particular implementation.

The `projection` attributes on each field tie each of those fields to the
particular projection for use by implementations. For example:

```rust
impl Player {
  #[projection(position)]
  fn get_position(&self) -> (f32, f32) { self.position }
}
```

This means that the `get_position` method acts on the `position` projection.
This means 1) it may only access fields which are part of that projection, and 2
in order to call it the `position` projection must be availabe for borrowing
(with `&self`) - it, it must not have been moved or be exclusively borrowed.
(See below for more details.)

An alternative method:
```rust
#[projection(position)]
pub fn get_position_ref(&self) -> &(f32, f32) { &self.position }
```
returns a reference. This reference is specifically a reference to the
`position` projection of the structure. Desugared it looks like:
```rust
#[projection(position)]
pub fn get_position_ref<'a>(&'a @position self) -> &'a @position (f32, f32) { &self.position }
```

If we had a corresponding:
```rust
#[projection(position)]
pub fn get_position_mut(&mut self) -> &mut (f32, f32) { &mut self.position }
```

and the caller:
```rust
  let pos = player.get_position_ref(); // OK: Shared borrow of `position` projection
  let pos_mut = player.get_position_mut(); // BAD: `pos` is already a shared borrow of `position` projection
```

but if we also have
```rust
#[projection(velocity)]
pub fn get_velocity(&self) -> (f32, f32) { self.velocity }
```
then this would be fine:
```rust
  let pos_mut = player.get_position_mut(); // OK: exclusive borrow of `position`
  let vel = player.get_velocity(); // OK: `velocity` projection independent of `position
```

We said above that the projections are part of the interface, but don't
constrain the implementation. For example, we could change `Player` with:
```rust
#[projection(position)]
#[projection(velocity)]
struct Player {
  #[projection(position)]
  position: (f32, f32),
  #[projection(velocity)]
  speed: f32,
  #[projection(velocity)]
  direction: f32,
}

impl Player {
  #[projection(velocity)]
  pub fn get_velocity(&self) -> (f32, f32) { ... }
  #[projection(velocity)]
  pub fn get_speed_mut(&mut self) -> f32 { &mut speed }
}
```

This presents almost the same API as before, except we're also exposing the speed via `get_speed_mut`.
```rust
  let speed_mut = player.get_speed_mut(); // OK: got an exclusive reference to `velocity` projection
  let get_velocity = player.get_velocity(); // BAD: `velocity` already borrowed exclusively
```

We can also factor the `speed` and `direction` fields as their own projections
while still keeping `velocity`:

```rust
#[projection(position)]
#[projection(speed)]
#[projection(direction)]
#[projection(velocity: speed + direction)]
struct Player {
  #[projection(position)]
  position: (f32, f32),
  #[projection(speed)]
  speed: f32,
  #[projection(direction)]
  direction: f32,
  // XXX need a velocity placeholder field?
}

impl Player {
  #[projection(velocity)]
  pub fn get_velocity(&self) -> (f32, f32) { ... }
  #[projection(speed)]
  pub fn get_speed_mut(&mut self) -> f32 { &mut speed }
}
```

From a user's perspective, `location` and `velocity` projections still exist,
but the implementation has changed to use a polar (speed and direction)
representation. We've also added `speed` and `direction` projections to expose
this. (Note that adding projections should be backwards compatible, but removing
them isn't.)

The result is that a method acting on the `velocity` projection must have access
to both the speed and direction projections. For example:
```rust
  let speed_mut = player.get_speed_mut(); // OK: got an exclusive reference to `speed` projection
  let get_velocity = player.get_velocity(); // BAD: `velocity` not available because component `speed` is already borrowed
```

All structures have the default `ALL` projection, which includes all other
projections. Any method which doesn't have an explicit projection attribute is
taken as `ALL`. This is consistent with Rust before projections, where, say, a
`&self` parameter was taken to borrow the entire structure.

## Projections from fields

A structure may want to expose a projection from one of its own field's structures. For example:
```rust
#[projection(position)]
#[projection(velocity)]
struct Player {
  #[projection(position = Entity::position)]
  #[projection(velocity = Entity::velocity)]
  entity: Entity
}
```

## Projection sets

The `@projection` syntax is a special case of a more general syntax of the form `@(set_expression)`:
```bnf
set_expression ::= "(" set_expression ")"
                | set_expression + set_expression
                | set_expression - set_expression
                | projection_name
```

The parentheses are needed if the set_expression is anything other than a
literal projection name.

This means you can express a negative expression such as `@(ALL - velocity)` to
express a Player structure which has no velocity.

## Projections across APIs

Projection annotations need not be tied to intrinsic methods. For example:
```
fn do_something_to_player(player: & @velocity Player) -> & @speed f32 { ... }
```

The projections are tied to the lifetimes of the references. For example, given
a function with the signature:
```rust
fn get_position_ref(player: & @position Player) -> &(f32, f32) { ... }
```
Since this is using lifetime elision, the lifetime of the `player` parameter is
inferred for the return value. The projection `position` is attached to a
`Player` reference, so its inferred to be `Player::position`. As a result the
return reference is `& '1 @Player::position (f32, f32)` (using `'1` for the
elided lifetime).

## Moved projections

TBD

- a function taking a structure by value (eg `self` or other non-reference
  parameter) can specify it only needs specific projections to be present.
- A return by value (`Self`) can specify which projections are present
- Code within the method can assign to non-present fields like initialization

## Projections and Traits

A trait implementation for a type can be over a projection. For example:
```rust
#[projection(position, velocity)]
impl Move for Player {
  fn move(&mut self);
}
```

means that calling `<Player as Move>::move` only interacts with the `position`
and `velocity` projections. The `&mut self` receiver is treated as `&mut
@(position + velocity) self`.

However, because trait objects erase the underlying type, they also erase the
projections. Therefore coercing a `&mut Player` to `&mut dyn Move` is an
operation on `&mut @all Player`. This means that the coercion fails if there are
any outstanding borrowed or moved projections on the object.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

TBD

- projections are named, all structures have an `all` projection which covers
  the whole thing; all other projections are a subset of this
- References are bound to a projection. Projections are attached to lifetimes (named or elided)
- Also owned objects are labelled with a set of present/missing projections for partial moves
- Existing partial move/borrows are reimplemented in terms of projections
  (basically a projection per field with an internal name)
- subtyping relationship between different projections

Projection naming:
- Canonical name is path::of::type::projection
  - This makes them have the same form as field names. Are they in the same
    namespace or not? I can see it being useful to be able to give a projection
    and a field the same name, so different namespaces. It could be confusing
    though, like traits and their types.
  - How to name the projection of a specific field (eg for "re-exporting" a
    projection from an inner type). Referencing a field name would make more
    sense than type (since there could be multiple fields with the same type,
    and you want re-project state from just one of them)
  - 
# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

- Complex addition to the language
- Does it cohere with the rest of the language?
- Can this be approximated with existing language facilities?
- Can it be implemented effectively in a library (eg Bevy ECS)?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Traits as projections

The set relationship between projections is reminiscent of how traits are
defined: it's common to use a collection of narrowly scoped traits to expose
specific aspects of a more complex type.

This raises the possibility that one could use traits as the direct
representation of a projection, rather than having to define a whole new
language concept.

However, I think this runs aground when considering how it must interact with
the borrow checker. In this design, references are augmented with the set of
projections being borrowed by the reference. I don't see any way of doing that
with a trait-related mechanism.

# Prior art
[prior-art]: #prior-art

## View types

[Niko](https://smallcultfollowing.com/babysteps/blog/2021/11/05/view-types/)
discusses this problem and proposes "view types". I think this key distinction
between view types and this proposal is that projections are not directly tied
to field names. Directly referencing field names in the API of a type means that
it's very coupled to internal implementation details.

The `&{project,...} something` syntax is probably worth considering though.


Also [this
post](https://internals.rust-lang.org/t/notes-on-partial-borrows/20020) is an
interesting summary of the field and proposal. It uses `&<...>` syntax for
viewed references; I'm not sure if this is more or less ambiguous than `{}`.

## Structural inheritance

In some ways this is similar to a structural subclassing mechanism. For example
the `Player` example might be implemented in C++ with:
```c++
class Speed { ... };
class Direction { ... };
class Velocity: public Speed, public Direction { ... };
class Inventory { ... };
class Player: public Velocity, public Inventory { ... };
```

But this is much more limited - fundamentally projections are a "has-a"
relationship rather than an "is-a" relationship.


## Bevy's ECS mechanisms

Bevy implements an elaborate system for disaggregating state via it's
Entity/Component/System (ECS) mechanism. An "entity" is more or less equivalent
to a projection, in that it represents a specific piece of state. These are
logically grouped into "components" to form an aggregate bundle of state, which
are then operated on by systems. 

In order to support this, it implements an elaborate query mechanism, partially
implemented at compile time, partially at runtime. This allows a piece of code
to specify which specific pieces of state it wants to operate on (shared or
exclusively) and the runtime makes sure that everything is sequenced
appropriately.

This mechanism requires the whole program to conform to this design, which means
its very hard to retrofit into an existing design. It's also not appropriate for
embedded systems (it requires an allocator, threading, and a lot more).


# Unresolved questions
[unresolved-questions]: #unresolved-questions

- This currently focuses on `struct`s. Could it be extended to cover other
  aggregate types? What is partial borrow/move currently supported on?
- How does it this interact/relate with pin projections? Could they be
  implemented in terms of this?
- The idea of attaching a set of projections to a reference is similar to Niko's
  view types?
- This attempts to cover partial borrows and moves. Moves are raise their own
  set of tricky questions - should they be separated and handled once borrows are worked out?
  Specific issues:
  - How does one indicate a move?
  - How do you indicate a partial `self`?
  - If a method requires a partial `self`, but the object passed in isn't
    partial, is that a compile time error? The extra fields are Dropped as part
    of the call? (I think they have to be unless we want in-struct drop flags
    again)
  - Does this handle re-populating moved fields?
- Syntax:
  - The attribute syntax isn't awful to start with
  - But I'm really unsure about a sigil-based approach for annotating references
  - And see the partial-move questions above
- What other things would this enable? Does it solve more problems?


# Future possibilities
[future-possibilities]: #future-possibilities

- Some way of extending it to trait objects
- Extending to partial moves if we defer them