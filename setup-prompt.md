# CC-Workflow Setup — Paste this entire file into any Claude Code chat to install

You are performing a zero-configuration installation of the cc-workflow-subagent-magic system. Execute every step below without asking the user any questions. All paths are resolved dynamically at runtime.

---

## STEP 0 — LOCATE THE REPO

Locate the `claude-code-workflow` repository on this machine. This is the directory containing this setup-prompt.md file (or the directory passed as context). The user cloned this repo to some local path — find it.

Strategy:
1. The file you are reading IS this setup-prompt.md. Its parent directory is the repo root.
2. If you cannot determine the file's location, search for a directory named `claude-code-workflow` containing a `README.md` and a `.claude/commands/` subfolder.
3. Common locations to check: `%USERPROFILE%\Documents\`, `%USERPROFILE%\`, `~/Documents/`, `~/`.

Store the repo root as REPO_PATH. Verify it contains `.claude/commands/01-intake.md` — if not, output:
```
ERROR: Could not locate the claude-code-workflow repo. Ensure you are running this from within the cloned repo directory.
```
Stop.

---

## STEP 1 — LOCATE GLOBAL .CLAUDE FOLDER

Detect the global `.claude` folder dynamically. Check each path in order and use the first one that exists:

**Windows:**
1. `%APPDATA%\Claude\` — check if this directory exists
2. `%USERPROFILE%\.claude\` — check if this directory exists
3. `%LOCALAPPDATA%\Claude\` — check if this directory exists

**macOS / Linux:**
1. `~/.claude/` — check if this directory exists

The correct global `.claude` path is the one that exists and where Claude Code stores its global commands. Verify by checking if a `commands/` subfolder is present or creatable.

Store as GLOBAL_CLAUDE_PATH.

If no valid global `.claude` path is found:
```
ERROR: Cannot locate global .claude folder. Ensure Claude Code is installed.
Checked: %APPDATA%\Claude\, %USERPROFILE%\.claude\, %LOCALAPPDATA%\Claude\, ~/.claude/
```
Stop.

---

## STEP 2 — CREATE REQUIRED DIRECTORIES

Create the following directories if they do not already exist. Do not fail if they already exist.

1. `[GLOBAL_CLAUDE_PATH]/commands/` — global commands directory
2. `[REPO_PATH]/logs/` — execution logs directory
3. `[REPO_PATH]/archives/` — self-iteration archive directory

---

## STEP 3 — INSTALL GLOBAL COMMANDS

Copy the following files from the repo to the global .claude commands directory.

For each file: read the source file using the Read tool, then write it to the destination using the Write tool. Do not modify any content — copy verbatim.

**Source → Destination:**
1. `[REPO_PATH]/.claude/commands/01-intake.md` → `[GLOBAL_CLAUDE_PATH]/commands/01-intake.md`
2. `[REPO_PATH]/.claude/commands/02-extract-profile.md` → `[GLOBAL_CLAUDE_PATH]/commands/02-extract-profile.md`
3. `[REPO_PATH]/.claude/commands/03-build-prompt.md` → `[GLOBAL_CLAUDE_PATH]/commands/03-build-prompt.md`
4. `[REPO_PATH]/.claude/commands/04-refinep.md` → `[GLOBAL_CLAUDE_PATH]/commands/04-refinep.md`
5. `[REPO_PATH]/.claude/commands/05-split-orc-v2.md` → `[GLOBAL_CLAUDE_PATH]/commands/05-split-orc-v2.md`
6. `[REPO_PATH]/.claude/commands/06-seq-init.md` → `[GLOBAL_CLAUDE_PATH]/commands/06-seq-init.md`

**Global entry point command (already in global commands — verify or install):**
7. If `[GLOBAL_CLAUDE_PATH]/commands/cc-workflow.md` does not exist:
   Read `[REPO_PATH]/.claude/commands/cc-workflow.md` and write to `[GLOBAL_CLAUDE_PATH]/commands/cc-workflow.md`.
   (If cc-workflow.md is not in the repo .claude/commands/, it may have been installed separately — skip this step without error.)

**Global audit command (verify or install):**
8. If `[GLOBAL_CLAUDE_PATH]/commands/cc-audit.md` does not exist:
   Read `[REPO_PATH]/.claude/commands/cc-audit.md` and write to `[GLOBAL_CLAUDE_PATH]/commands/cc-audit.md`.
   (If cc-audit.md is not in the repo .claude/commands/, skip without error.)

---

## STEP 4 — INITIALIZE LOG FILES

For each log file: check if it exists first. If it already exists, DO NOT overwrite it — existing log data must be preserved.

1. If `[REPO_PATH]/logs/archive-log.json` does not exist:
   Write it with content: `{"schema_version": 1, "runs": []}`

2. If `[REPO_PATH]/logs/global-state-log.json` does not exist:
   Write it with content: `{"schema_version": 1, "runs": []}`

---

## STEP 5 — CREATE ARCHIVE PLACEHOLDER

If `[REPO_PATH]/archives/.gitkeep` does not exist:
  Write it with empty content.

---

## STEP 6 — VERIFY INSTALLATION

Verify each of the following files exists and is non-empty. If any file is missing, report it explicitly.

**Global commands (in [GLOBAL_CLAUDE_PATH]/commands/):**
- [ ] 01-intake.md
- [ ] 02-extract-profile.md
- [ ] 03-build-prompt.md
- [ ] 04-refinep.md
- [ ] 05-split-orc-v2.md
- [ ] 06-seq-init.md

**Log files:**
- [ ] [REPO_PATH]/logs/archive-log.json
- [ ] [REPO_PATH]/logs/global-state-log.json

**Archive directory:**
- [ ] [REPO_PATH]/archives/ (directory exists)

---

## STEP 7 — CONFIRM SUCCESS

Output exactly:

```
CC-WORKFLOW INSTALLATION COMPLETE
==================================
Repo path:          [REPO_PATH]
Global .claude:     [GLOBAL_CLAUDE_PATH]

Installed commands:
  [GLOBAL_CLAUDE_PATH]/commands/01-intake.md
  [GLOBAL_CLAUDE_PATH]/commands/02-extract-profile.md
  [GLOBAL_CLAUDE_PATH]/commands/03-build-prompt.md
  [GLOBAL_CLAUDE_PATH]/commands/04-refinep.md
  [GLOBAL_CLAUDE_PATH]/commands/05-split-orc-v2.md
  [GLOBAL_CLAUDE_PATH]/commands/06-seq-init.md

Global entry point: /cc-workflow [available if cc-workflow.md was installed]
Global audit:       /cc-audit    [available if cc-audit.md was installed]

Log files initialized:
  [REPO_PATH]/logs/archive-log.json
  [REPO_PATH]/logs/global-state-log.json

You're ready. To use the system:
  1. Open any Claude Code project
  2. Run: /cc-workflow [your brain dump, file path, or idea]
  3. When complete, run the generated /N-magic command in a fresh chat
  4. To iterate the system on itself: run /cc-audit
```
