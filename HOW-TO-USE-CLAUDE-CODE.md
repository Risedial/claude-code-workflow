# How to Use Claude Code — Brain Dump to Full Execution

## The Process at a Glance

**Brain dump → Intake (research + Q&A) → Extract Profile (JSON + vision) → Build Exec Prompt → Refine → Split into Steps → Initialize Runner → Execute**

Each section below is numbered. Follow them in order. Do not skip steps. Fresh chat = clear the chat and open a new one.

---

## STEP 1 — Run /01-intake

**What this does:** Researches your domain (4–8 web searches), extracts intent signals from your input, and asks you 4–10 targeted multiple-choice questions. Captures everything into a structured intake document. Nothing is interpreted yet — this step only gathers.

**Command to type:**

```
/01-intake [paste your raw idea, brain dump, or voice transcript here]
```

Or from a file:

```
/01-intake file:path/to/notes.md
```

Or with prior chat history for context:

```
/01-intake ({path/to/chat-history.md}) [raw idea text]
```

**What to expect:**
1. Claude silently researches your domain (no output shown)
2. A series of popup dialogs appear with multiple-choice questions — answer each one
3. Claude writes the intake document to your working directory

**Output file:** `[slug]-intake.md` — note the exact filename and path.

**When done:** Clear the chat and open a new one.

---

## STEP 2 — Run /02-extract-profile

**What this does:** Reads the intake document from Step 1 and maps its content to two output files — the JSON profile and the vision document. No research, no questions. Fast, focused extraction.

**Command to type:**

```
/02-extract-profile [path-to-intake-file]
```

Example:

```
/02-extract-profile "C:\Users\Alexb\...\[slug]-intake.md"
```

**Output files:**
- `[slug]-vision.md` — 5-section vision document
- `[slug]-profile.json` — 14-key structured context profile

Note both exact file paths.

**When done:** Clear the chat and open a new one.

---

## STEP 3 — Build the Execution Prompt

**What this does:** Takes the two files from Step 2 and produces a single self-contained execution prompt file.

**Command to type:**

```
/03-build-prompt [vision-file-path] [profile-file-path]
```

Example:

```
/03-build-prompt "[slug]-vision.md" "[slug]-profile.json"
```

**Output:** `exec-[slug].md`

Note the exact path.

**When done:** Clear the chat and open a new one.

---

## STEP 4 — Run /04-refinep

**What this does:** Takes the execution prompt from Step 3 and upgrades it to production-grade quality using Anthropic best practices. Output is a polished, self-contained `.md` prompt file.

**Command to type:**

```
/04-refinep [path-to-exec-file]
```

Example:

```
/04-refinep "C:\Users\Alexb\...\exec-[slug].md"
```

**Output file:** `[subject]_refined.md` — note its exact name and path.

**When done:** Clear the chat and open a new one.

---

## STEP 5 — Run /05-split-orc-v2

**What this does:** Takes the refined prompt from Step 4 and decomposes it into numbered batch files inside a `splits/` folder. These batches are what gets executed step-by-step.

**Command to type:**

```
/05-split-orc-v2 [path-to-refined-prompt]
```

Example:

```
/05-split-orc-v2 "C:\Users\Alexb\...\[subject]_refined.md"
```

**Output folder it creates:**

```
splits/[prompt-name-without-extension]/
  ├── 01-[label].md
  ├── 02-[label].md
  ├── ...
  └── README.md
```

Note the **full absolute path** to the splits folder. You'll need it in Step 6.

**When done:** Clear the chat and open a new one.

---

## STEP 6 — Run /06-seq-init

**What this does:** Scans the splits folder from Step 5, builds a state-tracking file, and auto-generates a runner command (`/111111111111`) you'll use to execute each step one by one in fresh chats.

**Command to type:**

```
/06-seq-init [absolute-path-to-splits-folder]
```

Example:

```
/06-seq-init "C:\Users\Alexb\...\splits\subject-refined"
```

**Output files it creates:**
- `seq-state.json` — tracks which steps are done/pending
- `.claude/commands/111111111111.md` — your runner command (auto-generated, no edits needed)

**When done:** Clear the chat and open a new one.

---

## STEP 7 — Execute the Steps (Repeat Until Done)

**What this does:** Runs exactly one step per fresh chat. Each run reads the state, executes the next pending prompt batch, updates the state file, and reports what's next.

**Open a fresh chat. Type:**

```
/111111111111
```

That's it. No arguments. No file paths.

**After it finishes:**
- It will tell you which step just ran and what step is next.
- **Clear the chat and open a new one.**
- Type `/111111111111` again.
- Repeat until it reports: **All steps complete.**

---

## Quick Reference Table

| Step | Command | Argument |
|------|---------|----------|
| 1 | `/01-intake` | Raw idea text or file:path |
| 2 | `/02-extract-profile` | `[slug]-intake.md` |
| 3 | `/03-build-prompt` | `[slug]-vision.md [slug]-profile.json` |
| 4 | `/04-refinep` | `exec-[slug].md` |
| 5 | `/05-split-orc-v2` | `[subject]_refined.md` |
| 6 | `/06-seq-init` | absolute path to splits folder |
| 7 | `/111111111111` | (no argument — repeat in fresh chats) |

---

## Rules

- **Every step runs in its own fresh chat.** Never chain steps in the same chat.
- **Always note output file paths** before clearing a chat — you need them in the next step.
- **Step 7 repeats** until all batches complete. The state file tracks progress automatically.
- The intake document (`[slug]-intake.md`) is your source of truth. Keep it. If something downstream looks wrong, it traces back to the intake.
- If you want to pick up where you left off on a previous run, just open a fresh chat and type `/111111111111` — it reads the state file and resumes.
