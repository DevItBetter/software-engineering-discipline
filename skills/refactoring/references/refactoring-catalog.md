# Refactoring Catalog (Extended)

A working subset of Fowler's catalog (2nd ed.), grouped by what they do. Each entry: the smell that triggers it, and a sketch of the move.

## Composing methods

### Extract Function
**When:** A piece of code has a name worth giving it; or a function is too long; or you want to comment a section (the comment becomes the function name).
**Move:** Cut the section into a new function with parameters for what it needs and a return for what it produces. Replace the original with a call.
**Watch:** If the only available name is mechanical (`doThingTwo`, `helperLoop`), don't extract — the abstraction has no concept behind it.

### Inline Function
**When:** A function's body is as clear as its name — it adds nothing.
**Move:** Replace each call with the body. Delete the function.

### Extract Variable
**When:** An expression is hard to read or computed in multiple places nearby.
**Move:** Pull it into a named variable. The name is the documentation.

### Inline Variable
**When:** A variable is used once or twice and the expression is clearer.
**Move:** Replace uses with the expression. Delete the variable.

### Change Function Declaration (Rename / Reorder Params)
**When:** A function's name doesn't reflect what it does; or its parameters are in a confusing order.
**Move:** Rename, reorder. The IDE does this safely; manual is a bug source.

### Encapsulate Variable
**When:** A piece of data is accessed from many places and you need to add behavior (validation, derivation, observability) on access.
**Move:** Replace direct access with getter/setter (or property), then add the behavior in one place.

### Rename Variable
The single most-used and most-undervalued refactor. Do it constantly. A better name eliminates a comment, a confusion, or a bug.

### Introduce Parameter Object
**When:** A group of parameters travel together (data clump).
**Move:** Define a class/struct/record. Replace the parameter group with the new type at every call site.

### Combine Functions into Class
**When:** Several functions all operate on the same data. They want to be a class.
**Move:** Make them methods on a new class that holds the data.

### Combine Functions into Transform
**When:** Several functions derive different values from the same input. (Reads as duplicated parsing/setup.)
**Move:** A single transform function that produces the enriched record once.

### Split Phase
**When:** A function does X-then-Y, with X producing data that Y consumes — but the two phases are tangled.
**Move:** Make the intermediate explicit. The first phase produces it; the second phase takes it. Each phase is now testable separately.

## Moving features

### Move Function / Method
**When:** A method uses another class's data more than its own (Feature Envy).
**Move:** Put it where the data lives. Adjust the parameter list.

### Move Field
**When:** A field is read/written more by another class than the one it's on.
**Move:** Move it. Update accessors at the new home.

### Move Statements into Function / Out of Function
**When:** Setup or cleanup that always happens around a function call belongs inside it (or vice versa).

### Slide Statements
**When:** Related code is separated by unrelated statements.
**Move:** Move related statements together. Often a precursor to extraction.

### Replace Inline Code with Function Call
**When:** A piece of code already exists as a function elsewhere.
**Move:** Use the existing function. Watch for accidentally-different behavior.

## Organizing data

### Replace Primitive with Object
**When:** A primitive type (string, int) is used to represent a domain concept with rules.
**Move:** Define a value type. The type encodes the rules. Now invalid values can't exist.
Examples: `EmailAddress` instead of `string`; `Money` (with currency) instead of `decimal`; `OrderId` instead of `string` (avoids accidentally passing a customer ID).

### Replace Type Code with Subclasses
**When:** A class has an `enum`/`type` field and behavior conditional on it. Often a `kind` field.
**Move:** Replace with subclasses. Behavior dispatches via polymorphism.

### Replace Type Code with State/Strategy
**When:** Same situation but the type can change at runtime, so subclassing the whole object is wrong.
**Move:** Extract the variable behavior to a Strategy object that can be swapped.

### Encapsulate Collection
**When:** A class exposes a collection directly. Callers can mutate it; the owning class can't enforce invariants.
**Move:** Return an unmodifiable view. Add `add()`/`remove()` methods that enforce invariants.

## Simplifying conditional logic

### Decompose Conditional
**When:** A complex `if` body or `else` body is doing nontrivial work.
**Move:** Extract each branch into a named function. The conditional becomes `if (eligible) doRefund() else logIneligible()`.

### Consolidate Conditional Expression
**When:** Multiple `if` statements check the same precondition with the same body.
**Move:** Combine into one. Or extract the condition into a named predicate.

### Replace Nested Conditional with Guard Clauses
**When:** Many levels of nesting.
**Move:** Each precondition becomes a guard at the top of the function. The happy path ends up at the function's natural indentation.

### Replace Conditional with Polymorphism
**When:** Behavior dispatches on a type code via `if`/`switch`/`isinstance`.
**Move:** Each branch becomes a method on a subclass. Caller calls the method; dispatch is automatic.

### Introduce Special Case (Null Object)
**When:** Many call sites check for the absence/special case of a value.
**Move:** Provide an object that represents the special case and behaves sensibly. Callers use it without checking.

### Replace Error Code with Exception (and vice versa)
**When:** Error handling style doesn't match the language's idiom or the situation. Use exceptions for unusual conditions; use return values for expected outcomes.

### Replace Exception with Precheck
**When:** An exception is being used to signal a condition the caller could check first.
**Move:** Add a check method. Caller checks, then proceeds without try/except. Use exceptions for *exceptional* situations, not normal flow control.

## Refactoring APIs

### Separate Query from Modifier
**When:** A function both returns a value and mutates state. Easy to misuse.
**Move:** Two functions: one pure query, one mutator.

### Parameterize Function
**When:** Two functions are nearly identical except for one value.
**Move:** Combine into one function with an extra parameter.

### Remove Flag Argument
**When:** A boolean parameter switches between two behaviors.
**Move:** Two functions, one per behavior. Call sites are now self-documenting.

### Preserve Whole Object
**When:** A function takes several fields off an object as separate parameters.
**Move:** Pass the object. (Watch: can become Stamp Coupling if the function uses few of the object's fields. Use judgment.)

### Replace Parameter with Query
**When:** A parameter could be derived inside the function.
**Move:** Compute it inside. Caller doesn't have to compute it.

### Replace Query with Parameter
**When:** A function reads from global state or external resources.
**Move:** Pass what it needs. Now the function is pure (or nearly so) and testable.

### Remove Setting Method (Make Immutable)
**When:** A field shouldn't change after construction.
**Move:** Remove the setter. Set in the constructor only. Caller must construct with the right value.

### Replace Constructor with Factory Function
**When:** Construction is nontrivial, or different paths produce different "kinds."
**Move:** Static factory functions named for what they produce (`User.fromEmail()`, `User.fromOAuth()`).

## Dealing with inheritance

### Pull Up Method / Field
**When:** Two subclasses have the same method/field.
**Move:** Pull it up to the superclass.

### Push Down Method / Field
**When:** A method/field on a superclass is used by only some subclasses.
**Move:** Push it down to where it's used.

### Replace Subclass with Delegate
**When:** Inheritance was used for code reuse and the "is-a" relationship doesn't hold (Refused Bequest).
**Move:** Use composition. The class delegates to a strategy object instead of being a subclass.

### Replace Inheritance with Delegation
**When:** A class extends another only to reuse a few methods, but doesn't honor the parent's contract.
**Move:** Hold an instance of the would-be parent as a field. Forward only the methods that are actually used.

### Replace Superclass with Delegate
**When:** Superclass-subclass relationship is causing more problems than it solves.
**Move:** Convert to composition. Subclass becomes a class that holds an instance of the former superclass.

## When you're stuck

If a piece of code feels like it needs refactoring but you can't see the move, try these in order:

1. **Read the smell catalog** in the SKILL.md. Match what you see to a named smell. The named smell maps to one or more moves.
2. **Add characterization tests** to capture current behavior (in case the refactor breaks something).
3. **Extract Function aggressively.** Find any nameable subsection and extract it. Even if the result isn't the final design, the named pieces will reveal the structure.
4. **Inline aggressively.** If the structure is wrong, sometimes the right first move is to flatten everything (inline several functions) so you can see the actual logic, then re-extract along the right axes.
5. **Walk away and come back.** Refactoring is partly pattern-matching; sleep helps.
6. **Ask someone else.** Pair on it.

If after all of this the move still isn't clear, the issue may be a design problem (rather than a code smell). See `software-design-principles`.
