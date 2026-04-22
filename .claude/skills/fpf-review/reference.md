# FPF Reference — Solution Design

> Distilled from [ailev/FPF](https://github.com/ailev/FPF) for use in solution design.
> Covers: A.1, A.1.1, A.2, A.3, A.7, A.15

---

## 1. Core Types (A.1)

FPF uses a three-tier root ontology to avoid category errors:

```
U.Entity     — anything that can be individuated. No structural assumptions.
  └─ U.Holon — entity with part-whole structure. Simultaneously a whole and a part.
       ├─ U.System   — physical/operational holon. Can ACT. Bears roles, executes methods.
       └─ U.Episteme — knowledge holon (model, spec, doc). CANNOT act. Passive content.
```

**Key rule:** Behavioural roles attach ONLY to `U.System`. An `U.Episteme` cannot "decide", "execute", or "process". If a sentence reads "the document decided" — it violates A.1.

**Inside/Outside test** — to decide if entity E belongs inside holon H:
1. Dependency: removing E breaks a core invariant of H?
2. Interaction: E participates in causal loops wholly within H's boundary?
3. Emergence: E contributes to a novel collective property of H?

Fail all three → E is outside.

---

## 2. Bounded Context (A.1.1)

A `U.BoundedContext` is an explicit semantic frame: a named scope where terms, roles, and rules have defined local meaning.

**Components:**
- **Glossary** — local vocabulary ("ticket" means X here, not Y)
- **Invariants** — local rules that must hold within this context
- **Roles** — role taxonomy valid only inside this context
- **Bridges** — explicit mappings to other contexts (with loss/fit notes)

**Critical distinctions:**
- Domain ≠ Bounded Context. "Healthcare" is a domain family label. `Hospital.OR_2025` is a bounded context.
- Contexts are FLAT — no context inherits from another. Cross-context relations = explicit Bridges only.
- Every `U.RoleAssignment` references exactly one `U.BoundedContext`.

**Anti-patterns:**
| Anti-pattern | Fix |
|---|---|
| Using "Healthcare" where a specific context is needed | Use specific context id |
| Same term across contexts assumed equal | Publish explicit Bridge with loss/fit note |
| "Sub-contexts" with inheritance | Remove; express via Bridges only |
| "Design context" vs "Runtime context" as separate contexts | Use DesignRunTag on artifacts; keep semantic frame fixed |

**In code:** Accessing another context's internals without a Bridge = boundary violation. A module boundary in code maps directly to a Bounded Context boundary — crossing it without an explicit interface is an implicit cross-context dependency.

---

## 3. Strict Distinctions — A.7 (Clarity Lattice)

The most important FPF guard. Prevents four categories of confusion:

### 3.1 Role vs Method vs Work

| Term | What it is | Design/Run |
|------|-----------|-----------|
| **Role** | Contextual mask/position a system can bear | Design |
| **Method** | Abstract way-of-doing; timeless capability | Design |
| **MethodDescription** | Recipe/SOP/algorithm — an Episteme describing a Method | Design |
| **Capability** | System's ability to perform a Method | Design |
| **WorkPlan** | Schedule of intended Work occurrences | Design |
| **Work** | Dated, resource-consuming execution — what actually happened | Run |

**Chef analogy:**
- `ChefRole` = Role (job title)
- Cookbook = MethodDescription (recipe)
- Chef's skill = Capability
- Cooking Tuesday evening = Work (actual occurrence)

**Guards:**
- Never use MethodDescription as evidence that Work was done ("paper compliance")
- Never present Method (capability) as if it has happened
- "Process" is ambiguous — resolve to Method/MethodDescription (recipe), WorkPlan (schedule), or Work (run)

### 3.2 System vs Episteme (who can act)

- **System** — the only holon that can bear behavioural roles and execute Method/Work
- **Episteme** — cannot act; changed by a system acting on its carriers
- **Carrier** — physical/digital file that holds an Episteme (≠ the Episteme itself)

**Guards:**
- "The spec was updated" → state which carrier changed
- "The model decided" → invalid; a system bearing a role decided

### 3.3 Set vs Collective System

- **Set/Collection** — mathematical grouping, no joint behaviour
- **Collective System** — a system with boundary, roles, and methods (e.g., a team)

If a grouping is expected to act → model it as a Collective System.

---

## 4. Role Taxonomy (A.2)

A `U.Role` is a context-bound mask a holon wears. It is NOT behaviour, NOT a structural part, NOT a type.

**Assignment notation:** `Holder#Role:Context@Window`
- Example: `PaymentService#TransformerRole:BillingContext`
- Example: `DataPipeline#ObserverRole:MonitoringContext`

**Three kernel roles (substrate-neutral):**
| Role | Intent | System example | Software example |
|------|--------|---------------|-----------------|
| `TransformerRole` | Changes other holons via Method/Work | Robot arm | API handler, data transformer, build system |
| `ObserverRole` | Collects evidence/metrics | Sensor | Logger, metrics collector, audit service |
| `SupervisorRole` | Governs subordinate holons | PLC | Orchestrator, scheduler, config manager |

**Key rules:**
- Roles are NEVER structural parts — never put a role in a component hierarchy
- Same system can bear multiple roles in the same context (if compatible)
- "The service decides" → a service bearing TransformerRole decides
- No self-transformation: "the system configures itself" must be split into Regulator (acting) + Regulated (target)

---

## 5. Role–Method–Work Alignment (A.15)

The canonical chain from context to execution:

```
U.BoundedContext
  └─ defines → U.Role
                  └─ RoleAssignment (Holder#Role:Context)
                       └─ governs → U.Work (run-time)
                                      └─ isExecutionOf → U.MethodDescription
```

**Common violations:**
1. **Role-as-Part** — putting a Role inside a structural bill-of-materials (Role ≠ component)
2. **Spec-as-Execution** — citing MethodDescription as evidence work was done
3. **Capability-as-Work** — confusing ability to do with actually doing
4. **Work-without-Context** — logging Work with no link to Role/Method

**Design-time vs Run-time:**
- Design-time: Role, Method, MethodDescription, Capability, WorkPlan
- Run-time: Work (dated, resource-consuming, with start/end)

---

## 5a. Safe Vocabulary (A.3 Didactic Dictionary)

Before naming things, map common words to FPF terms:

| Everyday word | FPF term | Why |
|--------------|----------|-----|
| Process, Workflow, SOP, Algorithm, Script | `MethodDescription` | Design-time recipe; lives in Tᴰ |
| Operation, Job, Run, Execution, Deploy | `Work` | Run-time occurrence; lives in Tᴿ |
| Function (what a system can do) | `Method` or `Capability` | Abstract capability |
| Service (entry point) | `U.RoleAssignment` + `MethodDescription` | Access spec |
| Document, Spec, Config, Schema | `Episteme` (+ its Carrier) | Knowledge artifact; cannot act |
| Team, Group | `Collective System` (if it acts) or `Set` (if it doesn't) | |

---

## 6. Key Failure Modes

| Failure | Symptom | FPF rule violated |
|---------|---------|------------------|
| Method = Tool | "We use Jira as our method" | A.7 / A.3 |
| MethodDescription = Evidence | "We have a spec, so it's done" | A.7:5.3 |
| Episteme acts | "The model decided X" | A.1, A.7:5.4 |
| Role = Component | Role placed in structural hierarchy | A.15 |
| Domain = Context | "Healthcare context" | A.1.1 |
| Implicit cross-context equivalence | Same term, different contexts, assumed equal | A.1.1 |
| Work without Role/Method trace | Execution with no governing context | A.15 |
| Set acts | "The team decided" without modeling as Collective System | A.7:5.6 |
| Abduction skipped | Solution proposed without naming the problem | B.5 |

---

## 8. Quick Reference — Safe Vocabulary

| Instead of... | Use... | Pattern |
|--------------|--------|---------|
| "Process" | Method (recipe), WorkPlan (schedule), or Work (execution) | A.15 |
| "Function" | Role (mask) or Method (behaviour under role) | A.7:5.2 |
| "System" for a theory/spec | Episteme | A.1 |
| "The document decided" | "System bearing [Role] updated the document" | A.1, A.7 |
| "Domain context" | Specific bounded context id | A.1.1 |
| "Sub-context" | Bridge between two contexts | A.1.1 |
| "It's done" (pointing at spec) | Work record with start/end and resource trace | A.15 |
