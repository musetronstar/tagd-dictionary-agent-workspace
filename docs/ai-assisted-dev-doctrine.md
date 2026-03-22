# AI-Assisted Development Doctrine

## Command Structure

|                                 Role                             |                       Owner                   |
|------------------------------------------------------------------|-----------------------------------------------|
| Mission, scope, constraints, acceptance criteria, review         | **Chat / LLM**                                |
| Code inspection, local decisions, implementation, test execution | **Coding Agent**                              |
| Standing orders, enduring doctrine, behavior contract            | **Repo artifacts** (AGENTS.md, tests, README) |

### Rule

* The LLM defines what must be true.
* The Coding Agent decides how to make it true.

---

## Responsibilities

### Chat / LLM
- Set direction and define the mission
- Scope which files may change
- State constraints and non-goals
- Define acceptance criteria
- Review diffs: boundaries, coupling, test quality, naming
- Issue the next mission based on review

The LLM operates at **architectural altitude**.  
It should not choreograph internal code moves unless correcting a specific known mistake.

### Coding Agent
- Read and inspect the code
- Make local implementation decisions
- Execute changes
- Run tests
- Report results concisely

The Coding Agent operates at **implementation altitude**.  
It should decide *how* to do the work within the given boundaries.

### Repo Artifacts
- Carry enduring doctrine so prompts stay short
- Define required behavior via tests
- Encode naming conventions, module boundaries, and design intent

---

## Prompt Doctrine

A good Coding Agent prompt contains exactly five things:

```
# Task
[One paragraph. What change is wanted.]

## Scope
* files the agent may change
* files the agent must not touch

## Constraints
* preserve behavior
* keep diff small and reviewable
* no new dependencies
* follow AGENTS.md
* follow interruptions introduced by user
  + comments or deleted code become design imperatives or contracts
  - do not fix broken code by reverse breaking changes introduced by user
  + do fix broken code by TDD (design by contract):
    1) write test according to new suggested change to design or behavior
    2) redesign/refactor around new tests and breaking changes
    3) pass tests according to new design/behaviour
* update tests as needed


### Interruptions by User
  + user comments or deleted code become design imperatives or contracts
  - do not fix broken code by reverse breaking changes introduced by user
  + do fix broken code by TDD (design by contract):
    1) write test according to new suggested change to design or behavior
    2) redesign/refactor around new tests and breaking changes
    3) pass tests according to new design/behaviour

## Acceptance criteria
* boundary condition
* behavior condition
* tests pass

## Deliverables
1. concise summary of changes
2. test results
3. open concerns
```

That is usually enough.

---

## What to Avoid

| Anti-pattern | Why it hurts |
|---|---|
| Prescribing exact placement of every function | Removes agent judgment; increases prompt noise |
| Repeating the same rule in multiple sections | Fatigues the reader; dilutes priority |
| Mixing strategy with implementation detail | Blurs command altitude; confuses scope |
| Multiple fallback options in one prompt | Makes the agent indecisive |
| Speculative architecture in a refactor prompt | Invites scope creep |

**Less is more.** A short, scannable prompt is faster for humans and more decisive for agents.

---

## Working Cycle

```
1. LLM writes short mission
2. Coding Agent implements and tests
3. LLM reviews the diff
4. LLM writes the next short mission
```

One mission per step. No all-in-one redesign prompts unless a full redesign is the intent.

When a task explicitly requires TDD sequencing, state the order:

1. unit or in-process integration tests first
2. implementation to pass those tests
3. system tests second
4. task is not complete until all required test layers pass

If a repo defines `make all` as the completion command, `make all` must run the
full required suite, not just build artifacts.

Prefer repo-owned local fixtures for normal tests over external mutable data
files. Use external files only for explicit integration coverage.

Mark aggregate make targets like `all`, `tests`, and `clean` as `.PHONY`.

Implement the specified external contract directly.
Do not preserve wrong internal formats and compensate for them in downstream code or tests.

## Reporting

When a task or user prompt changes one or more files, conclude the report with a suggested
concise git commit message.

---

## Mission Types

For most work, a prompt should name one of these:

- split responsibility
- tighten boundary
- add tests
- reduce coupling
- rename for clarity
- extract pure function
- preserve behavior while simplifying structure

Not *"redesign architecture"* unless you truly want a redesign.

---

## Altitude Check

Before sending a prompt, ask:

> *"Am I telling the agent what must be true, or am I telling it how to implement?"*

If you are specifying implementation, pull back up to architecture.  
The agent should own the how.
