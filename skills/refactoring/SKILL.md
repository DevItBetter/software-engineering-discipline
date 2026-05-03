---
name: refactoring
description: "How to refactor safely and effectively — recognize when, choose the right named move, work in small reversible steps, attack legacy code with seams and characterization tests. Use this skill whenever the task is changing the structure of code without changing its behavior, including questions like \"how should I refactor this\", \"how do I break up this class/function\", \"how do I make this testable\", \"how do I get this under control\", and reviews where the change would benefit from being preceded by a refactor. The companion skill `code-smells-and-antipatterns` is the diagnostic side (finding what's wrong, with concrete examples); this skill is the corrective side (the moves, the process, the discipline). Built on Fowler (Refactoring 2nd ed.), Beck (Tidy First?), Feathers (Working Effectively with Legacy Code)."

---

# Refactoring (Techniques and Discipline)

Refactoring is changing the structure of code without changing its behavior. The point is not aesthetics — it is to leave the code in a state where the *next* change is easy. Kent Beck's formulation: **make the change easy, then make the easy change.**

This skill is the *corrective* side of code health: the catalog of named moves, the discipline of small steps, and how to attack untested legacy code safely. For the *diagnostic* side — naming what's wrong, with concrete code examples — see the sibling skill `code-smells-and-antipatterns`. The two skills are designed to be used together.

## When to refactor

Three legitimate triggers, in priority order:

1. **You're about to make a change and the current code makes that change hard.** Refactor first to make the change easy, then make the easy change. This is the most common and most valuable refactor — it has a concrete payoff (the change is now possible), so it isn't speculative.

2. **You see a smell while you're already in the file for another reason.** The "boy scout rule": leave it cleaner than you found it. Small, targeted refactors with the next change.

3. **A specific design defect is causing recurring cost** — bug after bug clusters in the same module, every change to feature X requires touching files A/B/C/D, on-call keeps getting paged for the same kind of failure. Refactor with intent and a clear hypothesis about what cost will go down.

Bad triggers:

- "This doesn't match my preferred style." Not a refactor; a re-skin.
- "I read about a pattern I want to try." Don't.
- "This is old." Age is not a smell. Working code that hasn't been touched in three years and works correctly is *good*.
- "I want to apply SOLID more rigorously." Apply SOLID as a diagnostic, not a renovation project. See `software-design-principles`.

## Tidy First? (Kent Beck)

Beck's recent framing: when you want to make a change that touches messy code, you have three options.

- **Tidy first**: clean up, then change. One PR for the cleanup, one for the behavior change. Cleanup is reviewable in isolation; the behavior change is reviewable on tidy code.
- **Change first**: make the behavior change in the messy code, then clean up. Sometimes the right call when the behavior change is small and the cleanup might churn against in-flight work.
- **Don't tidy**: just change. Acceptable when you'll never touch this code again, or when the cleanup cost exceeds the value.

The disciplines:
- Keep tidying and behavior changes separate by default, ideally in separate commits and often separate PRs. Mixing them increases review and bisect cost; only combine them when the cleanup is tiny, local, and necessary to make the behavior change understandable.
- Each tidying is small enough that "did I break anything" has a fast answer (the test suite passing).
- Tidyings are **reversible** — if the tidy turns out to be wrong, you can roll back without losing the behavior change.

## How to refactor safely

The discipline that separates refactoring from "rewriting and hoping":

1. **Have a green test suite first.** Refactoring without tests is rearrangement; you can't tell if you broke anything. If the code has no tests, your first refactor is to add **characterization tests** — tests that capture current behavior, even if it's wrong, so you can detect changes.

   A characterization test is dumb on purpose. You aren't asserting what the code *should* do; you're snapshotting what it *currently* does, so a refactor that accidentally changes behavior is caught.

   ```python
   # The code has no tests. You need to refactor calculate_pricing.
   # Don't refactor yet. First, characterize:

   def test_characterize_calculate_pricing_basic():
       result = calculate_pricing(items=[{"sku": "A", "qty": 2, "price": 10}], customer_tier="standard")
       assert result == {"subtotal": 20, "discount": 0, "tax": 1.6, "total": 21.6}
       # ^ You didn't compute this; you ran the function and pasted what it returned.

   def test_characterize_calculate_pricing_gold_tier():
       result = calculate_pricing(items=[{"sku": "A", "qty": 2, "price": 10}], customer_tier="gold")
       assert result == {"subtotal": 20, "discount": 2, "tax": 1.44, "total": 19.44}

   # Now you have a safety net. Refactor; the test fails if behavior changes.
   ```

   If a characterization test surfaces a bug ("wait, that tax is wrong"), don't fix it during the refactor. Note it; finish the refactor (with the wrong-but-stable behavior); fix the bug as a separate change with the test updated.

2. **Make one small move at a time.** Each move is one of the named refactorings (Extract Function, Inline Variable, Move Method, Replace Conditional with Polymorphism, etc.). After each move, run the tests. If they fail, undo and try a smaller step.

3. **Don't mix refactors with behavior changes.** If you find a bug while refactoring, finish the refactor (or revert it), commit, then fix the bug as a separate change. Two PRs. Mixing makes both harder to review and harder to bisect.

4. **Commit frequently.** Each successful step gets a commit. Fewer "rebase nightmare" outcomes. Each commit message says what was refactored and why ("Extract Function: pull out customer normalization so we can reuse it in the import path").

5. **For larger refactors, use the Mikado Method.** When you start a refactor and discover it requires another refactor first, which requires another refactor first, write a tree of dependencies. Do the leaves first, working back to the root. Avoids the half-done-everything state.

6. **Use the IDE / language tools.** Renames, extractions, and moves done by an IDE are mechanically correct. Done by hand they're a bug source. For dynamic languages (Python, JS), test coverage matters even more because the IDE has less to work with.

## The refactoring catalog (working subset)

For each smell, the standard moves. The smells themselves — with concrete before/after code examples — live in the sibling `code-smells-and-antipatterns` skill.

### Composing methods

- **Extract Function** — pull a coherent section into a new function with a name. The single most-used refactoring. Don't extract if the only available name is mechanical.
- **Inline Function** — when a function is just a confusing layer over its body, fold it back in.
- **Extract Variable** — pull an expression into a named variable; the name documents the meaning.
- **Inline Variable** — when the variable is a thin alias for a clearer expression.
- **Change Function Declaration** — rename, reorder parameters, change parameter type. IDE-assisted when possible.
- **Encapsulate Variable** — replace direct field access with accessors so behavior can be added on access.
- **Rename Variable / Function / Class** — the most underrated refactor. Do it constantly.
- **Introduce Parameter Object** — when parameters travel together, name the bundle.
- **Combine Functions into Class** — several functions on the same data want to be a class.
- **Combine Functions into Transform** — several functions deriving values from the same input → one transform.
- **Split Phase** — when one function does X-then-Y with a tangled intermediate, split into two with the intermediate explicit.

### Moving features

- **Move Function / Method** — Feature Envy fix; behavior moves to where its data lives.
- **Move Field** — field moves to where it's mostly used.
- **Move Statements into / out of Function** — setup or cleanup that always wraps a call belongs inside it (or vice versa).
- **Slide Statements** — group related statements together; precursor to extraction.
- **Replace Inline Code with Function Call** — when the function already exists.

### Organizing data

- **Replace Primitive with Object** — domain concept gets a type. `EmailAddress` not `string`; `Money` not `decimal`; `OrderId` not `string`. The type encodes the rules.
- **Replace Type Code with Subclasses** — class with a `kind` field and conditional behavior → subclasses with polymorphism.
- **Replace Type Code with State/Strategy** — same situation but the type can change at runtime.
- **Encapsulate Collection** — return an unmodifiable view; expose `add`/`remove` methods that enforce invariants.

### Simplifying conditional logic

- **Decompose Conditional** — extract complex `if`/`else` bodies into named functions.
- **Consolidate Conditional Expression** — multiple `if`s with the same body merge into one.
- **Replace Nested Conditional with Guard Clauses** — preconditions become early returns; happy path lives at natural indentation.
- **Replace Conditional with Polymorphism** — type-based dispatch becomes virtual methods.
- **Introduce Special Case (Null Object)** — provide an object representing absence/special case; callers don't check.
- **Replace Error Code with Exception (and vice versa)** — match the language idiom and the situation.
- **Replace Exception with Precheck** — when the exception is being used to signal a condition the caller could test.

### Refactoring APIs

- **Separate Query from Modifier** — split functions that both return and mutate.
- **Parameterize Function** — combine near-duplicates with a parameter for the difference.
- **Remove Flag Argument** — boolean parameters split into named functions.
- **Preserve Whole Object** — pass the object instead of pulling fields off it (use judgment — can become Stamp Coupling).
- **Replace Parameter with Query** — derive instead of pass when the function has access.
- **Replace Query with Parameter** — pass instead of derive when isolation/testability matters.
- **Remove Setting Method (Make Immutable)** — set in the constructor only.
- **Replace Constructor with Factory Function** — when construction is non-trivial.

### Dealing with inheritance

- **Pull Up Method / Field** — duplicated in subclasses → in the superclass.
- **Push Down Method / Field** — used by some subclasses → only in those.
- **Replace Subclass with Delegate** — inheritance was for code reuse, not is-a → composition.
- **Replace Inheritance with Delegation** — extension only used a few methods; hold the parent as a field.
- **Replace Superclass with Delegate** — same medicine the other direction.

## The Mikado Method (for big refactors)

When a desired change requires a prerequisite change which requires a prerequisite change…

1. Try the goal change directly. Note what breaks.
2. Don't fix the breaks; revert. Write down the prerequisite changes you discovered as a tree.
3. Try the first prerequisite. If it's still too big, recurse — note its prerequisites and revert.
4. Eventually you reach a leaf change you can do without breaking anything. Do it. Commit.
5. Try the next-leaf-up. Now it works (its dependency is in place). Commit.
6. Work back up the tree. The original goal becomes possible.

Avoids the "I started this refactor three days ago and now nothing compiles" failure mode.

## Working with legacy code (Feathers' framing)

Feathers' definition of legacy code: **code without tests.** Whether it's old or new, written by someone else or by you yesterday — if no test tells you it works, it's legacy.

The fundamental problem: to change legacy code safely you need tests; to add tests you usually need to change the code. The cycle: change-to-test, test-to-change.

The way out is **seams** — places where you can alter behavior in a test context without editing production code at that point. Common seams:

- **Object seams** — function takes a dependency as a parameter; tests pass a fake.
- **Subclass seams** — method is virtual; tests subclass and override.
- **Module-import seams** (Python, Ruby, JS) — replace the import in the test environment.
- **Link seams** (C/C++) — link a different implementation.

When code has no seams, the first move is to introduce one. The smallest possible change that creates a seam is the right starting point.

### Sprout and Wrap

When you can't safely refactor the legacy code itself, do new work alongside it and integrate at one point.

- **Sprout Method** — new behavior in a new method on the existing class, tested independently. Old method calls it.
- **Sprout Class** — new behavior in a new class, tested independently. The legacy class delegates to it. Use when the existing class is so tangled that adding a method to it requires understanding too much.
- **Wrap Method** — preserve the old method's name and signature; rename the existing implementation; the new method calls the old and adds the new behavior. Callers see the same surface; the new behavior runs around or before/after the old.
- **Wrap Class** (a.k.a. decorator) — wrap the legacy class in a new class with the same interface. The wrapper adds behavior; the legacy class is untouched. Useful when the new behavior needs to apply to many call sites without editing each.

The point is identical to seams: minimize the change to untested code. Sprout and Wrap let new code be tested cleanly without first earning the right to refactor the old.

### Characterization tests

Before changing untested legacy code, write tests that pin down its current behavior — not the behavior you wish it had, the behavior it actually has, including the bugs.

The procedure:

1. Write a test that exercises the code with a concrete input.
2. Assert what you *think* it should produce.
3. Run the test. Whatever it actually produces is the answer; update the assertion to that.
4. Repeat across the input space you care about.

You now have a safety net. You can refactor; if behavior changes, the tests fail and you know. Once the refactor is in and the seams are good, *then* you can decide which of the pinned behaviors are bugs and fix them deliberately, one at a time, with the tests as guide.

This is uncomfortable — you are codifying behavior you may know is wrong. Do it anyway. The alternative is changing the code blind.

## When *not* to refactor

Refactoring is not free. Skip or defer when:

- **The code is about to be deleted.** Don't tidy a module slated for removal in the next sprint.
- **You have no tests and can't add them.** Without a safety net, "refactor" becomes "rewrite-and-pray." Add characterization tests first or leave it.
- **The change is purely cosmetic and you're about to merge a feature.** Mixing tidying with behavior change makes the diff hard to review and hard to bisect. (Beck's *Tidy First?* discipline: tidy in a separate commit, ideally a separate PR.)
- **You don't understand the code yet.** Read it. Then refactor. Renaming things you don't understand turns one bug into many.
- **Someone else is in flight on the same code.** Coordinate, or wait.

## Reference library

- `references/refactoring-catalog.md` — extended catalog of named refactorings with mechanics, when each applies, and the order to do them.
- `references/legacy-code-strategies.md` — Feathers' seams, dependency-breaking moves, characterization-test patterns, and the change-point / inflection-point analysis for deciding where to invest.

## Sibling skills

- `code-smells-and-antipatterns` — the diagnostic side. Start there to name what's wrong; come here to fix it.
- `software-design-principles` — the underlying frame. Most refactorings exist to attack a cohesion, coupling, or complexity problem named there.
- `testing-discipline` — refactoring depends on tests. Characterization tests, mutation testing, and what-to-test guidance live there.
- `engineering-discipline` — how to communicate a refactor in a PR, when to escalate from refactor to redesign conversation.
