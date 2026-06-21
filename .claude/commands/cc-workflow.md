---
name: cc-workflow
description: "Global entry point command for the cc-workflow-subagent-magic system. Accepts any brain dump (file path, tag, or raw text) as argument. Spawns 6 subagents sequentially: intake → extract-profile → build-prompt → refinep → split-orc-v2 → seq-init. All inter-step data travels via files on disk — never via parent context. Produces a numbered N-magic.md project command on completion. Accessible from any Claude Code workspace."
argument-hint: "[file path, tag, or raw brain dump text]"
allowed-tools: Agent, Read, Write, Glob
---

# CC-WORKFLOW — Global Entry Point

You are the parent orchestrator for the cc-workflow-subagent-magic system. Your ONLY job is deterministic routing and delegation. You do NOT perform domain reasoning, write subagent instructions from scratch, or process output content. You spawn subagents in sequence, pass file paths between them, and generate the final magic command.

---

## PHASE 0 — SETUP

**Resolve working directory:**
The working directory is where Claude Code is currently running. All output files will be written here. Store as WORK_DIR.

**Resolve command repository path:**
Locate the cc-workflow command files. Check these paths in order:
1. Look for the directory containing this command file — its parent chain likely includes the repo.
2. Check environment: look for a directory named `claude-code-workflow` under common locations (Documents, home directory).
3. The repo path is the directory containing `.claude/commands/01-intake.md`.
Store as REPO_PATH.

**Resolve global .claude path:**
Detect the global .claude folder dynamically at runtime. Check in order:
1. `%APPDATA%\Claude\` (Windows)
2. `%USERPROFILE%\.claude\` (Windows fallback)
3. `~/.claude/` (macOS/Linux)
The correct path is the one that contains a `commands/` subfolder with this file (cc-workflow.md) in it.
Store as GLOBAL_CLAUDE_PATH.

**Parse brain dump input:**
BRAIN_DUMP = $ARGUMENTS
If BRAIN_DUMP is empty: spawn a subagent to ask the user for input using AskUserQuestion, then proceed.

**Initialize working files:**
All step output JSON files will be written to WORK_DIR by the subagents.
Clarification mode file: WORK_DIR/step-00-clarification-mode.json (written by step 01 subagent).

---

## PHASE 1 — SPAWN SUBAGENT 01: INTAKE

Spawn a subagent using the Agent tool:
- instruction: "Execute the intake command at [REPO_PATH]/.claude/commands/01-intake.md with this brain dump as input: [BRAIN_DUMP]. Write all output files to [WORK_DIR]. Write step-01-intake.json to [WORK_DIR]/step-01-intake.json when complete."
- The subagent will ask the user the clarification mode question (the ONLY interactive pause allowed in this workflow).
- When the subagent completes, read [WORK_DIR]/step-01-intake.json.
- Extract and record: slug, intake_file, clarification_mode_file, clarification_mode, completed_at.
- Do NOT read the full intake document — work only from the JSON summary.

**Error guard:** If step-01-intake.json does not exist after the subagent reports complete, output:
`ERROR: Step 01 subagent did not write step-01-intake.json. Cannot proceed to step 02.`
Stop.

---

## PHASE 2 — SPAWN SUBAGENT 02: EXTRACT-PROFILE

Spawn a subagent using the Agent tool:
- instruction: "Execute the extract-profile command at [REPO_PATH]/.claude/commands/02-extract-profile.md with this input file path: [WORK_DIR]/step-01-intake.json. Write all output files to the same directory as the intake file. Write step-02-profile.json when complete."
- When the subagent completes, read [WORK_DIR]/step-02-profile.json.
- Extract and record: slug, profile_file, vision_file, confidence_score, completed_at.
- Do NOT read the full profile JSON or vision document.

**Error guard:** If step-02-profile.json does not exist after completion, output:
`ERROR: Step 02 subagent did not write step-02-profile.json. Cannot proceed to step 03.`
Stop.

---

## PHASE 3 — SPAWN SUBAGENT 03: BUILD-PROMPT

Spawn a subagent using the Agent tool:
- instruction: "Execute the build-prompt command at [REPO_PATH]/.claude/commands/03-build-prompt.md with this input file path: [WORK_DIR]/step-02-profile.json. Write all output files to the same directory. Write step-03-prompt.json when complete."
- When the subagent completes, read [WORK_DIR]/step-03-prompt.json.
- Extract and record: slug, prompt_file, completed_at.

**Error guard:** If step-03-prompt.json does not exist, output:
`ERROR: Step 03 subagent did not write step-03-prompt.json. Cannot proceed to step 04.`
Stop.

---

## PHASE 4 — SPAWN SUBAGENT 04: REFINEP

Spawn a subagent using the Agent tool:
- instruction: "Execute the refinep command at [REPO_PATH]/.claude/commands/04-refinep.md with this input file path: [WORK_DIR]/step-03-prompt.json. Write all output files to the same directory. Write step-04-refined.json when complete."
- When the subagent completes, read [WORK_DIR]/step-04-refined.json.
- Extract and record: refined_file, completed_at.

**Error guard:** If step-04-refined.json does not exist, output:
`ERROR: Step 04 subagent did not write step-04-refined.json. Cannot proceed to step 05.`
Stop.

---

## PHASE 5 — SPAWN SUBAGENT 05: SPLIT-ORC-V2

Spawn a subagent using the Agent tool:
- instruction: "Execute the split-orc-v2 command at [REPO_PATH]/.claude/commands/05-split-orc-v2.md with this input file path: [WORK_DIR]/step-04-refined.json. Write all output files to disk. Write step-05-batches.json when complete."
- When the subagent completes, read [WORK_DIR]/step-05-batches.json.
- Extract and record: prompt_type, splits_folder, batch_count, completed_at.

**Error guard:** If step-05-batches.json does not exist, output:
`ERROR: Step 05 subagent did not write step-05-batches.json. Cannot proceed to step 06.`
Stop.

---

## PHASE 6 — SPAWN SUBAGENT 06: SEQ-INIT

Spawn a subagent using the Agent tool:
- instruction: "Execute the seq-init command at [REPO_PATH]/.claude/commands/06-seq-init.md with this input file path: [WORK_DIR]/step-05-batches.json. Write seq-state.json and the 111111111111.md runner to the project .claude/commands/ directory. Write step-06-runner.json when complete."
- When the subagent completes, read [WORK_DIR]/step-06-runner.json.
- Extract and record: sequence_name, seq_state_file, runner_file, total_steps, completed_at.

**Error guard:** If step-06-runner.json does not exist, output:
`ERROR: Step 06 subagent did not write step-06-runner.json. Cannot generate magic command.`
Stop.

---

## PHASE 7 — GENERATE MAGIC COMMAND

**Enumerate existing magic commands:**
Use Glob to list all files matching `[WORK_DIR]/.claude/commands/*-magic.md`.
Also check `[WORK_DIR]/.claude/commands/` for any file matching `[0-9]*-magic.md`.
Find the highest existing integer N. If no magic files exist, N = 0.
Set MAGIC_N = N + 1.

**Verify no collision:**
Confirm `[WORK_DIR]/.claude/commands/[MAGIC_N]-magic.md` does not exist. If it does, increment MAGIC_N until a non-colliding integer is found.

**Create .claude/commands/ if missing:**
If `[WORK_DIR]/.claude/commands/` does not exist, create it.

**Write magic command file:**
Write `[WORK_DIR]/.claude/commands/[MAGIC_N]-magic.md` with this content (substituting all values):

```markdown
---
name: [MAGIC_N]-magic
description: "Magic command [MAGIC_N] for the [SEQUENCE_NAME] sequence. Parent-agent-plus-subagents architecture. Executes the 111111111111.md runner via subagents in a fresh chat, advancing one step per subagent. Requires seq-state.json and 111111111111.md to exist — generated by /cc-workflow."
allowed-tools: Agent, Read, Write
---

# MAGIC COMMAND [MAGIC_N] — [SEQUENCE_NAME]

You are the parent orchestrator for magic command execution. Your ONLY job is to drive the 111111111111.md runner via sequentially spawned subagents, one step at a time, until all steps in seq-state.json are complete.

---

## PHASE 0 — PREREQUISITE CHECK

Before spawning any subagent, verify both files exist:
- seq-state.json in the current working directory
- .claude/commands/111111111111.md in the current working directory

To check: use the Read tool to attempt reading both files.

If seq-state.json is missing:
  HALT. Output: BLOCKED: seq-state.json not found in current directory. Run /cc-workflow first to generate the runner and state file. Do not proceed.

If .claude/commands/111111111111.md is missing:
  HALT. Output: BLOCKED: 111111111111.md runner not found in .claude/commands/. Run /cc-workflow first to generate the runner. Do not proceed.

Read seq-state.json. Extract:
- totalSteps
- completedSteps (array)
- pendingSteps (array)
- sequenceName

If pendingSteps is empty:
  Output: SEQUENCE ALREADY COMPLETE. All [totalSteps] steps of [sequenceName] are done. Nothing to execute.
  Stop.

---

## PHASE 1 — EXECUTE RUNNER STEPS VIA SUBAGENTS

For each pending step (iterate through pendingSteps in order):

1. Re-read seq-state.json to get current state before each spawn.
2. If pendingSteps is now empty: stop — sequence complete.
3. Spawn a subagent using the Agent tool:
   - instruction: "Execute the /111111111111 runner command located at .claude/commands/111111111111.md in the current working directory. Read seq-state.json to determine the current step. Execute exactly ONE step. Update seq-state.json when done. Report the step ID completed and any files written."
4. After the subagent reports complete:
   - Read seq-state.json again.
   - Confirm the step that was pending is now in completedSteps.
   - If not confirmed: output WARNING: subagent did not update seq-state.json for step [ID]. Continuing to next spawn but state may be inconsistent.
5. Record the completed step ID and any artifacts.

Repeat until pendingSteps is empty.

---

## PHASE 2 — COMPLETION REPORT

Display:
```
MAGIC COMMAND [MAGIC_N] COMPLETE
================================
Sequence: [SEQUENCE_NAME]
Total steps executed: [N]
All pending steps: [list of completed step IDs]

Artifacts produced (from seq-state.json artifacts.filesWritten):
[list each file]

seq-state.json: [path]
```
```

---

**Verify the magic command file was written:**
Read `[WORK_DIR]/.claude/commands/[MAGIC_N]-magic.md`. Confirm it exists and is non-empty.

---

## PHASE 8 — FINAL REPORT

Display to user:
```
CC-WORKFLOW COMPLETE
====================
Brain dump processed: [first 100 chars of BRAIN_DUMP]...
Slug: [slug]
Clarification mode: [clarification_mode]

Pipeline steps completed:
  01 - Intake:          [WORK_DIR]/[slug]-intake.md
  02 - Profile:         [WORK_DIR]/[slug]-profile.json
  03 - Prompt:          [WORK_DIR]/exec-[slug].md
  04 - Refined prompt:  [WORK_DIR]/[refined filename]
  05 - Splits:          [splits_folder] ([batch_count] batches, [prompt_type])
  06 - Runner:          [runner_file]

Magic command created: [WORK_DIR]/.claude/commands/[MAGIC_N]-magic.md

To execute the workflow:
  Open a fresh Claude Code chat in this project directory.
  Paste: /[MAGIC_N]-magic
```
