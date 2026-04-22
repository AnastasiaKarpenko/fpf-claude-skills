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
