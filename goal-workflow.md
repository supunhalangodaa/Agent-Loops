# Authoring brief — `/goal` as a dynamic workflow

This is **not** a command or a subagent. It's a brief you hand to Claude so it *writes* a
dynamic-workflow script that codifies the same loop as `/goal`, reusing the five role prompts
in `.claude/agents/`. The difference from `/goal`: the loop, the cycle counter, and the gate
live in the workflow **script** (real control flow), not in Claude's context — so the gate
can't be skipped or "declared done" early, and dozens of tasks can run in parallel.

## How to use this brief

1. In Claude Code (v2.1.154+, workflows enabled), run:

   ```
   ultracode: write a reusable dynamic workflow from the brief in @.claude/goal-workflow.md
   ```

   (`ultracode` is the opt-in keyword; "run a workflow" / "use a workflow" works too.)
2. Claude writes the script and shows the planned phases for approval. Approve it.
3. When a run does what you want, open `/workflows`, select it, press `s`, and save to
   `.claude/workflows/` as `goalflow` (project) or `~/.claude/workflows/` (personal).
4. From then on: `/goalflow "<objective>"` — the objective arrives as the `args` global.

## What the workflow script must do

Reuse the existing role prompts as the system prompts for the agents it spawns — read each from
`.claude/agents/{planner,researcher,executor,reviewer,gatekeeper}.md`. Do **not** reinvent the
roles; the workflow only changes who holds the loop.

**Input contract (`args`):** `args` is either the objective string, or an object
`{ objective, maxCycles = 3, autorun = false, model }`. If `args` is undefined, stop and ask
for an objective.

**Phase 1 — Plan.** Spawn one `planner` agent on the objective. Parse its output into a
structured array of tasks in a script variable: `{ id, role, what, depends_on[], accept }`,
plus `open_questions`.

**Plan checkpoint (workflows can't take mid-run input).**
- If `autorun` is false (default): emit the plan + open_questions as the run's result and
  **stop here**. The human reads it and re-invokes with `autorun: true` to execute. This is the
  "sign-off between stages" pattern — planning is its own workflow stage.
- If `autorun` is true: continue straight through. (The pre-run approval card is the human gate.)

**Phase 2 — Execute, in dependency order, in code.** Topologically sort tasks by `depends_on`.
Run independent tasks at the same level **concurrently** (respect the runtime's 16-concurrent
cap; never exceed it). For an `execute` task that needs context, spawn a `researcher` first and
pass its summary into the executor.

**Phase 3 — The gate, as a real loop (this is the whole point).** For each execute task, the
script runs this loop in code, not as agent instructions:

```
cycle = 0
loop:
  result   = spawn executor(task, accept, researcherSummary, lastFailure)
  verdict  = spawn gatekeeper(task)        // runs the repo's real build/test/lint/typecheck
  if verdict == PASS: break
  cycle += 1
  if cycle >= maxCycles: mark task ESCALATED; break
  lastFailure = verdict.failingOutput      // fed into the next executor spawn
```

The exit condition is the gatekeeper's verdict — a deterministic `PASS`, never an agent's
opinion. A task is only "done" when its gate loop broke on PASS.

**Phase 4 — Review against the goal.** After execution, spawn `reviewer` with the **original
objective** and the diff. Record `aligned | drifted` + specifics in a script variable. A
`drifted` verdict re-enters the gate loop for the affected task with the drift reason.

**Phase 5 — Save what worked.** The script itself can't touch the filesystem — so spawn a small
agent whose only job is to append one dated entry to `.claude/goal-memory.md`: objective,
approach that worked, what failed and why, repo-specific gotcha.

**Phase 6 — Report only what matters.** Synthesize the final result from script variables:
`{ doneVerified, needsEyes (reviewer flags + ESCALATED tasks + ungated parts), leftOut }`.
That object is the only thing that lands back in the session.

## Cost and safety rails (state these in the script)

- Route the `researcher` stage to a cheaper model where the runtime allows; keep `planner` and
  `reviewer` on the strong model. Every agent uses the session model unless the script routes
  the stage otherwise.
- Honor the runtime caps: ≤16 concurrent agents, ≤1000 agents total per run.
- The spawned agents inherit your tool allowlist and run in `acceptEdits`. Before a long run,
  add the build/test/lint commands the `gatekeeper` needs to your allowlist so it doesn't
  prompt mid-run.
- Gauge spend on a slice first (one directory / one task) before pointing it at the whole repo.
