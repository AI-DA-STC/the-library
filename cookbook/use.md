# Use a Skill from the Library

## Context
Pull a skill, agent, or prompt from the catalog into the local environment. If already installed locally, check for a version change and ask before overwriting.

## Input
The user provides a skill name or description.

## Steps

### 1. Sync the Library Repo
Pull the latest catalog before reading:
```bash
cd <LIBRARY_SKILL_DIR>
git pull
```

### 2. Find the Entry
- Read `library.yaml`
- Search across `library.skills`, `library.agents`, and `library.prompts`
- Match by name (exact) or description (fuzzy/keyword match)
- If multiple matches, show them and ask the user to pick one
- If no match, tell the user and suggest `/library search`

### 3. Resolve Dependencies
If the entry has a `requires` field:
- For each typed reference (`skill:name`, `agent:name`, `prompt:name`):
  - Look it up in `library.yaml`
  - If found, recursively run the `use` workflow for that dependency first
  - If not found, warn the user: "Dependency <ref> not found in library catalog"
- Process all dependencies before the requested item

### 4. Determine Target Directory
- Read `default_dirs` from `library.yaml`
- If user said "global" or "globally" → use the `global` path
- If user specified a custom path → use that path
- Otherwise → use the `default` path
- Select the correct section based on type (skills/agents/prompts)

### 5. Check Installed Version

Determine whether this item is already installed:
- Check if `<target_directory>/<name>/` exists (for skills) or `<target_directory>/<name>.md` exists (for agents/prompts)

**If not installed** → skip to step 6, no confirmation needed.

**If already installed**, compare versions:

Read the local version from the installed file's frontmatter:
```bash
grep '^version:' <target_directory>/<name>/SKILL.md | head -1 | sed 's/version:[[:space:]]*//'
```

Fetch the remote version without a full clone:
- For GitHub sources: use `gh api` to read just the frontmatter file:
  ```bash
  gh api repos/<org>/<repo>/contents/<file_path> --jq '.content' | base64 -d \
    | grep '^version:' | head -1 | sed 's/version:[[:space:]]*//'
  ```
- For local sources: read the file directly.

Then show the user one of these messages and **wait for confirmation**:

- If remote version differs from local:
  ```
  wiki_brain is installed at v<local>. v<remote> is available.
  Update to v<remote>? [y/N]
  ```
- If versions are the same:
  ```
  wiki_brain is already at v<version>. Re-install anyway? [y/N]
  ```

If the user says **no** (or anything other than y/yes): stop here and report:
```
Kept installed version: <name> v<local> at <target_directory>/<name>/
```

### 6. Fetch from Source

**If source is a local path** (starts with `/` or `~`):
- Resolve `~` to the home directory
- Get the parent directory of the referenced file
- For skills: copy the entire parent directory to the target:
  ```bash
  cp -R <parent_directory>/ <target_directory>/<name>/
  ```
- For agents: copy just the agent file to the target:
  ```bash
  cp <agent_file> <target_directory>/<agent_name>.md
  ```
- For prompts: copy just the prompt file to the target:
  ```bash
  cp <prompt_file> <target_directory>/<prompt_name>.md
  ```
- If the agent or prompt is nested in a subdirectory under the `agents/` or `commands/` directories, copy the subdirectory to the target as well, creating the subdir if it doesn't exist. This is useful because it keeps the agents or commands grouped together.

**If source is a GitHub URL**:
- Parse the URL to extract: `org`, `repo`, `branch`, `file_path`
  - Browser URL pattern: `https://github.com/<org>/<repo>/blob/<branch>/<path>`
  - Raw URL pattern: `https://raw.githubusercontent.com/<org>/<repo>/<branch>/<path>`
- Determine the clone URL: `https://github.com/<org>/<repo>.git`
- Determine the parent directory path within the repo (everything before the filename)
- Clone into a temporary directory:
  ```bash
  tmp_dir=$(mktemp -d)
  git clone --depth 1 --branch <branch> https://github.com/<org>/<repo>.git "$tmp_dir"
  ```
- Copy the parent directory of the file to the target:
  ```bash
  cp -R "$tmp_dir/<parent_path>/" <target_directory>/<name>/
  ```
- Clean up:
  ```bash
  rm -rf "$tmp_dir"
  ```

**If clone fails (private repo)**, try SSH:
  ```bash
  git clone --depth 1 --branch <branch> git@github.com:<org>/<repo>.git "$tmp_dir"
  ```

### 7. Verify Installation
- Confirm the target directory exists
- Confirm the main file (SKILL.md, AGENT.md, or prompt file) exists in it
- Report success with the installed path

### 8. Confirm
Tell the user:
- What was installed/updated and where
- The version that is now installed
- Any dependencies that were also installed
- Whether this was a fresh install or an update
