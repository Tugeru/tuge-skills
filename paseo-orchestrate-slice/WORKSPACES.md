# Paseo Workspace Lifecycle

Use this reference whenever the orchestrator creates, bootstraps, recovers, or archives a slice worktree workspace.

The orchestrator owns the workspace lifecycle. Workers do not create or archive their own workspaces.

`$ROOT`, `$REPO`, `$BASE`, and `$BASE_SHA` come from SKILL.md step 1.

The installed `paseo` CLI is the authority for command syntax and response shapes. Confirm flags with `paseo <command> --help` and read identifiers from JSON responses; never guess field names.

## Hard limit

Before creating anything, count active slice worktree workspaces. If 2 already exist, stop — READY tickets remain queued until a slot frees.

## 1. Refresh the integration base

```bash
git fetch origin "$BASE"
BASE_SHA="$(git rev-parse "origin/$BASE")"
```

Branch from the fetched `origin/$BASE`, never a stale local checkout.

## 2. Create the workspace

Branch name: `<type>/<issue>-<slug>`. Default `feat/`; use `fix/` for bug tickets, `docs/` for documentation-only tickets, `chore/` for maintenance, `hotfix/` for urgent fixes.

```bash
paseo workspace create \
  --isolation worktree \
  --mode branch-off \
  --new-branch "<type>/<issue>-<slug>" \
  --base "origin/$BASE" \
  --title "slice-<issue>" \
  --worktree-slug "slice-<issue>" \
  --json
```

From the JSON result, retain the `workspaceId`, the worktree path (`cwd`), and the branch.

The repository's committed `paseo.json` configures the worktree bootstrap: `worktree.setup` runs once after the worktree is created (dependency install, env files, migrations). Wait for setup to finish before declaring the workspace bootstrapped.

## 3. Bootstrap secrets and dependencies

The repository convention: copy the canonical environment and verify dependencies.

```bash
SOURCE_ENV="$ROOT/.env.local"
WORKTREE="<returned worktree path>"

test -f "$SOURCE_ENV" || { echo "Missing canonical .env.local: $SOURCE_ENV" >&2; exit 1; }
test -d "$WORKTREE" || { echo "Missing worktree: $WORKTREE" >&2; exit 1; }

install -m 600 "$SOURCE_ENV" "$WORKTREE/.env.local"
```

Treat `.env.local` contents as secret: never print, prompt with, log, or commit them.

Verify Git ignores it:

```bash
git -C "$WORKTREE" check-ignore -q .env.local || {
  echo ".env.local is not ignored by Git; refusing worker launch." >&2
  exit 1
}
```

Install dependencies only when the setup hook did not:

```bash
test -d "$WORKTREE/node_modules" || pnpm --dir "$WORKTREE" install --frozen-lockfile
```

No silent fallback to a lockfile-mutating install. If the lockfile is inconsistent, stop and surface the repository problem.

## 4. Bootstrap sanity check

```bash
test -d "$WORKTREE"
test -f "$WORKTREE/.env.local"
test -d "$WORKTREE/node_modules"
git -C "$WORKTREE" status --short --branch
```

Require the checked-out branch to be the slice branch, and `.env.local` to be absent from Git status. This establishes a runnable development worktree only — `/paseo-ship-slice` owns `pnpm test`, `pnpm lint`, and `pnpm build`.

## 5. Bootstrap failure

If any step fails:

1. Do not start a worker.
2. Do not create another workspace for the same slice.
3. Keep the failed workspace for diagnosis.
4. Report the failed command and shortest useful error.
5. Leave the ticket READY or mark orchestration attention required.

The failed workspace still counts toward the two-worktree ceiling until archived.

## 6. Recover an interrupted workspace

If the workspace exists but its worker is gone, do not create another one. Find it:

```bash
paseo workspace ls --json          # title "slice-<issue>"; retain workspaceId + cwd
paseo ls -a -g --json --label "slice=<issue>"   # find the worker agent
```

Inspect the worktree:

```bash
git -C "$WORKTREE" status --short --branch
git -C "$WORKTREE" log --oneline -10
```

Inspect GitHub issue and PR state. Verify bootstrap:

```bash
test -f "$WORKTREE/.env.local"
test -d "$WORKTREE/node_modules"
```

If `.env.local` is missing, copy it as in step 3. Do not overwrite an existing `.env.local` during ordinary recovery. If dependencies are absent, run the frozen install again.

If the worker agent still exists, resume it:

```bash
paseo send "<agent-id>" "<resume prompt>"
```

Otherwise relaunch in the existing workspace:

```bash
paseo run \
  --background \
  --provider "<provider/model>" \
  --title "slice-<issue>" \
  --workspace "<workspace-id>" \
  --label "slice=<issue>" \
  "<resume prompt>"
```

Resume prompt:

```text
Resume GitHub slice #<issue> under parent #<parent>.

This is an existing Paseo worktree workspace from an interrupted worker. Do not restart the implementation.

Reconstruct current state from:

- git status/log/diff
- GitHub issue
- existing PR
- OpenSpec artifacts
- CI/review state

Continue:

/paseo-ship-slice <issue> orchestrated

Do not merge. Stop only at MERGE_READY.
```

Never discard uncommitted work automatically.

## 7. Cleanup

Archive a slice workspace only after GitHub confirms its PR is merged.

```bash
git -C "$WORKTREE" status --short
```

If clean:

```bash
paseo workspace archive "<workspace-id>"
```

Archiving closes the workspace's agents and terminals; Paseo removes the managed worktree after the final workspace reference is archived. The branch itself is not deleted.

If the checkout is unexpectedly dirty, stop, preserve it, and report the files. Never archive merely to free a concurrency slot when uncommitted work exists.

After archiving, verify the workspace no longer appears in `paseo workspace ls --json`. Only then is the execution slot free.
