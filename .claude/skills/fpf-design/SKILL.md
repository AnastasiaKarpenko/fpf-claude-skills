---
name: fpf-design
description: Apply FPF (First Principles Framework) principles to analyze problems or validate designs. Two modes — analyze: takes a problem description and structures it through FPF lens; validate: takes an existing design and checks it against FPF principles.
argument-hint: analyze "<problem>" | validate "<design>"
allowed-tools: Read
---

Apply FPF (First Principles Framework) to the input based on the mode in $ARGUMENTS.

Read [reference.md](reference.md) before proceeding — it contains the FPF distillation you need.

---

## Mode: analyze

**Trigger:** $ARGUMENTS starts with `analyze` or no mode is specified and a problem is described.

**Goal:** Take a raw problem or design challenge and structure it using FPF concepts. Return a clear FPF-structured design proposal.

**Steps — execute in order:**

### Step 1: Identify the System-of-Interest
- What is the primary holon being designed or changed?
- Is it a `U.System` (operational, can act) or `U.Episteme` (knowledge artifact)?
- Name it explicitly. If it's ambiguous, propose alternatives.

### Step 2: Define the Bounded Context
- What is the semantic frame for this design?
- Name the context (specific, not a domain family label)
- List key terms that need local definitions (Glossary)
- State at least one invariant that must hold
- Are there adjacent contexts that need Bridges?

### Step 3: Map Roles
- Who or what acts in this context?
- List `Holder#Role:Context` assignments
- Remember: Role is a mask, not the behavior itself

### Step 4: Identify Methods and Work Products
- What are the Methods (ways of doing)?
- What are the MethodDescriptions (recipes/SOPs)?
- What is the expected Work (run-time outputs with criteria)?
- Distinguish design-time (Method, MethodDescription, WorkPlan) from run-time (Work)

### Step 5: Apply Strict Distinctions check
- Is anything being treated as acting that can't act (Episteme with behavior)?
- Is a MethodDescription being used as evidence of Work?
- Are cross-context terms translated via explicit Bridges?
- Is "process" resolved to Method/WorkPlan/Work?

### Step 6: Reasoning path (B.5)
State plainly:
- What is the hypothesis being proposed?
- What would break if it's wrong? (consequences to check)
- How would you know it works? (validation signal)

**Output format:**
```
## System-of-Interest
[name, type: System/Episteme, boundary]

## Bounded Context
[context id, key glossary entries, at least one invariant, bridges if needed]

## Roles
[Holder#Role:Context for each actor]

## Method–Work
[MethodDescriptions (recipes), expected Work (outputs with criteria)]

## FPF distinctions applied
[what was clarified, any violations found and resolved]

## Hypothesis & validation
[what's being proposed → how to verify it]
```

**Software example:**

Input: `analyze "we need to split our monolith payment service into smaller services"`

Output:
```
## System-of-Interest
PaymentService (U.System), boundary: all payment transaction logic

## Bounded Context
BillingContext: "transaction" = financial record; "payment" = authorized transfer
Invariant: a transaction cannot move to Settled without an Authorization record

## Roles
BillingService#TransformerRole:BillingContext (processes payments)
AuditService#ObserverRole:BillingContext (records evidence)
Gateway#SupervisorRole:BillingContext (routes and governs)

## Method–Work
MethodDescriptions: AuthorizePayment SOP, SettleTransaction SOP
Expected Work: AuthorizePayment execution → AuthorizationRecord (with timestamp, amount, status)

## FPF distinctions applied
- "The config decides routing" → violation fixed: Gateway#SupervisorRole decides, config is MethodDescription
- "Payment service" resolved to Collective System with explicit roles

## Hypothesis & validation
Hypothesis: splitting by bounded context (BillingContext / FraudContext) reduces coupling
Verify: each context can be deployed independently without shared schema changes
```

---

## Mode: validate

**Trigger:** $ARGUMENTS starts with `validate`.

**Goal:** Check an existing design or decision against FPF principles. Return a conformance report.

**Checklist — check each item:**

### Actors and boundaries
- [ ] Every acting entity can actually act on its own (not a document, config, or spec)
- [ ] Boundaries are declared — what is inside/outside?
- [ ] Groups expected to act are modeled as units with their own boundary and roles (not just a list)

### Semantic scope
- [ ] Terms are defined locally — the same word in two different parts of the system may mean different things
- [ ] Cross-boundary term alignment is explicit, not assumed
- [ ] Each Role is defined inside a specific scope, not globally
- [ ] Scopes are independent — one module/service does not silently inherit the rules or definitions of another just because they share a namespace or package
- [ ] A file or document is not the same as what it represents — a schema file is not the data contract, a config file is not the logic, a test file is not evidence that tests pass

### Clarity of concepts
- [ ] Role (what position something holds) ≠ Method (what it does in that position)
- [ ] A recipe/spec/SOP is not evidence that the work was actually done
- [ ] Documents, configs, schemas cannot act, decide, or execute — only systems can
- [ ] "Process" is ambiguous — resolve to: recipe (how to do it), schedule (when to do it), or execution (what happened)

### Design vs runtime
- [ ] Each execution traces back to: who did it → in what role → following which recipe
- [ ] Design-time artifacts (roles, recipes, plans) are not confused with run-time events (what actually happened)
- [ ] No "paper compliance" — having a spec does not mean the work was done

### Reasoning
- [ ] New proposals name the problem they solve (not solution in search of a problem)
- [ ] Design decisions state how they would be validated

**Software-specific checks:**
- [ ] "The database decides X" → violation (database = Episteme; a service bearing a role decides)
- [ ] "The config controls behavior" → config is MethodDescription, a system enacts it
- [ ] "The pipeline processes itself" → no self-transformation; split into Transformer + Target
- [ ] "Service A calls Service B" → is the Role assignment explicit? Is the context declared?
- [ ] API contract treated as evidence work is done → MethodDescription ≠ Work

### Before reporting a violation — verify in context (mandatory)

For each candidate violation found above, before including it in the report:
1. **Read the call sites** — how is the artifact actually used by callers?
2. **Read surrounding code** — are there adjacent log lines, guards, or comments that already resolve the apparent violation?
3. **Check the runtime path** — does the violation actually occur at runtime, or is it dead code / an unreachable branch?

Only report a violation if it holds after this check. If surrounding context resolves it, note it as compliant with an explanation. A violation that exists only when reading an artifact in isolation — but is disambiguated by its call site — is not a real violation.

**Output format:**
```
## Conformance Report

### ✅ Compliant
[what is correctly modeled]

### ⚠️ Violations
[plain-language description of the violation — suggested fix]

### Recommendation
[what to fix before committing this design]
```

**Software example:**

Input: `validate "The Scheduler system reads the job config and decides when to run tasks. The pipeline executes itself based on the trigger."`

Output:
```
## Conformance Report

### ✅ Compliant
- Scheduler is modeled as a System (can act) ✓

### ⚠️ Violations
1. "reads the config and decides" — a config file cannot decide anything, it has no agency. Fix: the Scheduler decides by following the job config as a recipe.
2. "pipeline executes itself" — nothing can transform itself. Fix: split into Scheduler (the one acting) and Pipeline (the target being triggered).

### Recommendation
Rewrite as: Scheduler#SupervisorRole:PipelineContext enacts TriggerJob MethodDescription → produces Work record per scheduled run.
```
