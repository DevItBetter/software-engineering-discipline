# SOLID, With Nuance

SOLID is a useful diagnostic vocabulary. It is a terrible mechanical checklist. The principles, sharpened and with their criticism literature acknowledged.

## Single Responsibility (S)

**Stated narrowly (Robert Martin):** "A class should have one, and only one, reason to change."

**Stated as it is sometimes used:** "A class should do one thing." This is the misapplication. By this reading, every class with two methods violates SRP.

**The honest test:** if you can identify two distinct stakeholder/domain concerns that drive change in this module, it is two modules pretending to be one. Concerns to look for:

- Different audiences. Persistence (DBA cares) and presentation (UX cares) are different concerns; mixing them is divergent change waiting to happen.
- Different rates of change. A pricing rule that changes monthly and a tax calculation that changes annually do not belong in one module unless their coupling is essential.
- Different abstraction levels. Orchestration (high-level workflow) and step implementation (low-level details) are different responsibilities.

**The Dan North critique:** SRP as stated is "pointlessly vague" — "responsibility" is undefined and infinitely subdividable. The honest version is "one reason to change," scoped to the actual axes of change in your domain.

**When to flag SRP in review:**
- A file changes often, for reasons that have nothing to do with each other.
- A class has methods that operate on disjoint subsets of its data.
- You can't state the class's purpose in one sentence without "and" or "or."

**When NOT to flag SRP:**
- A file is "long" but every part serves one purpose.
- A class has many methods but they all elaborate one concept.

## Open/Closed (O)

**Stated:** "Software entities should be open for extension, but closed for modification."

**The honest reading:** at predictable variation points, design extension mechanisms so that adding a new variant doesn't require modifying existing code. *At predictable variation points.* Not everywhere.

**The misapplication:** treating O/C as "every concrete class must have an interface." This is how you get codebases with `IUserService` extended by exactly one `UserService`, with the interface adding zero hiding and a layer of indirection cost.

**When to flag O/C:**
- Every time a new variant is added (new payment method, new report format, new export type), the same `if/elif` chain has to grow. This is a real O/C violation; the variation point is open. Extract a strategy or polymorphic dispatch.
- A new feature requires modifying many existing classes' methods. Indicates the variation point is in the wrong place.

**When NOT to flag O/C:**
- Code that hasn't actually been extended yet. "We might need to extend it" is YAGNI bait.
- Stable parts of the system that have never had a second variant.

## Liskov Substitution (L)

**Stated (Liskov, sharply):** if a program uses an object of type T, substituting any subtype S of T should not break the program. Subtypes must honor the supertype's contract.

**This is the most concrete and least controversial of the SOLID principles** — and the most-violated in practice. Common violations:

- A subclass throws an exception that the parent doesn't document. Callers that catch the parent's exceptions don't catch the child's.
- A subclass strengthens a precondition (rejects inputs the parent accepts). Callers using the parent's contract pass valid inputs that the child rejects.
- A subclass weakens a postcondition (returns less than the parent promises).
- A subclass has different side effects, ordering, or threading behavior than the parent.
- The classic example: `Square extends Rectangle`. Setting the width of a `Rectangle` doesn't change the height; setting the width of a `Square` does. Substituting a `Square` where the code expects a `Rectangle` produces wrong results.

**When to flag LSP:**
- Override that catches and silently swallows an exception the base class would propagate.
- Override that returns null/None where the base method documents non-null.
- Override that has different time complexity in a way callers depend on (per Hyrum's Law).
- Mock or fake in tests that doesn't honor the real contract — leads to "passes the tests, breaks in prod."

## Interface Segregation (I)

**Stated:** "Clients should not be forced to depend on methods they do not use."

**The honest reading:** when designing an interface, design it from the consumer's perspective. If different consumers need different subsets of the operations, those subsets are separate interfaces (which a single implementation can implement).

**The misapplication:** splitting interfaces so finely that every operation has its own one-method interface. The cost of indirection now exceeds the benefit of segregation.

**When to flag ISP:**
- Tests have to mock methods the code-under-test doesn't call. Smell of an over-broad interface.
- A consumer takes a `Repository<T>` but only calls `find()` — it should take a `Finder<T>`. The current type lets it accidentally call `delete()` and gives the reviewer no clue what was meant.
- "Service" objects with 30 methods.

**When NOT to flag ISP:**
- Small, cohesive interfaces with 3–6 methods that are all genuinely used by all consumers.

## Dependency Inversion (D)

**Stated:** "High-level modules should not depend on low-level modules. Both should depend on abstractions."

**The honest reading:** at architectural seams (domain ↔ persistence, domain ↔ HTTP, domain ↔ external services), the volatile detail should depend on the stable abstraction, not the other way around. The domain logic shouldn't know it's storing data in PostgreSQL; the PostgreSQL adapter should know how to fulfill what the domain needs.

**The misapplication (Dan North's "DI obsession"):** inverting *every* dependency, regardless of stability or volatility. Now `OrderTotalCalculator` takes an `IAdditionStrategy` — adding indirection for nothing.

**When to flag DI:**
- Domain logic imports from infrastructure (e.g., the order processor imports from the database driver). The dependency direction is backwards; reversing it via an interface lets the domain stay stable while the infrastructure varies.
- A high-level module is impossible to test because it directly instantiates its low-level dependencies. Inverting the dependency makes test substitution clean.
- A module needs to support multiple implementations (test fake, real, etc.) and currently has the dependency hardcoded.

**When NOT to flag DI:**
- A module depends on a stable utility (a date library, a string formatter). Inverting buys nothing.
- A module instantiates one specific concrete dependency that is unlikely to vary. Forcing DI here produces ceremony with no payoff.

## The meta-rules

1. **Cite the specific principle and the specific violation.** "Violates SRP" alone is too vague. "Violates SRP — this class is changed both for tax-rule updates and for invoice-format updates, and those concerns have completely different stakeholders" is actionable.

2. **Don't use SOLID to justify an abstraction the code doesn't need yet.** SOLID describes shapes that pay off when there is genuine variation. Speculative SOLID is over-engineering with a fancy name.

3. **The principles are interrelated and sometimes in tension.** Aggressive ISP can fight DRY (more interfaces to maintain). Aggressive DI can fight readability (everything is injected, nothing is direct). Use judgment; the principles are guides, not laws.

4. **Cohesion and coupling cover most of what SOLID covers, more directly.** When in doubt, fall back to "is this cohesive?" and "is this coupled appropriately?" — those questions are sharper.
