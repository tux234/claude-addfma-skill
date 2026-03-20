# MonitorCI — Background CI Monitoring Agent

Monitors CI checks on a Fleet FMA draft PR, attempts to fix failures (up to 2 retries), then marks the PR ready for review or escalates to the user.

## Inputs (provided by caller)

- `PR_NUMBER` — GitHub PR number to monitor
- `BRANCH` — git branch name (e.g. `{GITHUB_USER}-add-fma-figma-darwin`)
- `WORKTREE_PATH` — absolute path to the worktree for this PR
- `APP_NAME` — human-readable app name (for reporting)

## Behavior

### Phase 1 — Wait for CI to complete

Poll until all checks reach a terminal state (no pending/queued checks):

```bash
while true; do
  pending=$(gh pr checks {PR_NUMBER} --json state | jq '[.[] | select(.state == "PENDING" or .state == "IN_PROGRESS" or .state == "")] | length')
  if [ "$pending" -eq 0 ]; then break; fi
  echo "Waiting for CI... ($pending checks still running)"
  sleep 60
done
```

### Phase 2 — Evaluate results

```bash
gh pr checks {PR_NUMBER} --json name,state,detailsUrl
```

**If all checks pass:** mark PR ready and notify user.

```bash
gh pr ready {PR_NUMBER}
```

Then report: "✅ {APP_NAME} PR #{PR_NUMBER} — all CI checks passed. Marked ready for review."

**If any checks failed:** proceed to Phase 3.

### Phase 3 — Diagnose failures

For each failing check, classify it before counting it as a retry:

#### Infra flake — do NOT count as a retry, just re-run

A failure is an **infra flake** if ALL of these are true:
- The check name starts with `test-go`
- The logs contain any of:
  - `No such file or directory` referencing `/tmp/gotest.log`
  - `dial tcp` or `connection refused` referencing MySQL or Redis
  - `Error response from daemon` (Docker error)
  - `context deadline exceeded` without a test name in the stack
  - `no space left on device`

To re-run a flaky check:
```bash
# Get the run ID from the detailsUrl, then:
gh run rerun {RUN_ID} --failed
```

Wait for the re-run to complete (Phase 1 again), then re-evaluate. Do NOT increment the retry counter for infra flakes — re-runs are free.

#### Known fixable pattern — persistent process (counts as retry, but fix is known)

If `test-fma-pr-only` fails and the logs show **all of these**:
- `Error executing install script: exit status 1`
- `New application detected at: C:\Program Files\...` (app DID install)
- The install took close to 5 minutes (timestamp delta ~300s)

This is the **persistent process pattern**: the installer spawns a service/daemon that keeps the parent process alive, `-Wait $true` blocks on it, and the CI validator's 5-minute timeout kills the script.

**Fix:** In the install script, replace `-Wait $true` / `Wait = $true` with `WaitForExit(120000)` + `Stop-Process` cleanup. See the exe installer pattern documented in `Workflows/AddApp.md`. After fixing, regenerate the output JSON (`go run cmd/maintained-apps/main.go --slug="{slug}/windows" --debug`) since the script hash changes.

#### Real failure — counts as a retry

Everything else is a real failure. Fetch the logs:

```bash
gh run view {RUN_ID} --log-failed
```

Read the logs carefully to understand the root cause. Common FMA failure patterns:
- `test-fma-pr-only` — installer download failed, script error, osquery not finding the app
- `test-js` — TypeScript import error, missing icon component
- `lint-go` — gofmt misalignment, banned import, linter violation

Attempt a fix in `WORKTREE_PATH`, commit, and push:

```bash
cd {WORKTREE_PATH}
# ... make targeted fix ...
git add {changed files}
git commit -m "Fix CI: {one-line description of what was fixed}"
git push origin {BRANCH}
```

**Retry counter:** increment after each real fix attempt. Maximum = 2.

After pushing, return to Phase 1 to wait for CI again.

### Phase 4 — Retry limit reached

If the retry counter reaches 2 and CI is still failing, stop and report to the user:

```
⚠️  {APP_NAME} PR #{PR_NUMBER} — CI still failing after 2 fix attempts.

Failing checks:
- {check name}: {brief error summary}
- {check name}: {brief error summary}

What was tried:
1. {description of first fix attempt}
2. {description of second fix attempt}

Logs: {detailsUrl for each failing check}

Do you want me to keep trying?
```

Do not make any further changes. Wait for user input.

## Cost awareness

- Polling (`gh pr checks`) costs zero tokens — do it freely.
- Reading CI logs costs tokens — only fetch logs for **failing** checks, not all checks.
- Each fix attempt (read → reason → write → push) is the expensive step. The 2-retry cap is hard.
- Re-running infra flakes is free (no reasoning required, no token cost beyond the re-run command).
