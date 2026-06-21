---
name: cc-audit
description: "Global self-iteration audit command for the cc-workflow-subagent-magic system. Run in a fresh chat when you want the system to audit itself and improve. Presents all project-level N-magic.md commands via AskUserQuestion multiple choice. Audits global-state-log.json execution history. Produces a braindump file. Automatically re-enters /cc-workflow with the braindump as input. Gracefully handles first run when no state log exists."
allowed-tools: AskUserQuestion, Read, Write, Glob, Agent
---

# CC-AUDIT — Self-Iteration Audit Command

You are the self-iteration audit orchestrator for the cc-workflow-subagent-magic system. Your job: enumerate magic commands, audit history, produce a braindump, and feed it back into the entry-point workflow to generate a self-improvement command.

---

## PHASE 0 — RESOLVE PATHS

**Resolve repo path dynamically:**
The cc-workflow-subagent-magic repo contains this audit command system. Locate the repo by checking common paths:
1. Look for `claude-code-workflow` directory under `%USERPROFILE%\Documents\` (Windows)
2. Look for `claude-code-workflow` directory under `%USERPROFILE%\` (home)
3. Look for `claude-code-workflow` directory under `~/Documents/` (macOS/Linux)
4. Look for any directory containing `logs/global-state-log.json` or `logs/archive-log.json`
Store as REPO_PATH.

**Resolve working project path:**
The current working directory is the project the user is auditing. Store as WORK_DIR.

**Resolve global .claude path:**
Detect dynamically:
1. `%APPDATA%\Claude\` (Windows) — check if commands/ subfolder exists here
2. `%USERPROFILE%\.claude\` (Windows fallback) — check if commands/ exists
3. `~/.claude/` (macOS/Linux)
Store the path that contains this cc-audit.md file as GLOBAL_CLAUDE_PATH.

**State log paths:**
STATE_LOG = [REPO_PATH]/logs/global-state-log.json
ARCHIVE_LOG = [REPO_PATH]/logs/archive-log.json

---

## PHASE 1 — ENUMERATE MAGIC COMMANDS

Use Glob to list all files matching `[WORK_DIR]/.claude/commands/*-magic.md`.
For each file found, extract the N from the filename (e.g., `1-magic.md` → N=1).

If no magic command files are found:
  Output: No magic commands found in [WORK_DIR]/.claude/commands/. Run /cc-workflow first to generate a magic command for this project.
  Stop.

Build a list of magic command options for the multiple-choice question.

---

## PHASE 2 — SELECT MAGIC COMMAND

Ask the user using AskUserQuestion:
- question: "Which magic command would you like to audit?"
- header: "Select Magic Command"
- options: [one option per magic file found, label = filename, description = "Audit [N]-magic.md and generate a self-improvement workflow for it"]
- multiSelect: false

Store the selected magic command filename as SELECTED_MAGIC.
Store its full path as SELECTED_MAGIC_PATH.

---

## PHASE 3 — READ EXECUTION HISTORY

**Check state log existence:**
Attempt to read STATE_LOG.

If STATE_LOG does not exist (first run):
  FIRST_RUN = true
  Initialize STATE_LOG with baseline entry:
  ```json
  {
    "schema_version": 1,
    "runs": [
      {
        "run_id": 1,
        "run_type": "baseline",
        "magic_command": "[SELECTED_MAGIC]",
        "timestamp": "[ISO8601 current timestamp]",
        "prior_run_id": null,
        "changes_applied": [],
        "outcome": "baseline — no prior history to compare",
        "notes": "First audit run. This entry establishes the baseline state. No changes will be proposed on the first run — the system records starting state only."
      }
    ]
  }
  ```
  Write this JSON to STATE_LOG.
  Output: First audit run detected. Initializing global-state-log.json with baseline entry. No changes will be proposed — this run records starting state only.

If STATE_LOG exists (subsequent runs):
  FIRST_RUN = false
  Read STATE_LOG. Parse runs array.
  Filter runs where `magic_command` matches SELECTED_MAGIC.
  Store as RELEVANT_RUNS.

---

## PHASE 4 — PRODUCE BRAINDUMP

BRAINDUMP_FILE = [REPO_PATH]/braindump-audit-[timestamp].md

Write BRAINDUMP_FILE with these sections:

```markdown
# CC-Workflow Self-Iteration Audit Braindump

> Generated: [ISO8601 timestamp]
> Magic command audited: [SELECTED_MAGIC]
> Audit type: ["First run — baseline" if FIRST_RUN else "Iteration run"]

---

## What We Are Auditing

This braindump is a self-improvement input for the cc-workflow-subagent-magic system. The goal is to analyze the system's own behavior and propose targeted improvements.

## Selected Magic Command

- File: [SELECTED_MAGIC_PATH]
- Command: /[SELECTED_MAGIC without .md extension]

## Current System State

[If FIRST_RUN:]
This is the first audit run. No execution history exists to compare against. The system has been installed and is in its initial state. This braindump records the starting configuration.

Proposed focus for initial self-improvement:
- Review all 6 pipeline commands (01-intake through 06-seq-init) for any gaps in the file-based I/O protocol
- Verify that the clarification mode gate is correctly propagated to all subagent steps
- Verify that the magic command prerequisite checks are in place
- Review the cc-workflow entry point for any hardcoded path references
- Assess whether the archive system is initialized correctly

[If NOT FIRST_RUN:]
## Execution History for [SELECTED_MAGIC]

[For each entry in RELEVANT_RUNS, list:]
- Run ID: [run_id]
- Timestamp: [timestamp]
- Changes applied: [changes_applied]
- Outcome: [outcome]
- Notes: [notes]

## Pattern Analysis

[Analyze RELEVANT_RUNS for patterns:]
- Which changes improved outcomes?
- Which changes introduced regressions?
- What remains unchanged across multiple runs (stable = valuable)?
- What changed frequently (unstable = may need redesign)?

## Proposed Improvement Areas

[Based on history analysis, list specific areas to improve:]
[If no clear pattern yet: focus on the initial setup quality and any known edge cases from the vision document]

## System Files to Review

- [REPO_PATH]/.claude/commands/01-intake.md
- [REPO_PATH]/.claude/commands/02-extract-profile.md
- [REPO_PATH]/.claude/commands/03-build-prompt.md
- [REPO_PATH]/.claude/commands/04-refinep.md
- [REPO_PATH]/.claude/commands/05-split-orc-v2.md
- [REPO_PATH]/.claude/commands/06-seq-init.md
- [GLOBAL_CLAUDE_PATH]/commands/cc-workflow.md
- [GLOBAL_CLAUDE_PATH]/commands/cc-audit.md
- [SELECTED_MAGIC_PATH]

## Constraints for Self-Improvement

- Do not introduce external dependencies
- Do not break any existing behavioral output of the 6 pipeline commands
- Archive every file before modifying it — write to [REPO_PATH]/archives/ first
- Update archive-log.json after each file is archived
- Append to global-state-log.json after all changes are applied
- Clarification mode gate must remain intact across all subagent steps
- All paths must remain dynamically resolved — no hardcoded absolute paths
```

---

## PHASE 5 — RE-ENTER ENTRY POINT

If FIRST_RUN = true:
  Display:
  ```
  CC-AUDIT COMPLETE (First Run — Baseline)
  ==========================================
  State log initialized: [STATE_LOG]
  Braindump written: [BRAINDUMP_FILE]

  This was the first audit run. The global-state-log.json has been initialized with the baseline state.
  No changes will be applied on the first run.

  To run a full self-improvement cycle, run cc-audit again after using the system on at least one project.
  The system will then have history to compare against.
  ```
  Stop.

If FIRST_RUN = false:
  Spawn a subagent using the Agent tool:
  - instruction: "Execute the cc-workflow global entry point command. Use the file at [BRAINDUMP_FILE] as the brain dump input (pass it as 'file:[BRAINDUMP_FILE]'). This is a self-improvement audit run — the workflow will analyze the cc-workflow-subagent-magic system itself and produce improvements."

  After the subagent completes, append a new entry to STATE_LOG runs array:
  ```json
  {
    "run_id": [next integer],
    "run_type": "self_iteration",
    "magic_command": "[SELECTED_MAGIC]",
    "timestamp": "[ISO8601 current timestamp]",
    "prior_run_id": [previous run_id],
    "changes_applied": ["self-improvement workflow generated via cc-workflow"],
    "outcome": "pending — review new magic command output",
    "notes": "Braindump: [BRAINDUMP_FILE]"
  }
  ```

  Display:
  ```
  CC-AUDIT COMPLETE
  =================
  Braindump: [BRAINDUMP_FILE]
  Re-entered /cc-workflow with braindump as input.
  State log updated: [STATE_LOG]

  The cc-workflow pipeline ran on the system itself.
  A new magic command has been generated for the self-improvement sequence.
  Review the new magic command output in the cc-workflow completion report above.
  ```
