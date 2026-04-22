---
name: fpf-review
description: Review code for FPF design violations — responsibility boundaries, passive-vs-active confusion, design/runtime mixing. Complements CodeRabbit: focuses on design intent and business logic correctness, not syntax or style.
argument-hint: [file or directory path, or leave empty to review current git diff]
allowed-tools: Read Grep Bash
---

Review code for FPF design violations. Read [reference.md](reference.md) before proceeding.

**Scope:** Design and business logic correctness only.
Do NOT check: syntax, style, naming conventions, performance, security, test coverage — CodeRabbit handles those.

---

## Step 1: Get the code to review

If $ARGUMENTS is empty — run `git diff HEAD` to get current changes.
If $ARGUMENTS is a file path — read that file.
If $ARGUMENTS is a directory — grep for violation patterns across files in that directory.

```bash
git diff HEAD          # uncommitted changes
git diff main...HEAD   # all changes on current branch vs main
```

---

## Step 2: Scan for FPF violation patterns

Look for these patterns in the code:

### Pattern 1 — Passive object that acts
Data classes, DTOs, config objects, schemas that have methods making decisions.

**Signals to grep for:**
- Config/settings class with methods like `decide`, `resolve`, `handle`, `process`, `apply`
- DTO or record class with conditional business logic
- Schema or model class that calls other services

```
Examples:
  class JobConfig { shouldRun() {...} }        ← config decides → violation
  class UserDTO { isEligible() {...} }          ← DTO judges → check context
  data class Schema { fun validate() {...} }    ← depends: pure structural check = ok, business rule = violation
```

### Pattern 2 — Mixed design-time and runtime in one class
A class that is simultaneously the recipe, the execution record, and the actor.

**Signals:**
- Class named `XxxProcessor`, `XxxPipeline`, `XxxWorkflow` that also stores execution state (timestamps, status, results)
- A "Service" class that both describes how to do something AND stores what happened
- Methods like `process()` that both execute logic AND write to a result field on `this`

```
Examples:
  class PaymentProcessor {
    var status: String        ← execution state (Work)
    fun process() { ... }     ← execution (Work)
    val rules = ...           ← recipe (MethodDescription)
  }                           ← all three mixed → violation
```

### Pattern 3 — Boundary leak
One module directly accessing the internals of another instead of going through its interface.

**Signals:**
- Import of an `internal` package class from outside that package
- Direct database table access from a service that doesn't own that table
- Reaching into another service's domain model (not its API/contract)

```
Examples:
  // In OrderService:
  import com.company.payment.internal.PaymentRecord  ← internal class → violation
  paymentRepository.findAll()                        ← another service's repo → violation
```

### Pattern 4 — Same term, different meaning across modules
The same class name, field name, or concept used with different semantics in different parts of the system — with no explicit translation.

**Signals:**
- `status` field with different possible values in different modules
- `User` class that means different things in auth vs billing vs notifications
- `Event` used as both a domain event and a UI event without separation

### Pattern 5 — Spec treated as execution evidence
A specification, plan, or contract cited as if it proves work was done.

**Signals in code:**
- A test annotated `@Ignore` or `@Disabled` but counted in coverage
- A TODO comment used as a placeholder instead of an actual implementation
- An interface with a no-op implementation that silently passes all calls

### Pattern 6 — Business rule in the wrong layer
Domain or business logic placed in infrastructure code (controller, repository, DTO, mapper) instead of the service or domain layer.

**Signals:**
- Validation beyond input format (e.g. "user must have active subscription") in a controller or DTO
- Business decision (e.g. "if order status is X, do Y") inside a repository method
- Conditional domain logic in a mapper or serializer

```
Examples:
  fun toDto(order: Order): OrderDto {
    if (order.status == CANCELLED && order.total > 0) refund()  ← business rule in mapper → violation
  }

  @PostMapping("/pay")
  fun pay(...) {
    if (user.balance < amount) throw InsufficientFundsException()  ← domain rule in controller → violation
  }
```

### Pattern 7 — Orchestration mixed with business logic
A single service both coordinates other services AND implements domain rules. These are different responsibilities and change for different reasons.

**Signals:**
- A service calls 3+ other services AND contains non-trivial conditional logic
- Methods named `process()` or `execute()` that both delegate to other services and compute domain results
- Business calculations inside a class whose main job is sequencing calls

```
Examples:
  class OrderService {
    fun submitOrder(order: Order) {
      val discount = if (order.items.size > 10) 0.1 else 0.0   ← domain rule
      paymentService.charge(order.total * (1 - discount))       ← orchestration
      inventoryService.reserve(order.items)                     ← orchestration
      notificationService.notify(order.userId)                  ← orchestration
    }
  }
  ← discount logic should be in a domain/pricing service, not here
```

### Pattern 8 — Domain object crossing module boundary without translation
An object from one module's domain model is passed directly into another module instead of being mapped to the receiving module's own model.

**Signals:**
- A class from `com.company.orders` used directly as a parameter or return type in `com.company.billing`
- Response object from one service returned as-is from another service's method
- Shared "common" model used across multiple bounded contexts with no explicit mapping

```
Examples:
  // In BillingService:
  fun charge(order: orders.Order): Receipt  ← orders.Order crosses into billing → violation
  // Fix: map to billing.PaymentRequest first
```

### Pattern 9 — Business rule duplicated across the codebase
The same business rule expressed in multiple places. When the rule changes, all copies must be updated — and they won't be.

**Signals:**
- Identical or near-identical conditionals in more than one service (e.g. `if (user.tier == PREMIUM)`)
- The same validation logic in both the API layer and the domain layer with no shared source
- Status transition rules repeated in multiple methods instead of centralized in one place

```
Examples:
  // In OrderController:
  if (order.status != PENDING) throw InvalidStateException()
  // Also in OrderService:
  if (order.status != PENDING) throw InvalidStateException()   ← duplicated rule → violation
  // Fix: one method `order.requirePending()` or a state machine
```

---

## Step 3: Verify each candidate violation in context (mandatory)

Before flagging anything:
1. **Read the call site** — how is this class/method actually used?
2. **Read surrounding code** — is there a wrapper/adapter that corrects the apparent violation?
3. **Check if it's dead code** — is the pattern even reachable at runtime?

Only report if the violation holds after this check.

---

## Step 4: Output

Format each finding as a PR-style comment. Plain language — no FPF jargon unless it adds precision.

```
## FPF Design Review

### ✅ Looks good
[what is well-separated and correctly modeled]

### ⚠️ Issues

**[File:line]**
What: [plain description of the problem]
Why it matters: [what breaks or becomes harder if left as-is]
Fix: [concrete suggestion — rename, move logic, split class, etc.]

### Not checked
Syntax, style, performance, security, test coverage → handled by CodeRabbit.
```

---

## Example

Input: `/fpf-review src/scheduler/`

Output:
```
## FPF Design Review

### ✅ Looks good
- SchedulerService is cleanly separated from JobConfig — config is passed in, not embedded
- JobRecord correctly stores only execution data (timestamps, status, output)

### ⚠️ Issues

**JobConfig.kt:34**
What: `JobConfig.shouldRunNow()` makes a scheduling decision based on current time.
Why it matters: Config is passive data — putting decision logic here means the decision
  can't be tested without a real config, and it's invisible to anyone reading SchedulerService.
Fix: Move `shouldRunNow()` to SchedulerService, pass config as input parameter.

**PipelineRunner.kt:67**
What: PipelineRunner imports `internal.StepRegistry` from the transform module directly.
Why it matters: This creates a hidden dependency on transform internals —
  if StepRegistry moves or changes, PipelineRunner breaks silently.
Fix: Access StepRegistry through TransformService's public interface, not the internal package.

### Not checked
Syntax, style, performance, security → handled by CodeRabbit.
```
