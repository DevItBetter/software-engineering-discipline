---
name: software-design-principles
description: Senior-engineer guidance on software design — modularity, cohesion, coupling, abstraction, complexity, SOLID-with-nuance, DRY/KISS/YAGNI, naming, and readability. Use this whenever the task involves designing a new module, evaluating whether an abstraction is pulling its weight, deciding how to decompose a system into pieces, judging the quality of an existing design, naming things, or asking "is this a good design" / "should I extract this" / "is this the right level of abstraction" / "how should I split this up." Also use it when reviewing code with design implications, when refactoring decisions are being made, and whenever a design feels off but the cause is hard to articulate. This skill teaches the vocabulary to name what is wrong.
---

# Software Design Principles

Software design is the discipline of choosing what to make explicit, what to hide, what to couple, and what to keep separate — under uncertainty, with incomplete information, knowing the design will be wrong in some way and need to evolve.

The job is not to apply principles. The job is to make a system that costs less to change tomorrow than it does today. Principles are tools for diagnosing and explaining what's getting in the way of that.

This skill gives you the vocabulary, the diagnostics, and the trade-offs. The vocabulary matters because "this feels off" is not actionable; "this is a shallow module that's leaking implementation through its interface" tells the author exactly what to change.

## The one thing that matters most: complexity

Per Ousterhout's *A Philosophy of Software Design*: **a system is complex when it is hard to understand and hard to change.** That's it. Every other principle exists to attack one of those two.

Complexity has two causes:

1. **Dependencies** — a piece of code cannot be understood, modified, or tested in isolation; you have to know about other things first. Dependencies create *change amplification* (one logical change requires edits in many places) and *cognitive load* (a developer must hold many things in mind at once).
2. **Obscurity** — important information is not obvious. The interface looks innocent, but there's an undocumented invariant; the function takes a `Config`, but only three of its 15 fields are read. Obscurity creates *unknown unknowns* — the developer doesn't know what they need to know.

When evaluating a design, ask: **does this reduce dependencies and obscurity, or does it just rearrange them?**

A module that hides a complicated implementation behind a simple, honest interface reduces complexity. A module that adds three layers of indirection so that the same complexity is now spread across more files reduces nothing — it adds it.

See `references/complexity-and-deep-modules.md` for the full Ousterhout framing.

## The most important design idea: deep modules

A **deep module** has a small, simple interface that hides a large amount of useful work. The Unix file API (`open`, `read`, `write`, `close`, `seek`) is the canonical example: five functions, behind which sit filesystems, page caches, permission systems, journaling, NFS, FUSE, and decades of engineering.

A **shallow module** has a wide or complicated interface that hides little. A wrapper class with one public method per private method is the canonical anti-example. So is a "configuration" object with 30 fields each of which has its own getter/setter.

The "depth" of a module is roughly: **functionality hidden ÷ interface surface**. Maximize depth. Avoid creating modules whose only contribution is to require their callers to learn another vocabulary.

This is the single most reliable test of whether an abstraction is pulling its weight. If the abstraction is shallow — if you have to learn its API and *also* understand the thing underneath to use it correctly — it is making things worse.

See `references/complexity-and-deep-modules.md`.

## Simple is not the same as easy

From Rich Hickey's *Simple Made Easy*: simplicity and ease are different things and conflating them produces bad designs.

- **Simple** means *one fold* — not interleaved with other concerns. The opposite of simple is **complex** (literally "braided together"). Simplicity is objective, structural, and about what a thing *is*.
- **Easy** means *near at hand* — familiar, available, requires little to start. Easy is subjective and contextual.

Many things that are *easy* are *not simple*. ORMs are easy; they complect (braid together) data access, schema knowledge, transaction control, lazy loading, and identity. Mutable shared state is easy; it complects time, identity, and value.

Many things that are *simple* are *not easy*. Pure functions over immutable data are simple; they take effort to learn and structure your code around.

When designing, prefer simple even when it's not easy. The cost of "easy" compounds; the cost of "learning the simple thing" is paid once.

A heuristic from Hickey: **what does this complect?** If the answer is "many things," try to disentangle them. State complects time and identity. Inheritance complects type, behavior, and identity. Frameworks often complect structure, lifecycle, and configuration.

See `references/the-simple-vs-easy-distinction.md`.

## Cohesion and coupling

Two ideas, originally from Stevens, Myers, and Constantine in the 1970s, that have not been improved upon:

- **Cohesion** is the degree to which the things inside a module belong together. High cohesion: a module does one thing, and everything in it serves that thing. Low cohesion: the module is a junk drawer.
- **Coupling** is the degree to which a module depends on other modules. Loose coupling: the module communicates through narrow, stable interfaces. Tight coupling: the module reaches into another module's internals, depends on its data structures, calls its private methods, depends on its file layout.

The targets:

- **High cohesion within modules.** Things that change together live together.
- **Loose coupling between modules.** Things that change for different reasons depend on each other through stable interfaces, not implementation details.

This is the entire content of "Single Responsibility Principle" stated honestly. SRP is "high cohesion + low coupling, named." See `references/cohesion-and-coupling.md`.

A useful diagnostic when something feels wrong: **do you keep changing the same files for unrelated reasons?** That's low cohesion (divergent change). **Do you keep needing to change many files for one logical change?** That's tight coupling (shotgun surgery). Each is a smell with a known fix in `refactoring-and-code-smells`.

## Information hiding (Parnas, 1972)

The criterion for decomposing a system into modules is **what each module hides**. Modules should be designed around the **decisions most likely to change**, with each module hiding such a decision behind a stable interface.

This is older than SOLID, deeper than SOLID, and produces better designs than SOLID when applied honestly. The questions to ask of any proposed module:

- What decision is hidden behind this interface?
- Is that decision *likely to change*?
- If it changes, does the change stay inside this module?

If the answers are "nothing in particular," "no," and "no," the module isn't doing the work modules are supposed to do.

The application: **expose intent, hide mechanism.** A module's interface should let callers express *what* they want; the module decides *how*. Anything that exposes the *how* (storage layout, transport protocol, caching strategy, threading model) is a leak.

## SOLID — useful as a diagnostic, dangerous as a checklist

SOLID is a memorable acronym. Applied as a diagnostic — "is the dependency direction wrong here?" — it is useful. Applied as a checklist — "every class must have an interface, every interface must be split, every dependency must be injected" — it produces over-engineered, indirection-heavy code that is harder to read and harder to change.

The honest version:

- **Single Responsibility** — a module should have one *reason to change*. Not "do one thing." Not "have one method." A reason to change is a stakeholder, a domain area, an axis of variation. Mixing presentation with persistence is two reasons; two methods on related data is one. (Dan North calls SRP "pointlessly vague" — it is, when stated as "one responsibility." It is sharp when stated as "one reason to change.")
- **Open/Closed** — extension points should reduce churn at predictable variation points. They should not be added speculatively. Most code is closed-by-default and that is correct.
- **Liskov Substitution** — subtypes/implementations must honor the base contract: same errors, same side effects, same ordering, same nullability. Violations of LSP are real bugs, not academic.
- **Interface Segregation** — callers should not depend on methods they don't use. The smell is "I had to mock five methods to test this even though my code only calls one."
- **Dependency Inversion** — high-level policy should not depend on volatile low-level details. The valuable case is at architectural seams (domain ↔ database, domain ↔ HTTP transport). Inverting every dependency is what Dan North calls "DI obsession."

When you cite SOLID, cite it sharply. "This violates LSP because the override silently swallows the exception that the base class documents" is useful. "This needs more SOLID" is not.

See `references/solid-with-nuance.md` for the principles done right and the common ways they are misapplied.

## DRY, KISS, YAGNI, POLA — the famous principles, done right

These are widely quoted and widely misapplied. The honest versions:

**DRY (Don't Repeat Yourself).** From *The Pragmatic Programmer*: "Every piece of knowledge must have a single, unambiguous, authoritative representation within a system." DRY is about *knowledge*, not *characters*. Two functions with the same five lines that change for different reasons are not violating DRY. Two functions that encode the same business rule and must stay synchronized *are* violating DRY, even if they look different. The fix to a real DRY violation is to put the rule in one place. The fix to fake DRY (pure visual similarity) is to leave it alone — premature DRY couples unrelated concepts.

**KISS (Keep It Simple).** Choose the simpler design unless you have a concrete reason to choose the complex one. The reason must be more specific than "we might need flexibility." (See YAGNI.) Note that "simple" here means Hickey's simple — un-complected — not "few lines of code." A 500-line function can be simpler than a 50-file inheritance hierarchy.

**YAGNI (You Aren't Gonna Need It).** Don't build for hypothetical future requirements. Build for the requirements in front of you, in a way that doesn't make future change harder. The tax of carrying speculative features and abstractions is paid every day until they're either used or removed; the cost of adding a needed feature later is usually small if the design isn't actively obstructing it. (Note that YAGNI is about *features* and *speculative abstractions* — it is not an excuse for shipping code that's known-bad on a hot path.)

**POLA (Principle of Least Astonishment).** A function, type, or API should behave the way its name and signature suggest. Surprising behavior — silent failures, hidden side effects in getters, magic global state, non-obvious defaults — destroys the reader's ability to reason locally. Surprise is the most expensive thing in a codebase, because it forces every reader to verify their assumptions before they can trust anything.

See `references/dry-kiss-yagni-pola.md`.

## Abstractions that pay rent

An abstraction has a cost: every reader must learn it before they can read the code that uses it. The abstraction has to be worth that cost.

An abstraction pays rent when:

- It hides genuine complexity behind a simpler interface (deep module).
- It captures a real domain concept that recurs (Ubiquitous Language, per DDD).
- It enforces an invariant that would otherwise have to be re-checked at every call site.
- It enables a real, current variation point (multiple implementations exist *now*, not "someday").

An abstraction does not pay rent when:

- It has one consumer and the second is hypothetical.
- It exists because "we should have an interface" with no concrete reason.
- It hides nothing — the interface is the same shape as the implementation.
- It is named with a generic word like `Manager`, `Handler`, `Helper`, `Service`, `Util`, `Common`, `Wrapper`, `Strategy`, or `Factory` with no domain noun. (Generic names are a tell that the abstraction has no real concept behind it.)

When in doubt: **inline the implementation, ship it, extract when you have a second concrete consumer.** This is what Greg Young, Sandi Metz, and many others advocate, and it is repeatedly validated. Premature abstraction is more expensive than duplication.

See `references/abstractions-that-pay-rent.md`.

## Naming and readability

Names are the API of your code to its readers. Bad names cost more than any other single readability issue, because they are everywhere and they cause every reader to mentally re-check what each thing means.

A good name:

- Is **specific to the domain**, not generic. `customerId` beats `id`. `pendingOrders` beats `data`.
- Reveals **intent**, not mechanism. `isEligibleForRefund` beats `checkRefundFlag`. `currentRetryAttempt` beats `tmpInt`.
- Is **honest**. `getUser` should not delete the user; `validateOrder` should not silently mutate the order; `calculate` should not hit the network.
- Is **searchable**. `OrderStatus.PENDING_REVIEW` beats `4`. Magic values without names destroy the reader's ability to grep.
- **Matches the codebase's conventions**. If everywhere else uses `customer`, don't introduce `client` for the same concept.
- **Reads naturally at the call site.** `if (user.canEdit(post))` reads. `if (Permissions.check(user, post, ACTION_EDIT))` does not.

A bad name to flag in review (always):
- Names that lie. (Function does more than the name says.)
- Names that pun. Same word for two concepts in the same codebase.
- Names that include type info redundantly. `userList`, `customerObj`, `boolFlag`. Let the type system do that.
- Names that abbreviate domain concepts. `cust`, `ord`, `usr`. Saves keystrokes; costs every reader.
- Generic placeholders that escaped review. `tmp`, `data`, `info`, `manager`, `helper`, `util`.

See `references/naming-and-readability.md` for the depth treatment, including function shape, function length, and the "comments for why, code for what" principle.

## When to extract a function vs. when to leave inline

A perennial design question. The honest heuristic:

**Extract when the inline code has a name worth giving it.** If you can describe the inline block with a precise, intent-revealing name — `isAuthorizedForRefund`, `normalizePhoneNumber` — extract. The name is the abstraction; extraction makes the call site self-documenting.

**Don't extract when the only available name is mechanical.** `doFirstStep`, `helperForLoop`, `processItem` — these are signs that the extraction is purely physical, not conceptual. The reader gains nothing by jumping; they lose context.

The Sandi Metz rule of "≤5 lines per method" is a useful provocation but applied dogmatically it produces functions named for their position in another function. Use the rule as a *prompt* — "is there a real concept here that wants extracting?" — not as a constraint.

## Encapsulate what varies, not what doesn't

A foundational design heuristic. The parts of your system that change frequently or that have alternative implementations are the ones that should hide behind interfaces. The parts that are stable can be used directly.

Examples:
- Storage backend (likely to change) → repository / gateway interface.
- HTTP transport (changes per environment) → adapter.
- Pricing rules (change frequently per business) → strategy or rule engine.
- Standard library data structures (don't change) → use directly. No `IList<T>` wrapping `List<T>` wrapping `ArrayList`.

Inverting a stable dependency adds indirection cost with no flexibility benefit. Inverting a volatile dependency lets the volatile thing change without dragging the rest of the system with it.

## Conway's Law applies to design

> "Organizations which design systems are constrained to produce designs which are copies of the communication structures of these organizations." — Melvin Conway, 1968.

If two teams own the two halves of a system, the seam between those halves will be the most stable boundary in the design — even if it's the wrong boundary. Conversely, parts of the system owned by one team will be tightly coupled even when they shouldn't be.

When designing, consider the team topology. A module boundary that doesn't match a team boundary will erode under cross-team friction. A team boundary that doesn't match a module boundary will produce coordinated changes across teams (a smell).

This is not a defeatist law; it's actionable. Sometimes the right design move is to change the team structure (or the team's API) before changing the code structure.

## When the design is wrong

Signs that a design is fighting you:

- One change requires touching many unrelated files (low cohesion or shotgun surgery).
- Many small changes require touching the same file for unrelated reasons (god module, divergent change).
- Tests for one piece of behavior require setting up many other pieces (tight coupling).
- The same logic exists in multiple places and you keep forgetting to update one of them (real DRY violation).
- New developers can't find where to make a change (low discoverability — module names don't match concepts).
- Reading one function requires opening five others to understand what it does (no real abstraction; just code spread thin).
- The "obvious" place to put a new feature isn't where it ends up going (the design's mental model doesn't match the domain).

When you see these in review, the right response is often "this needs a design conversation, not a line-comment fix." See `engineering-discipline` for how to escalate from review to design.

## Reference library

- `references/complexity-and-deep-modules.md` — Ousterhout's framing in depth: complexity, dependencies, obscurity, deep vs shallow modules, define-errors-out-of-existence, tactical vs strategic.
- `references/the-simple-vs-easy-distinction.md` — Hickey's framing: complecting, what each common construct complects, the test of "what does this make harder to change later."
- `references/cohesion-and-coupling.md` — Stevens/Myers/Constantine and Parnas, with modern restatement.
- `references/solid-with-nuance.md` — each principle done right, with the common misapplications and the criticism literature (Dan North, DHH, Hickey).
- `references/dry-kiss-yagni-pola.md` — the famous principles done sharply, with the common misuses.
- `references/abstractions-that-pay-rent.md` — when an abstraction earns its keep, and the names that signal it doesn't.
- `references/naming-and-readability.md` — names, function shape, comments-for-why, the squint test, idiomatic local consistency.
- `references/conways-law-and-team-topology.md` — how organization shapes design, and when to change the org.
    