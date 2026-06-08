# `/goal` — a coordinated multi-agent loop for Claude Code

Turn an objective into a coordinated loop instead of one big prompt. You stop answering and
start orchestrating: decompose → assign roles → execute in order → review against the goal →
**gate** before shipping → save what worked → report only what matters. You step in at
decision points, not every step.

## What's in here

```
.claude/
├── commands/
│   └── goal.md          # the orchestrator — runs on the main thread, holds the loop + gate + memory
├── agents/
│   ├── planner.md       # decompose objective → task graph (roles, deps, acceptance checks)   [read-only]
│   ├── researcher.md    # gather context for an executor                                      [read-only]
│   ├── executor.md      # implement one task to its acceptance check                          [read/write]
│   ├── reviewer.md      # judge alignment with the ORIGINAL objective                         [read-only]
│   └── gatekeeper.md    # run real build/test/lint — the loop's source of truth               [read + bash]
└── goal-workflow.md     # brief to turn the loop into a saved DYNAMIC WORKFLOW (loop+gate in code)
```

Two ways to run the same loop: as the in-context `/goal` command (cheap, single objective), or
as a **dynamic workflow** where the loop and gate become real code that scales to dozens of
parallel tasks (see "Run it as a dynamic workflow" below).

## Install

Copy the `.claude/` tree into the root of your repo (project scope, committed so the team
shares it), or into `~/.claude/` (personal, available in every project). Then:

```
/goal "migrate the TestMatrix coupling out of the orchestrator into its own boundary"
```

Verify the pieces loaded: run `/agents` to see the five subagents in the Library tab, and type
`/` to confirm `goal` appears.

## How it runs

1. **Prime** — reads `.claude/goal-memory.md` + `CLAUDE.md` so it doesn't re-learn known gotchas.
2. **Decompose** — `planner` returns a task graph with a concrete acceptance check per task.
3. **Checkpoint** — shows you the plan and *waits*. The cheapest correction you'll ever make.
4. **Execute** — dispatches `executor`s in dependency order, in parallel where tasks are independent; `researcher` feeds context where needed.
5. **Review** — `reviewer` checks the work against the *original* objective (intent, not just "runs").
6. **Gate** — `gatekeeper` runs the repo's real verification. `FAIL` → back to the executor with the *specific* reason. Capped at 3 cycles, then it escalates to you.
7. **Save** — appends what worked / what failed to `.claude/goal-memory.md`.
8. **Report** — done & verified / needs your eyes / left out. No play-by-play.

## The design, in one paragraph

The thing that makes this more than a fancy `while` loop is the **separation of review and
gate**. Review is judgment (did we build the right thing?); the gate is deterministic (does it
actually build, typecheck, lint, pass tests?). The loop's source of truth is the *objective*
gate — never the model grading its own homework. If you can't write an objective gate for a
piece of work, the loop says so out loud instead of pretending it passed. That, plus the
dependency-ordered dispatch and the persistent memory, is what lets you step out of the
bottleneck without flying blind.

## Walkthrough

A real run on a coupled monorepo. You touch the keyboard twice: to answer one fork in the
plan, and to read the report. Everything between runs without you.

You type one line:

```
/goal "stop TestMatrix from reaching into Orchestrator internals — put an
enforced import boundary between them with import-linter, and fix the one
real violation it surfaces"
```

**Phases 0–1 — prime + decompose (no input from you).** It reads `goal-memory.md` and
`CLAUDE.md`, dispatches `planner`, then shows the plan and stops:

```
PLAN — 4 tasks

  t1  research  map every TestMatrix → Orchestrator import; flag real internal
                reaches vs. public-API use      · depends: none
                accept: list of importers, each tagged real/ok
  t2  execute   add import-linter "forbidden" contract: testmatrix may not
                import orchestrator.internals    · depends: t1
                accept: `lint-imports` reports exactly the real violation(s)
  t3  execute   refactor the real violation behind the public interface
                depends: t2
                accept: `lint-imports` passes AND `pytest tests/test_matrix` green
  t4  review    confirm the boundary matches intent    · depends: t3

OPEN QUESTION
  One-way (TestMatrix can't import Orchestrator) or mutual? They imply
  different refactors.   → approve / edit / cancel?
```

**Decision point #1 — the only place you think hard.** You answer the fork:

```
one-way for now. mutual is a bigger refactor — log it as a follow-up, don't do it.
```

**Phases 3–5 — execute, review, gate (no input from you).** Note the gate failing and looping
back with a *specific* reason, not a restart:

```
researcher → only 1 of 3 imports is a real reach (runner.py → internals.scheduler);
             other 2 use the public API — leave them
executor   → added importlinter contract; refactored runner.py to public schedule()
gatekeeper → FAIL
  ran: lint-imports ✓ · pytest tests/test_matrix ✗ (1 failed)
  test_runner.py:42  ImportError: cannot import '_Scheduler' from internals
  (the test was reaching past the boundary too)
  ↻ cycle 1/3 — sending failure back to executor

executor   → updated the test to use public schedule() + a fixture
gatekeeper → PASS   (lint-imports ✓ · pytest tests/test_matrix ✓ 12 passed)
```

The gate caught what a "looks right" review never would: the *test* also violated the
boundary. Because the failure was specific, the executor fixed that one thing instead of
re-planning.

**Phases 4, 6, 7 — review, save, report:**

```
REPORT
Done & verified
  • contract enforces testmatrix ✗→ orchestrator.internals
  • the one real violation now goes through the public API
  • proved by: lint-imports + pytest tests/test_matrix (green)
Needs your eyes
  • reviewer: the other 2 imports are "ok" only by convention — the contract
    forbids .internals, not every private module. Enforced for this case, not airtight.
Left out / follow-ups
  • mutual boundary (you scoped it out) — logged in goal-memory.md
```

Two moments of your attention across a task that touched config, source, and tests and
survived a failed gate.

**How to drive it well**
- **Put the gate in the objective.** "fix the *one real* violation it surfaces" gave the
  planner a runnable acceptance check. "clean up the coupling" has no gate — the gatekeeper
  would mark most of it *ungated* and hand the judgment back to you. Sharper criteria → more
  of the work the loop can own.
- **Trust the checkpoint, distrust the silence.** If the plan is wrong, every downstream cycle
  is wasted compute. The 30-second plan read is the highest-leverage thing you do.
- **Know when not to use it.** Open-ended, taste-driven work ("should we split the monorepo at
  all?") has no gate — that's a conversation, not a `/goal`. The Fast path handles true
  one-liners. The loop is for the middle: real objective, real dependencies, real gate.

## Run it as a dynamic workflow

`/goal` runs the loop in Claude's context — Claude decides each step, and the gate is enforced
by *instruction*. That's perfect for one bounded objective. For big work (a 500-file migration,
a service-wide audit, a bug sweep), there's a stronger option: a **dynamic workflow**, where
Claude writes a JavaScript orchestration script and a background runtime executes it. The loop,
the cycle counter, and the gate live in the script's control flow — so the gate is enforced by
*code*, not instruction. It can't be skipped, and the run can fan out to many parallel agents.

You don't hand-write that script; Claude writes it from a brief. `.claude/goal-workflow.md` is
that brief — it points Claude at the same five agents and specifies the orchestration (notably,
the gate as a real `loop until PASS` condition).

```
# 1. have Claude author the workflow from the brief
ultracode: write a reusable dynamic workflow from the brief in @.claude/goal-workflow.md

# 2. once a run does what you want: /workflows → select it → press s → save as "goalflow"

# 3. thereafter, run it with the objective as args:
/goalflow "port the CoverageRegistry module from Pydantic v1 to v2 across the repo"
```

What changes versus the in-context version:

- **The gate becomes structural.** The script loops `executor → gatekeeper` and only breaks on a
  real `PASS`. The "reviewed 35 of 50 and declared done" failure mode is gone — a `for` loop
  doesn't get tired.
- **The plan checkpoint can't pause mid-run.** Workflows take no mid-run input. So the brief
  defaults to *plan-and-stop*: the first invocation returns the plan, you review it, then
  re-invoke with `autorun: true` to execute end-to-end. (Planning as its own stage is the
  documented pattern for between-stage sign-off.)
- **Cost goes up, a lot.** Many parallel agents burn far more tokens than a conversation. Gauge
  it on one directory before the whole repo; the runtime caps a run at 16 concurrent / 1000
  total agents, and `/workflows` shows live token usage so you can stop early.

Requires Claude Code v2.1.154+ with dynamic workflows enabled (research preview; on Pro, toggle
it in `/config`). Available on the API and Bedrock too — relevant if you ever drive this from
the Agent SDK rather than the CLI.

Rule of thumb: reach for `/goal` by default; reach for `/goalflow` when the task is too big for
one context to coordinate and you want the orchestration itself codified and rerunnable.

## Tuning

- **Models** — orchestrator + planner run on `opus` (coordination/judgment), executors and
  judges on `sonnet`, research on `haiku`. Swap to `inherit` to follow your main session, or
  set `CLAUDE_CODE_SUBAGENT_MODEL` to change the default for all of them.
- **Loop cap** — edit "Max 3 cycles" in `goal.md`. Higher for fuzzy work, lower to fail fast.
- **Checkpoints** — `goal.md` pauses after the plan and on destructive actions. Add/remove
  pause points there to taste.
- **Tools** — each agent's `tools:` line is a deliberate allowlist (reviewers and planner can't
  write; gatekeeper can run bash). Tighten or widen per your trust level.
- **Worktree isolation** — for parallel executors that touch the filesystem, add
  `isolation: worktree` to `executor.md` so concurrent tasks don't stomp each other.

## Porting it elsewhere

The shape is platform-agnostic — decompose, assign, coordinate, gate, report. Only the
plumbing changes:

- **Codex** — keep `goal.md`'s body as a custom prompt/command. Codex doesn't have Claude
  Code's `.claude/agents/` filesystem dispatch, so collapse the five agents into named
  *phases* the single orchestrator runs in sequence (or use Codex's own task/subtask
  mechanism). The gate still has to shell out to your real build/test commands — that part
  is identical.
- **Your own agent stack (Agent SDK / FORGE-style)** — map each `.md` to a worker with the
  same role contract: planner emits the task DAG, executors are dispatched per node, the
  gatekeeper is a deterministic verifier wired to CI commands, and the orchestrator owns the
  loop + checkpoints. The agent files here double as ready-made system prompts for those workers.

The one rule that travels everywhere: **the gate must be objective and the feedback cheap.**
Everything else is tuning.
