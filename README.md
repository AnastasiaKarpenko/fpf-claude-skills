# fpf-claude-skills

Two Claude Code skills for catching design problems — before you write code and after.

| Skill | When to use | What you get |
|-------|-------------|-------------|
| `/fpf-design` | You have a problem or a design idea and want to think it through before coding | Structured breakdown: what acts, what's passive, where boundaries are, what's a guess vs what's decided |
| `/fpf-review` | You have a diff or a file and want to check the structure of the implementation | PR-style report: what's clean, what's tangled, concrete fix suggestions |

**The short version:** `/fpf-design` is for thinking. `/fpf-review` is for reviewing.

Based on [ailev/FPF](https://github.com/ailev/FPF). Lightweight — ~4.5K tokens of distilled principles, not the full 1.2M-token spec.

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

## /fpf-design — design thinking

Use this when you're about to build something and want to think through the structure first — or when you've described a design and want to catch problems before writing code.

```
/fpf-design analyze we need to split the data transformation layer — right now one service
  handles ingestion, validation, and enrichment and it's getting hard to change

/fpf-design validate The Scheduler reads the job config and decides when to run tasks.
  The pipeline executes itself based on the trigger.

/fpf-design analyze how to design a review process for data quality across multiple upstream sources
```

The more context you give, the more precise the output.

### analyze mode

Describe a problem. Returns a structured breakdown:
- **What is being built** — the thing you're designing and where its boundary is
- **Who does what** — which components act, which just hold data, who coordinates
- **Local vocabulary** — what terms mean in this context, and how they map to adjacent systems
- **Design vs runtime** — what's a recipe/plan vs what's an actual execution with output
- **What's unclear** — assumptions and open questions surfaced explicitly

### validate mode

Describe an existing design. Returns a **Conformance Report**:
- ✅ what is correctly modeled
- ⚠️ issues with a plain-language explanation of what's wrong and a suggested fix
- recommendation before committing the design

### Example output (validate mode)

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

## /fpf-review — code review

Code review has a dirty secret: most automated tools (linters, static analyzers, CodeRabbit) are great at catching *how* the code is written, but none of them ask *whether the design makes sense*. That's the gap this skill fills.

It looks at the structure of your changes and asks: are responsibilities placed where they belong? Do module boundaries hold? Is business logic where it should be — or scattered across layers that shouldn't know about it?

```
/fpf-review                   # reviews current git diff
/fpf-review src/scheduler/    # reviews a specific file or directory
```

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

Returns a PR-style report:
- ✅ what is well-separated and correctly structured
- ⚠️ issues with plain-language explanation and concrete fix suggestion
- what was not checked (so you know what's still on you)

---

## Background: why FPF?

FPF (First Principles Framework) is a framework by [Anatoly Levenchuk](https://github.com/ailev/FPF) that defines universal distinctions for thinking about systems and design. These skills apply a distilled subset of it — you don't need to know FPF to use them.

Core ideas behind the checks:

- **Services act. Documents don't.** A service can execute logic. A config file, a spec, a schema — these describe what to do, they can't do anything on their own.
- **Having the recipe doesn't mean dinner was made.** A spec, a workflow description, a plan — these are design-time artifacts. Actual work has a timestamp, a cost, and an output. Confusing the two is one of the most common sources of "we thought it was done" bugs.
- **Terms mean different things in different contexts.** Making context boundaries explicit prevents silent semantic drift between teams or modules.

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
