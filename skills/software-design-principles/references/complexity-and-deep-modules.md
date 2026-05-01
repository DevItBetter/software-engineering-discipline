# Complexity and Deep Modules

This is John Ousterhout's framing from *A Philosophy of Software Design*. It is the single most useful modern framework for diagnosing design problems and explaining what to change.

## Complexity is what makes software hard to change

Ousterhout's working definition: **complexity is anything related to the structure of a software system that makes it hard to understand and modify.**

Note what this excludes: a system can be large without being complex (if its parts are simple and well-isolated), and a system can be small but complex (if everything is tangled).

Complexity is the *enemy*. Every other principle in this skill exists to attack one symptom of complexity.

## The three symptoms of complexity

When complexity is present, you observe one or more of:

1. **Change amplification.** A simple-sounding change requires modifications in many places. Adding a new field requires touching the model, the DTO, the validator, the serializer, the controller, the database migration, the API documentation, three tests, and the client SDK. The conceptual change was small; the actual change was huge.

2. **Cognitive load.** A developer needs to know a lot before they can make a change safely. The function's behavior depends on global state set somewhere else; the abstraction has invariants that aren't checked but must be maintained; the order of method calls matters but isn't enforced.

3. **Unknown unknowns.** The developer doesn't know what they need to know. They make a change that looks correct in isolation but breaks something three modules away because of a hidden coupling. This is the worst symptom because there is no way to defend against it locally — the system itself doesn't tell you what to look at.

When you find yourself saying "this looks fine but I'm worried I'm missing something," you're staring at unknown unknowns. The fix is design that makes dependencies explicit and surprising-behavior impossible.

## The two causes of complexity

Per Ousterhout, complexity has two root causes. Every design move either reduces these or doesn't.

### Cause 1: Dependencies

A dependency exists when one piece of code cannot be understood, modified, or used in isolation from another. Some dependencies are essential (the system has to do *something*); the goal of design is to:

- Eliminate dependencies that aren't essential.
- Make the dependencies that remain **explicit and obvious**.
- Push the unavoidable dependencies to **stable, well-defined interfaces**.

Common kinds of dependency that are easy to miss:

- **Temporal coupling.** Methods must be called in a specific order. (`init()` before `start()` before `process()`. Forgetting the order produces a runtime error or, worse, silently wrong behavior.) A module that requires a sequence is leaking its state machine.
- **Hidden global state.** Function A modifies a global; function B reads it. The contract isn't visible at the call site.
- **Implicit type contracts.** Two modules pass dictionaries; the receiving module relies on specific keys being present. The contract is in the receiver's code and the sender's code; nothing connects them.
- **Convention coupling.** "If you add a new handler class, name it `XxxHandler` and put it in `handlers/`, and the registry will pick it up." The new file's existence is now load-bearing.

### Cause 2: Obscurity

Obscurity is when the information you need isn't visible. It is closely related to dependencies (a hidden dependency is obscure) but stands on its own. Examples:

- A function takes a `Config` object but only reads three of the 30 fields. The reader can't tell what the function actually depends on.
- A method is documented as `// returns the user`, but it also marks the user as logged in as a side effect.
- An exception type is `RuntimeException` (not a domain-specific exception); the catch site has no idea what went wrong.
- A magic number `0.95` appears in the code with no name; the reader has no idea why.

Obscurity feeds unknown unknowns. The reader either reads more carefully than necessary (high cost), reads less carefully than necessary (bugs), or both at different times (worst of both).

## Deep modules vs shallow modules

The most useful single Ousterhout idea. **A module's depth is the ratio of functionality it provides to the size of its interface.**

Deep modules are good. They hide a lot of complexity behind a small surface. Examples:

- **Unix file API.** Five primary functions (`open`, `read`, `write`, `close`, `seek`) hide filesystems, page caches, permission checks, journaling, network filesystems, FUSE drivers, and much more. The user learns five functions; the user does not need to know about any of the rest unless something specific demands it.
- **A garbage collector.** From the application's view, an interface that is essentially "allocate a thing" hides one of the most sophisticated subsystems in the runtime.
- **A SQL database.** The interface (`query`, `execute`, `commit`, `rollback`) is simple. The B-trees, query planner, MVCC, write-ahead log, replication, and disk I/O sit behind it.

Shallow modules are bad. They have wide interfaces relative to the work they do. Examples:

- **A wrapper class with one public method per private method.** Adds nothing; reader must learn the wrapper's vocabulary *and* understand what it wraps.
- **A class with 15 getters and setters.** Exposes its entire state. There is no hiding; the object is a dictionary with extra steps.
- **A "configuration" object with 30 boolean flags.** The interface is the implementation; nothing is hidden; the reader must learn all 30 flags to use it.
- **A "service" that orchestrates by calling methods on its dependencies and returning their results unmodified.** A pass-through. The reader gains nothing by going through it.

The depth test: **does using this module require knowing less than using the things it wraps?** If no, the module is making things worse.

## Pass-through methods are the most common shallow-module symptom

A pass-through method exists when a method does little or nothing besides calling another method, often with the same signature.

```python
class OrderService:
    def __init__(self, repo): self._repo = repo
    def get_order(self, id): return self._repo.get_order(id)
    def save_order(self, order): return self._repo.save_order(order)
```

This is shallow. The reader has to learn `OrderService` and *then* read `repo` to understand anything. The wrapper hides nothing. The fix is usually one of:

- Delete the wrapper; let callers use the underlying thing.
- If the wrapper exists for dependency injection, make it actually do something useful (validation, caching, retry, observability).
- If the wrapper exists for "future flexibility," delete it and add it back when the second consumer arrives.

## Information leakage

A module *leaks information* when knowledge of its internal design appears in another module. The classic example: a module reads a config file, and four other modules also know the format of that config file. Now the format can't be changed without touching all five places.

Information leakage is the inverse of information hiding. The cure is to identify the leaked decision and put a single owner in charge of it.

## Tactical vs strategic programming

Two stances toward design:

- **Tactical.** "Just get this working." Each change is the minimum needed to ship the current task. Design debt accumulates because nobody is paying for the cleanup. Eventually the system's complexity dominates and every change becomes expensive.
- **Strategic.** Each change is the minimum needed to ship the current task *plus* the design improvement that the current task surfaces an opportunity for. Design quality is a continuous investment, not a one-time refactor.

Strategic doesn't mean over-engineering. It means: when you're already touching a piece of code and you can see how to leave it slightly cleaner than you found it, do so. Pay the small ongoing cost so you don't owe the lump sum.

The trap: tactical programmers feel productive (lots of features shipped) and look productive (lots of commits) but produce systems that get progressively harder to work in. Strategic programmers look slightly slower in the short term and dramatically faster over years.

This connects to Kent Beck's "make the change easy, then make the easy change." If the change is hard, the right move is often to first do a small refactor that makes it easy, then make the now-easy change.

## Define errors out of existence

A specific Ousterhout move worth highlighting. Many "errors" exist only because the API was designed to make them possible.

Examples:

- A `delete(file)` method that throws if the file doesn't exist requires every caller to wrap in try/except. A `delete(file)` method that returns silently (or returns a `Removed` / `WasNotPresent` value) eliminates the error class entirely.
- A queue with a separate `is_empty()` and `dequeue()` requires the caller to check, then dequeue, with a race between them. A `dequeue() -> Optional[T]` eliminates the race.
- A type that requires several setup calls before use can be replaced with a constructor that does the setup atomically. The "uninitialized" state no longer exists.
- A function that takes nullable inputs and is undefined-behavior on null can be replaced with a function whose type signature forbids null (or with two functions, one for present and one for absent).

The general move: **redesign the interface so the error condition cannot be expressed.** This eliminates a whole class of bugs at design time.

## What this means for review

When evaluating a design or a diff:

- **Name the complexity.** Is the issue change amplification, cognitive load, or unknown unknowns? "I had to read three files to understand what this function does" is cognitive load.
- **Find the dependency or obscurity.** Don't just say "this is complex." Say "this depends on X being initialized first, but the dependency is not visible at the call site" or "this method takes a `Config` but only reads `Config.timeout`; the rest of the config is irrelevant noise to the reader."
- **Apply the depth test to new modules.** Is the interface smaller than the work hidden? If not, the module is shallow and probably shouldn't exist as a separate module.
- **Watch for pass-through methods, getters/setters that mirror state, and "service" classes that don't service anything.**
- **Consider whether errors can be defined out of existence** rather than handled.
