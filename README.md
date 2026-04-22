# fpf-design — Claude Code Skill

A Claude Code skill that helps you design solutions and validate architecture decisions using **FPF (First Principles Framework)** — a set of universal distinctions that keeps your thinking precise and your models category-error-free.

Two modes:
- **`analyze`** — describe a problem, get a structured FPF-based design
- **`validate`** — describe an existing design, get a conformance report with violations and fixes

Based on [ailev/FPF](https://github.com/ailev/FPF).

---

## Why this exists

Design discussions often get stuck because people use the same words to mean different things:

- "Process" — is it a recipe, a schedule, or something that already ran?
- "The service decides" — can a service actually decide, or does something else?
- "We have a spec, so it's done" — is a specification evidence that work happened?

FPF gives you a precise vocabulary for these distinctions. This skill applies that vocabulary automatically — in analyze mode it structures your problem, in validate mode it catches the mistakes before they become architecture debt.

---

## What is FPF?

FPF (First Principles Framework) is a framework by [Anatoly Levenchuk](https://github.com/ailev/FPF) that defines universal distinctions for correct thinking about systems, knowledge, and action. It's domain-agnostic — it applies equally to software architecture, process design, data systems, and organizational design.

Core ideas in plain language:

- **Systems act. Documents don't.** A service can execute logic. A config file, a spec, a schema — these are knowledge artifacts that describe what to do. They can't decide anything on their own.
- **Recipe ≠ cooking.** A MethodDescription (SOP, algorithm, workflow) is design-time knowledge. Work is the actual run-time execution with a timestamp and resource cost. Having the recipe doesn't mean dinner was made.
- **Role ≠ structure.** A role is a contextual mask a system wears (e.g. `PaymentService#TransformerRole`). It's not a component in a hierarchy and not a type.
- **Local meaning.** Terms mean different things in different contexts. FPF makes context boundaries explicit so there's no silent semantic drift between teams or services.

---

## Installation

**Personal (available in all projects):**
```bash
git clone https://github.com/your-username/fpf-design-skill
cp -r fpf-design-skill/.claude/skills/fpf-design ~/.claude/skills/fpf-design
```

**Project-specific:**
```bash
cp -r fpf-design-skill/.claude/skills/fpf-design your-project/.claude/skills/fpf-design
```

Restart Claude Code after installing.

---

## Usage

You can write short or long descriptions — the more context you give, the more precise the output.

```
/fpf-design analyze we need to split the data transformation layer — right now one service handles ingestion, validation, and enrichment and it's getting hard to change

/fpf-design validate The Scheduler reads the job config and decides when to run tasks. The pipeline executes itself based on the trigger.

/fpf-design analyze how to design a review process for data quality across multiple upstream sources
```

### analyze mode

Returns:
- **System-of-Interest** — what is being designed, its type and boundary
- **Bounded Context** — local vocabulary, invariants, cross-context bridges
- **Roles** — who acts and in what capacity (`Holder#Role:Context`)
- **Method–Work** — recipes (design-time) vs outputs with criteria (run-time)
- **FPF distinctions applied** — what was clarified or corrected
- **Hypothesis & validation** — what's being proposed and how to verify it

### validate mode

Returns a **Conformance Report**:
- ✅ what is correctly modeled
- ⚠️ violations with the FPF rule broken and a suggested fix
- recommendation before committing the design

---

## Example output (validate mode)

Input:
```
/fpf-design validate The pipeline executes itself based on the trigger config.
```

Output:
```
## Conformance Report

### ⚠️ Issues found

1. "The pipeline executes itself"
   Nothing can trigger itself — there's always an external actor causing the action.
   Fix: split into two things — the Scheduler (which triggers) and the Pipeline (which gets triggered).
   Rewrite: "The Scheduler triggers the Pipeline when conditions in the config are met."

2. "The config decides"
   A config file is a passive document — it can't make decisions.
   What decides is the Scheduler, using the config as a set of instructions.
   Fix: "The Scheduler reads the config and decides when to trigger a run."

### Recommendation
Rewrite as: Scheduler reads config → Scheduler triggers Pipeline → Pipeline run is recorded with timestamp and outcome.
```

---

## File structure

```
.claude/skills/fpf-design/
├── SKILL.md       — skill instructions and methodology
└── reference.md   — distilled FPF principles (self-contained, no external dependency)
```

---

## License

MIT
