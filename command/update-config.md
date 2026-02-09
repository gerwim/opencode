---
description: Update opencode-config repository and manage symlinks to ~/.config/opencode
---

# Update Opencode Config

You are helping the user update their opencode-config repository and manage symlinks.

## Configuration

Before starting, determine the opencode-config directory. Set this once and use it for all subsequent steps:

**Linux/macOS:**
```bash
OPENCODE_CONFIG_DIR="$HOME/opencode-config"
```

**Windows (PowerShell):**
```powershell
$OPENCODE_CONFIG_DIR = "$env:USERPROFILE\opencode-config"
```

If the user's repo is at a different path, substitute accordingly.

## Overview

This command will:
1. Detect the user's platform and validate the repository
2. Check for updates from the remote default branch and show a changelog
3. Ask user to confirm the update
4. Perform git pull if confirmed
5. Detect and symlink any new opencode-compatible folders

## Step 0: Detect Platform and Validate Repository

Before running any shell commands, detect the user's platform so you use the correct commands throughout.

**Linux/macOS:**
```bash
uname -s
```
This returns `Linux` or `Darwin`. Both use the same commands unless noted otherwise.

**Windows (PowerShell):**
```powershell
$env:OS
```
Returns `Windows_NT` on Windows.

Use the platform result to choose between `bash` and `powershell` code blocks for all subsequent steps.

### Validate Repository

Verify the config directory is a git repository with an `origin` remote:

**Linux/macOS:**
```bash
cd "$OPENCODE_CONFIG_DIR"
git rev-parse --is-inside-work-tree >/dev/null 2>&1 || { echo "Not a git repository"; exit 1; }
git remote get-url origin >/dev/null 2>&1 || { echo "No origin remote configured"; exit 1; }
```

**Windows (PowerShell):**
```powershell
Set-Location $OPENCODE_CONFIG_DIR
if (-not (git rev-parse --is-inside-work-tree 2>$null)) { throw "Not a git repository" }
if (-not (git remote get-url origin 2>$null)) { throw "No origin remote configured" }
```

### Detect Default Branch

Detect the remote default branch rather than assuming `master`:

**Linux/macOS:**

```bash
cd "$OPENCODE_CONFIG_DIR"
DEFAULT_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
if [ -z "$DEFAULT_BRANCH" ]; then
  if git show-ref --verify --quiet refs/remotes/origin/main; then
    DEFAULT_BRANCH="main"
  else
    DEFAULT_BRANCH="master"
  fi
fi
```

**Windows (PowerShell):**

```powershell
Set-Location $OPENCODE_CONFIG_DIR
$DEFAULT_BRANCH = (git symbolic-ref refs/remotes/origin/HEAD 2>$null) -replace '^refs/remotes/origin/', ''
if ([string]::IsNullOrWhiteSpace($DEFAULT_BRANCH)) {
  if (git show-ref --verify --quiet refs/remotes/origin/main) {
    $DEFAULT_BRANCH = 'main'
  } else {
    $DEFAULT_BRANCH = 'master'
  }
}
```

Use `$DEFAULT_BRANCH` in place of hardcoded `master` for all subsequent git commands.

## Step 1: Check for Updates

> **Note:** Steps 1-3 use only `git` commands, which work identically across all platforms. Platform-specific blocks are omitted where the commands are the same.

First, check for uncommitted changes so the user knows early whether an update can proceed:

```bash
cd "$OPENCODE_CONFIG_DIR"
git status --porcelain
```

If `git status --porcelain` output is not empty, warn the user:
> "Local changes detected in the config repo. You'll need to commit or stash them before updating."

Continue to show the changelog (so the user can decide whether to deal with local changes), but note that the update cannot proceed until the working tree is clean.

Then fetch the latest state from the remote and auto-fix `origin/HEAD` for future runs, and check for new commits:

```bash
cd "$OPENCODE_CONFIG_DIR"
git fetch --prune origin
git remote set-head origin --auto >/dev/null 2>&1
git log "HEAD..origin/$DEFAULT_BRANCH" --oneline --no-decorate
```

If the output is empty (no new commits), inform the user:
> "Your opencode-config is already up to date. No updates available."

Then skip to Step 4 (symlink check) to ensure any new folders are still properly linked.

If there are new commits, format the output as a bulleted changelog. Example:
> **Available Updates:**
> - abc1234 Add new agent configuration
> - def5678 Update command templates
> - ghi9012 Fix skill documentation

## Step 2: Ask for Confirmation

Present the changelog to the user and ask:

> "The above updates are available. Would you like to apply them?"

Wait for the user's response. If they say no, respond with:
> "Update cancelled. Your configuration remains unchanged."

Then continue to Step 4 (symlink check).

## Step 3: Perform Update

If the user confirms, verify the working tree is clean (as detected in Step 1). If uncommitted changes were found earlier and have not been resolved, **stop** and ask the user to commit or stash local changes before updating. Do **not** proceed to the pull.

Only if the working tree is clean, perform the pull:

```bash
cd "$OPENCODE_CONFIG_DIR"
git pull --ff-only origin "$DEFAULT_BRANCH"
```

If `git pull --ff-only` fails due to non-fast-forward, report that local and remote history diverged and ask the user how they want to reconcile.

On success, report:
> "Configuration updated successfully!"

## Step 4: Detect New Symlink Candidates

After the pull (or if no updates were available), check for new folders that need symlinking.

### List directories in the config repo

**Linux/macOS:**
```bash
cd "$OPENCODE_CONFIG_DIR"
ls -1d */ 2>/dev/null | sed 's|/$||' | grep -v '^\.' | sort
```

**Windows (PowerShell):**
```powershell
Set-Location $OPENCODE_CONFIG_DIR
Get-ChildItem -Directory | Where-Object { $_.Name -notlike '.*' } | Sort-Object Name | Select-Object -ExpandProperty Name
```

### List directories already symlinked in ~/.config/opencode/

**Linux/macOS:**
```bash
mkdir -p "$HOME/.config/opencode"
ls -1d "$HOME/.config/opencode"/*/ 2>/dev/null | while read -r d; do
  if [ -L "${d%/}" ]; then
    basename "${d%/}"
  fi
done | sort
```

**Windows (PowerShell):**
```powershell
$targetDir = "$env:USERPROFILE\.config\opencode"
if (-not (Test-Path $targetDir)) { New-Item -ItemType Directory -Path $targetDir -Force | Out-Null }
Get-ChildItem -Path $targetDir -Directory | Where-Object {
  $_.LinkType -eq 'SymbolicLink' -or $_.LinkType -eq 'Junction'
} | Sort-Object Name | Select-Object -ExpandProperty Name
```

### Compare the lists

Compare the two lists to find directories that:
1. Exist in the config repo
2. Are NOT already symlinked in ~/.config/opencode/
3. Are opencode-compatible (see criteria below)

### Opencode-Compatible Criteria (Strict)

Use a strict allowlist-first approach to avoid suggesting unrelated folders.

A directory is considered opencode-compatible only if it matches one of these rules:
1. **Exact name allowlist**: `agent`, `command`, `skills`, `prompts`, `workflows`, `templates`
2. **Known child-patterns**: contains at least one immediate child directory named one of:
   `agent`, `command`, `skills`, `prompts`, `workflows`, `templates`
3. **Opencode marker files at top level**: contains one of these files directly under the folder:
   `OPENCODE.md`, `AGENTS.md`, `COMMANDS.md`, `SKILLS.md`

> **Maintenance note:** If the opencode ecosystem adds new folder conventions or marker files, update these allowlists accordingly.

Do **not** treat "contains any `.md` file anywhere" as sufficient by itself.

Suggested checks for a given `<folder>`:

**Linux/macOS:**
```bash
# Rule 2: known child-patterns (immediate children only)
ls -1d "$OPENCODE_CONFIG_DIR/<folder>"/agent \
       "$OPENCODE_CONFIG_DIR/<folder>"/command \
       "$OPENCODE_CONFIG_DIR/<folder>"/skills \
       "$OPENCODE_CONFIG_DIR/<folder>"/prompts \
       "$OPENCODE_CONFIG_DIR/<folder>"/workflows \
       "$OPENCODE_CONFIG_DIR/<folder>"/templates 2>/dev/null | grep -qE '.' && echo "match"

# Rule 3: marker files at top level only
ls "$OPENCODE_CONFIG_DIR/<folder>"/OPENCODE.md \
   "$OPENCODE_CONFIG_DIR/<folder>"/AGENTS.md \
   "$OPENCODE_CONFIG_DIR/<folder>"/COMMANDS.md \
   "$OPENCODE_CONFIG_DIR/<folder>"/SKILLS.md 2>/dev/null | grep -qE '.' && echo "match"
```

**Windows (PowerShell):**
```powershell
# Rule 2: known child-patterns (immediate children only)
$childPatterns = @('agent','command','skills','prompts','workflows','templates')
$hasChild = Get-ChildItem -Path "$OPENCODE_CONFIG_DIR\<folder>" -Directory |
  Where-Object { $childPatterns -contains $_.Name }
if ($hasChild) { Write-Output "match" }

# Rule 3: marker files at top level only
$markerFiles = @('OPENCODE.md','AGENTS.md','COMMANDS.md','SKILLS.md')
$hasMarker = Get-ChildItem -Path "$OPENCODE_CONFIG_DIR\<folder>" -File |
  Where-Object { $markerFiles -contains $_.Name }
if ($hasMarker) { Write-Output "match" }
```

If no folders match these strict rules, report no candidates rather than proposing broad guesses.

## Step 5: Interactive Symlink Selection

If there are new compatible folders found, present them to the user:

> **New folders detected that can be symlinked:**
> The following folders in the config repo are not yet linked to ~/.config/opencode/:
>
> [ ] 1. custom-skills
> [ ] 2. new-commands
> [ ] 3. my-templates
>
> Select which folders to symlink (space to toggle, enter to confirm, or type 'all' to select all):

Use the question tool to let the user select which folders to symlink. Allow multiple selection.

If no new folders are detected, report:
> "All folders are already properly symlinked. No action needed."

## Step 6: Create Symlinks

Before creating each symlink, check for name collisions with existing non-symlink directories:

**Linux/macOS:**
```bash
if [ -e "$HOME/.config/opencode/<folder>" ] && [ ! -L "$HOME/.config/opencode/<folder>" ]; then
  echo "WARNING: $HOME/.config/opencode/<folder> already exists as a regular directory"
fi
```

**Windows (PowerShell):**
```powershell
$target = "$env:USERPROFILE\.config\opencode\<folder>"
if ((Test-Path $target) -and -not (Get-Item $target).LinkType) {
  Write-Warning "$target already exists as a regular directory"
}
```

If a regular (non-symlink) directory already exists at the target path, warn the user and ask whether to skip it, back it up, or remove it before symlinking. Do **not** create a symlink inside the existing directory.

For each folder the user selected (with no collision), create the symlink:

**Linux/macOS:**
```bash
ln -s "$OPENCODE_CONFIG_DIR/<folder>" "$HOME/.config/opencode/<folder>"
```

Verify each symlink was created successfully by checking the symlink target matches the expected path:

```bash
[ -L "$HOME/.config/opencode/<folder>" ] && \
  [ "$(readlink "$HOME/.config/opencode/<folder>")" = "$OPENCODE_CONFIG_DIR/<folder>" ] && \
  echo "OK" || echo "FAILED"
```

**Windows (PowerShell):**
```powershell
try {
    New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.config\opencode\<folder>" -Target "$OPENCODE_CONFIG_DIR\<folder>"
} catch {
    Write-Warning "SymbolicLink failed (may need Developer Mode or admin). Falling back to Junction..."
    New-Item -ItemType Junction -Path "$env:USERPROFILE\.config\opencode\<folder>" -Target "$OPENCODE_CONFIG_DIR\<folder>"
}
```

Verify each link was created successfully:

```powershell
(Get-Item "$env:USERPROFILE\.config\opencode\<folder>").Target
```

The link is valid only if the output matches the expected target path.

## Step 7: Summary

Provide a final summary of what was done:

**If updates were applied:**
> **Update Summary:**
> - Pulled latest changes from origin/$DEFAULT_BRANCH
> - Updated X commit(s)
> - Created X new symlink(s): [list folders]

**If no updates but symlinks added:**
> **Update Summary:**
> - Configuration was already up to date
> - Created X new symlink(s): [list folders]

**If nothing changed:**
> **Update Summary:**
> - Configuration is up to date
> - All folders are properly symlinked
> - No actions needed

## Error Handling

If any command fails, report the error clearly:

- **No git repository**: "The config directory does not appear to be a git repository. Please ensure it's properly initialized."
- **No origin remote**: "No origin remote is configured. Please add one with: git remote add origin <url>"
- **Git fetch/pull errors**: "Failed to update repository. Error: [error message]"
- **Dirty working tree**: "Local changes detected in the config repo. Please commit or stash before updating."
- **Non-fast-forward pull**: "Cannot fast-forward to origin/$DEFAULT_BRANCH because histories diverged. Please reconcile branches before updating."
- **Permission errors (Linux/macOS)**: "Permission denied when creating symlinks. You may need to check directory permissions."
- **Symlink permission error (Windows)**: "Creating symbolic links requires Developer Mode enabled or administrator privileges. Attempting junction fallback..."
- **Junction creation error (Windows)**: "Failed to create junction. Ensure the target path is a local directory (junctions do not work with network paths)."
- **Directory collision**: "A regular directory already exists at ~/.config/opencode/<folder>. Please back it up or remove it before symlinking."

## Notes

- The command should be idempotent - running it multiple times should be safe
- Existing symlinks should never be overwritten or modified
- Only directories should be symlinked, never individual files
- Hidden directories (starting with .) should be ignored
- On Windows, junctions behave like symlinks for directories but do not require elevated privileges. They only work for local paths (not network shares).
- PowerShell resolves `$HOME` and `~` to the user's profile directory, equivalent to `$env:USERPROFILE`
- Path separators: use `/` on Linux/macOS and `\` on Windows (PowerShell accepts both, but `\` is conventional)

## Platform Reference

| Operation | Linux/macOS | Windows (PowerShell) |
|---|---|---|
| Set config dir | `OPENCODE_CONFIG_DIR="$HOME/..."` | `$OPENCODE_CONFIG_DIR = "$env:USERPROFILE\..."` |
| Detect platform | `uname -s` | `$env:OS` |
| List directories | `ls -1d */` | `Get-ChildItem -Directory` |
| Check if symlink | `[ -L path ]` | `(Get-Item path).LinkType` |
| Create directory | `mkdir -p path` | `New-Item -ItemType Directory -Force` |
| Create symlink | `ln -s target link` | `New-Item -ItemType SymbolicLink` |
| Symlink fallback | n/a | `New-Item -ItemType Junction` |
| Read symlink target | `readlink path` | `(Get-Item path).Target` |
| Verify symlink | `readlink` output matches expected target | `.Target` matches expected path |
