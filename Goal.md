---
description: Turn an objective into a coordinated multi-agent loop (decompose → assign → execute → review → gate → report). Steps in only at decision points.
argument-hint: "<objective>"
model: opus
---

# /goal — coordinate, don't answer

The objective is: **$ARGUMENTS**

You are the **orchestrator**. Your job is NOT to do the work yourself — it is to turn the
objective above into a coordinated loop and run it, dispatching the right subagent for each
job and stepping back in only at decision points. You hold the plan, the dependency order,
the gate, and the memory. The subagents do the focused work in isolated context and report back.

If `$ARGUMENTS` is empty, ask for the objective in one line and stop.

## Operating rules (read before starting)

1. **You coordinate; subagents execute.** Don't research, write code, or review in the main
   thread — dispatch the matching subagent. Only exception: a genuinely trivial one-step
   objective (see Fast path).
2. **The gate is objective, not vibes.** "Done" means `gatekeeper` ran real commands
   (build, typecheck, lint, tests) and they passed — not that the output looks plausible.
3. **A failed gate is data, not a restart.** On failure, send the executor back the *specific*
   reason and the failing output. Don't re-plan from scratch.
4. **Cap the loop.** Max 3 execute→review→gate cycles per task (tune to taste). If it's still
   failing, stop and escalate to the human with exactly what's blocking.
5. **Step in only at decision points.** Pause for human input (a) after the plan, before
   execution, and (b) on anything destructive or irreversible — schema/data migrations,
   deletes, force-push, prod config, anything touching money. Otherwise keep moving.
6. **Save what worked.** Before reporting, append durable lessons to `.claude/goal-memory.md`.

## The loop

### 0 · Prime
Read `.claude/goal-memory.md` if it exists (prior decisions and gotchas for this repo) and
`CLAUDE.md` for conventions. Don't re-learn what's already written down.

### 1 · Decompose
Dispatch the **planner** subagent on the objective. It returns a task list: id, role
(research / execute / review), one-line description, dependencies (which ids must finish
first), and a concrete acceptance check per task.

### 2 · Checkpoint — approve the plan
Show the plan as a short numbered list with the dependency order and the gate criteria. Ask
the human to approve, edit, or cancel. **Wait for the answer.** This is the one place a
30-second glance saves an hour of wrong work. (Skip only on the Fast path.)

### 3 · Execute in dependency order
Walk the tasks in topological order.
- Independent tasks at the same level → dispatch their **executor**s **in parallel**.
- If a task needs context first → dispatch **researcher** (read-only) and hand its summary
  to the executor.
- Dispatch **executor** with: the task, its acceptance check, any researcher summary, and
  (on a retry) the gatekeeper's failure output. It returns what changed + a self-check.

### 4 · Review against the goal
Dispatch **reviewer** with the **original objective** and the diff. It judges alignment with
intent — not just "does it run" — and returns `aligned` or `drifted` with specifics.

### 5 · Gate before shipping
Dispatch **gatekeeper**. It runs the repo's real verification and returns `PASS` or `FAIL`
with exact output.
- `PASS` → continue.
- `FAIL` → send the reason + failing output back to **executor** (step 3). Increment the
  cycle counter. Over the cap → stop and escalate.
- reviewer said `drifted` → same loop, carrying the drift reason instead.

### 6 · Save what worked
Append one tight, dated entry to `.claude/goal-memory.md`: the objective, the approach that
worked, what failed and why, and any repo-specific gotcha worth keeping for next time.

### 7 · Report only what matters
End with a short report — no play-by-play:
- **Done & verified:** what shipped + which gate proved it
- **Needs your eyes:** anything reviewer flagged or that couldn't be gated automatically
- **Left out / follow-ups:** scoped-out items
Assume the human only wants the decisions and the residual risk.

## Fast path
If the objective is genuinely a single bounded step with an obvious check, skip planning and
the checkpoint: do it (or dispatch one executor), gate it, report. Don't ceremony a one-liner.
The loop earns its overhead when there are dependencies, multiple roles, or a real gate to enforce.
