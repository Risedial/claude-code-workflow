# claude-code-workflow

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Requires Claude Code](https://img.shields.io/badge/Requires-Claude%20Code-blueviolet)](https://claude.ai/code)
[![Workflow Version](https://img.shields.io/badge/Workflow-v3.0-blue)](./HOW-TO-USE-CLAUDE-CODE.md)

**Turn any brain dump into a fully executed workflow with exactly two commands â€” in a self-improving system that never needs external dependencies.**

This repo transforms a 6-step Claude Code workflow pipeline into a fully automated, two-command experience. A parent agent spawns 6 subagents in deterministic sequence, passes each subagent's structured output to the next via files on disk, and completes the entire workflow in a single chat session â€” without the user touching any intermediate step.

---

## What the System Does

The cc-workflow-subagent-magic system has three interlocking components:

### 1. Entry Point Command (`/cc-workflow`)

A global Claude Code command accessible from any workspace. You invoke it with any brain dump â€” a file path, a tag, or raw text. The parent agent spawns 6 subagents in sequence:

```
brain dump
    â”‚
    â–¼
[01] intake â†’ step-01-intake.json
    â”‚
    â–¼
[02] extract-profile â†’ step-02-profile.json
    â”‚                   [slug]-profile.json
    â”‚                   [slug]-vision.md
    â–¼
[03] build-prompt â†’ step-03-prompt.json
    â”‚               exec-[slug].md
    â–¼
[04] refinep â†’ step-04-refined.json
    â”‚           [subject]_refined.md
    â–¼
[05] split-orc-v2 â†’ step-05-batches.json
    â”‚               splits/[folder]/ (numbered batch files)
    â–¼
[06] seq-init â†’ step-06-runner.json
                seq-state.json
                .claude/commands/111111111111.md
    â”‚
    â–¼
N-magic.md (written to project .claude/commands/)
```

Each subagent writes its output to a named JSON file on disk. The parent reads the file path â€” not the full content â€” and passes it to the next subagent. This file-based handoff keeps the parent context window manageable across 6+ sequential calls.

### 2. Magic Command (`/N-magic`)

At the end of the `/cc-workflow` run, a numbered project-level magic command is written to your project's `.claude/commands/` directory (e.g., `1-magic.md`, `2-magic.md`). The number is collision-safe â€” the system enumerates existing magic files before assigning the next integer.

Paste `/N-magic` into a fresh chat. The parent agent drives the `111111111111.md` runner via subagents, advancing one step at a time until all implementation artifacts are produced.

### 3. Self-Iteration Audit Command (`/cc-audit`)

A separate global command that lists all project-level magic commands via a multiple-choice question. It audits the execution history from `global-state-log.json`, produces a braindump file, and feeds it back into `/cc-workflow` â€” causing the system to analyze and improve itself.

Before any file is modified during self-iteration, it is duplicated into a sequentially numbered archive folder. A JSON log maps each archived file to its execution run, timestamp, and before/after state.

---

## The Two-Command Workflow

```bash
# Command 1 â€” in any Claude Code project chat:
/cc-workflow [your brain dump, file path, or idea]

# Command 2 â€” in a fresh Claude Code chat:
/N-magic
```

That is the entire user-facing interface. All 6 pipeline steps run automatically between the two commands.

---

## Install via Setup Prompt

1. Clone this repo to any local path:
   ```bash
   git clone https://github.com/[your-username]/claude-code-workflow
   ```

2. Open any Claude Code chat.

3. Copy the entire contents of `setup-prompt.md` from the repo root.

4. Paste it into the chat.

Claude will locate the global `.claude` folder dynamically, copy all system files, initialize the log files, and confirm success. Zero path inputs. Zero configuration choices. Works on any machine.

---

## How Magic Commands Work

Each `/cc-workflow` run generates one numbered magic command in your project's `.claude/commands/` directory. The numbering is sequential and collision-safe:

- First run: `1-magic.md`
- Second run on same project: `2-magic.md`
- And so on â€” no overwrites, no collisions

The magic command uses the same parent-agent-plus-subagents architecture as the entry point. When you paste `/N-magic` in a fresh chat:

1. The parent agent checks that `seq-state.json` and `111111111111.md` exist (halts with a clear diagnostic if either is missing)
2. For each pending step in `seq-state.json`, it spawns one subagent
3. Each subagent executes exactly one step of the runner
4. After all steps complete, the parent reports the full artifact list

---

## How Self-Iteration Works

Run `/cc-audit` in a fresh chat at any time:

1. A multiple-choice list shows all magic commands in the current project
2. You select which one to audit
3. The system reads `global-state-log.json` for that command's execution history
4. A braindump file is generated summarizing what worked, what failed, what changed
5. The braindump is fed into `/cc-workflow` â€” the pipeline runs on the system itself
6. A self-improvement sequence is generated and executed
7. Every modified file is archived first; the archive log is updated

**First run behavior:** If `global-state-log.json` does not exist yet, the system initializes it with a baseline entry (run #1) and treats the first audit as a state-capture operation â€” no changes are proposed. Future runs compare against this baseline.

---

## Archive and State Log

### Archive system
- Location: `[repo]/archives/`
- Structure: numbered folders per self-iteration run (`01/`, `02/`, `03/`...)
- Rule: archived files are never modified â€” each run always creates a new archive folder
- Purpose: full rollback capability without git or external services

### Archive log
- Location: `[repo]/logs/archive-log.json`
- Maps each archived file to: execution run ID, timestamp, before/after state
- Never overwritten â€” only appended

### Global state log
- Location: `[repo]/logs/global-state-log.json`
- Accumulates cross-project self-iteration history indefinitely
- Records what changed, what worked, and what regressed in each run
- Used by `/cc-audit` to identify patterns across many iterations

---

## Clarification Mode

When you run `/cc-workflow`, the very first question the intake subagent asks is:

> "Do you want me to ask clarifying questions during this workflow?"

- **Yes:** The intake subagent runs web research and asks 4-10 targeted clarifying questions before writing the intake document. Best for complex or ambiguous goals.
- **No:** All 6 subagent steps run with zero interactive pauses. Best when you want the full workflow to run hands-free.

The clarification mode flag is written to `step-00-clarification-mode.json` and propagated to every downstream subagent. When mode = no, no subagent in the chain can trigger an interactive pause.

---

## File Structure

```
claude-code-workflow/
â”œâ”€â”€ setup-prompt.md                    # One-paste installer
â”œâ”€â”€ README.md                          # This file
â”œâ”€â”€ HOW-TO-USE-CLAUDE-CODE.md          # Detailed guide
â”œâ”€â”€ logs/
â”‚   â”œâ”€â”€ archive-log.json               # Maps archived files to runs
â”‚   â””â”€â”€ global-state-log.json          # Cross-project iteration history
â”œâ”€â”€ archives/
â”‚   â”œâ”€â”€ 01/                            # Archived files from iteration run 1
â”‚   â”œâ”€â”€ 02/                            # Archived files from iteration run 2
â”‚   â””â”€â”€ ...                            # Immutable â€” never modified after creation
â””â”€â”€ .claude/
    â””â”€â”€ commands/
        â”œâ”€â”€ 01-intake.md               # Pipeline step 1
        â”œâ”€â”€ 02-extract-profile.md      # Pipeline step 2
        â”œâ”€â”€ 03-build-prompt.md         # Pipeline step 3
        â”œâ”€â”€ 04-refinep.md              # Pipeline step 4
        â”œâ”€â”€ 05-split-orc-v2.md         # Pipeline step 5
        â””â”€â”€ 06-seq-init.md             # Pipeline step 6

Global .claude folder (installed by setup-prompt.md):
~/.claude/commands/  (or %APPDATA%\Claude\commands\ on Windows)
    â”œâ”€â”€ cc-workflow.md                 # Global entry point command
    â”œâ”€â”€ cc-audit.md                    # Global self-iteration audit command
    â”œâ”€â”€ 01-intake.md through 06-seq-init.md

Per-project (generated at runtime):
[project]/.claude/commands/
    â”œâ”€â”€ 1-magic.md                     # Generated by first /cc-workflow run
    â”œâ”€â”€ 2-magic.md                     # Generated by second run
    â””â”€â”€ 111111111111.md                # Generated by 06-seq-init

[project]/
    â”œâ”€â”€ [slug]-intake.md               # Output of step 01
    â”œâ”€â”€ [slug]-vision.md               # Output of step 02
    â”œâ”€â”€ [slug]-profile.json            # Output of step 02
    â”œâ”€â”€ exec-[slug].md                 # Output of step 03
    â”œâ”€â”€ [subject]_refined.md           # Output of step 04
    â”œâ”€â”€ splits/[folder]/               # Output of step 05
    â””â”€â”€ seq-state.json                 # Output of step 06 (live execution state)
```

---

## Prerequisites

- **[Claude Code CLI](https://claude.ai/code)** â€” installed and authenticated
- **Claude Code subscription** â€” Pro or above (Max recommended for 6+ sequential subagent chains)
- A brain dump. A voice note transcript. A document. Anything works.

---

## Contributing

This framework is actively used and self-improving. If you have improvements:

1. Fork the repo
2. Create a branch: `git checkout -b feature/your-improvement`
3. Make changes to `.claude/commands/`
4. Update this README if the system behavior changes
5. Open a pull request

---

## License

MIT â€” see [LICENSE](./LICENSE).

Use it, fork it, improve it, ship it.