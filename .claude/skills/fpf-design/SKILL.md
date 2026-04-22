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

### Holon & Boundary (A.1)
- [ ] Every acting entity is typed as `U.System` (not generic Holon or Episteme)
- [ ] Boundaries are declared — what is inside/outside?
- [ ] Groups expected to act are modeled as Collective Systems (not sets)

### Bounded Context (A.1.1)
- [ ] Context is specific (not a domain family label like "Healthcare")
- [ ] Cross-context term alignment uses explicit Bridges
- [ ] No context hierarchy — contexts are flat
- [ ] Each Role is defined inside a context

### Strict Distinctions (A.7)
- [ ] Role (mask) ≠ Method (behavior)
- [ ] MethodDescription (recipe) ≠ evidence that Work was done
- [ ] Episteme does not "act", "decide", or "execute"
- [ ] Carrier (file/document) ≠ Episteme (knowledge content)
- [ ] "Process" is resolved to Method / WorkPlan / Work

### Role–Method–Work (A.15)
- [ ] Each Work traces back to: Role → RoleAssignment → MethodDescription
- [ ] Design-time artifacts (Role, Method, MethodDescription) are not confused with run-time (Work)
- [ ] No "paper compliance" — spec existence ≠ work done

### Reasoning (B.5)
- [ ] New claims name the problem they solve (not solution-in-search-of-a-problem)
- [ ] Design decisions state how they would be validated

**Software-specific checks:**
- [ ] "The database decides X" → violation (database = Episteme; a service bearing a role decides)
- [ ] "The config controls behavior" → config is MethodDescription, a system enacts it
- [ ] "The pipeline processes itself" → no self-transformation; split into Transformer + Target
- [ ] "Service A calls Service B" → is the Role assignment explicit? Is the context declared?
- [ ] API contract treated as evidence work is done → MethodDescription ≠ Work

**Output format:**
```
## Conformance Report

### ✅ What's correct
[what is correctly modeled, in plain language]

### ⚠️ Issues found
For each issue:
- What the problem is — in plain, non-jargon language
- Why it matters (what breaks if left as-is)
- Concrete fix — a rewritten sentence or design change

### Recommendation
[one clear action to take before committing this design]
```

**Important:** Explain violations in plain language first. Use FPF terms only when they add precision, and always follow with a plain explanation in parentheses.

**Software example:**

Input: `validate "The Scheduler system reads the job config and decides when to run tasks. The pipeline executes itself based on the trigger."`

Output:
```
## Conformance Report

### ✅ What's correct
- Scheduler is treated as an active system that can run things ✓

### ⚠️ Issues found

1. "The config decides when to run"
   A config file is a passive document — it can't make decisions.
   What actually decides is the Scheduler, using the config as instructions.
   Fix: "The Scheduler reads the job config as a set of rules and decides when to trigger runs."

2. "The pipeline executes itself"
   Nothing can trigger itself — there's always an external actor causing the action.
   Fix: split into two things — the Scheduler (which triggers) and the Pipeline (which gets triggered).
   Rewrite: "The Scheduler triggers the Pipeline when conditions in the job config are met."

### Recommendation
Rewrite as: Scheduler reads job config → Scheduler triggers Pipeline → Pipeline run is recorded with timestamp and outcome.
```
