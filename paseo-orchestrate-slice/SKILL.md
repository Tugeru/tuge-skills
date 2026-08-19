---
name: paseo-orchestrate-slice
description: Coordinate a GitHub parent tracker through its vertical slices with Paseo — dispatch under a hard two-worktree ceiling, supervise workers, serialize merges, verify the assembled OpenSpec change.
argument-hint: "<parent-issue> [provider/model]"
disable-model-invocation: true
---

# Orchestrate Slices with Paseo

Coordinate all vertical slices under one GitHub parent issue until the complete change is merged and verified.

This session is the **orchestrator**. It never implements application code.

Workers implement exactly one slice each through:

```text
/paseo-ship-slice <issue> orchestrated
```

One slice = one GitHub child issue = one Paseo worktree workspace = one branch = one PR.

Load:

- [WORKSPACES.md](WORKSPACES.md) whenever creating, bootstrapping, recovering, or archiving a slice workspace.
- [SUPERVISE.md](SUPERVISE.md) after workers are launched and while monitoring, rebasing, reviewing, or merging them.

## Hard concurrency limit

At most **2 active slice worktree workspaces** may exist concurrently for this orchestration run. This is a hard ceiling, not a default.

Only slice implementation workspaces count; the orchestrator's own workspace does not.

A workspace still consumes its slot while its slice waits for CI, review, rebase, the merge gate, or a user decision — and while it is MERGE_READY or blocked, until deliberately archived.

Capacity examples:

- 0 active + 5 READY → create at most 2.
- 1 active + 4 READY → create at most 1.
- 2 active + 8 READY → create none.

READY slices beyond the ceiling stay queued. A slot frees only after its PR is confirmed merged and its workspace is archived.

## Authority

Use each system for one kind of truth:

- **OpenSpec** — requirements, design, capability specs, implementation tasks, final verification.
- **GitHub** — parent and child issues, blocker relationships, PR state, CI/reviews, merged/completed state.
- **Paseo** — live workspaces, live workers, worker lifecycle state, agent-parent relationships.

Do not substitute agent prose for durable GitHub state.

## 1. Orient

Resolve the parent issue, repository, repository root, integration branch, and worker provider/model.

```bash
ROOT="$(git rev-parse --show-toplevel)"
REPO="$(gh repo view --json nameWithOwner -q '.nameWithOwner')"
BASE="$(gh repo view --json defaultBranchRef -q '.defaultBranchRef.name')"
git fetch origin "$BASE"
BASE_SHA="$(git rev-parse "origin/$BASE")"
```

Resolve `$BASE` from the repo; do not assume it is `main`.

Require:

```bash
paseo daemon status
gh auth status
paseo provider ls
```

Worker provider/model: the invocation argument when given; otherwise the first ready provider in `paseo provider ls` with its default mode. Confirm model IDs with `paseo provider models <provider>` when needed.

This session must be a Paseo-managed agent (`test -n "$PASEO_AGENT_ID"`) so launched workers become your subagents; otherwise always pass `--workspace` explicitly and report the missing parentage.

Done when: parent, repo, root, `$BASE` and `$BASE_SHA`, and worker provider are known, and the daemon answers with readable provider and workspace lists.

## 2. Read the parent change

Read the parent issue and every child issue. Prefer GitHub-native structure (sub-issues, blocked-by relationships); fall back to parent task-list references or `## Blocked by` sections in child bodies only when necessary.

Read the OpenSpec change the tracker points at — at minimum `openspec/changes/<change>/proposal.md`, `design.md`, `specs/`, and `tasks.md`.

The ticket graph controls execution order; OpenSpec controls what the assembled change must accomplish.

Done when: every child and its blockers are recorded, and the OpenSpec change is read.

## 3. Reconstruct runtime state

Every invocation must support resuming an interrupted orchestration run. Inspect:

```bash
paseo workspace ls --json
paseo ls -a -g --json
```

And on GitHub: existing slice PRs, merged PRs, closed children, open children.

Worker identities are deterministic:

```text
workspace title: slice-<issue>
branch:          slice/<issue>-<slug>
worktree slug:   slice-<issue>
agent title:     slice-<issue>
agent label:     slice=<issue>
```

Match every existing workspace and agent back to its child issue before creating anything. Match agents by `--label slice=<issue>` or by their `cwd` (the worktree path from `paseo workspace ls`). If a workspace exists but its worker is gone, classify it ACTIVE and recover it via WORKSPACES.md — never create a duplicate workspace because the orchestrator restarted.

Done when: every existing workspace, agent, PR, and child issue is matched up and recorded.

## 4. Classify every child

Every child is exactly one of:

- **DONE** — PR merged and child issue completed.
- **BLOCKED** — open, with at least one declared blocker still open.
- **ACTIVE** — open, with an active slice workspace, live worker, implementation PR, or recoverable interrupted work.
- **READY** — open, all blockers completed, no active slice workspace.

The ordered READY set is the **frontier**.

No distributed claim comments, no worker leases: the orchestrator is the sole dispatcher for this parent.

Done when: every child classified exactly once, and the frontier is ordered by tracker position.

## 5. Determine available capacity

Count active slice worktree workspaces from `paseo workspace ls --json` — workspaces titled `slice-*` with worktree isolation whose slice PR is not yet merged.

```text
capacity = 2 - active_slice_workspaces
```

- capacity ≤ 0 → dispatch nothing.
- capacity == 1 → dispatch at most one READY slice.
- capacity == 2 → dispatch at most two READY slices.

Done when: capacity is computed from the current workspace list.

## 6. Select the next wave

READY means eligible, not parallel-safe. Before selecting two slices together, check their scopes, OpenSpec tasks, expected files/modules, and shared contracts against these high-contention surfaces:

- Prisma models and Supabase migrations
- shared database invariants
- auth/authorization boundaries
- shared domain types and central validation schemas
- package configuration
- heavily shared components or the same feature module
- wide refactors

If overlap is uncertain, serialize. Prefer the earliest READY slice in tracker order. Select no more slices than capacity permits.

Done when: the wave is selected with the overlap check recorded.

## 7. Create and bootstrap selected workspaces

For each selected slice, load WORKSPACES.md and complete create + bootstrap before starting its worker. A failed bootstrap means no worker launch; keep the workspace for diagnosis and do not auto-create a replacement.

Done when: every selected slice has a bootstrapped worktree workspace, or the run stopped on a failed bootstrap.

## 8. Start the worker

Worker contract: `/paseo-ship-slice <issue> orchestrated` — no lease audit, no own branch, no self-merge; stop at MERGE_READY. If the installed `/paseo-ship-slice` has no `orchestrated` mode, stop: a worker would cut its own branch and merge its own PR.

```bash
paseo run \
  --background \
  --provider "<provider/model>" \
  --title "slice-<issue>" \
  --workspace "<workspace-id>" \
  --label "slice=<issue>" \
  "<prompt>"
```

Retain the returned `agentId`; the worker runs in its slice workspace and, from this session, becomes your subagent.

Prompt the worker:

```text
You are the implementation worker for GitHub slice #<issue> under parent #<parent>.

You own ONLY #<issue>. The orchestrator already created and bootstrapped this Paseo worktree workspace (title "slice-<issue>", branch slice/<issue>-<slug>).

Invoke:

/paseo-ship-slice <issue> orchestrated

Rules:

- Work only on this slice.
- Do not create another branch, worktree, or workspace.
- Do not discover, claim, or implement sibling slices.
- Do not modify or close the parent.
- Implement production-ready code and complete required verification and code review.
- Open or update exactly one PR for this slice.
- Resolve required CI and review findings.
- DO NOT merge the PR.

When the PR is completely ready for the orchestrator's merge gate, report:

MERGE_READY: issue #<issue>, PR #<pr>, head <sha>

If blocked by a genuine product/spec/domain decision, state the exact decision required and wait.
```

If two slices were selected, launch both before entering supervision.

Done when: every selected worker is running in its slice workspace with the prompt delivered.

## 9. Supervise the active wave

Load SUPERVISE.md and supervise all active workers until a slice merges and frees a slot, a material user decision is required, or an unrecoverable environment/tooling failure occurs.

Done when: one of the three exit conditions is reached.

## 10. After every successful merge

Re-run steps 3–6: reconstruct state, reclassify every child, recompute capacity, select and dispatch the next wave. Do not trust the previous frontier.

Done when: the new wave is dispatched, or step 11's exit applies.

## 11. When no READY slice exists

- ACTIVE slices remain → continue supervising them.
- BLOCKED slices remain → name their blockers and continue supervising any ACTIVE work.
- No ACTIVE or READY slices but BLOCKED children remain → stop and report the blocking graph.

Do not invent work to fill capacity.

Done when: one of the three exits is taken.

## 12. Complete the parent

Implementation is complete only when every child is DONE, every slice PR is merged, no live slice worker remains, and no slice workspace remains needing recovery.

Then run:

```text
/openspec-verify-change <change>
```

**Passes** — confirm the parent's acceptance criteria, close the parent only when satisfied, and report: merged slices, merged PRs, final integration SHA, OpenSpec verification result, and any intentionally deferred work.

**Fails** — do not patch application code from the orchestrator. Determine whether the failure is an incomplete existing slice, a cross-slice integration defect, a missing requirement, or a new follow-up slice; create or propose the smallest appropriate follow-up work item. Do not close the parent.

Done when: the parent change is merged into the integration branch, final OpenSpec verification passes, and the report is delivered.

## Failure behavior

- **Paseo daemon unavailable** — stop orchestration; never launch unmanaged background workers outside Paseo.
- **GitHub unavailable** — stop state-changing orchestration actions; never guess tracker or PR state.
- **Worker crashes** — recover its existing workspace via WORKSPACES.md; never create a new one.
- **User decision required** — pause only the affected slices unless the decision changes the shared OpenSpec contract; other independent workers continue.
