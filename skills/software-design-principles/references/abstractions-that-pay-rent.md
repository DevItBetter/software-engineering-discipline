# Abstractions That Pay Rent

Every abstraction has a cost: every reader must learn it before they can read the code that uses it. Every test must accommodate it. Every change must consider it. The abstraction has to be worth that cost.

This file is about how to tell whether an abstraction earns its keep, and what to do when it doesn't.

## What an abstraction costs

The fixed cost: a reader who didn't know about the abstraction must learn it. They must learn its name, its interface, its preconditions, its postconditions, and any non-obvious invariants. Until they do, they cannot read code that uses it.

The variable cost: every change to the surrounding code must consider whether it interacts with the abstraction's contract. Changes that "should" be local turn into changes that ripple through the abstraction's clients.

For a deep abstraction (small interface, lots hidden) the cost is paid once, the benefit is paid every time. For a shallow abstraction (big interface, little hidden), the cost is paid repeatedly and the benefit is small.

## The five tests an abstraction should pass

### 1. Does it hide a real, current decision?

The decision should:
- Already exist (not "we might decide this differently in the future").
- Be one that callers would otherwise have to know about (not a trivial wrapping).
- Be one that's likely to change (information hiding, per Parnas).

If the decision is hypothetical, the abstraction is speculative — flag it.
If the decision is trivial, the abstraction is shallow — flag it.
If the decision is permanent, the abstraction adds indirection without flexibility — flag it.

### 2. Does it represent a real domain concept?

The best abstractions name something the domain experts would recognize: `Order`, `Reservation`, `Invoice`, `RetryBudget`, `BoundedContext`. These have meaning beyond the code.

Bad abstraction names are generic structural words: `Manager`, `Handler`, `Processor`, `Helper`, `Service`, `Util`, `Common`, `Wrapper`, `Strategy`, `Factory`, `Provider`, `Resolver`. Each of these signals that the author didn't have a real concept in mind and reached for a generic noun.

A useful test: read the abstraction's name to a domain expert (or to a developer unfamiliar with this codebase). Do they form a correct mental model from the name alone? If not, the name is doing no work — and probably the abstraction isn't either.

### 3. Does it have multiple consumers, or one with a near-term plan for a second?

The "rule of three" (Sandi Metz, others): inline once. Inline twice. Extract on the third.

The two-cases case is the hardest call. If the second consumer is *imminent and concrete* (in the same PR, in a planned next PR), extract. If the second consumer is "hypothetical" or "we might want to," inline.

Premature abstraction is more expensive than duplication because: duplication is visible (you can see the two copies) and easy to consolidate later; abstraction is invisible coupling (hidden behind the interface) and hard to undo.

### 4. Does it enforce an invariant that would otherwise be re-checked everywhere?

A real win for abstraction: take a property that every caller would otherwise need to enforce (validation, authorization, normalization, retry, transactional boundary) and move it inside the abstraction. Now the property holds by construction; callers can't get it wrong.

Examples:
- A `Money` type that prevents adding two amounts in different currencies.
- A `SafeSql` type that can only be constructed via parameterized statements.
- A `LoggedInUser` type that can only be obtained from a session that has been authenticated.
- A `NonEmptyList<T>` type that has no representation for the empty case.

These abstractions earn their keep by making certain bugs unrepresentable. Cite "make illegal states unrepresentable" (Yaron Minsky) when promoting one of these.

### 5. Does it shrink, not grow, the cognitive surface for the caller?

A pass test: a caller of the abstraction needs to know less than a caller working without it. Specifically, the caller can use the abstraction correctly without reading its implementation.

A fail test: the caller has to read the implementation to know what to pass, what edge cases to handle, what errors might come back, or what state the object will be in afterwards.

The fail signature is "leaky abstraction" — the caller is forced to know about the supposedly-hidden details.

## Names that signal the abstraction probably doesn't earn its keep

Generic structural names that are usually bad on their own (and need a domain noun added or to be deleted):

- **Manager** — what does it manage? "OrderManager" almost always wants to be a verb (`OrderProcessor` is barely better; `OrderRepository` is honest if it's persistence). Often the right move is to delete it and put the methods on the entity.
- **Helper / Util / Common / Misc / Shared** — junk drawers. Each function in a `helpers.py` should live with the concept it helps.
- **Service** — sometimes legitimate (in a layered architecture where it has specific meaning), often a placeholder for "I needed somewhere to put this." `OrderService` is suspicious; `OrderProcessor` or `OrderApplicationService` (DDD term-of-art) is more honest.
- **Handler** — usually a function in disguise. `WebhookHandler` is fine if it's the entry point of an HTTP endpoint; `OrderHandler` is suspicious.
- **Processor** — better than Manager, still generic. Often wants to be a verb.
- **Wrapper** — almost always a smell. If you're wrapping for cross-cutting concerns (logging, retry, caching), the wrapper should be named for what it adds (`LoggingFooClient`, `CachingFooClient`).
- **Factory** — sometimes legitimate (for nontrivial construction); often shallow. A factory that just calls `new` adds nothing.
- **Provider / Resolver** — usually shallow. Replace with a function or a direct call.

If you see one of these names in a review, ask: what concept is being named here? Could the concept be named directly? Could the abstraction just not exist?

## Specific anti-patterns

**The "I" interface with one implementation.** `IOrderRepository` extended only by `OrderRepository`. The interface is overhead. Use the concrete type. Add the interface when the second implementation arrives.

**The "Strategy" pattern with one strategy.** Same problem. Either inline the strategy or delete the indirection.

**The "Service / Repository / Controller" stack with no logic in the service.** The service is a pass-through from the controller to the repository. Remove the service; the controller can call the repository directly.

**The "Manager" / "Helper" with state.** A manager that holds state probably wants to be the entity itself. (E.g., `UserManager.delete(user)` should probably be `user.delete()` — Tell Don't Ask.)

**The "rich Configuration object" with 30 fields, used by 8 modules each of which reads 3 fields.** Each module should take exactly the values it needs. The shared config object creates stamp coupling and Hyrum's-law surface.

**The "abstract base class" with one concrete subclass.** Abstract is speculative until it isn't. Make it concrete; promote later.

## When you find an abstraction that doesn't pay rent

Three options, in order of preference:

1. **Delete it.** Inline its uses. The code gets shorter, the reader has less to learn, the test surface shrinks.

2. **Demote it from a class/interface to a function.** Most "Helper" classes are function collections.

3. **Make it earn its keep.** If you suspect a real concept is *almost* there, see if a name change or a slight expansion of responsibility produces a deep, named domain concept.

If none of these work, leave it alone but flag it for the next PR. Removing abstractions is a refactor, not a code-review action.
