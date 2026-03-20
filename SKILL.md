---
name: AddFMA
description: Fleet Maintained Apps workflow — add new or regenerate existing macOS/Windows apps in the Fleet FMA catalog. USE WHEN adding a new FMA, adding a Fleet maintained app, adding macOS app to fleet, adding Windows app to fleet, regenerating FMA output, updating FMA JSON, re-running FMA ingester, or regenerating darwin.json or windows.json.
---

# AddFMA

Manages the Fleet Maintained Apps (FMA) catalog: adding new apps and regenerating existing outputs.

## Customization

**Before executing, check for user customizations at:**
`~/.claude/skills/PAI/USER/SKILLCUSTOMIZATIONS/AddFMA/`

If this directory exists, load and apply any `PREFERENCES.md` found there.

## Voice Notification

**When executing a workflow, do BOTH:**

1. **Send voice notification**:
   ```bash
   curl -s -X POST http://localhost:8888/notify \
     -H "Content-Type: application/json" \
     -d '{"message": "Running the WORKFLOWNAME workflow in the AddFMA skill to ACTION", "voice_id": "fTtv3eikoepIosk8dTZ5", "voice_enabled": true}' \
     > /dev/null 2>&1 &
   ```

2. **Output text notification**:
   ```
   Running the **WorkflowName** workflow in the **AddFMA** skill to ACTION...
   ```

## Workflow Routing

| Workflow | Trigger | File |
|----------|---------|------|
| **AddApp** | "add new FMA", "add macOS app", "add Windows app", "new maintained app" | `Workflows/AddApp.md` |
| **RegenerateOutput** | "regenerate FMA output", "re-run ingester", "regenerate darwin.json", "update FMA JSON" | `Workflows/RegenerateOutput.md` |

## Examples

**Example 1: Add a new macOS app**
```
User: "Add Figma as a new Fleet maintained app for macOS"
→ Invokes AddApp workflow
→ Creates worktree, looks up Homebrew token, creates input JSON
→ Tests installer URL, adds enricher if needed
→ Runs ingester, commits, opens PR
```

**Example 2: Add a Windows app**
```
User: "Add Notion for Windows as an FMA"
→ Invokes AddApp workflow
→ Looks up winget PackageIdentifier, creates input JSON
→ Runs ingester, commits, opens PR
```

**Example 3: Regenerate existing output**
```
User: "Regenerate the Warp darwin.json"
→ Invokes RegenerateOutput workflow
→ Runs ingester for warp/darwin in current worktree
→ Commits and pushes updated output
```

## Quick Reference

- **Fleet repo:** Set via `FLEET_REPO` in PREFERENCES.md
- **Worktrees:** `.worktrees/` inside repo root
- **macOS inputs:** `ee/maintained-apps/inputs/homebrew/{app}.json`
- **Windows inputs:** `ee/maintained-apps/inputs/winget/{app}.json`
- **Ingester:** `go run cmd/maintained-apps/main.go --slug="{slug}" --debug`
- **Outputs:** `ee/maintained-apps/outputs/{app}/darwin.json` or `windows.json` — NEVER hand-edit
- **Branch pattern:** `{GITHUB_USER}-add-fma-{app-name}-{platform}`
- **GitHub user:** Set via `GITHUB_USER` in PREFERENCES.md
