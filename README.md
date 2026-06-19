# claude-code-workflow

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Requires Claude Code](https://img.shields.io/badge/Requires-Claude%20Code-blueviolet)](https://claude.ai/code)
[![Workflow Version](https://img.shields.io/badge/Workflow-v2.0-blue)](./HOW-TO-USE-CLAUDE-CODE.md)

**Turn a raw brain dump into fully executed work — without losing context or momentum.**

Most AI-assisted development stalls at the same point: you have an idea, you write a rough prompt, the context window fills up, the model loses the thread, and you're back to square one. This framework solves that with a six-command pipeline that takes your messy, unstructured thinking and systematically refines it into parallelizable, state-tracked execution — one clean step at a time.

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
┌─────────────────────────────────────────────────────────────────────┐
│                    CLAUDE CODE WORKFLOW PIPELINE                    │
└─────────────────────────────────────────────────────────────────────┘

  YOUR BRAIN                                             EXECUTED WORK
  ─────────                                             ─────────────
  "I want to                                            ✓ Step 1 done
   build a thing                                        ✓ Step 2 done
   that does..."                                        ✓ Step 3 done
        │                                               ✓ Step 4 done
        ▼                                                      ▲
  ┌───────────┐                                                │
  │  /intent  │  ──► [slug]-vision.md                         │
  │  analyze- │  ──► [slug]-profile.json                      │
  │  draft    │                                                │
  └───────────┘                                                │
        │                                                      │
        ▼                                                      │
  ┌───────────┐                                                │
  │  /intent  │  ──► exec-[slug].md                           │
  │  build-   │      (self-contained execution prompt)         │
  │  prompt   │                                                │
  └───────────┘                                                │
        │                                                      │
        ▼                                                      │
  ┌───────────┐                                                │
  │ /refinep  │  ──► [subject]_refined.md                     │
  │           │      (production-grade prompt)                 │
  └───────────┘                                                │
        │                                                      │
        ▼                                                      │
  ┌───────────┐      splits/[folder]/                         │
  │ /split-   │  ──► ├── 01-[name].md                        │
  │ orc-v2    │      ├── 02-[name].md                        │
  │           │      ├── 03-[name].md                        │
  └───────────┘      └── ...                                  │
        │                                                      │
        ▼                                                      │
  ┌───────────┐  ──► seq-state.json                           │
  │ /seq-init │  ──► .claude/commands/111111111111.md         │
  └───────────┘      (auto-generated runner command)           │
        │                                                      │
        ▼                                                      │
  ┌───────────┐                                                │
  │/111111111 │  ──► executes one step ──────────────────────►│
  │  111      │      opens fresh chat
  │ (repeat)  │      reads state
  └───────────┘      runs next batch
                     updates seq-state.json
```

---

## Prerequisites

- **[Claude Code CLI](https://claude.ai/code)** — installed and authenticated
- **Claude Code subscription** — Pro or above (Max recommended for long execution chains)
- **Node.js / Python** — depending on what your splits actually execute
- A brain dump. A voice note transcript. A document you wrote at 2am. Anything works.

---

## Quick Start

Run these six commands in order. Each one feeds the next.

```bash
# 1. Extract structured intent from your raw brain dump
/intent:analyze-draft

# 2. Build a self-contained execution prompt from the vision + profile
/intent:build-prompt [slug]-vision.md [slug]-profile.json

# 3. Refine the prompt to production-grade quality
/refinep exec-[slug].md

# 4. Decompose the refined prompt into parallelizable or sequential batch files
/split-orc-v2 [subject]_refined.md

# 5. Initialize state tracking and generate the runner command
/seq-init splits/[folder]/

# 6. Execute one step at a time — repeat in a fresh Claude chat each time
/111111111111
```

After step 5, `/111111111111` is auto-generated. Repeat step 6 in a **fresh chat session** for each step until `seq-state.json` shows all steps complete.

---

## Command Reference

| Command | Input | Output | What it does |
|---|---|---|---|
| `/intent:analyze-draft` | Paste raw brain dump / voice note in chat | `[slug]-vision.md`, `[slug]-profile.json` | Extracts structured goals, constraints, and a psychoanalysis layer from unstructured input |
| `/intent:build-prompt [vision] [profile]` | `[slug]-vision.md` + `[slug]-profile.json` | `exec-[slug].md` | Assembles a self-contained, fully-specified execution prompt |
| `/refinep [prompt-file]` | Any `.md` prompt file | `[subject]_refined.md` | Applies Anthropic prompt engineering best practices to make the prompt production-grade |
| `/split-orc-v2 [refined-file]` | `[subject]_refined.md` | `splits/[folder]/` with numbered `.md` batch files | Decomposes the prompt into TYPE-A (parallel) or TYPE-B (sequential with checkpoints) execution units |
| `/seq-init [splits-folder]` | Path to a numbered splits folder | `seq-state.json`, `.claude/commands/111111111111.md` | Scans splits, initializes state tracking, and generates the `/111111111111` runner command |
| `/111111111111` | *(no arguments)* | Completed work + updated `seq-state.json` | Reads current state, executes exactly one step, and saves progress |

### Split Types

`/split-orc-v2` supports two execution models:

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

- **Steps 1–5** happen once, upfront. They are pure planning and decomposition.
- **Step 6** (`/111111111111`) executes in a fresh chat every time.

Each fresh chat session:
1. Reads `seq-state.json` to know exactly where it left off
2. Loads only the single batch file it needs to execute
3. Does the work
4. Updates `seq-state.json` with the result
5. Exits cleanly

No context from previous steps bleeds in. No 10,000-token preamble. Just the current step, done well.

### The State Machine

`seq-state.json` is the source of truth. It tracks:
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
        ├── intent/
        │   ├── analyze-draft.md       # Step 1: brain dump → vision + profile
        │   └── build-prompt.md        # Step 2: vision + profile → exec prompt
        ├── refinep.md                 # Step 3: prompt → refined prompt
        ├── split-orc-v2.md            # Step 4: prompt → numbered batch files
        ├── seq-init.md                # Step 5: splits → state + runner
        └── 111111111111.md            # Step 6: auto-generated runner (one step/chat)
```

**Generated artifacts (not committed, gitignored by default):**

```
[your-project]/
├── [slug]-vision.md                   # Output of analyze-draft
├── [slug]-profile.json                # Output of analyze-draft
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
