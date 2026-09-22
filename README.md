# Agent TDD Loop

This directory contains a model-routed development loop:

```text
PLAN -> TEST -> IMPLEMENT -> SIMPLIFY -> REVIEW
                     ^          |          |
                     +----------+----------+
```

Each stage runs in a fresh Claude Code or Cursor Agent process. State and
handoffs are stored under `.loop-state/`, so stages do not share conversation
history. Test results, simplification feedback, and review findings determine
the next state mechanically.

Run commands from the repository or spawned worktree root:

```bash
.loop/bin/loop <command>
```

## Typical workflow

### Start a loop

```bash
.loop/bin/loop init --goal "Add rate limiting to the search API"
.loop/bin/loop run
```

The first `run` executes the planner and pauses for approval. Review:

```bash
$EDITOR .loop-state/subtasks.json
```

Then either approve it:

```bash
.loop/bin/loop approve-plan
.loop/bin/loop run
```

Or reject it with requested changes:

```bash
.loop/bin/loop revise-plan \
  --feedback "Combine ST-2 and ST-3. Do not add a database migration."
.loop/bin/loop run
```

For longer feedback:

```bash
.loop/bin/loop revise-plan --feedback-file /tmp/plan-feedback.md
.loop/bin/loop run
```

The rejected plan is retained in
`.loop-state/artifacts/plan-revision-N.json`. The planner receives the feedback
and writes a replacement plan, which pauses for approval again.

### Monitor and recover

```bash
.loop/bin/loop status
```

`runner=live` means a `loop run` process is active. `runner=not running` means
the state exists but no agent is currently working.

If a runtime or authentication failure blocks the loop, fix the underlying
problem and run:

```bash
.loop/bin/loop resume --reason "Cursor authentication fixed"
.loop/bin/loop run
```

`resume` changes state from `blocked` to `running`; it does not launch a
process. Only `loop run` starts work.

### Close a loop

```bash
.loop/bin/loop close --reason "Merged in PR #123"
```

This archives the active state, plan, verdicts, and artifacts under
`.loop-state/archive/<loop-id>/`. It does not remove a git worktree or branch.

## Commands

### `init`

Create loop state in the current checkout.

```bash
.loop/bin/loop init --goal "Required goal"
```

Options:

- `--goal TEXT` — required description of the desired change.
- `--jira KEY` — Jira key for the planner to fetch, such as `PSI-1234`.
- `--context TEXT` — additional planner context.
- `--context-file PATH` — append a file's contents to planner context.
- `--force` — discard an existing active state instead of requiring `close`.

Examples:

```bash
.loop/bin/loop init \
  --goal "Implement the ticket" \
  --jira PSI-1234 \
  --context-file ./architecture-notes.md
```

Prefer `close` over `init --force` when the prior run should be retained.

### `run`

Drive the current state headlessly until one of these conditions occurs:

- the plan needs approval;
- all subtasks complete;
- a stage blocks;
- a configured retry or iteration limit is reached.

```bash
.loop/bin/loop run
```

Run it again after approving or revising a plan, or after `resume`.

### `status`

Show the current stage, subtask statuses, feedback, and process liveness.

```bash
.loop/bin/loop status
```

Important states:

- `running` — eligible to run; check `runner=live` to know if it is active.
- `awaiting_plan_approval` — use `approve-plan` or `revise-plan`.
- `blocked` — resolve the reported issue, then use `resume` and `run`.
- `done` — all planned subtasks were approved.

### `approve-plan`

Accept `.loop-state/subtasks.json` and start its first subtask.

```bash
.loop/bin/loop approve-plan
.loop/bin/loop run
```

This command is valid only when status is `awaiting_plan_approval`.

### `revise-plan`

Reject the current plan and return specific feedback to a fresh planner run.

```bash
.loop/bin/loop revise-plan --feedback "Split the API and queue work."
```

Options:

- `--feedback TEXT` — inline revision instructions.
- `--feedback-file PATH` — revision instructions from a file.

At least one option is required. Both may be supplied. Afterward, run
`.loop/bin/loop run`; the revised plan will pause for approval again.

### `context`

Add context to an initialized loop before planning completes.

```bash
.loop/bin/loop context --jira PSI-1234
.loop/bin/loop context --context "Do not add a migration."
.loop/bin/loop context --context-file ./api-contract.md
```

This is intended for a loop still in `PLAN` before it has produced subtasks.
Use `revise-plan` once a plan is awaiting approval.

### `resume`

Clear a `blocked` state after its cause has been fixed.

```bash
.loop/bin/loop resume --reason "Claude permissions updated"
.loop/bin/loop run
```

Options:

- `--reason TEXT` — optional audit note stored in loop history.

### `close`

Archive the current loop and clear active state.

```bash
.loop/bin/loop close --reason "Completed"
```

Options:

- `--reason TEXT` — optional close reason stored in loop history.

The command refuses to close while a live runner is detected.

### `spawn`

Create an isolated branch and git worktree for a concurrent loop.

```bash
.loop/bin/loop spawn report-agent \
  --goal "Scaffold the report agent" \
  --branch PSI-15917-scaffold-report-agent \
  --runtime claude \
  --jira PSI-15917
```

Arguments and options:

- `name` — required short loop name; used by default directory and branch.
- `--goal TEXT` — required goal.
- `--branch NAME` — branch name; defaults to `loop/<name>`.
- `--base REF` — source ref; defaults to `HEAD`.
- `--dir PATH` — worktree path; defaults to `../<repo>-<name>`.
- `--runtime claude|cursor` — runtime for this worktree only.
- `--jira KEY` — Jira context for the planner.
- `--context TEXT` — additional planner context.
- `--context-file PATH` — planner context loaded from a file.

Start the spawned loop using the paths printed by the command:

```bash
cd ../phoenix-report-agent
.loop/bin/loop run
```

Each concurrent loop needs its own worktree. Branches alone do not isolate
working files or tests.

When finished:

```bash
cd ../phoenix-report-agent
.loop/bin/loop close --reason "Merged"

cd ../phoenix
git worktree remove ../phoenix-report-agent
git branch -d PSI-15917-scaffold-report-agent
```

Do not normally delete a worktree directory with `rm`; that leaves stale git
worktree metadata. If it was deleted manually, run `git worktree prune`.

### `next` and `advance`

These support an interactive/manual loop instead of `run`.

```bash
.loop/bin/loop next
```

`next` prints the current action, model, effort, subtask, output path, and
prompt path. Complete that stage in an interactive agent, then:

```bash
.loop/bin/loop advance
```

`advance` applies the same test and verdict gates used by headless mode.

## Configuration

Edit `.loop/config.json`.

### Runtime and per-stage models

```json
{
  "runtime": "claude",
  "stages": {
    "planner": {
      "models": {
        "claude": "sonnet",
        "cursor": "kimi-k3-high"
      },
      "effort": {
        "claude": "high"
      }
    }
  }
}
```

- `runtime` selects `claude` or `cursor` for the loop.
- `models` selects a model per stage and runtime.
- Claude `effort` accepts `low`, `medium`, `high`, `xhigh`, or `max`.
- Cursor thinking level is generally encoded in its model slug.

The five configurable stage keys are `planner`, `tester`, `implementor`,
`simplifier`, and `reviewer`.

Spawned worktrees receive a copy of `.loop/` at spawn time. Later changes to
the main checkout's config or prompts do not update existing worktrees.

### Gates and limits

Useful settings:

- `test_command_default` — fallback test command; testers should normally emit
  a narrower command.
- `require_plan_approval` — pause after planning when true.
- `notify_on_finish` — macOS notification on done, blocked, or plan ready.
- `commit_per_subtask` — automatically commit each approved subtask when true.
- `max_test_attempts` — allowed tester retries when tests pass prematurely.
- `max_implement_attempts` — implementation retries while tests remain red.
- `max_simplify_cycles` — simplifier-to-implementor kickbacks.
- `max_review_cycles` — reviewer-to-implementor kickbacks.
- `max_iterations` — total stage iterations for one invocation.
- `stage_timeout_seconds` — timeout for each headless agent process.

## State and artifacts

Active runtime files live under `.loop-state/`:

```text
.loop-state/
├── state.json
├── subtasks.json
├── run.pid
├── verdicts/
├── artifacts/
└── archive/
```

`.loop-state/` should remain gitignored. `.loop/` contains the reusable driver,
configuration, and stage prompts.

## Troubleshooting

### Status says running, but nothing is happening

If `status` reports `runner=not running`, start it:

```bash
.loop/bin/loop run
```

### `approve-plan` says no plan is awaiting approval

The planner has not produced a plan yet, or the loop is in another state.
Check:

```bash
.loop/bin/loop status
```

If it shows `stage=PLAN`, `status=running`, and `runner=not running`, run
`.loop/bin/loop run` first.

### Headless agent authentication or permission failure

Authenticate the selected CLI directly, verify it can run, then:

```bash
.loop/bin/loop resume --reason "Authentication fixed"
.loop/bin/loop run
```

### Concurrent tests interfere

Worktrees isolate files, not shared Docker services or databases. Prefer
narrow unit-test commands, stagger database-backed gates, or run separate
Compose projects for true test-environment isolation.
