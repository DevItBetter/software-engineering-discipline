# DRY, KISS, YAGNI, POLA — Done Right

These four principles are the most quoted and the most misapplied. Each, sharpened.

## DRY — Don't Repeat Yourself

**The original (Hunt & Thomas, *The Pragmatic Programmer*):** "Every piece of *knowledge* must have a single, unambiguous, authoritative representation within a system."

**The common misreading:** "If two pieces of code look the same, they must be combined."

The misreading produces over-coupling. Two functions that *currently* look the same but exist for different reasons — one calculates VAT for the EU, another calculates state sales tax for the US — will eventually diverge, and the premature DRY-fication will block the divergence or twist the abstraction into knots to accommodate it.

**What DRY actually attacks:**

- A business rule encoded in two places that must stay in sync. (E.g., "minimum order amount is $5" appears in the validator, the UI, and the email template.)
- A serialization shape duplicated across a writer and a reader.
- A constant repeated as a literal everywhere instead of having a name.
- A piece of policy (authorization rule, retry budget, feature flag) duplicated.

**What DRY does not attack:**

- Two functions whose lines coincidentally look similar but encode different concepts.
- Two test cases that use the same setup boilerplate. (The boilerplate is *meant* to be obvious; abstracting it makes tests harder to read.)
- Two configuration files for two different environments.

**The diagnostic:** is this the *same knowledge* expressed twice, or two *different things* that happen to look alike? If it's the same knowledge, fix it (extract, link, derive one from the other). If it's two different things, leave them alone.

The Sandi Metz aphorism that captures this: **"duplication is far cheaper than the wrong abstraction."** When uncertain, prefer duplication and let the right abstraction emerge from the third or fourth concrete example. Premature abstraction is harder to undo than duplication.

## KISS — Keep It Simple

**The original (Lockheed, attributed to Kelly Johnson):** "Keep it simple, stupid." (The "stupid" was a noun about the system, not the engineer — make it simple enough that a stupid system survives a stupid moment from a smart engineer.)

**The honest reading:** when you have two designs that meet the requirement, prefer the one with less internal complexity (in Hickey's sense — un-complected, fewer interleaved concerns). The complex one needs an explicit justification: a current requirement, a near-term known requirement, or a measured constraint.

**The misreading:** "fewer lines of code is always better." This rewards clever one-liners and dense ternary chains that are objectively simpler-looking and subjectively impossible to read. A simple design is often *more* lines than a complex one — explicit code is longer than implicit code.

**The diagnostic:** can a new contributor read this and form a correct mental model in one pass? Can they predict the behavior on inputs they didn't test? If yes, it's simple in the way that matters. If no, it has hidden complexity even if it's terse.

## YAGNI — You Aren't Gonna Need It

**The original (Extreme Programming):** "Always implement things when you actually need them, never when you just foresee that you need them."

**The honest reading:** every speculative feature, configuration option, abstraction layer, or hook carries an ongoing cost — every reader must learn it, every change must consider it, every test must accommodate it. That cost is paid every day. The cost of adding the feature later, *if* it turns out to be needed, is usually much smaller than the running tax.

**Common YAGNI violations to flag:**

- A new interface with one implementation and a hypothetical second.
- A configuration option that is never set to anything but its default.
- A "plugin system" with one plugin.
- A field on a data type "for future use."
- A retry / cache / rate-limiter / timeout layer added "just in case" without measurement.
- An async/queue indirection added "for scalability" before there's a load problem.
- Generics or type parameters where every consumer uses the same concrete type.

**Common false-positive YAGNI accusations to push back on:**

- Reasonable defensive design at trust boundaries (input validation isn't speculative; the input is real).
- Schema design that anticipates a known near-term requirement.
- Modular design that doesn't add layers — just keeps things separable that could otherwise become entangled.
- Observability hooks that exist because production *will* fail (it always does).

The pithy version: **YAGNI is about feature speculation, not about engineering hygiene.** Don't use YAGNI to argue away tests, observability, or trust-boundary validation.

## POLA — Principle of Least Astonishment

**Stated:** components of a system should behave in a way that the user (caller, reader, operator) expects, given the name and signature.

**Why it matters:** surprise is the most expensive property of a codebase. Every reader must verify their assumptions before trusting anything; every change must consider what surprising behavior callers may have come to depend on.

**Common POLA violations:**

- A function that mutates its input without saying so. (`sortInPlace` is honest; `sort` that mutates is astonishing.)
- A getter that has a side effect (logs the access, mutates state, hits the network).
- A function whose name suggests purity but that in fact reads global state, time, or environment.
- A boolean parameter whose default value flips behavior in a surprising way (`exists(path, follow_symlinks=True)` is fine; `delete(path, recursive=True)` as a default is astonishing).
- An exception type that doesn't match the actual failure (raising `ValueError` for a network failure).
- Silent truncation, silent rounding, silent type coercion.
- A return value that means different things based on input shape.
- An iterator that mutates the underlying collection.
- A method named `validate` that returns a normalized value (now it does two things and the second is invisible).

**The diagnostic:** read the name and signature only. Predict the behavior. Now read the body. Was your prediction right? If not, the name lies — fix the name, fix the signature, or fix the behavior so they agree.

**A specific application — honest defaults:** the default behavior of an API should be the behavior the caller most likely wants and the safest behavior under uncertainty. Defaults that "do the powerful thing silently" violate POLA. (Compare: `git push` originally pushed all matching branches; that was eventually changed because users were astonished when it pushed branches they didn't intend to push.)

## A combined check

When evaluating any abstraction, function, or interface, run all four:

- **DRY violation?** Is this duplicating *knowledge* that must stay synchronized?
- **KISS violation?** Is the design more complex than the requirement justifies, with no concrete reason for the extra complexity?
- **YAGNI violation?** Does this build for a hypothetical, not-yet-real, requirement?
- **POLA violation?** Does the name/signature suggest one thing while the behavior does another?

If any answer is yes, that is the finding. Name it, with the specific evidence.
