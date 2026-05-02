---
name: code-smells-and-antipatterns
description: "Identify what's wrong with code by name. The diagnostic library — every code smell and antipattern has a name, a concrete code example, the cost it imposes, and the fix. Use this skill whenever the task involves looking at code and asking \"what's off here?\", \"why does this feel wrong?\", \"what should I be flagging?\", or evaluating whether a piece of code is healthy. Use it in code reviews, refactoring planning, design discussions, and technical-debt audits. Built on Fowler (Refactoring 2nd ed.), Refactoring.Guru's catalog, and the wider antipattern literature. The companion skill `refactoring` is the corrective side; this skill is the diagnostic side."

---

# Code Smells and Antipatterns

A code smell is a surface symptom that suggests deeper trouble. The smell isn't always a bug, but it raises the probability of bugs, makes change more expensive, and erodes the team's ability to reason about the code. Naming the smell sharply — "this is Feature Envy" — does three things: it tells the author *what specifically* is wrong, it points at the named fix, and it teaches.

This skill is the diagnostic library. Each smell has a code example, the cost it imposes, and a pointer to the named refactoring(s) that fix it. For the *application* of those refactorings — when to refactor, how to do it safely, the discipline of small steps — see the sibling skill `refactoring`.

For AI-specific failure modes (hallucinated APIs, fabricated tests, etc.), see `ai-coding-antipatterns`. The two skills overlap a little; this one focuses on smells that show up in any code, AI-authored or not.

## How to use this skill

When reviewing code:

1. **Skim for smells.** Most reviews find one or two; some find more. Each one named.
2. **Tie each smell to the cost it imposes.** "Long Function" alone is not a finding; "Long Function — this 200-line function bundles validation, transformation, and persistence; a change to any of the three forces you to read all three" is.
3. **Map to the fix.** Each smell has standard refactorings. Cite them.
4. **Don't pile on.** A 30-line PR with five smells is one finding ("design needs rework — see `engineering-discipline` on escalating to a redesign conversation"), not five.

## The classic catalog (Fowler buckets)

### Bloaters — code that has grown too large

#### Long Function

A function that has accumulated too much logic. Common signs: more than ~30 lines in most languages, multiple levels of indentation, comments that introduce sections, mixing levels of abstraction.

```python
# Bad
def process_order(order_data):
    # validate
    if not order_data.get("customer_id"):
        raise ValueError("missing customer")
    if not order_data.get("items"):
        raise ValueError("no items")
    for item in order_data["items"]:
        if item["quantity"] <= 0:
            raise ValueError(f"bad quantity for {item['sku']}")
    # transform
    customer_id = order_data["customer_id"]
    total = 0
    line_items = []
    for item in order_data["items"]:
        price = lookup_price(item["sku"])
        line_items.append({"sku": item["sku"], "qty": item["quantity"], "price": price})
        total += price * item["quantity"]
    # persist
    db.execute("INSERT INTO orders ...", customer_id, total)
    order_id = db.last_id()
    for li in line_items:
        db.execute("INSERT INTO order_items ...", order_id, li["sku"], li["qty"], li["price"])
    # notify
    send_email(customer_id, total)
    return order_id

# Better
def process_order(order_data):
    validated = validate_order(order_data)
    priced = price_order(validated)
    order_id = persist_order(priced)
    notify_customer(priced.customer_id, priced.total)
    return order_id
```

**Cost:** mixing concerns means a change to validation, pricing, or notification touches the same function and risks unrelated breakage. Each piece is hard to test in isolation.

**Fix:** Extract Function for each cohesive section. Each extraction earns its name; if the only available name is mechanical (`doStepThree`), don't extract.

#### Long Parameter List

More than three or four parameters. Easy to mix up at the call site (especially with same-typed parameters). Often grows by accretion as new "options" are added.

```typescript
// Bad
function sendEmail(
  to: string,
  from: string,
  subject: string,
  body: string,
  cc: string[],
  bcc: string[],
  attachments: File[],
  isHtml: boolean,
  priority: "low" | "high",
  trackOpens: boolean,
  scheduledAt: Date | null
) { /* ... */ }

// Call site — what does each argument mean?
sendEmail("a@b.com", "c@d.com", "hi", "...", [], [], [], true, "high", false, null);

// Better
type SendEmailRequest = {
  to: string;
  from: string;
  subject: string;
  body: string;
  format?: "text" | "html";
  cc?: string[];
  bcc?: string[];
  attachments?: File[];
  priority?: "low" | "high";
  trackOpens?: boolean;
  scheduledAt?: Date;
};

function sendEmail(req: SendEmailRequest) { /* ... */ }

// Call site — meaning is at the call
sendEmail({ to: "a@b.com", from: "c@d.com", subject: "hi", body: "...", format: "html", priority: "high" });
```

**Cost:** call sites are unreadable; argument-order mistakes type-check; refactoring the signature touches every caller.

**Fix:** Introduce Parameter Object. Use named arguments where the language supports them. For booleans, see Boolean Trap below.

#### Large Class

A class that's grown to do too much. Symptom: state fields that are unrelated; methods that operate on disjoint subsets of fields; no single sentence describing what the class is for.

**Cost:** divergent change (the file gets edited for unrelated reasons), low cohesion, often high coupling because it touches everyone.

**Fix:** Extract Class along the fields-and-methods clusters. Sometimes Extract Subclass when the class has type-conditional behavior.

#### Primitive Obsession

Using primitives (`string`, `int`, `bool`, `Map`) for things that have rules — domain concepts that aren't enforced.

```python
# Bad
def transfer(from_account: str, to_account: str, amount: float, currency: str): ...

# This compiles fine but ships a bug:
transfer("acc-123", "acc-456", 100, "EUR")  # OK
transfer("acc-456", "acc-123", 100, "USD")  # also OK, but USD account doesn't accept USD
transfer("acc-123", "acc-456", 100, "EUR")  # type-checks, but account/currency rules live elsewhere

# Better
class AccountId(str): pass
class Money:
    def __init__(self, amount: Decimal, currency: Currency):
        if amount < 0: raise ValueError("negative")
        self.amount = amount
        self.currency = currency

def transfer(from_: AccountId, to: AccountId, amount: Money): ...

transfer(AccountId("acc-123"), AccountId("acc-456"), Money(Decimal("100"), Currency.EUR))
# Wrong order won't typecheck; can't add Money(EUR) to Money(USD); can't have negative Money
```

**Cost:** invalid states are representable; the rules live in scattered validation code; type-checking can't help.

**Fix:** Replace Primitive with Object. The type encodes the rules; bugs become unrepresentable.

#### Data Clumps

The same group of values keeps traveling together as parameters. They want to be a class.

```javascript
// Bad — these always appear together
function rect(x, y, width, height) {}
function moveRect(x, y, width, height, dx, dy) {}
function scaleRect(x, y, width, height, factor) {}

// Better
class Rect {
  constructor(public x: number, public y: number, public width: number, public height: number) {}
  move(dx: number, dy: number) { /*...*/ }
  scale(factor: number) { /*...*/ }
}
```

**Cost:** changing the group (adding `z`, splitting `width` and `height` apart) means touching every signature. The shared concept is hidden.

**Fix:** Introduce Parameter Object or Extract Class.

### Object-orientation abusers

#### Switch Statements / `isinstance` chains

Behavior dispatching on a closed set of types.

```python
# Bad
def calculate_pay(employee):
    if employee.kind == "salaried":
        return employee.monthly_salary
    elif employee.kind == "hourly":
        return employee.hours * employee.rate
    elif employee.kind == "commission":
        return employee.base + employee.sales * employee.rate
    else:
        raise ValueError(f"unknown kind: {employee.kind}")

# Better — polymorphism
class Employee:
    def calculate_pay(self) -> Decimal: raise NotImplementedError

class Salaried(Employee):
    def calculate_pay(self) -> Decimal: return self.monthly_salary

class Hourly(Employee):
    def calculate_pay(self) -> Decimal: return self.hours * self.rate

class Commission(Employee):
    def calculate_pay(self) -> Decimal: return self.base + self.sales * self.rate
```

**Cost:** adding a new type forces edits to every conditional. Easy to forget one. Fowler's "Repeated Switches" — the same conditional structure appears in many places, and a new variant has to be added in all of them.

**Fix:** Replace Conditional with Polymorphism. (For sealed unions / discriminated unions in functional-style code, exhaustive pattern matching is fine; the language enforces total coverage.)

#### Refused Bequest

A subclass overrides parent methods to do nothing or throw — the "is-a" relationship doesn't hold.

```java
// Bad
class Bird {
    void fly() { /* ... */ }
}
class Penguin extends Bird {
    void fly() { throw new UnsupportedOperationException("penguins can't fly"); }
}
```

**Cost:** Liskov violation; callers handed a `Bird` cannot trust the contract.

**Fix:** Replace Subclass with Delegate, or restructure so the hierarchy honors the contract (`Bird` doesn't have `fly()`; only `FlyingBird` does).

### Change preventers — design that makes change expensive

#### Divergent Change

One module changes for many unrelated reasons.

```python
# Bad — UserManager is edited for tax-rule updates AND email-template changes AND auth changes
class UserManager:
    def calculate_tax(self, user): ...
    def send_welcome_email(self, user): ...
    def authenticate(self, user, password): ...
    def reset_password(self, user): ...
    def render_invoice(self, user): ...
```

**Cost:** the file is a coordination hotspot. Multiple unrelated stakeholders edit it. Merge conflicts cluster here. The class has no single purpose statable in one sentence.

**Fix:** Extract Class along the responsibility lines. `TaxCalculator`, `WelcomeMailer`, `Authenticator`, `InvoiceRenderer`.

#### Shotgun Surgery

Inverse of Divergent Change: one logical change requires touching many files.

Add a new field to `User` → touch the model, the DTO, the validator, the serializer, the controller, the migration, the API doc, three tests, the client SDK. The change is conceptually small; the actual change is huge.

**Cost:** every change is a multi-file coordination. Easy to miss a file. New developers can't find where things live.

**Fix:** Move Function / Move Field to consolidate; sometimes Combine Functions into Class. The change is to reduce the number of places one logical concept lives. Sometimes signals a deeper Conway's-Law mismatch (see `software-design-principles`).

### Dispensables — code that adds no value

#### Comments narrating code

```python
# Bad
# Increment the counter
counter += 1
# Check if the counter exceeds the limit
if counter > MAX:
    # Reset the counter
    counter = 0
```

The comments restate the next line. They add nothing.

**Fix:** delete. Or replace with a comment about *why*: "Counter rolls over to keep buckets bounded; the actual cardinality is tracked elsewhere." Or extract a function with a name that captures the intent: `increment_with_rollover()`.

#### Duplicate Code

The same logic in multiple places.

```typescript
// Bad — both paths encode the same business rule
function applyDiscount(order) {
  if (order.customerTier === "gold" && order.total > 100) return order.total * 0.9;
  return order.total;
}
function quoteShipping(order) {
  if (order.customerTier === "gold" && order.total > 100) return 0;  // free shipping for big gold orders
  return STANDARD_SHIPPING;
}
```

The condition `customerTier === "gold" && total > 100` lives in two places. If the rule changes, both must update.

**Fix:** Extract Function for the predicate (`isQualifiedGoldOrder(order)`) so the rule lives in one place. (Be careful: only extract if it's the *same knowledge* — see DRY discussion in `software-design-principles`. Two functions that look similar but encode different concepts shouldn't be merged.)

#### Lazy Class

A class that doesn't do enough to justify existing.

```javascript
// Bad
class UserName {
  constructor(public firstName: string, public lastName: string) {}
}
// Used like this:
const u = new UserName("Ada", "Lovelace");
console.log(u.firstName, u.lastName);
```

**Fix:** Inline Class. Just use the fields. (If there *would be* behavior — formatting, validation, comparison — the class earns its keep.)

#### Speculative Generality

Hooks, parameters, abstractions for hypothetical future needs. The most common form: an interface with one implementation and a planned-but-not-built second.

```typescript
// Bad — the interface has one implementer and the second is hypothetical
interface IPaymentGateway {
  charge(amount: Money): Promise<Receipt>;
}
class StripeGateway implements IPaymentGateway { /*...*/ }

// Used as:
const gateway: IPaymentGateway = new StripeGateway();
gateway.charge(amount);
```

**Cost:** every reader has to learn the interface; tests have to mock or fake it; the abstraction enforces nothing real.

**Fix:** Inline. Use `StripeGateway` directly. When a second gateway actually arrives, *then* extract the interface.

#### Dead Code / Commented-out Code

Unreachable, unused, or commented-out code.

**Fix:** delete. Version control remembers. Commented-out code is worse than deleted code because it makes readers wonder if they should uncomment it.

### Couplers — code that knows too much about other code

#### Feature Envy

A method uses another object's data more than its own.

```ruby
# Bad — Order.total reaches into the customer for everything
class Order
  def total
    base = items.sum(&:price)
    if @customer.tier == "gold" && @customer.years_active > 5
      base *= 0.9
    end
    if @customer.address.country == "DE"
      base *= 1.19  # VAT
    end
    base
  end
end

# Better — move the customer-specific logic to the customer
class Order
  def total
    base = items.sum(&:price)
    base = @customer.apply_discount(base)
    base = @customer.apply_taxes(base)
    base
  end
end
```

**Cost:** logic that should be on the customer lives on the order; changes to customer rules now require touching the order class.

**Fix:** Move Method to where the data lives.

#### Inappropriate Intimacy

Two classes know each other's internals.

```python
# Bad
class Order:
    def total(self):
        return sum(item.price * item.quantity for item in self.items)
        # ... but Customer reaches into Order:
class Customer:
    def total_spent(self):
        return sum(o.items[0].price for o in self.orders)  # accesses Order.items[0]
```

**Cost:** changes to one class break the other; the boundary is fictional.

**Fix:** define an explicit interface between them, or move the behavior to where the data is.

#### Message Chains (Train Wrecks)

```python
# Bad
employee.department.manager.address.city.zip_code
```

The caller has learned the entire object graph. Any change to the structure breaks every caller.

**Fix:** Hide Delegate — `employee.department_zip_code()` (Tell Don't Ask). Sometimes the underlying data should be modeled differently entirely.

#### Middle Man

A class that mostly delegates to another.

```java
// Bad
class OrderService {
  private OrderRepository repo;
  public Order get(String id) { return repo.find(id); }
  public void save(Order o) { repo.save(o); }
  public void delete(String id) { repo.delete(id); }
}
```

**Cost:** reader has to learn the wrapper *and* read the underlying class.

**Fix:** Remove Middle Man — let callers use the underlying class. (If the wrapper exists for a reason — DI seam, validation, observability — make it actually do that work.)

## Beyond the classic catalog

### Boolean Trap

Multiple boolean parameters whose meaning at the call site is unclear.

```typescript
// Bad
exportData(true, false, true, false);  // what does each mean?

// Slightly better — named args
exportData({ includeArchived: true, includeMetadata: false, asJson: true, compress: false });

// Often best — split functions named for the behavior
exportArchivedAsJson();
```

**Cost:** combinatorial behavior at call sites; easy to mix up; refactoring is risky because the call site type-checks any way the booleans are arranged.

**Fix:** named arguments where supported, or split into separate functions, or use enums.

### Mysterious Name

A name that doesn't tell you what the thing does.

```python
# Bad
def process(d, t):  # what's d? what's t? what's process?
    ...

result = process(get_data(), 0.5)
```

**Cost:** every reader has to verify the meaning. Cumulative reading cost across the codebase is enormous.

**Fix:** Rename. The most underrated and most valuable refactor.

### Magic Values

Literal numbers or strings with hidden meaning.

```python
# Bad
if user.status == 3:
    grant_access()

if attempts > 5:
    lock_account()
```

**Fix:** Replace Magic Literal with named constants.
```python
ACTIVE_STATUS = 3
MAX_FAILED_ATTEMPTS = 5
```
Better still, use an enum for closed sets of statuses.

### Hidden Side Effects

A function that looks pure but mutates state, hits the network, logs, or otherwise has effects beyond its return value.

```python
# Bad — looks like a query
def get_user(id):
    user = db.find(id)
    user.last_accessed = now()
    db.save(user)  # surprise: mutates the user
    return user
```

**Fix:** Separate Query from Modifier. Two functions: `get_user(id)` (pure query) and `mark_user_accessed(id)` (explicit mutator).

### Temporal Coupling

A sequence of calls that must happen in a specific order, with no enforcement.

```python
# Bad
client = HttpClient()
# you must call .configure() before .connect() before .request()
# nothing in the API tells you that
client.configure(opts)
client.connect()
result = client.request("/foo")
```

**Fix:** make the order explicit in the type — `HttpClient.create_and_connect(opts).request("/foo")`. Or design the API so each method only works after the prerequisites are satisfied (typestate pattern).

### God Object / God Module

A class or module that knows or does too much. Symptom: hundreds of methods, central to everything, changes for every reason.

**Fix:** decompose by responsibility. See `software-design-principles` on cohesion and coupling.

### Sequential Coupling (Accidental Sequencing)

Code that only works if called in a particular sequence the caller has to know about. Closely related to Temporal Coupling; often appears as "init, then start, then process, then stop."

**Fix:** combine the sequence into one operation, or use a state machine that prevents invalid orderings.

### Anemic Domain Model

A domain class with only data and no behavior; all the logic lives in "service" classes.

```java
// Anemic
class Order {
  // only fields, getters, setters
}
class OrderService {
  void calculateTotal(Order o) { /* ... */ }
  void applyDiscount(Order o, Discount d) { /* ... */ }
  void cancel(Order o) { /* ... */ }
}
```

**Cost:** the domain rules are scattered; the `Order` class doesn't enforce its own invariants; "Tell Don't Ask" violated everywhere.

**Fix:** move the relevant behavior onto `Order` itself — `o.calculateTotal()`, `o.applyDiscount(d)`, `o.cancel()`. Service objects make sense for cross-aggregate orchestration; not for things one entity could do to itself.

(Caveat: in some idioms — Go, Rust, functional languages — methods on a struct are essentially what services are in Java. The smell is about scattering behavior away from data, not specifically about which language construct holds the behavior.)

### Stamp Coupling

A function takes a whole structure when it only uses two fields.

```python
# Bad
def calculate_shipping(user_profile):
    return BASE_RATE if user_profile.address.country == "US" else INTERNATIONAL_RATE

# Better
def calculate_shipping(country: str):
    return BASE_RATE if country == "US" else INTERNATIONAL_RATE
```

**Cost:** function is now coupled to the entire shape of `user_profile`; refactoring the profile breaks shipping.

**Fix:** pass exactly what's needed. (Counter-tension: too narrow can produce Long Parameter List. Use judgment.)

### Yo-Yo Problem (deep inheritance)

Reading the code requires bouncing up and down the inheritance hierarchy: behavior of `methodX` lives partly in `Base`, partly in `Middle`, partly in `Concrete`. The reader has to keep four files open.

**Fix:** flatten the hierarchy. Replace Inheritance with Delegation. Composition makes the dependencies explicit at one level.

### Loops where you wanted a pipeline

```python
# Bad
result = []
for item in items:
    if item.is_valid:
        normalized = normalize(item)
        if normalized.score > THRESHOLD:
            result.append(normalized.value)

# Better
result = [n.value for item in items
          if item.is_valid
          for n in [normalize(item)]
          if n.score > THRESHOLD]
# or, more readable in many languages:
valid = (item for item in items if item.is_valid)
normalized = (normalize(item) for item in valid)
high_score = (n for n in normalized if n.score > THRESHOLD)
result = [n.value for n in high_score]
```

The pipeline reads as a sequence of operations on the data; the imperative loop hides that structure.

**Fix:** Replace Loop with Pipeline.

### Defensive over-handling

Code wrapped in try/except (or null checks) for failures that don't actually happen, or for "errors" that shouldn't be silently swallowed.

```python
# Bad
def get_email(user):
    try:
        return user.email
    except Exception:
        return None
```

If `user` doesn't have `.email`, that's a bug; silently returning `None` hides it.

**Fix:** narrow the exception type to the specific expected failure; for programmer errors, let them propagate. See `error-handling-and-resilience`.

### Yes, And: scope creep

A change that does the requested thing *and* renames variables, restructures a loop, adds type hints, "improves" error messages, adopts a new pattern from a blog post — none of which were requested.

**Cost:** diff is unreviewable. Each unrequested change carries risk. Bisect is poisoned. Reverting one piece is hard.

**Fix:** split into focused PRs. The requested change is one. Each ancillary improvement is another.

## How to use smells in code review

A finding looks like:

> **Long Parameter List** — `sendEmail` has 11 parameters; the call at `email.ts:42` is unreadable. Replace with a `SendEmailRequest` parameter object (Introduce Parameter Object).

Not:

> Bit messy here, could be cleaner.

The named smell, the file:line, the cost, the fix. Three sentences max per finding. Multiple findings in a small change usually means the right move is "request changes; consider redesign" rather than five separate line comments.

For deciding *whether* a smell is worth flagging, see `engineering-discipline` — particularly the triage rubric (Blocking / Important / Suggestion / Nit). Most smells are Suggestion or Important; few are Blocking. Don't block on smells unless they constitute a real risk.

## Reference library

- `references/extended-smell-catalog.md` — additional smells and antipatterns from the broader literature, with examples.

## Sibling skills

- `refactoring` — the corrective side. Each smell here maps to one or more named refactorings there.
- `software-design-principles` — the underlying frame. Most smells are surface symptoms of cohesion/coupling/complexity issues.
- `engineering-discipline` — how to communicate a smell finding effectively.
- `ai-coding-antipatterns` — failure modes specific to AI-generated code; complementary to this skill.
