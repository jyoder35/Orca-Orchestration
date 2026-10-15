---
name: orca-dag
description: "Plan software work as an Orca orchestration task DAG. Use when the user wants to break a feature or project into an executable graph of tasks with dependencies and approval gates in Orca, then visualize and fire it. Teaches the exact `orca orchestration` CLI commands to build the DAG, the spec-writing conventions, and how to open the orca-dag viewer so the user can pick a harness per node and fire tasks."
---

# orca-dag

Refine a software requirement through conversation and land it as an **Orca orchestration task DAG** (directed acyclic graph). You own **building and reshaping the graph**; **execution is triggered by the user in the `orca-dag` viewer** — the viewer advances the whole graph automatically, in parallel, along its dependencies. It is not fired node by node.

Every planning result **must be written into Orca's orchestration state** (by running `orca orchestration` commands), not left as chat text — the viewer polls Orca live and draws whatever is there.

## Tools you may use
- **Bash** to run the `orca` CLI and read/write orchestration state.
- **Read / Write / Edit** to write planning docs in the working directory (`docs/PRD.md`, `docs/TECH_SPEC.md`).
- Do not run destructive commands unrelated to this task (`rm`, `git push`, deleting files, …).

## Preflight
```bash
orca status --json      # runtime.state should be "ready"; if not, ask the user to run `orca open` first
                        # runtime.appVersion must be >= 1.4.160 (the Run/Dispatch contract)

orca skills get orchestration --json        # REQUIRED. See below.
```

**Always load `orca skills get orchestration` before building a DAG.** This skill is a
graph-building helper, not the execution authority. The orchestration guide is served by
the `orca` binary itself (version-matched, cannot drift) and is the only place that
documents what to do when execution goes wrong: `worker-start` receipts and their
`failedStage` / `residualResources`, the layered liveness model, and the recovery rules.
`orca skills get orchestration --references` lists seven reference docs; load
`references/recovery-and-cleanup.md` the moment any dispatch misbehaves.

The stub at `~/.claude/skills/orchestration/SKILL.md` is only a pointer. Reading it is not
a substitute for running the command.

## Workflow (three phases, all in conversation)
1. **Requirement clarification (PRD)**: align on the goal, MVP scope, and explicit non-goals with short questions — one key question at a time. If MVP is enough, plan only P0; don't over-design. Once agreed, write `docs/PRD.md`.
2. **Technical design (TECH_SPEC)**: stack, data model (down to fields), module interfaces (pseudocode). Write `docs/TECH_SPEC.md`.
3. **Decompose into a task DAG**: split the design into parallel/serial subtasks and create the tasks and dependencies with the commands below. **This step is the required output.**

**For review/revise work on code that already exists** (page critique, bug-fix rounds,
"here's what I don't like"), skip PRD/TECH_SPEC entirely — they are for greenfield builds.
Phase 1 becomes "agree on the critique", phase 2 becomes "write the critique to a dated
Markdown file with an explicit scope boundary", and phase 3 is the same. See the
back-and-forth section below.

## Writing the DAG into Orca (the core)

### Step 0: create a Run first (required on Orca ≥ 1.4.160)

Tasks are **no longer global**: every task belongs to a Run, and `task-create` / `task-list` fail with `run_required` when no Run is bound. So **start every plan by opening a fresh Run** to hold this DAG:

```bash
orca orchestration run-create --objective "<one sentence on what this plan does>" --json
```

The response carries `result.run.id` (shaped like `run_xxxxxxxx`). `run-create` binds the **current terminal** as that Run's coordinator, so later `task-create` calls don't need `--run`.

**If you are not running inside an Orca terminal** (e.g. Claude Desktop or Cursor driving the
DAG), `run-create` fails with `no_active_sender_terminal`. Create a blank Orca terminal to act
as the coordinator and send from it:
```bash
orca terminal create --worktree active --title "DAG coordinator" --json   # note its handle
orca orchestration run-create --objective "..." --from <coordinator_handle> --json
```

> One Run holds one DAG. Orca itself doesn't enforce this (a Run is just a namespace), but the viewer renders "one Run = one graph". To replan, open a new Run — **never** use `reset`.

### Step 1: create tasks one by one

Dependencies between tasks are passed to `--deps` as a **JSON array** of the depended-on task ids.

A root task (no deps):
```bash
orca orchestration task-create \
  --task-title "Scaffold the project" \
  --spec "Initialize project structure and dependencies; produce package.json, src/, .gitignore. Acceptance: npm test runs." \
  --json
```
`result.task.id` in the response is the task id (shaped like `task_xxxxxxxx`).

Depending on an earlier task (put its id into `--deps`):
```bash
orca orchestration task-create \
  --task-title "Implement the data model" \
  --spec "Implement models and migrations per the TECH_SPEC data model. Acceptance: migrations run, unit tests included." \
  --deps '["task_c8df9d97"]' --json
```
Multiple deps: `--deps '["task_aaa","task_bbb"]'`.

**Remember every returned task id** — later tasks reference them in `--deps`, which is what builds the correct DAG. After creating a batch, self-check:
```bash
orca orchestration task-list --run <run_id> --json     # verify the dependencies are right
```
Deps may only point at tasks **within the same Run**.

### Spec-writing rules (important)
Every subtask's `--spec` must be **self-contained and independently executable**, so the future executing agent **never has to ask questions or enter plan mode**:
- **Inputs**: what it depends on, which files/interfaces to read.
- **Outputs**: which files to create/modify, what the deliverable is.
- **Acceptance criteria**: what "done" means (runnable tests, observable behavior).
- Use imperative sentences; avoid vague phrasing like "investigate" or "as appropriate".
- **Side effects outside the files**: if a change reaches anything external (a webhook
  payload, a CRM field mapping, legal or compliance text, a shared check script), name it in
  the brief and get the user's decision before firing. Don't leave it for a worker to
  discover. Adding a "last name" form field changed what a lead webhook sent; the CRM
  workflow had to map the new key, or the value would have been silently dropped.

## When human approval is needed

> ⚠️ Read the gate warning in the viewer section below first. Gates are swept to
> `"approved"` when the viewer binds as coordinator, so a gate created before execution
> does not survive. Use `task-update --status blocked` for a pause that holds.

At key points (e.g. "approve the design before execution"), create a decision gate. It blocks its task, and the viewer surfaces approve/reject buttons:
```bash
orca orchestration gate-create \
  --task <task_id> \
  --question "Approve the TECH_SPEC and move to execution?" \
  --options '["approved","rejected"]' --json
```

## After the DAG is built: open the viewer and let Orca execute
Once the DAG has taken shape, ask the user to open the viewer, and **tell them the Run id** (they pick it in the viewer's top bar):
```bash
npx orca-dag    # run in the current project directory; serves http://localhost:8787 and opens the browser
```
(If they already have it running — likely, since that command is also what installed this skill — they just need to reselect the Run.)

**The viewer only lives while its terminal does.** `Failed to fetch`, or localhost:8787 not
reachable, means the process is gone (usually because its tab was closed). Start it again
in a tab the user keeps open. **If you are driving the workers yourself** (see the defect
section below), the viewer is watch-only: tell the user **not** to click "▶ Run with Orca"
on that Run. Starting it runs `run-use` for the viewer, which fences you and revokes every
live worker's capability (see the 🛑 note below).
In the viewer the user will: **pick your new Run in the top bar** → **watch the DAG live** → **choose a harness per node** (claude / codex / opencode / grok …, or a default fallback) → click **"▶ Run with Orca"** → **the viewer uses `worker-start` to spin up workers in dependency-parallel, waits for `worker_done`, and advances the whole graph** → **resolve approval gates**.

In other words: **execution is the viewer's job, not yours.** Your responsibility ends at "the DAG is correct".

⚠️ **You will get fenced — this is normal.** When the user starts execution, the viewer binds the Run's coordinator to its own terminal. From then on **your** mutations against that Run (`task-create` / `gate-resolve` / `dispatch`) fail with `consumer_fenced`. To take it back:

```bash
orca orchestration run-use --id <run_id> --json     # re-bind yourself as coordinator
```

Reads are unaffected — `task-list --run <id>` / `gate-list --run <id>` always work.

🛑 **NEVER `run-use` while any dispatch is in flight.** Reclaiming the binding bumps the
Run's `consumer_generation`, which **revokes the capability of every live worker** — they
become zombies: `status: pending`, `capability_revoked_at` set, no heartbeat, holding their
tasks hostage forever. The viewer then fails to recover them (`cannot move to ready while
Dispatch ... is active`) and the whole graph deadlocks.

Before any `run-use`, prove nothing is running:
```bash
orca orchestration worker-list --json | grep -E 'dispatchStatus|workerState'
```
Only reclaim when every dispatch is `failed`/`completed`. If you created zombies anyway,
clear them with `worker-abandon --dispatch <ctx_id>` (fences without claiming the process
stopped, touches no files), then reset task statuses by hand. `worker-abandon` takes **no
`--from`** (only `--dispatch`), unlike most coordinator mutations, so don't pass one.

You and the user can keep adjusting the DAG in conversation, but **make graph edits while
the graph is idle**, then let the user hit Run.

⚠️ **Decision gates are unreliable as a pause.** Observed twice on 1.4.217: the viewer
resolves every `pending` gate to `"approved"` about one second after it binds as
coordinator — before any work runs. A gate created pre-execution is therefore spent on the
next Run click, and recreating it just gets it eaten again. To actually hold a node, set
its status directly (`task-update --id <task> --status blocked`) and flip it when ready; a
task status is not swept by the gate sweep. Better still, don't make a human pause
load-bearing — make the node read-only so running it early is harmless.

## 🔴 Known defect: dispatched prompts are pasted but never submitted

**Orca 1.4.217 injects the task preamble into the worker agent and does not press Enter.**
Diagnosed and proven on 2026-09-30. The orchestration stack is otherwise fine — Runs,
DAGs, deps, capabilities, `worker_done`, auto-settlement all work. Only the submit
keystroke is missing.

Two faces of the same bug:

| Launch mode | Symptom |
|---|---|
| **structured** (Orca's default for new agent tabs) | Silent infinite hang. `stage: dispatch_input`, `state: start_unknown`, `heartbeat: null`. No error, no output, task stuck forever. |
| **TUI** (`--terminal <handle>`) | Prompt visibly sits unsent in the input box. `stage: turn_start_unobserved`. |

**Do not diagnose this as your graph being wrong, as a harness problem, or as an auth
problem.** It reproduces identically on both `claude` and `codex`. Do not relaunch — the
orchestration guide is explicit that a non-zero `worker-start` must not be retried.

**Working path today — dispatch into a visible TUI and nudge it:**
```bash
# 1. visible agent tab in the current worktree (this IS the "messaging window")
#    --command is a raw command: Orca does NOT add its per-agent default args here,
#    so pass the permission flag yourself or the worker stalls on approvals / read-only.
orca terminal create --worktree active --title "Codex worker"   --command "codex --dangerously-bypass-approvals-and-sandbox" --focus --json
#    Claude worker instead:  --command "claude --dangerously-skip-permissions"
orca terminal wait --terminal <term_handle> --for tui-idle --timeout-ms 120000 --json

# 2. dispatch into that exact terminal instead of letting Orca spawn a structured session
orca orchestration worker-start --task <task_id> --terminal <term_handle> \
  --worktree current --timeout-ms 120000 --json

# 3. THE NUDGE — press Enter, because Orca didn't
orca orchestration worker-start ... ; orca terminal send --terminal <term_handle> --enter --json

# 4. watch / read it work
orca terminal read --terminal <term_handle> --screen --json
```
(`--wait-submit` requires `--text`; for a bare Enter use `--enter` alone.)

**Read the screen before every Enter: a dialog is not the input box.** The first time an
agent opens in a folder it may show a trust prompt instead of its input box. Claude Code's
"Quick safety check: Is this a project you trust?" has **"No, exit" selected by default**, so
the Enter nudge quits Claude and the dispatch dies (seen 2026-10-01, abandoned dispatch,
bare PowerShell left behind). After `terminal wait --for tui-idle`, run
`terminal read --screen`; if it shows a trust or consent dialog, choose "Yes, I trust this
folder" in the tab (by hand, or send a Down arrow then Enter) and confirm the input box is
showing **before** `worker-start`. Pre-trusting the folder once by hand avoids it.

**One Enter is sometimes not enough.** If the input box still shows `[Pasted Content]` after
the nudge, the paste was collapsed and not sent; send `--enter` a second time. Confirm with
`terminal read --screen` that the agent actually started working.

With the nudge, the full supervised loop completes normally: the worker reads the TASK
block, does the work, calls `orchestration check`, sends `worker_done` with its dispatch
capability, and the task auto-settles to `completed`. Verified end to end in 24s.

Because of this defect, **the viewer's "▶ Run with Orca" cannot be trusted to start
work.** After the user fires it, verify a turn actually began rather than assuming:
```bash
orca orchestration worker-list --json | grep -E 'workerState|stage'
```
`start_unknown` at `dispatch_input` means nothing is running. Say so immediately instead
of waiting.

## Back-and-forth between Claude and Codex (review/revise rounds)

**Roles are not fixed.** The examples below assume Claude orchestrates and Codex implements.
Either agent can orchestrate: read "Claude" as *the orchestrator in the chat* and "codex" as
*the worker harness*. If Codex orchestrates, the worker is usually `claude` (launch it with
`--dangerously-skip-permissions`), and Codex does its own review rounds in its chat.

For "have Codex fix it and toss it back to me", the shape is **bounded rounds**, because a
DAG is acyclic and cannot express a loop. Do not try to encode a cycle.

**Round pattern** — one implement node and one review node per round, chained:
```
spec/feedback (Claude, in chat)
   └─> R1 implement (codex)  ─> R1 review (Claude)
          └─> R2 implement (codex)  ─> R2 review (Claude)
```
Two rounds is usually right. Create R2's nodes up front; if R1 passes review, mark R2
`completed` with a result saying "not needed, R1 accepted" rather than deleting them
(tasks cannot be deleted).

**Keep Codex's context across rounds by reusing its terminal.** This is what makes it feel
like a conversation instead of a cold restart:
```bash
orca orchestration worker-start --task <round2_task> --terminal <same_codex_term> --json
```
The orchestration guide calls this reuse-after-settlement; choose the terminal's next owner
*before* acking the `worker_done` delivery, and `worker-retain` it if the user wants the tab
to stay open.

**The orchestrator's review nodes belong in conversation, not in a worker.** Running
"Claude reviews this" as a background agent when Claude is already in the chat is pure
indirection, and it
costs the user the ability to react. Do the review in chat, then
`task-update --id <review_task> --status completed --result '{"summary":"<verdict>"}'`.

**Waiting on a worker.** `orchestration check --wait` blocks until a message arrives. Claude
can run it as a background task and be notified. An orchestrator whose shell commands time
out (Codex) should loop shorter waits instead of one long one, and treat each empty return as
a checkpoint, not a failure:
```bash
orca orchestration check --wait --types "worker_done,escalation,question" --timeout-ms 300000 --json
```
Between waits, `orca terminal read --terminal <worker_term> --screen --json` shows whether
the worker is still busy.

**Mid-task questions genuinely go both ways.** A dispatched worker blocks on the `ask`
command from its injected preamble; the coordinator answers with
`orchestration reply --id <message_id> --body "..."`. Tell Codex in the spec to use `ask`
rather than guessing when a constraint is ambiguous — that is the real back-and-forth
channel, and it is better than a round trip through a new node.

**Write the critique to a file, not into the spec.** For page/design feedback, put the
findings in a dated Markdown file and point every node at it as the contract. Specs cannot
be edited after creation, so a file is the only thing you can correct between rounds. Give
the file an explicit scope boundary naming what must NOT change — otherwise Codex will
"helpfully" fix the most interesting problem you deliberately excluded.

**Make the boundary machine-checkable.** Workers break prose "do not touch" lists. Where you
can, back the boundary with a small check script (IDs that must exist, verbatim legal text,
untouched tracking and payload keys) and run it yourself after every round. Revert a round
that fails; don't patch forward. A 32-check script kept a landing page's tracking, webhook
payload and legal copy intact through six rounds by two agents (2026-10-01).

**Before round 1, make sure the verification tools show the thing under review.** A
screenshot script stopped above the contact form both reviewers were asked to cut, so each
wrote its own capture workaround. Fix the tool once, up front. Then require every visual
claim in a round entry to cite a screenshot or a measurement. Codex caught a "fits on one
line" claim that the phone crop showed as two lines.

**Give rounds a fixed log format in the contract file.** Each round appends:
- **critique of the previous round**: adversarial (what got worse, what is still unclear),
  pushing back with reasons;
- **changes made**, each tied to a critique point;
- **deliberately not changed**, and why;
- the **check-script result**.

The last round ends with **open disagreements for the user**, so neither reviewer wins
quietly. This format is what made the reviewers catch each other: Codex rewrote an approved
headline to fit a product name in, and Claude restored it in the next round.

**Copy the user's words exactly, and record reversals.** The orchestrator added a period
when it copied the user's byline into the contract, and the reviewers spent two rounds
removing and restoring it. Quote requirements verbatim. When reviewers dispute the contract
itself, ask the user and fix the file; don't let rounds argue it out. When new feedback
reverses an earlier contract item, write the reversal into the file ("this reverses item 4;
the user's call") so a later round doesn't bring the old rule back.

## Ending a Run
- Mark leftover nodes `completed` with a result that says why (e.g. "not needed, R2
  accepted"), so the graph reads as finished.
- **Tell the user which tabs to close**, by title: the coordinator terminal, finished worker
  tabs, and any bare shell left by a dead dispatch. Close the ones you created once their
  dispatches have settled. Tabs pile up fast across Runs (one 2026-10-01 session had eight
  open, including a coordinator from an earlier Run), and a stale coordinator makes it
  unclear which Run is live.
- **Write a handoff note into the contract file**: the Run id and whether it finished, which
  terminals can be closed, launch flags that turned out to be needed, and any user feedback
  not yet applied. The next orchestrator may be a different agent (Codex or Cursor) with no
  memory of this chat, and the file is the only thing it will read.

## Boundaries and known constraints
- Focus on **planning + graph building**. Prefer letting the viewer execute, but while the
  submit defect above is unfixed you may have to drive `worker-start` + the Enter nudge
  yourself. Tell the user you are doing that and why; don't silently take over.
- **If `worker-start` exits non-zero, do not relaunch.** Read the receipt's `failedStage`
  and `residualResources`, then load `references/recovery-and-cleanup.md` from the
  orchestration guide. Retrying blind strands terminals and multiplies zombies.
- **Absence is not proof.** A timeout, an empty `check`, or a missing heartbeat is a
  checkpoint, not a failure. Only positive proof of exit (`exited` liveness, the worker's
  own observation, or a transcript whose final turn sent no `worker_done`) authorizes
  stop / abandon / retry.
- **Never pipe `orchestration check --wait` through `head`** (or anything that closes the pipe
  early). `head` exits after its line count and ends the wait before a worker reports. Run it
  unpiped, or capture to a file and read that.
- **`worker_done`'s `filesModified` under-reports.** Treat it as a hint, not the change set.
  Before reviewing or accepting a node, run `git diff --stat` in the worker's worktree.
- **Codex workers in Orca worktrees need `--dangerously-bypass-approvals-and-sandbox`.**
  Without it Codex can only read files there and PowerShell is blocked. It is now set as
  Orca's default Codex argument; check it is still there if a Codex node can't write. Orca
  runs Codex from its own home (`%APPDATA%\orca\codex-runtime-home\home`), not `CODEX_HOME`,
  so editing `AI\.codex\config.toml` does not affect it. The root cause of the read-only limit
  is unconfirmed. The Codex sandbox also can't write to `%TEMP%`: put any output paths in a
  spec inside the worktree.
- **Same-file work must be serial.** Parallel workers in one worktree clobber each other.
  Group fixes by region (JS / markup / CSS) and chain them, telling each node what the
  previous one already did. Fan out only across genuinely separate files or worktrees.
- **Created tasks cannot be edited**: `orca orchestration task-update` only changes `--status` / `--result`. **There is no interface to edit spec/title/deps**, and no command to delete a single task. Get the graph right on the first pass where possible.
- **To redraw the DAG, open a new Run — never `reset`.** `orca orchestration reset --tasks` has **no `--run` scope**: it clears the entire local orchestration database, deleting other Runs' tasks too. The correct move is `run-create` a new Run and rebuild; the old Run stays as history.
- **Decision gates are Run-scoped, but the flags differ by direction**: `gate-list` (read) takes `--run <id>`; `gate-resolve` (mutation) takes **no `--run`** — it locates the gate via `--from <handle>` (a coordinator terminal bound to the Run) plus the globally unique `--id`. That's why the viewer first binds a coordinator with `run-use`, then resolves via `--from`.
- **`orca orchestration run` / `run-stop` / `coordinator-start` / `coordinator-stop` are retired.** Calling them has no effect; they only return a "go read the orchestration skill" notice. Don't use them.
- Don't run destructive or off-task system commands.

## Communication style
- Follow the user's language. Concise and direct.
- After each batch of task creation/changes, summarize the DAG's current shape in one sentence (what runs in parallel, what is serial) — the user is watching it appear in the viewer.
