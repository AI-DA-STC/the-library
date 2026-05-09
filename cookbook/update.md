# Update an Installed Item

## Context
Check one or all installed items for newer versions and offer to update each one. Unlike `/library use`, this command is explicitly for items already installed — it errors if the named item is not installed, and it shows a clear before/after version for every update.

With no argument: check all installed items.
With a name: check only that item.

## Input
- `/library update` — check all installed items for updates
- `/library update <name>` — check one specific item

## Steps

### 1. Sync the Library Repo
```bash
cd <LIBRARY_SKILL_DIR>
git pull
```

### 2. Resolve Scope

**If a name was given:**
- Look it up in `library.yaml`
- If not found in catalog: "No catalog entry for '<name>'. Use `/library add` to register it first."
- Determine its target directory (global if `--global` was passed, otherwise default)
- If not installed locally: "wiki_brain is not installed. Use `/library use <name>` to install it first."
- Proceed with just this one item.

**If no name was given:**
- Read all entries in `library.yaml`
- For each entry, check if it is installed in the default or global directory
- Collect only the installed ones
- If nothing is installed: "No items are currently installed."

### 3. Check Versions

For each item in scope, read the locally installed version:
```bash
grep '^version:' <target_directory>/<name>/SKILL.md | head -1 | sed 's/version:[[:space:]]*//'
```
(For agents/prompts that have no version field, treat local version as "unknown".)

Fetch the remote version without a full clone:
- For GitHub sources:
  ```bash
  gh api repos/<org>/<repo>/contents/<file_path> --jq '.content' | base64 -d \
    | grep '^version:' | head -1 | sed 's/version:[[:space:]]*//'
  ```
- For local sources: read the file directly.

### 4. Show Update Summary

Display a table of all items in scope:

```
Available updates:

  Name        Installed   Latest   Status
  ─────────── ─────────── ──────── ────────────────
  wiki_brain  v0.1.0      v0.1.1   update available
  other_skill v0.2.3      v0.2.3   up to date
```

If nothing has an update available: "All installed items are up to date." and stop.

### 5. Confirm Each Update

For each item that has an update available, ask individually:

```
Update wiki_brain v0.1.0 → v0.1.1? [y/N]
```

If the user says **no**: skip that item, report "Kept v<local>".
If the user says **yes**: proceed to fetch for that item (step 6).

If there are multiple updates, a user can also say "all" or "yes to all" at the first prompt to approve all remaining updates without further prompting.

### 6. Fetch Updates

For each approved item, follow the same fetch steps as `cookbook/use.md` § Fetch from Source.

### 7. Report Results

```
Update complete:

  wiki_brain   v0.1.0 → v0.1.1  ✓ updated
  other_skill  v0.2.3            – skipped
```
