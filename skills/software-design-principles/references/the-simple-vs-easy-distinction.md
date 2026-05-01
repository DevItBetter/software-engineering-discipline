# Simple vs Easy

From Rich Hickey's *Simple Made Easy* (Strange Loop, 2011). The talk is the most influential single piece of design thinking of the last 15 years. The framing here is the lens that catches what other design vocabulary misses.

## The two distinctions

**Simple** vs **complex**. Etymology: *simple* = "one fold." *Complex* (from *complecti*) = "braided together." Simplicity is **objective**: it is a property of the structure of the thing. Either it is interleaved with other concerns or it isn't.

**Easy** vs **hard**. Etymology: *easy* = "near at hand." Easy is **subjective**: it is about familiarity, accessibility, and proximity to what you already know.

The two axes are independent. You can have:

- Simple and easy: a small, well-known function in a familiar language.
- Simple and hard: pure functional dataflow over immutable values, in a paradigm you don't know yet.
- Complex and easy: an ORM that hides ten subsystems under a familiar dot-syntax.
- Complex and hard: an ORM that hides ten subsystems and you've never used it before.

## Why the distinction matters

The trap is choosing for *ease* and getting *complexity* you didn't see. The classic examples:

- **State / mutable variables.** Easy to use; complect time, identity, and value. Reasoning about a value-that-changes requires reasoning about *when* it changes and *who* sees what.
- **Inheritance.** Easy to set up (extend a class); complect type, behavior, identity, and namespace. Changes to the base class affect every subclass in non-obvious ways.
- **ORMs and ActiveRecord patterns.** Easy to start with; complect data access, schema knowledge, identity, transactions, and lazy loading. Result: queries hidden inside attribute access; N+1 problems; impossible-to-test boundaries.
- **Conditionals (`if/else`).** Easy; complect "what to do" with "when to do it." A scattered set of `if isinstance` checks complects the type-discrimination logic across many places.
- **Frameworks with lifecycle hooks.** Easy to start a project; complect structure, lifecycle, configuration, and conventions. Changes to one piece reverberate through the framework's hidden coordination.
- **Globals, singletons, service locators.** Easy; complect every consumer with the global's lifecycle, identity, and changes.

Conversely, things that are *not easy* but are *simple*:

- **Pure functions.** Each function is one thing, no interleaving with state, time, or identity.
- **Immutable values.** No interleaving of identity with current value.
- **Explicit message passing / queues.** No interleaving of "who sends" with "who receives."
- **Composition over inheritance.** Behavior added by combining things, not by inheriting.
- **Data > functions > macros.** Plain data is the simplest thing; functions are simpler than macros; macros are simpler than frameworks.

## The diagnostic question: "what does this complect?"

When evaluating a design move, ask what it braids together that could be independent.

- A class that holds state and contains business logic complects state, identity, and policy.
- An exception that contains a stack trace and a user-facing message complects diagnostics with presentation.
- An HTTP handler that calls the database directly complects transport, routing, business logic, and persistence.
- A "rich domain model" with persistence annotations complects domain logic with storage shape.

Each complecting carries a tax: you cannot change one without thinking about the others. Each *un*-complecting frees you to vary one independently.

## Modularity is necessary but not sufficient for simplicity

A common confusion: "I split this into 20 small files, so it's simple now." Not necessarily. Splitting a tangled mess into 20 tangled-with-each-other modules has not reduced complecting; it has only spread it across a wider surface, possibly making it worse.

Hickey's specific claim: simplicity is **enabled by** stratification (layers) and partitioning (modules), but is not the *same* as them. You can have small modules whose *interactions* are deeply complected. The simple test is whether you can reason about each module without knowing about the others. If you can't, the modularity hasn't bought you simplicity.

## "Simplicity is hard" — strategies for getting there

Per Hickey, simplicity requires *care*: the practitioner has to think about it, choose it, and resist easier alternatives.

Strategies that actually produce simplicity (not always practical, but worth recognizing):

- **Separate `what` from `how`.** Name the operation; let the implementation be free to vary.
- **Separate policy from mechanism.** The mechanism (the engine) and the policy (the rules) should be independently changeable.
- **Separate value from identity from time.** A bank account's *value* (current balance) is separable from its *identity* (which account) which is separable from *time* (when this value was current). Most languages complect these in the variable.
- **Separate data from functions from macros.** Use the simplest construct that suffices.
- **Prefer queues to direct calls** when you want sender and receiver to vary independently.
- **Prefer composition to inheritance.**
- **Prefer immutable data to mutable.**
- **Prefer narrow, named messages to wide, structural arguments.**

## The relationship to "deep modules"

Ousterhout and Hickey are looking at related things from different angles. A *deep module* hides complexity behind a simple interface. For the interface to actually be simple (not just small), the module must not complect concerns into the interface.

A bad shallow module complects: it exposes its internals via a wide interface. A bad deep module *can* exist too — one whose interface looks small but is actually complected. (E.g., a `process(data, mode, options, flags) -> result` where the four parameters secretly interact in 30 ways.) The depth is illusory; the complexity has been hidden by being smushed together rather than abstracted away.

The combined heuristic: a good module is **deep** (hides a lot) and the things it hides are **simple** (un-complected), so that *if* a caller needs to look inside, the inside is reasonable.

## Caveat: simplicity is not always the immediate goal

Hickey is making a structural argument about long-term cost. In practice, sometimes the right move is a temporary "easy" choice to ship and revisit. The point is to *recognize the trade-off* — to know when you're choosing easy because the schedule demands it, vs. believing you've chosen simple when you haven't.

The damage comes from confusing the two. "We chose this because it was simple" — when in fact it was easy and complected — is a misdiagnosis that prevents the team from understanding why the system is hard to change a year later.
