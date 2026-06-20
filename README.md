# claude-code-workflow

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Requires Claude Code](https://img.shields.io/badge/Requires-Claude%20Code-blueviolet)](https://claude.ai/code)
[![Workflow Version](https://img.shields.io/badge/Workflow-v2.0-blue)](./HOW-TO-USE-CLAUDE-CODE.md)

**Turn a raw brain dump into fully executed work — without losing context or momentum.**

Most AI-assisted development stalls at the same point: you have an idea, you write a rough prompt, the context window fills up, the model loses the thread, and you're back to square one. This framework solves that with a seven-command pipeline that takes your messy, unstructured thinking and systematically refines it into parallelizable, state-tracked execution — one clean step at a time.

No more 10,000-token context bleed. No more re-explaining what you want. No more half-finished outputs.

---

## Table of Contents

- [Pipeline Overview](#pipeline-overview)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Command Reference](#command-reference)
- [How It Works](#how-it-works)
- [File Structure](#file-structure)
- [HOW-TO-USE Guide](#how-to-use-guide)
- [Contributing](#contributing)
- [License](#license)

---

## Pipeline Overview

```
  YOUR BRAIN
  ─────────
  "I want to
   build a thing
   that does..."
        │
        ▼
  ┌───────────────┐
  │  /01-intake   │  ──► [slug]-intake.md
  │               │      (research + Q&A + signals, all explicit)
  └───────────────┘
          │
          ▼
  ┌───────────────┐
  │ /02-extract-  │  ──► [slug]-vision.md
  │ profile       │  ──► [slug]-profile.json
  └───────────────┘
          │
          ▼
  ┌───────────────┐
  │ /03-build-    │  ──► exec-[slug].md
  │ prompt        │      (self-contained execution prompt)
  └───────────────┘
          │
          ▼
  ┌───────────────┐
  │ /04-refinep   │  ──► [subject]_refined.md
  └───────────────┘
          │
          ▼
  ┌───────────────┐      splits/[folder]/
  │ /05-split-    │  ──► ├── 01-[name].md
  │ orc-v2        │      ├── 02-[name].md
  └───────────────┘      └── ...
          │
          ▼
  ┌───────────────┐  ──► seq-state.json
  │ /06-seq-init  │  ──► .claude/commands/111111111111.md
  └───────────────┘
          │
          ▼  (repeat in fresh chat per step)
  ┌───────────────┐
  │ /111111111111 │  ──► executes one step
  │   (repeat)    │
  └───────────────┘
```

---

## Prerequisites

- **[Claude Code CLI](https://claude.ai/code)** — installed and authenticated
- **Claude Code subscription** — Pro or above (Max recommended for long execution chains)
- **Node.js / Python** — depending on what your splits actually execute
- A brain dump. A voice note transcript. A document you wrote at 2am. Anything works.

---

## Quick Start

Run these seven commands in order. Each one feeds the next.

```bash
# 1. Research domain, extract signals, ask clarifying questions
/01-intake

# 2. Map intake document to JSON profile + vision document
/02-extract-profile [slug]-intake.md

# 3. Build a self-contained execution prompt from the vision + profile
/03-build-prompt [slug]-vision.md [slug]-profile.json

# 4. Refine the prompt to production-grade quality
/04-refinep exec-[slug].md

# 5. Decompose the refined prompt into parallelizable or sequential batch files
/05-split-orc-v2 [subject]_refined.md

# 6. Initialize state tracking and generate the runner command
/06-seq-init splits/[folder]/

# 7. Execute one step at a time — repeat in a fresh Claude chat each time
/111111111111
```

After step 6, `/111111111111` is auto-generated. Repeat step 7 in a **fresh chat session** for each step until `seq-state.json` shows all steps complete.

---

## Command Reference

| Command | Input | Output | What it does |
|---|---|---|---|
| `/01-intake [brain dump or file:path]` | Raw idea text, voice transcript, or file | `[slug]-intake.md` | Researches domain (4–8 web searches), extracts intent signals, asks 4–10 targeted clarifying Q&A questions. Captures all findings explicitly — no interpretation. |
| `/02-extract-profile [intake-file-path]` | `[slug]-intake.md` from /01-intake | `[slug]-vision.md`, `[slug]-profile.json` | Maps intake document content to JSON profile (14-key schema) and vision document (5-section format). No research or Q&A — pure structured extraction. |
| `/03-build-prompt [vision] [profile]` | `[slug]-vision.md` + `[slug]-profile.json` | `exec-[slug].md` | Assembles a self-contained, fully-specified execution prompt |
| `/04-refinep [prompt-file]` | Any `.md` prompt file | `[subject]_refined.md` | Applies Anthropic prompt engineering best practices to make the prompt production-grade |
| `/05-split-orc-v2 [refined-file]` | `[subject]_refined.md` | `splits/[folder]/` with numbered `.md` batch files | Decomposes the prompt into TYPE-A (parallel) or TYPE-B (sequential with checkpoints) execution units |
| `/06-seq-init [splits-folder]` | Path to a numbered splits folder | `seq-state.json`, `.claude/commands/111111111111.md` | Scans splits, initializes state tracking, and generates the `/111111111111` runner command |
| `/111111111111` | *(no arguments)* | Completed work + updated `seq-state.json` | Reads current state, executes exactly one step, and saves progress |

### Split Types

`/05-split-orc-v2` supports two execution models:

- **TYPE-A (Parallel generation)** — batches that can run simultaneously, e.g. writing copy for multiple pages at once
- **TYPE-B (Sequential implementation)** — ordered steps with checkpoints, e.g. scaffold → implement → test → integrate

---

## How It Works

### The Core Problem This Solves

Claude's context window is powerful but finite. When you try to execute a complex project in a single session, three things go wrong:

1. **Context bleed** — early decisions contaminate later ones as the window fills
2. **Drift** — the model loses track of constraints defined 50 turns ago
3. **Stall** — one failed step hangs the entire session

### The Solution: One Step Per Fresh Chat

This framework enforces a clean separation between *planning* and *execution*:

- **Steps 1–6** happen once, upfront. They are pure planning and decomposition.
- **Step 7** (`/111111111111`) executes in a fresh chat every time.

Each fresh chat session:
1. Reads `seq-state.json` to know exactly where it left off
2. Loads only the single batch file it needs to execute
3. Does the work
4. Updates `seq-state.json` with the result
5. Exits cleanly

No context from previous steps bleeds in. No 10,000-token preamble. Just the current step, done well.

### Why Three Commands Instead of One

The intake pipeline splits work across three focused sessions to avoid the two failure modes of the previous design:

1. **Inference from messy text** — the old `/intent:analyze-draft` had to guess intent from unstructured brain dumps. `/01-intake` instead does research and asks questions, so every value in the intake document is explicit — either directly stated or selected from a multiple-choice option.

2. **Session dilution** — combining research + Q&A + JSON extraction + vision writing + exec prompt assembly in one session puts too many cognitive tasks in one context window, increasing the chance of phases being skipped or abbreviated.

The three commands are:
- **/01-intake**: Gather only. Research the domain, extract signals, ask questions, write everything down explicitly.
- **/02-extract-profile**: Extract only. Read the intake document, map explicit content to JSON and vision formats. No research, no Q&A.
- **/03-build-prompt**: Assemble only. Read vision + JSON, produce the execution prompt.

The intake document ([slug]-intake.md) is the source of truth for the whole pipeline. It stores Q&A answers verbatim with full option descriptions, research findings organized by category, and signals classified explicitly.

### The State Machine

`seq-state.json` is the source of truth for execution. It tracks:
- Which steps exist
- Which are complete
- Which is next
- Any outputs or artifacts from each step

This means you can pause, resume days later, hand the project to someone else, or rerun a failed step — and the system always knows exactly where things stand.

### The Philosophy

> Decompose first. Execute serially. Never lose state.

Complex work fails not because AI isn't capable, but because the interface between human intent and AI execution is lossy. This pipeline tightens that interface at every stage: intent extraction, prompt engineering, decomposition, and stateful execution.

---

## File Structure

```
claude-code-workflow/
├── HOW-TO-USE-CLAUDE-CODE.md          # Full step-by-step guide
└── .claude/
    └── commands/
        ├── 01-intake.md               # Step 1: brain dump → intake document
        ├── 02-extract-profile.md      # Step 2: intake → vision + profile
        ├── 03-build-prompt.md         # Step 3: vision + profile → exec prompt
        ├── 04-refinep.md              # Step 4: prompt → refined prompt
        ├── 05-split-orc-v2.md         # Step 5: prompt → numbered batch files
        ├── 06-seq-init.md             # Step 6: splits → state + runner
        └── 111111111111.md            # Step 7: auto-generated runner (one step/chat)
```

**Generated artifacts (not committed, gitignored by default):**

```
[your-project]/
├── [slug]-intake.md                   # Output of /intake
├── [slug]-vision.md                   # Output of /extract-profile
├── [slug]-profile.json                # Output of /extract-profile
├── exec-[slug].md                     # Output of build-prompt
├── [subject]_refined.md               # Output of refinep
├── splits/
│   └── [folder]/
│       ├── 01-[step-name].md
│       ├── 02-[step-name].md
│       └── ...
└── seq-state.json                     # Live execution state
```

---

## HOW-TO-USE Guide

The full annotated walkthrough lives in [`HOW-TO-USE-CLAUDE-CODE.md`](./HOW-TO-USE-CLAUDE-CODE.md). It covers:

- When to use TYPE-A vs TYPE-B splits
- How to handle a failed step
- How to rerun a specific batch file manually
- Tips for writing better brain dumps (better input = better output at every stage)
- Common failure modes and how to recover

---

## Contributing

This framework is actively used and evolving. If you've built something on top of it, fixed an edge case, or have a command that belongs in this pipeline:

1. Fork the repo
2. Create a branch: `git checkout -b feature/your-command-name`
3. Add your command to `.claude/commands/` with a clear header comment block
4. Update the Command Reference table in this README
5. Open a pull request with a description of what problem it solves

Issues and discussions welcome. If something doesn't work as documented, file a bug — the pipeline should be reliable enough to trust.

---

## License

MIT — see [LICENSE](./LICENSE).

Use it, fork it, ship it.
