---
name: seq-init
description: One-time bootstrap. Pass a folder path containing numbered prompt files. Scans the folder, writes seq-state.json, and generates a dedicated no-argument runner command (saved as 111111111111.md in the project's .claude/commands/). After this runs, paste /111111111111 into fresh chats — no folder path ever needed again.
argument-hint: "[absolute folder path containing prompt sequence]"
allowed-tools: Read, Write, Glob
---

# SEQ-INIT — Sequence Bootstrapper

You are the Sequence Bootstrapper. Your only job is to scan a folder, initialize a state file, and write a dedicated runner command to disk. You do not execute any prompts. After you finish, the user has a single no-argument command they paste into fresh chats to advance the sequence one step at a time.

---

## ARGUMENT

Folder path: `$ARGUMENTS`

---

## STEP 0 — LOCATE PROJECT ROOT

Identify the current working directory (the directory where Claude Code is running — this is the project root).

Check that a `.claude/commands/` directory exists within it. If `.claude/` exists but `commands/` does not, create `commands/`. If neither exists, stop with:

```
ERROR: No .claude/ directory found in the current working directory.
This command must be run from a Claude Code project root.
```

Store the resolved absolute path to `.claude/commands/` as `[COMMANDS_DIR]`.

---

## STEP 1 — DISCOVER PROMPT FILES

**Priority 1 — Explicit manifest:**
Check if `$ARGUMENTS/_sequence.txt` exists. If yes, read it. Each non-blank line is a path to a prompt file (absolute or relative to `$ARGUMENTS/`). This defines execution order. Proceed to Step 2.

**Priority 2 — Auto-discover numbered files:**
List all `.md` files directly inside `$ARGUMENTS/` (non-recursive) whose filename begins with one or more digits. Sort alphabetically. These are the prompts in order. Proceed to Step 2.

**If no prompt files found under either rule:**
```
ERROR: No prompt files found in $ARGUMENTS
Expected: files named 01-*.md, 02-*.md, etc. — or a _sequence.txt manifest
```
Stop.

---

## STEP 2 — DERIVE SEQUENCE NAME

Take the last path segment of `$ARGUMENTS`. Lowercase it. Replace spaces, underscores, and dots with hyphens. Strip any trailing separators. This is `[SEQUENCE_NAME]`.

Examples:
- `C:\Users\Alex\brand-research` → `brand-research`
- `C:\Users\Alex\My Sequence` → `my-sequence`
- `C:\Users\Alex\icp_deep_dive` → `icp-deep-dive`

---

## STEP 3 — BUILD STEP REGISTRY

For each discovered prompt file (in order), assign:
- Index N: 1-based, zero-padded to 2 digits
- Step ID: `step-[NN]-[full filename without extension]`
  - Example: `01-research-icp.md` → `step-01-research-icp`
  - Example: `03_write-hooks.md` → `step-03-write-hooks`

Also record for each step its absolute file path.

---

## STEP 4 — WRITE seq-state.json

Write `$ARGUMENTS/seq-state.json` with this structure (substitute all actual values — no placeholders):

```json
{
  "version": "1.0.0",
  "sequenceFolder": "<absolute resolved path of $ARGUMENTS>",
  "sequenceName": "<SEQUENCE_NAME>",
  "runnerCommand": "/111111111111",
  "totalSteps": <N>,
  "completedSteps": [],
  "pendingSteps": ["step-01-...", "step-02-...", "..."],
  "stepFiles": {
    "step-01-...": "<absolute path to first prompt file>",
    "step-02-...": "<absolute path to second prompt file>"
  },
  "artifacts": {
    "filesWritten": []
  },
  "startedAt": "<current ISO 8601 timestamp>",
  "lastRunAt": null
}
```

---

## STEP 5 — WRITE THE RUNNER COMMAND

Write the following file to `[COMMANDS_DIR]/111111111111.md`.

Substitute every occurrence of `[SEQUENCE_NAME]` with the actual derived name and every occurrence of `[FOLDER_PATH]` with the absolute resolved path of `$ARGUMENTS`. Write literal values — no template placeholders in the output file.

--- BEGIN RUNNER TEMPLATE ---

```
---
name: 111111111111
description: Runner for "[SEQUENCE_NAME]" sequence. Reads seq-state.json, executes the next pending prompt, updates state. No arguments needed. One step per fresh chat.
allowed-tools: Read, Write, Glob
---

# SEQUENTIAL RUNNER — [SEQUENCE_NAME]

You are the Sequential Runner for the "[SEQUENCE_NAME]" sequence. Your only job is to execute ONE prompt, update the state file, and stop. One step per chat session. No exceptions.

Sequence folder: [FOLDER_PATH]
State file: [FOLDER_PATH]\seq-state.json

---

## STEP 1 — READ STATE

Read `[FOLDER_PATH]\seq-state.json` fully.

If the file does not exist:
  ERROR: seq-state.json not found
  Resolution: Run /seq-init [FOLDER_PATH] to reinitialize.
  Stop.

---

## STEP 2 — CHECK COMPLETION

If `pendingSteps` is empty, display:

  SEQUENCE COMPLETE
  =================
  Sequence: [SEQUENCE_NAME]
  All steps completed.

  Completed: [list completedSteps]

Then stop.

---

## STEP 3 — IDENTIFY NEXT STEP

Take the FIRST item from `pendingSteps`. This is `currentStepId`.

If `currentStepId` is also in `completedSteps`:
  ERROR: [currentStepId] appears in both pendingSteps and completedSteps.
  Resolution: Manually edit [FOLDER_PATH]\seq-state.json and remove it from one array.
  Stop.

Look up `stepFiles[currentStepId]` — this is the absolute path to the prompt file.
Read that file fully before proceeding.

Display:
  EXECUTING
  =========
  Step:     [currentStepId]
  Progress: [completedSteps.length + 1] of [totalSteps]
  File:     [stepFiles[currentStepId]]
  Remaining after this: [pendingSteps.length - 1]

---

## STEP 4 — EXECUTE

Execute the prompt file contents exactly as written. Treat it as a complete, self-contained instruction. Do not summarize, paraphrase, or skip sections. Execute every instruction in the file in the order given. If the prompt references other files, read them as directed.

---

## STEP 5 — UPDATE STATE

After the prompt completes (including any verification it requires), mutate `[FOLDER_PATH]\seq-state.json`:

1. Remove `currentStepId` from `pendingSteps`
2. Append `currentStepId` to `completedSteps`
3. Set `lastRunAt` to current ISO 8601 timestamp
4. Append any output files produced to `artifacts.filesWritten`

Write the updated JSON back to `[FOLDER_PATH]\seq-state.json`. This write is mandatory before reporting done.

---

## STEP 6 — REPORT

Display:
  STEP COMPLETE
  =============
  Completed: [currentStepId]
  Progress:  [completedSteps.length] of [totalSteps] done

If more steps remain:
  Next: [first item in updated pendingSteps]

  Clear this chat, open a fresh one, and paste:
    /111111111111

If this was the last step:
  SEQUENCE COMPLETE — All [totalSteps] prompts executed.

---

## HARD CONSTRAINTS

1. Execute exactly ONE step per session. Never advance beyond the current step.
2. Write seq-state.json before reporting done. If the write fails, report it explicitly.
3. Read the prompt file fully before executing. Never execute from memory.
4. If the step is already in completedSteps, do not re-execute — report inconsistency and stop.
5. Zero silent failures. Surface every missing file, bad path, or schema error with a named error and resolution instruction.
```

--- END RUNNER TEMPLATE ---

---

## STEP 6 — REPORT

Display:
```
INITIALIZED
===========
Sequence: [SEQUENCE_NAME]
Folder:   $ARGUMENTS
Steps:    [N]

Step list:
  [for each step: step-ID  →  filename]

Files written:
  $ARGUMENTS/seq-state.json
  [COMMANDS_DIR]/111111111111.md

Your runner command is ready. Paste this into a fresh chat to begin:

  /111111111111

Paste the same command into each subsequent fresh chat to advance one step at a time.
To reset: delete $ARGUMENTS/seq-state.json and run /06-seq-init $ARGUMENTS again.
```
