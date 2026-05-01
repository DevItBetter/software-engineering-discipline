# Naming and Readability

Names are the API of your code to its readers. Code is read far more than it is written, so cost paid in writing (slightly longer name, more careful structure) is repaid many times over in reading.

This file covers names, function shape, comments, and the readability heuristics that produce code humans can trust at a glance.

## Names

A good name has these properties:

- **Specific to the domain.** `customerId` beats `id`; `pendingOrders` beats `data`; `retryAttempts` beats `count`.
- **Reveals intent.** `isEligibleForRefund` beats `checkRefund`; `currentRetryAttempt` beats `i`.
- **Honest.** The name describes what the function does, including side effects. `getUser` should not log the user in. `validateOrder` should not silently mutate the order.
- **Searchable.** `OrderStatus.PENDING_REVIEW` is greppable; `4` is not.
- **Pronounceable.** `customerEmailAddress` reads aloud; `cstmrEmlAddr` does not.
- **Consistent with the codebase.** If everywhere else uses `customer`, don't introduce `client` for the same concept.
- **Reads naturally at the call site.** `if user.canEdit(post):` reads. `if Permissions.check(user, post, ACTION_EDIT):` does not.

### Specific name patterns

**Booleans:** prefix with `is`, `has`, `can`, `should`. `isActive`, `hasPermission`, `canEdit`, `shouldRetry`. Negative names (`isNotActive`) are confusing — invert and use the positive (`isInactive`).

**Functions that return:** verb phrases (`computeTotal`, `findUser`, `parseDate`).

**Functions that mutate or have side effects:** verb phrases that imply action (`saveOrder`, `sendEmail`, `markAsPaid`). Do not mix mutation with names that suggest queries (`getUser` should not save anything).

**Functions that "are" something:** noun phrases (`primaryAccount`, `defaultLocale`).

**Predicates:** as above with `is/has/can/should`.

**Collections:** plural (`orders`, `pendingMessages`). A variable named `orders` should not be a single order.

**Counts and indices:** suffix with `Count` or `Index`. `userCount`, `currentIndex`, `lastIndex`. Avoid `num`, `n`, `cnt`.

**Times and durations:** include the unit. `timeoutMs`, `expiresAtUtc`, `delaySeconds`. A `timeout: 30` is ambiguous; `timeoutMs: 30` is honest.

**Identifiers:** include the type. `customerId` not `customer` (when it's just the id), `orderUuid` not `order`. Saves the reader from cross-checking.

### Names to flag in review

**Names that lie.** Function does more than the name suggests, or different things in different cases. The most expensive bug; surrounding code can't be trusted.

**Names that pun.** Same word for two concepts. `connection` meaning both a TCP connection and a database connection in the same file.

**Names with redundant type info.** `userList`, `customerObject`, `boolFlag`. Let the type system carry that.

**Hungarian-like type prefixes.** `strName`, `iCount`. The type system has this.

**Generic placeholders.** `tmp`, `temp`, `data`, `info`, `value`, `obj`, `item`. Sometimes appropriate (loop variable in a 3-line block), almost always replaceable with a domain term.

**Misleading scope.** Single-letter names (`u`, `o`) outside of very small scopes. Conventional names (`i`, `j` for loop indices) are fine in narrow scopes; bad outside them.

**Domain abbreviations.** `cust`, `ord`, `usr`, `prod`, `qty`. Saves a few keystrokes for the writer; costs every reader who has to verify the meaning.

**The "smurf" pattern.** Every name in a module starts with the module name: `UserUserId`, `UserUserName`, `UserUserEmail`. If you find this, the prefix is contributing nothing — drop it.

**Numbered suffixes.** `process`, `process2`, `processFinal`, `processRealFinal`. Each version has a real concept; name them.

## Function shape

A well-shaped function:

- **Does one thing.** Stating its purpose in one verb-noun pair. If you need "and," it's two functions.
- **Is at one level of abstraction.** Either it orchestrates (calls higher-level operations) or it implements (does low-level work). Mixing produces "step 1 is two lines of detailed string manipulation, step 2 is `processOrder()`" — jarring to read.
- **Has a small parameter list.** Three or four max. More than that is a code smell (Long Parameter List). Group with a parameter object.
- **Returns one kind of thing.** `getUser` should not sometimes return a `User`, sometimes a list, sometimes `None`, sometimes an error code. Pick one.
- **Has clear preconditions and postconditions.** Either documented, type-encoded, or asserted at the boundary.

### Function length

Sandi Metz: ≤5 lines per method. This is provocative and useful as a *prompt* — when a function gets long, it's an opportunity to ask "is there a concept here that wants extracting?"

It is *not* a hard rule. A function whose body is a clear sequence of obvious steps may be 30 lines and unimprovable by extraction. A function whose body is 8 lines of dense, branching logic may be improved by extraction.

The honest test: **can you extract a portion and give it a precise, intent-revealing name?** If yes, extract. If the only available name is mechanical (`doFirstStep`, `helperFor`), don't.

### Guard clauses over nested conditionals

```python
# Bad — nested
def process(order):
    if order is not None:
        if order.is_valid:
            if order.has_payment:
                # ... actual work, indented four levels
                pass

# Better — guard clauses
def process(order):
    if order is None:
        return
    if not order.is_valid:
        raise InvalidOrderError(order)
    if not order.has_payment:
        return Pending(order)
    # ... actual work at the function's natural indentation
```

The guard-clause version is shallower (less indentation), each precondition is named (the conditions become documentation), and the "happy path" is visually distinct.

### Avoid the "arrow code" / "pyramid of doom" anti-pattern

Code that nests deeply (5+ indentation levels) is a sign of either too many concerns in one function (extract) or weak control flow (use early returns, polymorphism, or pattern matching).

## Comments

Comments explain *why*, not *what*. The code already shows what; the comment is for what the code can't show.

**Good comments:**

- The *reason* a non-obvious choice was made. ("We're using a linear scan instead of a hash lookup because the list is always small and allocating the hash dominated.")
- An invariant the code can't enforce. ("This list must be kept sorted because the matching algorithm is O(n) instead of O(n log n) when sorted.")
- A reference to external context. ("See RFC 1234 §4.2 for why we encode this way.")
- A warning about a subtle hazard. ("This must be called with the mutex held; we don't take it here because the only caller already holds it.")
- A TODO with a date, an owner, and a concrete trigger condition. (Not "TODO: clean this up" but "TODO(@jane, 2026-09): replace with the new API once the migration completes.")

**Bad comments to flag:**

- Comments that narrate the code. (`// increment counter` above `counter += 1`.) Delete.
- Comments that contradict the code. The code wins; the comment is a lie.
- Commented-out code. Delete; version control remembers.
- Stale TODOs with no owner. Delete or assign.
- Decorative banners (`############# SECTION ##############`). The code's structure should communicate this; if it can't, the file is too big.
- Comments at the top of every function repeating its name. Doc generators sometimes need these; check your project's convention. If they're not used, delete.

A heuristic: **if you can replace the comment with a better name or a tiny refactor, do that instead.** `// check if user is admin` becomes a function `isAdmin(user)`.

## The "squint test"

Sandi Metz's diagnostic: defocus your eyes. Look at the shape of the code, not the words. What you should see:

- Indentation that mostly stays at one or two levels (not pyramids).
- Function bodies that are short and similar in shape (not one 5-liner and one 200-liner in the same file).
- Visual sections of the file that align with conceptual sections.
- Imports grouped sensibly, not scattered.
- Test files that mirror the structure of the code under test.

If the squint test reveals chaos — wildly varying indentation, nested complexity, no visual structure — the code is hard to read regardless of how good any individual line is.

## Idiomatic local consistency

Match the codebase. Don't import patterns from your previous job, your favorite blog, or another language's conventions. The codebase has a voice; add to it, don't fight it.

If the codebase uses `snake_case` and you write `camelCase`, every reader has to context-switch. If the codebase prefers explicit returns and you start using clever expression-statements, every reader has to recalibrate.

When you genuinely need to introduce a new pattern (because the existing one is broken or absent), do it deliberately: discuss with the team, document the choice, and apply it consistently going forward.

## What to flag in review

- Names that lie, mislead, or pun.
- Functions that mix abstraction levels.
- Long parameter lists with similar-typed parameters (easy to mix up at the call site).
- Comments that narrate code (replace with names) or contradict code (the code is the truth).
- Stale TODOs.
- Code that fights the codebase's existing conventions for no documented reason.

What *not* to flag, ever, as blocking:
- Nits about variable name style (camelCase vs snake_case) when the project has no documented standard.
- Personal preferences about formatting (the formatter has the call here).
- Aesthetic objections to working, idiomatic, locally-consistent code.
