# Cohesion and Coupling

The two oldest design ideas that still apply unchanged. Originally articulated by Stevens, Myers, and Constantine ("Structured Design," 1974) and refined by Parnas ("On the Criteria To Be Used in Decomposing Systems into Modules," 1972).

## Cohesion

**Cohesion** is the degree to which the elements within a module belong together. High cohesion is good: a module does one coherent thing.

The classic ranking from worst to best (originally Constantine):

1. **Coincidental cohesion (worst).** Things grouped because someone happened to put them in the same file. `utils.py` is the canonical example. No principle holds the contents together.
2. **Logical cohesion.** Things grouped because they're "the same kind of thing" but operate on different data and are called in different contexts. (E.g., a `formatters` module with formatters for orders, customers, invoices, and emails — they share no state and no logic.)
3. **Temporal cohesion.** Things that happen at the same time but are otherwise unrelated. (E.g., an `init()` that opens the database, sets up logging, registers signal handlers, and primes the cache.)
4. **Procedural cohesion.** Things grouped because they're called in sequence as part of one task — but each step could equally serve a different sequence.
5. **Communicational cohesion.** Things that operate on the same data structure.
6. **Sequential cohesion.** Output of one step is input to the next.
7. **Functional cohesion (best).** Every element contributes to one well-defined task. The module's name is one verb-noun pair, and every element supports it.

A modern restatement, owed to Robert Martin's reframing of SRP: **a module should have one reason to change.** If the file gets edited for two unrelated reasons (a price-formatting tweak *and* an authentication change), you've got divergent change — low cohesion.

## Coupling

**Coupling** is the degree to which one module depends on another. Loose coupling is good: modules communicate through narrow, stable interfaces and don't reach into each other's internals.

The classic ranking from worst to best:

1. **Content coupling (worst).** Module A reads or modifies Module B's internal state directly. (`other_module._private_var = 7`.) Found a lot in dynamic languages because the language doesn't enforce the boundary.
2. **Common coupling.** Modules share access to a global mutable variable. Each module's behavior depends on whatever the others happened to do to the global.
3. **External coupling.** Modules share an externally-imposed format (e.g., both depend on the same protocol or file format). Necessary for interop, but a leak when used internally.
4. **Control coupling.** Module A passes a flag to Module B that controls B's internal flow. (`B.do_thing(mode='special')`.) The caller must know B's internals to pick the flag.
5. **Stamp coupling.** Module A passes a whole structure to B when B only needs a piece of it. B is now coupled to the structure's shape, not just the piece it uses.
6. **Data coupling.** Modules communicate through plain values — exactly the data needed, no more.
7. **Message coupling (best).** Modules communicate through well-defined messages with no shared state. The receiver decides what to do with the message; the sender doesn't know how it's processed.

In practice, the modern rules-of-thumb that capture most of this:

- **Don't reach across a module boundary into private state.** If you have to, the boundary is wrong; either expose the operation or move the calling code.
- **Pass exactly what the callee needs**, not the whole context object. (Stamp coupling is how innocent functions become tied to massive shared types.)
- **Don't use flags to alter behavior across the boundary.** If the caller would pass `mode='create'` vs `mode='update'`, expose two functions or two methods.
- **Globals are shared mutable state by another name.** Avoid them; if necessary, treat them as a real dependency.

## How cohesion and coupling fail together

Bad designs almost always have **low cohesion AND tight coupling**. The two reinforce each other: a module with low cohesion (does many unrelated things) ends up coupled to many other modules (because each unrelated thing has its own dependencies). Modules with high cohesion (do one thing) tend to have minimal coupling (because the thing they do has clean inputs and outputs).

When a design is going wrong, ask both questions:

- **Does this module do one coherent thing?** (Cohesion.)
- **Does this module depend only on what it actually needs?** (Coupling.)

If either answer is no, that's where the redesign starts.

## Connecting to information hiding (Parnas)

Parnas's 1972 paper added the criterion: **what does this module hide?**

- A module with high cohesion that hides a *real decision likely to change* is a good module.
- A module with high cohesion that hides nothing meaningful (or hides a decision that will never change) is overhead.

The specific test: list the design decisions in the system that are most likely to change. Each module should hide one of them — so when the decision changes, only one module changes. If a likely-to-change decision is *not* hidden in any single module, that's where the next refactor needs to land.

## Practical heuristics

- **The "what does this file do?" test.** State the file's purpose in one sentence with one verb. If you need "and" or "or," it's two files.
- **The "co-changed files" test.** Look at git history. If two files are almost always changed together, they probably belong in one module (or are missing a shared abstraction). If one file is changed for many unrelated reasons, it's two modules pretending to be one.
- **The "cross-cut your tests" test.** If testing one piece of behavior requires setting up many unrelated dependencies, the module under test is coupled to those dependencies in a way it shouldn't be.
- **The "interface vs implementation size" test.** A high-cohesion, loosely-coupled module has an interface much smaller than its implementation. A low-cohesion, tightly-coupled module has an interface roughly the size of its implementation (because everything is exposed).
