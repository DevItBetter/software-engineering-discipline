# Working with Legacy Code

Distilled from Michael Feathers, *Working Effectively with Legacy Code*. The honest definition of legacy code in this tradition: **code without tests.** Whether it's old or new, written by someone else or by you yesterday — if there's no test telling you it works, it's legacy.

## The fundamental problem

You want to change legacy code. To change it safely, you need tests. To test it, you usually need to change it (because untested code is often hard to test — entangled with globals, hard-coded dependencies, side effects in constructors, etc.). The dependency cycle: change-to-test, test-to-change.

The tradition's answer: find a **seam** — a place where you can alter behavior without editing the production code at that point — and use it to introduce tests, then refactor.

## Seams

A seam is a point where two pieces of code are connected, and where you can swap one for the other in a test context. Common seams:

- **Object seams.** A function takes a dependency as a parameter. In tests, pass a fake.
- **Preprocessor seams (C/C++).** `#ifdef TEST` blocks let test builds substitute different code.
- **Link seams.** Two implementations of the same library; the linker picks one.
- **Subclass seams.** A method is virtual; tests subclass to override it.
- **Module-import seams (Python, Ruby).** Replace the import in the test environment.

When code has no seams, your first move is usually to introduce one. The smallest possible change that creates a seam is the right starting point.

## The "sprout" and "wrap" patterns

When you can't safely refactor the legacy code itself, do the new work alongside it and integrate at one point.

- **Sprout Method.** New behavior goes into a new method on the existing class. Tested independently. Old method calls the new one.
- **Sprout Class.** New behavior goes into a new class. Tested in isolation. Old code instantiates and uses the new class at one well-defined point.
- **Wrap Method.** When you need to add behavior before/after an existing method, rename the old method, create a new method with the original name that calls the renamed-old plus the new behavior.
- **Wrap Class.** Wrap the legacy class in a new class that delegates and adds behavior.

Each pattern leaves the legacy code mostly alone (low risk) and concentrates the new behavior in tested, isolated code. The tested island can grow over time.

## Characterization tests

When you must change legacy code that has no tests:

1. **Write tests that capture the current behavior, even if the behavior is wrong.** The point is not to verify correctness — it's to detect change. If your refactor accidentally changes behavior, the characterization test will fail.
2. **Generate inputs systematically.** Boundary values, common cases, edge cases. Don't try to be clever — try to be comprehensive.
3. **Snapshot the outputs.** Record what the function currently produces. Use snapshot testing tools or just `assert output == "..."` with the literal value pasted in.
4. **Run the characterization tests.** They should pass (you captured the current behavior).
5. **Now refactor**, with the safety net.
6. **Once the refactor is complete, fix any actual bugs in a separate change** (with characterization tests updated to reflect the new correct behavior).

This is mechanical and ugly and exactly what you should do. Trying to refactor legacy code "by reading carefully" is how you ship subtle bugs.

## Common legacy code structures and how to attack them

### Big function with deeply embedded conditional logic

1. Identify a "section" of the function — a block bracketed by blank lines, a comment, or a logical step.
2. Extract Function on it. Run tests.
3. Repeat until the original function is a sequence of named calls.
4. Now you can see the structure. Often this reveals the right top-level design.

### God class with many responsibilities

1. Identify clusters of methods that operate on overlapping subsets of fields.
2. Each cluster is a candidate for Extract Class.
3. Pick the most isolated cluster first (smallest dependencies on the rest).
4. Move methods. Move fields. Run tests.
5. Now the original class is smaller; iterate.

### Static method that depends on global state

1. Identify the global dependencies the method uses.
2. Add a parameter for each (or a single context object). Default to the current global, so existing callers don't have to change.
3. Now the method is testable: pass a fake context.
4. Over time, migrate callers to pass the context explicitly. Eventually delete the default.

### Class with side effects in the constructor

(Constructor opens a database connection, makes an HTTP call, starts a thread.)

1. Prefer moving the side-effecting work into a factory function (`User.create(...)`) or injecting the already-created dependency.
2. Keep the constructor cheap: it stores parameters and establishes invariants, so the object can be constructed in tests without I/O.
3. If you must introduce a public `init()` as a temporary seam, treat it as technical debt: add tests for the two-phase lifecycle, keep production callers behind one factory, and remove the temporal coupling as soon as the seam has served its purpose.

### A class with a singleton dependency

1. Add a constructor parameter for the dependency. Default to the singleton (so existing callers don't change).
2. Now tests can pass a fake.
3. Over time, migrate callers to pass the dependency explicitly.

## When to rewrite vs refactor

Refactor when:
- The code mostly works.
- The structure is fixable by sequence of small changes.
- The team owns the code and has the bandwidth.

Rewrite when:
- The code is fundamentally wrong (uses the wrong data model, the wrong protocol, the wrong language for the job).
- Refactoring it would be more work than rewriting it from a clean specification.
- The behavior is well-understood (often *because* there's a long history of bugs and a clear specification of what it should do).

The wrong-pattern: rewrite because the existing code is "ugly" or "old." Joel Spolsky's "Things You Should Never Do, Part I" is the touchstone: rewrites are seductive and almost always more expensive than expected. The new code recreates the bugs of the old, lacks the accumulated knowledge of edge cases, and ships months later than promised. Only rewrite when the *substance* is wrong, not the *style*.

## The Strangler Fig pattern (for big migrations)

When a system needs to be replaced but can't be replaced atomically:

1. Build the new system alongside the old.
2. Route a small slice of traffic to the new system. Verify behavior.
3. Slice by slice, migrate functionality from old to new.
4. When the old system has no more responsibilities, decommission.

The pattern is named for the strangler fig tree, which grows around a host tree until the host dies. You don't take the system down; the new system gradually takes over.

For each migration step: keep the old code working until the new code has *fully* replaced it. Don't enter the half-done state where users have to know which system to use. The transition is invisible to them.

## What to flag in review

- A "rewrite" PR that doesn't have characterization tests of the old behavior. You don't know if you're matching it.
- A "refactor" PR that changes behavior. Two PRs.
- A new feature added to legacy code with no test of the new feature, on the basis of "the surrounding code has no tests." Wrong direction. The new code is the seed of the testable island.
- A migration that goes through a half-done state visible to users. Use Strangler.
- A rewrite that's larger than 2–3 weeks of work without behind-a-flag deployment to validate. You're shipping unverified code.
