# fpf-claude-skills

A collection of Claude Code skills that apply **FPF (First Principles Framework)** to software design and code review.

| Skill | What it does |
|-------|-------------|
| `/fpf-design` | Analyze a problem or validate a design description for FPF violations |
| `/fpf-review` | Review code for design violations — complements CodeRabbit |

Based on [ailev/FPF](https://github.com/ailev/FPF).

**Lightweight** — uses a ~4.5K-token distilled reference, not the full FPF spec (1.2M tokens). Best used on a single file or `git diff`, not entire directories.

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

You can install both skills or just the one you need.

**Both skills, personal (available in all projects):**
```bash
git clone https://github.com/AnastasiaKarpenko/fpf-claude-skills
cp -r fpf-claude-skills/.claude/skills/fpf-design ~/.claude/skills/fpf-design
cp -r fpf-claude-skills/.claude/skills/fpf-review ~/.claude/skills/fpf-review
```

**One skill only:**
```bash
git clone https://github.com/AnastasiaKarpenko/fpf-claude-skills
cp -r fpf-claude-skills/.claude/skills/fpf-design ~/.claude/skills/fpf-design   # design only
cp -r fpf-claude-skills/.claude/skills/fpf-review ~/.claude/skills/fpf-review   # review only
```

**Project-specific:**
```bash
cp -r fpf-claude-skills/.claude/skills/fpf-design your-project/.claude/skills/fpf-design
cp -r fpf-claude-skills/.claude/skills/fpf-review your-project/.claude/skills/fpf-review
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

---

## /fpf-review — code review

Code review has a dirty secret: most automated tools (linters, static analyzers, CodeRabbit) are great at catching *how* the code is written, but none of them ask *whether the design makes sense*. That's the gap this skill fills.

It looks at the structure of your changes and asks: are responsibilities placed where they belong? Do module boundaries hold? Is business logic where it should be — or scattered across layers that shouldn't know about it?

**What it catches:**

- A config object or DTO that contains decision logic — data that's making up its own mind
- A class that's simultaneously a recipe, an executor, and a result log — three jobs, one place
- A service reaching into another module's internals instead of going through its interface
- The same business rule copy-pasted in three places — waiting to drift apart
- Domain logic sitting in a controller or a mapper where it doesn't belong
- One service doing both orchestration and computation — coordination and business rules tangled together
- A domain object crossing a module boundary without being translated — leaking your internal model into someone else's context
- A TODO or disabled test presented as working functionality

Each finding comes with a plain-language explanation of what's wrong, why it matters, and a concrete suggestion for how to fix it.

**What it doesn't check** — and doesn't try to: syntax, style, naming, performance, security, test coverage. CodeRabbit handles those. And it can't tell you whether your business logic is *correct* — that requires knowing the domain and reading the ticket. That part stays with the human reviewer.

**What human reviewers still own:**
- Does this solution actually solve the stated problem?
- Are the business rules themselves right — edge cases, domain invariants, correctness?
- Does this change introduce risks in areas the diff doesn't show (downstream effects, data migrations)?
- Is the overall approach the right one, or is there a simpler design?

```
/fpf-review                   # reviews current git diff
/fpf-review src/scheduler/    # reviews a specific file or directory
```

Returns a PR-style report:
- ✅ what is well-separated and correctly structured
- ⚠️ issues with plain-language explanation and concrete fix suggestion
- what was not checked (so you know what's still on you)

---

## File structure

```
.claude/skills/
├── fpf-design/
│   ├── SKILL.md       — skill instructions and methodology
│   └── reference.md   — distilled FPF principles (self-contained)
└── fpf-review/
    ├── SKILL.md       — review instructions and violation patterns
    └── reference.md   — same FPF reference (shared)
```

---

## License

MIT
