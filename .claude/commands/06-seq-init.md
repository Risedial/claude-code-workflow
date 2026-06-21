---
name: seq-init
description: "Step 6 of the cc-workflow pipeline. Receives a file path to step-05-batches.json (written by split-orc-v2 subagent). Reads the splits folder path referenced within. Scans the folder, writes seq-state.json, and generates a dedicated no-argument runner command (saved as 111111111111.md in the project .claude/commands/). Writes step-06-runner.json as output signal to parent. All inter-step data travels via files on disk."
argument-hint: "[path to step-05-batches.json produced by /split-orc-v2 subagent]"
allowed-tools: Read, Write, Glob
---

# SEQ-INIT — Sequence Bootstrapper (cc-workflow pipeline subagent)

You are the Sequence Bootstrapper running as a subagent in the cc-workflow pipeline. Your job: read the splits folder path from the step-05 output JSON, scan the folder, initialize a state file, and write a dedicated runner command to disk. You do not execute any prompts.

---

## STEP 0 — FILE-BASED INPUT

Read the file at $ARGUMENTS. Parse as JSON.
Extract `splits_folder` — absolute path to the splits output folder.
This is your FOLDER_PATH. All subsequent steps use this path.

---

## STEP 1 — LOCATE PROJECT ROOT

Identify the current working directory (the directory where Claude Code is running — this is the project root).

Check that a `.claude/commands/` directory exists within it. If `.claude/` exists but `commands/` does not, create `commands/`. If neither exists, stop with:
```
ERROR: No .claude/ directory found in the current working directory.
This command must be run from a Claude Code project root.
```

Store the resolved absolute path to `.claude/commands/` as `[COMMANDS_DIR]`.

---

## STEP 2 — DISCOVER PROMPT FILES

**Priority 1 — Explicit manifest:**
Check if `[FOLDER_PATH]/_sequence.txt` exists. If yes, read it. Each non-blank line is a path to a prompt file. This defines execution order. Proceed to Step 3.

**Priority 2 — Auto-discover numbered files:**
List all `.md` files directly inside `[FOLDER_PATH]/` (non-recursive) whose filename begins with one or more digits. Sort alphabetically. These are the prompts in order. Proceed to Step 3.

**If no prompt files found:**
```
ERROR: No prompt files found in [FOLDER_PATH]
Expected: files named 01-*.md, 02-*.md, etc. — or a _sequence.txt manifest
```
Stop.

---

## STEP 3 — DERIVE SEQUENCE NAME

Take the last path segment of FOLDER_PATH. Lowercase it. Replace spaces, underscores, and dots with hyphens. Strip trailing separators. This is `[SEQUENCE_NAME]`.

---

## STEP 4 — BUILD STEP REGISTRY

For each discovered prompt file (in order), assign:
- Index N: 1-based, zero-padded to 2 digits
- Step ID: `step-[NN]-[full filename without extension]`
- Record absolute file path

---

## STEP 5 — WRITE seq-state.json

Write `[FOLDER_PATH]/seq-state.json`:
```json
{
  "version": "1.0.0",
  "sequenceFolder": "[absolute resolved path of FOLDER_PATH]",
  "sequenceName": "[SEQUENCE_NAME]",
  "runnerCommand": "/111111111111",
  "totalSteps": [N],
  "completedSteps": [],
  "pendingSteps": ["step-01-...", "step-02-..."],
  "stepFiles": {
    "step-01-...": "[absolute path to first prompt file]",
    "step-02-...": "[absolute path to second prompt file]"
  },
  "artifacts": {
    "filesWritten": []
  },
  "startedAt": "[current ISO 8601 timestamp]",
  "lastRunAt": null
}
```

---

## STEP 6 — WRITE THE RUNNER COMMAND

Write the following file to `[COMMANDS_DIR]/111111111111.md`.

Substitute every `[SEQUENCE_NAME]` with the actual derived name and every `[FOLDER_PATH]` with the absolute resolved path. Write literal values — no template placeholders in the output file.

```
---
name: 111111111111
description: Runner for "[SEQUENCE_NAME]" sequence. Reads seq-state.json, executes the next pending prompt, updates state. No arguments needed. One step per fresh chat.
allowed-tools: Read, Write, Glob
---

# SEQUENTIAL RUNNER — [SEQUENCE_NAME]

You are the Sequential Runner for the "[SEQUENCE_NAME]" sequence. Your only job is to execute ONE prompt, update the state file, and stop. One step per chat session. No exceptions.

Sequence folder: [FOLDER_PATH]
State file: [FOLDER_PATH]/seq-state.json

## STEP 1 — READ STATE
Read `[FOLDER_PATH]/seq-state.json` fully.
If the file does not exist: ERROR: seq-state.json not found. Resolution: Run /06-seq-init [FOLDER_PATH] to reinitialize. Stop.

## STEP 2 — CHECK COMPLETION
If `pendingSteps` is empty, display:
  SEQUENCE COMPLETE
  Sequence: [SEQUENCE_NAME]
  All steps completed.
  Completed: [list completedSteps]
Then stop.

## STEP 3 — IDENTIFY NEXT STEP
Take the FIRST item from `pendingSteps`. This is `currentStepId`.
If `currentStepId` is also in `completedSteps`:
  ERROR: [currentStepId] appears in both arrays.
  Resolution: Manually edit seq-state.json and remove it from one array. Stop.
Look up `stepFiles[currentStepId]` — absolute path to the prompt file. Read that file fully before proceeding.
Display:
  EXECUTING
  Step: [currentStepId]
  Progress: [completedSteps.length + 1] of [totalSteps]
  File: [stepFiles[currentStepId]]
  Remaining after this: [pendingSteps.length - 1]

## STEP 4 — EXECUTE
Execute the prompt file contents exactly as written. Treat it as a complete, self-contained instruction. Do not summarize, paraphrase, or skip sections. Execute every instruction in order.

## STEP 5 — UPDATE STATE
After the prompt completes, mutate `[FOLDER_PATH]/seq-state.json`:
1. Remove `currentStepId` from `pendingSteps`
2. Append `currentStepId` to `completedSteps`
3. Set `lastRunAt` to current ISO 8601 timestamp
4. Append any output files produced to `artifacts.filesWritten`
Write the updated JSON back. This write is mandatory before reporting done.

## STEP 6 — REPORT
Display:
  STEP COMPLETE
  Completed: [currentStepId]
  Progress: [completedSteps.length] of [totalSteps] done
If more steps remain:
  Next: [first item in updated pendingSteps]
  Clear this chat, open a fresh one, and paste: /111111111111
If this was the last step:
  SEQUENCE COMPLETE — All [totalSteps] prompts executed.

## HARD CONSTRAINTS
1. Execute exactly ONE step per session.
2. Write seq-state.json before reporting done.
3. Read the prompt file fully before executing.
4. If the step is already in completedSteps, do not re-execute.
5. Zero silent failures.
```

---

## STEP 7 — WRITE step-06 OUTPUT JSON

Write step-06-runner.json in the same directory as the input step-05-batches.json:
```json
{
  "step": "06-seq-init",
  "sequence_name": "[SEQUENCE_NAME]",
  "seq_state_file": "[absolute path to FOLDER_PATH/seq-state.json]",
  "runner_file": "[absolute path to COMMANDS_DIR/111111111111.md]",
  "total_steps": [N],
  "completed_at": "[ISO8601 timestamp]"
}
```

---

## STEP 8 — REPORT

Display:
```
INITIALIZED
===========
Sequence: [SEQUENCE_NAME]
Folder: [FOLDER_PATH]
Steps: [N]

Step list:
  [for each step: step-ID → filename]

Files written:
  [FOLDER_PATH]/seq-state.json
  [COMMANDS_DIR]/111111111111.md
  step-06-runner.json

Your runner command is ready. The parent agent will record this output.
To execute manually: paste /111111111111 into a fresh chat.
```