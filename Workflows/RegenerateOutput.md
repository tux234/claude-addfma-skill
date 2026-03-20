# RegenerateOutput Workflow

Regenerate the output JSON for an existing Fleet Maintained App using the ingester script.
Use this when bumping a version, after modifying an enricher, or to trigger CI.

## When to Use

- A new version of an existing app needs to be reflected in `outputs/`
- You modified an enricher and need to regenerate the output
- The output JSON was accidentally hand-edited and needs to be restored
- CI didn't trigger because the output file wasn't touched

---

## Step 1 — Confirm Slug and Working Directory

Identify:
- `FULL_SLUG` — e.g. `warp/darwin`, `1password/darwin`, `box-drive/windows`
- Working directory — should be inside a worktree or the repo root

The current worktree path follows: `{FLEET_REPO}/.worktrees/{branch-name}`
Fleet repo: set via `FLEET_REPO` in PREFERENCES.md

---

## Step 2 — Run the Ingester

From the worktree root:

```bash
go run cmd/maintained-apps/main.go --slug="{FULL_SLUG}" --debug
```

Watch for errors. The script will:
1. Fetch app metadata from Homebrew or winget
2. Apply any enrichers registered for this slug
3. Write/update `ee/maintained-apps/outputs/{APP_SLUG}/{darwin|windows}.json`

**Never manually edit the output JSON.** Always regenerate via this script.

---

## Step 3 — Verify Output

Read the generated file and confirm:
- `version` is current and correct (no leftover build tags like `.stable_`)
- `installer_url` is accessible with a plain GET
- `sha256` is either a real hash or `"no_check"`
- `install_script_ref` and `uninstall_script_ref` match expected scripts

```bash
cat ee/maintained-apps/outputs/{APP_SLUG}/{darwin|windows}.json
```

---

## Step 4 — Commit and Push

```bash
git add ee/maintained-apps/outputs/{APP_SLUG}/{darwin|windows}.json
git commit -m "Regenerate {App Name} output via ingester script"
git push
```

**Never add Co-Authored-By to commits.**

If the push is rejected (non-fast-forward), check for remote-only commits:
```bash
git log --oneline origin/{branch} -5
git rebase origin/{branch}
git push
```

---

## CI Note

The CI workflow `test-fma-darwin-pr-only.yml` triggers on changes to:
- `ee/maintained-apps/inputs/**`
- `ee/maintained-apps/outputs/**`
- `cmd/maintained-apps/validate/**`

If CI still doesn't trigger after pushing, the PR may be in a state where GitHub skipped the workflow due to a force push history. In that case, make a meaningful file change (e.g., add a comment to the input JSON) and push again.
