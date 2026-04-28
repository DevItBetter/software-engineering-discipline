---
name: refactoring-and-code-smells
description: Recognize code smells and apply the right refactoring move. Use this skill whenever the task involves refactoring, restructuring, cleaning up, modernizing, or untangling existing code — including questions like "how should I refactor this", "is this a code smell", "how do I break up this class/function", "how do I make this testable", "how do I get this under control", "what's wrong with this code", and reviews where the change would benefit from being preceded by a refactor. Also use it any time you're tempted to add complexity to existing complex code — the answer is often "refactor first, then make the easy change." Built on Fowler's Refactoring catalog (2nd ed.), with concrete smell→refactoring mappings and the discipline of when to refactor and when to leave alone.
---

# Refactoring and Code Smells

Refactoring is changing the structure of code without changing its behavior. The point is not aesthetics — it is to leave the code in a state where the *next* change is easy. Kent Beck's formulation: **make the change easy, then make the easy change.**

This skill teaches you (a) the canonical smell vocabulary so you can name what's wrong, (b) the catalog of moves to fix it, (c) the discipline of when to refactor and when to defer, and (d) how to do it safely.

## When to refactor

Three legitimate triggers, in priority order:

1. **You're about to make a change and the current code makes that change hard.** Refactor first to make the change easy, then make the easy change. This is the most common and most valuable refactor — it has a concrete payoff (the change is now possible), so it's not speculative.

2. **You see a code smell while you're already in the file for another reason.** The "boy scout rule": leave it cleaner than you found it. Small, targeted refactors with the next change.

3. **A specific design defect is causing a recurring cost** — bug after bug clusters in the same module, on-call keeps getting paged for the same kind of failure, every change to feature X requires touching files A, B, C, D. Refactor with intent and a clear hypothesis about what cost will go down.

Bad triggers:

- "This doesn't match my preferred style." Not a refactor; a re-skin.
- "I read about a pattern I want to try." Don't.
- "This is old." Age is not a smell. Working code that hasn't been touched in three years and works correctly is *good*.
- "I want to apply SOLID more rigorously." Apply SOLID as a diagnostic, not as a renovation project. See `software-design-principles` for SOLID-with-nuance.

## How to refactor safely

The discipline that separates refactoring from "rewriting and hoping":

1. **Have a green test suite first.** Refactoring without tests is rearrangement; you can't tell if you broke anything. If the code has no tests, your first refactor is to add characterization tests (Michael Feathers' term — tests that capture the current behavior, even if it's wrong, so you can detect changes).

2. **Make one small move at a time.** Each move is one of the named refactorings (Extract Function, Inline Variable, Move Method, Replace Conditional with Polymorphism, etc.). After each move, run the tests. If they fail, undo and try a smaller step.

3. **Don't mix refactors with behavior changes.** If you find a bug while refactoring, finish the refactor (or revert it), commit, then fix the bug as a separate change. Two PRs. Mixing them makes both harder to review and harder to bisect.

4. **Commit frequently.** Each successful step gets a commit. Fewer "rebase nightmare" outcomes.

5. **For larger refactors, use the Mikado Method.** When you start a refactor and discover it requires another refactor first, which requires another refactor first, write a tree of dependencies. Do the leaves first, working back to the root. This avoids the half-done-everything state.

6. **Use the IDE / language tools.** Renames, extractions, and moves done by an IDE are mechanically correct. Done by hand they're a bug source.

## The smell catalog

From Fowler's *Refactoring* (2nd ed.), grouped. Each smell maps to one or more refactorings.

### Bloaters — code that has grown too large

- **Long Function.** A function that has accumulated responsibilities. *Refactorings: Extract Function, Decompose Conditional, Replace Temp with Query.*
- **Long Parameter List.** Too many parameters; easy to mix up; hard to remember at the call site. *Refactorings: Introduce Parameter Object, Preserve Whole Object, Replace Parameter with Query.*
- **Large Class.** A class that's grown to do too much. *Refactorings: Extract Class, Extract Subclass, Extract Superclass.*
- **Primitive Obsession.** Domain concepts represented as raw strings, ints, or maps. (`emailAddress: string` invites passing any string; `EmailAddress` value type forbids invalid emails by construction.) *Refactorings: Replace Primitive with Object, Replace Type Code with Subclasses, Replace Conditional with Polymorphism.*
- **Data Clumps.** Same group of values travels together as parameters. They want to be a class. *Refactoring: Introduce Parameter Object, Extract Class.*

### Object-orientation abusers

- **Switch Statements / `isinstance` chains over a closed set of types.** Behavior dispatching on type. *Refactorings: Replace Conditional with Polymorphism, Replace Type Code with Subclasses, Replace Conditional with Strategy.*
- **Repeated Switches.** The same conditional structure appears in multiple places. Now adding a new case requires touching all of them. *Refactoring: Replace Conditional with Polymorphism.*
- **Refused Bequest.** A subclass overrides parent methods to do nothing or throw. The "is-a" relationship doesn't hold. *Refactorings: Push Down Method/Field, Replace Subclass with Delegate.*
- **Alternative Classes with Different Interfaces.** Two classes do similar things but their methods are named differently. *Refactorings: Rename Function, Move Function, Extract Superclass.*

### Change preventers — design that makes change expensive

- **Divergent Change.** One module changes for many unrelated reasons. (Low cohesion.) *Refactorings: Split Class, Split Phase, Move Function.*
- **Shotgun Surgery.** One conceptual change requires touching many modules. (Tight coupling, missing abstraction.) *Refactorings: Move Function, Move Field, Combine Functions into Class, Inline Class.*
- **Parallel Inheritance Hierarchies.** Adding a subclass to one hierarchy forces adding a corresponding subclass in another. *Refactoring: Move Function, Move Field to fold one into the other.*

### Dispensables — code that adds no value

- **Comments narrating code.** The comment explains what the next line does, not why. *Refactoring: Extract Function (give it the name the comment described), Rename Variable.*
- **Duplicate Code.** Same logic in multiple places. *Refactorings: Extract Function, Pull Up Method, Form Template Method.*
- **Lazy Class.** A class that doesn't do enough to justify existing. *Refactorings: Inline Class, Collapse Hierarchy.*
- **Data Class.** A class that holds data with no behavior. Often fine in some idioms (records, value objects). Smell when the data has obvious operations that should be on it but live elsewhere — Tell, Don't Ask. *Refactorings: Move Function, Encapsulate Record.*
- **Dead Code.** Unreachable, unused, commented-out. *Refactoring: Delete it. Version control remembers.*
- **Speculative Generality.** Hooks, parameters, abstractions for hypothetical future needs. *Refactorings: Inline Function, Inline Class, Collapse Hierarchy, Remove Dead Code.*

### Couplers — code that knows too much about other code

- **Feature Envy.** A method uses another object's data more than its own. *Refactorings: Move Function, Extract Function.*
- **Inappropriate Intimacy.** Two classes know each other's private details. *Refactorings: Move Function, Move Field, Change Bidirectional Association to Unidirectional, Replace Inheritance with Delegation.*
- **Message Chains.** `a.b().c().d().e()` — caller knows the entire object graph. *Refactoring: Hide Delegate.*
- **Middle Man.** A class that mostly delegates to another. *Refactoring: Remove Middle Man, Inline Function.*
- **Insider Trading.** Modules trade information through back-channels. Make the dependencies explicit and narrow.

### Other important smells (not in Fowler's original five buckets)

- **Mysterious Name.** The name doesn't tell you what the thing does. *Refactoring: Rename. The most common and most valuable refactor; fix the moment you notice.*
- **Global Data / Mutable State.** Shared mutable state with no clear ownership. *Refactoring: Encapsulate Variable, Replace Magic Literal, sometimes substantial restructuring.*
- **Loops where you wanted a pipeline.** Long imperative loops doing filter/map/reduce by hand. *Refactoring: Replace Loop with Pipeline.*
- **Boolean Trap.** Functions with several boolean parameters; combinatorial behavior at call sites. *Refactorings: Split Function (one per behavior), Replace Boolean with Enum, Introduce Parameter Object.*
- **Temporary Field.** A class field used only sometimes. *Refactoring: Extract Class, Introduce Special Case.*

## The most useful individual refactorings

Most refactoring is small moves. The ones to know cold:

- **Extract Function.** Pull a piece of code out and give it a name. The single most-used refactoring. The new function's *name* is the abstraction; if you can't name it, don't extract.
- **Inline Function.** The opposite. When a function is just a confusing layer over its body, fold it back in.
- **Rename Variable / Function / Class.** The most underrated refactor. A better name eliminates the need for a comment, makes the call site self-documenting, and improves grep. Do this constantly.
- **Move Function / Field.** When behavior or data lives in the wrong place. Often follows from a Feature Envy diagnosis.
- **Extract Class.** Carve out a coherent subset of a class's responsibilities into a new class.
- **Introduce Parameter Object.** When parameters travel together, give them a name as a struct/class.
- **Replace Conditional with Polymorphism.** When `if isinstance(...)` chains dispatch on type, replace with virtual methods on the type.
- **Replace Magic Literal.** Give the constant a name. `MIN_ORDER_AMOUNT = 5` rather than `5`.
- **Encapsulate Variable / Record.** Convert direct access to access through accessors so behavior can be added later.
- **Split Phase.** When a function does X-then-Y where X produces an intermediate that Y consumes, split into two functions with the intermediate explicit. Often clarifies a confused function dramatically.
- **Replace Loop with Pipeline.** A long imperative loop becomes `filter().map().reduce()` (or the language's equivalent). Often a big readability win when the operations are pure.

See `references/refactoring-catalog.md` for an extended catalog with examples.

## When NOT to refactor

- **You're about to ship a hot fix.** Fix, ship, refactor in a follow-up. Refactoring under deadline pressure is when bugs slip in.
- **The code is going to be deleted soon.** Refactoring code that's being decommissioned next quarter is wasted work.
- **You don't understand what the code does.** Read first. Add tests that capture current behavior. *Then* refactor. Refactoring code you don't understand is "rewriting and hoping."
- **The code works, isn't a bottleneck, and you're not going to touch the surrounding area.** Don't pick fights with code that's leaving you alone.
- **You'd be doing it during a code review of someone else's PR.** Wrong scope, wrong moment. File a follow-up.

## The Mikado Method (for big refactors)

When a desired change requires a prerequisite change, which requires a prerequisite change, which requires...

1. Try the goal change directly. Note what breaks.
2. Don't fix the breaks; revert. Write down the prerequisite changes you discovered.
3. Try the first prerequisite. If it's still too big, recurse — note its prerequisites and revert.
4. Eventually you reach a leaf change that you can do without breaking anything. Do it. Commit.
5. Try the next-leaf-up. Now it works (because its dependency is in place). Commit.
6. Work back up the tree. The original goal becomes possible.

This avoids the "I started this refactor three days ago and now nothing compiles" failure mode.

## Connecting to other skills

- For *recognizing* whether a smell is real, see `software-design-principles` (especially `complexity-and-deep-modules.md` and `cohesion-and-coupling.md`).
- For *naming* problems sharply when you find them in review, see `code-review-best-practices` (`red-flag-patterns.md`).
- For test discipline that makes refactoring safe, see `testing-discipline`.

## Reference library

- `references/refactoring-catalog.md` — extended catalog of moves, with the smell that triggers each and a brief example.
- `references/legacy-code-strategies.md` — how to add tests to untested code, the "seam" concept, characterization tests; condensed Feathers.
