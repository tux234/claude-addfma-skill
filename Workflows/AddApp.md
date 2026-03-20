# AddApp Workflow

Add a new app to the Fleet Maintained Apps catalog.

## Variables

- `FLEET_REPO` = from PREFERENCES.md (e.g. `/path/to/fleet/fleet`)
- `GITHUB_USER` = from PREFERENCES.md (e.g. `your-github-username`)
- `APP_SLUG` = filesystem-friendly app name (e.g. `figma`, `notion`)
- `BRANCH` = `{GITHUB_USER}-add-fma-{APP_SLUG}` (one branch for all platforms)
- `WORKTREE_PATH` = `{FLEET_REPO}/.worktrees/{APP_SLUG}`

---

## Step 1 — Gather App Info

If not already provided, ask the user for:
- App name (human-readable, e.g. "Figma")
- Platform: macOS, Windows, or both
- Any hints about the download URL or known quirks

---

## Step 2 — Create Worktree

From `FLEET_REPO`:

```bash
cd {FLEET_REPO}
git worktree add .worktrees/{APP_SLUG} -b {GITHUB_USER}-add-fma-{APP_SLUG} main
```

Always pass `main` as the base. Without it, the new branch inherits whatever the main worktree's current HEAD is — which may be another FMA branch — contaminating the new worktree with another app's files.

All subsequent file operations happen inside `WORKTREE_PATH`.

---

## Step 3 — Find App Metadata

### macOS (Homebrew)

1. Search Homebrew formulae: https://formulae.brew.sh/
2. Fetch the cask API:
   ```bash
   curl -s "https://formulae.brew.sh/api/cask/{token}.json" | jq '{token, version, url, sha256, artifacts}'
   ```
3. Note the `bundle_identifier` from artifacts — this goes in `unique_identifier`
4. Note the installer URL format (`dmg`, `pkg`, `zip`)

### Windows (Winget)

1. Search the winget-pkgs repo manifest: https://github.com/microsoft/winget-pkgs/tree/master/manifests
2. Find the `PackageIdentifier` (e.g. `Figma.Figma`)
3. Open the `.installer.yaml` to determine `InstallerType` and architecture
4. Note the `unique_identifier` (DisplayName as seen in Windows Add/Remove Programs — must match exactly)

**MANDATORY: Verify before claiming a platform is unavailable.** If you cannot find a package,
check the manifest directory directly via the GitHub API before stating "not available on Windows":
```bash
curl -s "https://api.github.com/repos/microsoft/winget-pkgs/contents/manifests/{letter}/{Publisher}" \
  | python3 -c "import json,sys; [print(i['name']) for i in json.load(sys.stdin) if isinstance(i,dict)]"
```
Do NOT state "not available on Windows" without verifying the directory returns 404 or only has Snapshot/Preview entries.

---

## Step 4 — Check if Enricher Is Needed (macOS only)

Test whether the Homebrew download URL works without special headers:

```bash
curl -sI "{homebrew_url}" | head -5
```

An enricher is needed if ANY of these are true:
- The URL requires `User-Agent: Homebrew` to return 200
- The URL redirects to a better/more stable CDN URL (override for reliability)
- The version string in Homebrew contains extra segments that don't match the app bundle version (e.g. `.stable_`, `.build_`, extra dots)
- A PKG installer is preferred over the DMG Homebrew provides

**⚠️ DMG-wrapping-PKG pattern**: Some casks download a `.dmg` that contains a `.pkg` inside
(e.g. Karabiner-Elements). Check the cask's `artifacts` for `{pkg: [...]}`:
```bash
curl -s "https://formulae.brew.sh/api/cask/{token}.json" | python3 -c "import json,sys; d=json.load(sys.stdin); [print('artifact:', a) for a in d.get('artifacts',[])]"
```
If the artifact is `{pkg: ['App.pkg']}` but the URL is a `.dmg`, you MUST provide a custom
`install_script_path` that mounts the DMG, installs the PKG, and ejects. Set `installer_format: pkg`
but DO NOT use the default template — it will fail on a DMG download.

**Enricher pattern** — override URL and/or clean version, then register in `main.go`:
```go
// ee/maintained-apps/ingesters/homebrew/external_refs/{app_name}.go
func {AppName}Installer(app *maintained_apps.FMAManifestApp) (*maintained_apps.FMAManifestApp, error) {
    // Set direct URL BEFORE modifying version (if version is used in URL)
    app.InstallerURL = "https://..."
    app.SHA256 = "no_check"
    // Clean version string if needed
    app.Version = strings.Replace(app.Version, ".stable_", ".", 1)
    return app, nil
}
```

Register in `ee/maintained-apps/ingesters/homebrew/external_refs/main.go`:
```go
"{APP_SLUG}/darwin": {{AppName}Installer},
```

**NEVER** add a `Headers` field to `FMAManifestApp`. If a URL requires headers, find the direct CDN URL instead (check where the Homebrew URL redirects to via `curl -sL --max-redirs 5 -o /dev/null -w "%{url_effective}" {url}`).

---

## Step 5 — Create Input JSON

### macOS — `ee/maintained-apps/inputs/homebrew/{APP_SLUG}.json`

```json
{
  "name": "App Name",
  "slug": "{APP_SLUG}/darwin",
  "unique_identifier": "com.example.AppName",
  "token": "homebrew-token",
  "installer_format": "dmg|pkg|zip",
  "default_categories": ["Developer tools|Browsers|Communication|Productivity"]
}
```

Add `install_script_path` or `uninstall_script_path` only if the generated scripts won't work (e.g. app has a pre/post uninstall requirement).

### Windows — `ee/maintained-apps/inputs/winget/{APP_SLUG}.json`

```json
{
  "name": "App Name",
  "slug": "{APP_SLUG}/windows",
  "package_identifier": "Publisher.AppName",
  "unique_identifier": "Exact DisplayName from Add/Remove Programs",
  "installer_arch": "x64",
  "installer_type": "msi|exe|msix",
  "installer_scope": "machine",
  "default_categories": ["Developer tools|Browsers|Communication|Productivity"]
}
```

For `.exe` installers you **must** provide `install_script_path` and `uninstall_script_path` in `inputs/winget/scripts/`.

#### Windows exe scripts — KISS first, complexity only when needed

**Before writing anything:** read the existing scripts in `ee/maintained-apps/inputs/winget/scripts/`. The pattern is already solved for most cases. Match the simplest script that covers your app's scope.

**Step 1 — determine install scope** (machine vs user):
```bash
# Check the winget installer manifest
curl -s "https://raw.githubusercontent.com/microsoft/winget-pkgs/master/manifests/{path}.installer.yaml" | grep -i scope
```
- `machine` scope → installs to `C:\Program Files`, registry in HKLM → use Chrome/VSCode as reference
- `user` scope → installs to `%LOCALAPPDATA%`, registry in HKCU → use Cursor as reference

**Step 2 — install script** (`{app}_install.ps1`):

Simple pattern (covers ~90% of cases):
```powershell
$exeFilePath = "$env:INSTALLER_PATH"
$process = Start-Process -FilePath $exeFilePath -ArgumentList "{silent flags}" -PassThru -Wait
Exit $process.ExitCode
```

Only use the timeout/kill pattern if the installer spawns a **persistent background service** that keeps the process alive indefinitely (e.g. RustDesk, Ollama). Do not use it by default.

**Step 3 — uninstall script** (`{app}_uninstall.ps1`):

Simple pattern — read registry, run uninstaller directly:
```powershell
$displayName = "Exact DisplayName from Add/Remove Programs"

$paths = @(
  'HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall',  # user-scope first
  'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall',
  'HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall'
)

$uninstall = $null
foreach ($p in $paths) {
  $items = Get-ItemProperty "$p\*" -ErrorAction SilentlyContinue |
    Where-Object { $_.DisplayName -eq $displayName }
  if ($items) { $uninstall = $items | Select-Object -First 1; break }
}

if (-not $uninstall) { Write-Host "Uninstaller not found"; Exit 1 }

# Kill running processes first
Stop-Process -Name "{app-process-name}" -Force -ErrorAction SilentlyContinue
Start-Sleep -Seconds 2

$cmd = if ($uninstall.QuietUninstallString) { $uninstall.QuietUninstallString } else { $uninstall.UninstallString }
$parts = $cmd.Split('"')
$exe = $parts[1]
$args = if ($parts.Length -eq 3) { $parts[2].Trim() } else { "" }

# Add any required silent flags for this app's uninstaller type:
# Chromium-based: --uninstall --force-uninstall
# NSIS/Inno: /S or /VERYSILENT /NORESTART
$args = "$args {silent flags}".Trim()

$p = Start-Process -FilePath $exe -ArgumentList $args -NoNewWindow -PassThru -Wait
Exit $p.ExitCode
```

**Do NOT use the scheduled task pattern** unless the uninstaller explicitly requires a different user context than the one running the script. It is complex, harder to debug, and rarely necessary. When in doubt, run directly with `-Wait`.

#### Uninstaller flag reference by type

| Uninstaller | Silent flags |
|-------------|-------------|
| Chromium-based (Chrome, Vivaldi, Brave) | `--uninstall --force-uninstall` |
| NSIS | `/S` |
| Inno Setup | `/VERYSILENT /NORESTART` |
| MSI (via msiexec) | `/quiet /norestart` |
| WiX | `/quiet` |

---

## Step 6 — Run Ingester

From `WORKTREE_PATH`:

```bash
go run cmd/maintained-apps/main.go --slug="{APP_SLUG}/darwin" --debug
go run cmd/maintained-apps/main.go --slug="{APP_SLUG}/windows" --debug  # if Windows
```

Watch the output for errors. Common issues:
- `token not found` → wrong Homebrew token
- `no installer found` → check `installer_format` and winget `installer_type`/`installer_arch`
- Hash mismatch → use `"sha256": "no_check"` in input, or add enricher with `app.SHA256 = "no_check"`

Verify the generated output at:
- macOS: `ee/maintained-apps/outputs/{APP_SLUG}/darwin.json`
- Windows: `ee/maintained-apps/outputs/{APP_SLUG}/windows.json`

Check that:
- `version` looks correct (not a Homebrew build tag)
- `installer_url` is accessible with a plain GET
- `sha256` is either a real hash or `"no_check"`

---

## Step 7 — Update apps.json

Add a description entry for each platform in `ee/maintained-apps/outputs/apps.json`, in alphabetical order by name:

```json
{
  "name": "App Name",
  "slug": "{APP_SLUG}/darwin",
  "platform": "darwin",
  "unique_identifier": "com.example.AppName",
  "description": "App Name is a short description ending with a period."
}
```

Follow sentence casing. End with a period.

**⚠️ ASCII-only rule — descriptions MUST use only ASCII characters (U+0000–U+007F).** Do NOT use:
- Curly/smart quotes (`'` `'` `"` `"`) — use straight quotes instead (`'` `"`)
- Em-dashes or en-dashes (`—` `–`) — use a hyphen-minus (`-`) instead
- Ellipsis (`…`) — write `...` instead
- Any other non-ASCII punctuation or symbols

**Why:** Go's `json.Encoder` escapes all non-ASCII code points as `\uXXXX` regardless of `SetEscapeHTML(false)`. When any new app is added, the entire `apps.json` is re-encoded. Non-ASCII characters in existing entries will be escaped on every subsequent ingester run, producing noisy diffs and corrupted human-readable output. Until a fix lands in the ingester, keep all descriptions to ASCII.

---

## Step 8 — Generate Icon

The icon generation script is **scoped to a single app** via the `-s` flag. Always use it — do not skip this step.

```bash
cd {WORKTREE_PATH}
tools/software/icons/generate-icons.sh -s {APP_SLUG}
```

This script produces exactly two things:
1. `frontend/pages/SoftwarePage/components/icons/{ComponentName}.tsx`
2. An updated entry in `frontend/pages/SoftwarePage/components/icons/index.ts`

**CRITICAL — only commit your app's icon files:**
```bash
git add frontend/pages/SoftwarePage/components/icons/{ComponentName}.tsx
git add frontend/pages/SoftwarePage/components/icons/index.ts
```

Do NOT `git add frontend/pages/SoftwarePage/components/icons/` — this will accidentally stage any other app's icon files that may have been generated. Stage individual files by name only.

The script may require a running app or PNG source. If it fails because the app is not installed, check if there is an existing PNG in `website/assets/images/` matching the app (e.g. `app-icon-{app-slug}-60x60@2x.png`). Pass it via the `-i` flag:
```bash
tools/software/icons/generate-icons.sh -s {APP_SLUG} -i website/assets/images/app-icon-{app-slug}-60x60@2x.png
```

---

## Step 9 — Pre-Commit Verification

**Do not skip this step.** Run all three checks before staging a single file. If any check fails, fix the issue and re-run. This is the last line of defense before the diff becomes someone else's problem.

### Check 1 — Scope: only this app's files are touched

```bash
cd {WORKTREE_PATH}
git diff --name-only origin/main | grep "ee/maintained-apps" | grep -v "apps\.json" | grep -v "{APP_SLUG}"
```

Expected output: **empty**. Any line that appears belongs to a different app and must not be committed. If files from another app appear, your worktree was likely branched off the wrong base. Stop. Investigate before proceeding.

### Check 2 — apps.json: no existing entries were modified

```bash
python3 - <<'EOF'
import json, subprocess, sys

main_json = subprocess.check_output(
    ["git", "show", "origin/main:ee/maintained-apps/outputs/apps.json"]
)
with open("ee/maintained-apps/outputs/apps.json") as f:
    branch_json = f.read()

main_apps = {a["slug"]: a for a in json.loads(main_json)["apps"]}
branch_apps = {a["slug"]: a for a in json.loads(branch_json)["apps"]}

problems = []
for slug, app in main_apps.items():
    if slug in branch_apps and branch_apps[slug] != app:
        problems.append(slug)

if problems:
    print("FAIL: existing apps.json entries were modified:")
    for p in problems:
        print(f"  {p}")
    sys.exit(1)

added = [s for s in branch_apps if s not in main_apps]
print(f"OK: {len(added)} new entries added: {added}")
EOF
```

Expected: `OK: N new entries added: [...]` — only your app's slugs. Any `FAIL` line means an existing entry was accidentally changed (encoding issue, text-edit gone wrong, wrong base branch). Restore those entries before committing.

### Check 3 — descriptions are ASCII-only

```bash
python3 - <<'EOF'
import json, sys

with open("ee/maintained-apps/outputs/apps.json") as f:
    data = json.load(f)

problems = []
for app in data["apps"]:
    desc = app.get("description", "")
    if any(ord(c) > 127 for c in desc):
        problems.append((app["slug"], repr(desc)))

if problems:
    print("FAIL: non-ASCII characters found in descriptions:")
    for slug, desc in problems:
        print(f"  {slug}: {desc}")
    sys.exit(1)

print("OK: all descriptions are ASCII-only")
EOF
```

Expected: `OK: all descriptions are ASCII-only`. Any `FAIL` means a description contains a curly quote, em-dash, or other non-ASCII character. Fix it before committing.

---

## Step 10 — Commit and Push

```bash
cd {WORKTREE_PATH}
git add ee/maintained-apps/inputs/ \
        ee/maintained-apps/outputs/ \
        ee/maintained-apps/ingesters/ \  # if enricher was added
        frontend/pages/SoftwarePage/components/icons/{ComponentName}.tsx \
        frontend/pages/SoftwarePage/components/icons/index.ts
git commit -m "Add {App Name} as Fleet maintained app ({macOS/Windows/macOS and Windows})"
git push -u origin tux234-add-fma-{APP_SLUG}
```

**Never add Co-Authored-By to commits.**

---

## Step 11 — Open Draft PR

Always open as a **draft** PR. The CI monitor will mark it ready once checks pass.

```bash
gh pr create \
  --draft \
  --assignee {GITHUB_USER} \
  --title "Add {App Name} as a fleet-maintained app" \
  --body "$(cat <<'EOF'
## Summary

- Adds {App Name} ({macOS/Windows/macOS and Windows}) to the Fleet maintained apps catalog
- Input: `ee/maintained-apps/inputs/{source}/{APP_SLUG}.json`
- Output generated via ingester script

## Validation checklist

- [ ] App can be downloaded using manifest URL
- [ ] App installs successfully using manifest install script
- [ ] App exists in software inventory after install
- [ ] App uninstalls successfully using manifest uninstall script
EOF
)"
```

---

## Step 12 — Hand Off to Main Session for CI Monitoring

**IMPORTANT: Subagents cannot spawn background agents.** Do not attempt to spawn the CI monitor yourself — it will silently fail.

When you finish this workflow, return the following to the main session:

```
✅ {App Name} PR #{PR_NUMBER} opened as draft
Branch:   tux234-add-fma-{APP_SLUG}
Worktree: {WORKTREE_PATH}
→ Ready for CI monitor
```

The main session will spawn a background monitor agent using `Tools/MonitorCI.md`. That agent will:
1. Poll CI until all checks are terminal
2. Re-run infra flakes (Docker/MySQL `test-go` failures) for free — no retry counted
3. Apply known fix patterns (e.g. persistent process) — counts as retry 1
4. Attempt one more fix on continued failure — counts as retry 2
5. After 2 retries with CI still red: halt and report back to the main session for human decision
6. On all green: run `gh pr ready {PR_NUMBER}` and notify the user

Do NOT run `gh pr ready` yourself.

---

## Notes

- CI workflow `test-fma-darwin-pr-only.yml` triggers automatically on changes to `ee/maintained-apps/outputs/**`
- If CI doesn't trigger, check that the output file was committed (it must change to trigger the path filter)
- PRs stay in draft until CI passes — the monitor agent marks them ready automatically
