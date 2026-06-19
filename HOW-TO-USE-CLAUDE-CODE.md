# How to Use Claude Code — Brain Dump to Full Execution

## The Process at a Glance

**Brain dump → Analyze → Build Prompt → Refine → Split into Steps → Initialize Runner → Execute**

Each section below is numbered. Follow them in order. Do not skip steps. Fresh chat = clear the chat and open a new one.

---

## STEP 1 — Write Your Brain Dump

Before opening Claude Code, write down everything you want to do. No structure needed. Voice notes, fragments, ideas, goals — all of it.

Save it as a `.md` file. Example:

```
C:\Users\Alexb\My Drive\website-outreach-system\_User-sandbox\_June 19\raw-vision.md
```

You already have this file. Use it.

---

## STEP 2 — Analyze the Draft

**What this does:** Extracts real intent, goals, and structure from your raw brain dump. Produces two output files you'll use in Step 3.

**Command to type:**

```
/intent:analyze-draft
```

**Then immediately paste** the full contents of your brain dump file below the command (copy-paste the text — do not use a file path argument for this one). Or type the argument path if the skill accepts it:

```
/intent:analyze-draft C:\Users\Alexb\My Drive\website-outreach-system\_User-sandbox\_June 19\raw-vision.md
```

**Output files it creates (note these names):**
- `[slug]-vision.md` — human-readable vision document
- `[slug]-profile.json` — structured JSON context profile

The `[slug]` is auto-generated from your content (e.g., `facebook-dashboard-agentic-plan`). Look at what files were created in the working directory and note the exact filenames.

**When done:** Note the two output file paths. Then **clear the chat and open a new one.**

---

## STEP 3 — Build the Execution Prompt

**What this does:** Takes the two files from Step 2 and produces a single self-contained execution prompt file ready to run in a fresh Claude session.

**Command to type:**

```
/intent:build-prompt [vision-file-path] [profile-file-path]
```

Replace the placeholders with the actual file paths from Step 2. Example using the existing files in this folder:

```
/intent:build-prompt "C:\Users\Alexb\My Drive\website-outreach-system\_User-sandbox\_June 19\facebook-dashboard-agentic-plan-vision.md" "C:\Users\Alexb\My Drive\website-outreach-system\_User-sandbox\_June 19\facebook-dashboard-agentic-plan-profile.json"
```

**Output file it creates:**
- `exec-[slug].md` — the full execution prompt (e.g., `exec-facebook-dashboard-agentic-plan.md`)

Note the exact path of this file.

**When done: Clear the chat and open a new one.**

---

## STEP 4 — Refine the Prompt

**What this does:** Takes the execution prompt from Step 3 and upgrades it to production-grade quality using Anthropic best practices. Output is a polished, self-contained `.md` prompt file.

**Command to type:**

```
/refinep [path-to-exec-file]
```

Example:

```
/refinep "C:\Users\Alexb\My Drive\website-outreach-system\_User-sandbox\_June 19\exec-facebook-dashboard-agentic-plan.md"
```

**Output file it creates:**
- `[subject]_refined.md` in the root working directory — note its exact name and path.

**When done: Clear the chat and open a new one.**

---

## STEP 5 — Split into Parallel/Sequential Batches

**What this does:** Takes the refined prompt from Step 4 and decomposes it into numbered batch files inside a `splits/` folder. These batches are what gets executed step-by-step.

**Command to type:**

```
/split-orc-v2 [path-to-refined-prompt]
```

Example:

```
/split-orc-v2 "C:\Users\Alexb\My Drive\website-outreach-system\[subject]_refined.md"
```

(Use whatever filename `refinep` produced in Step 4.)

**Output folder it creates:**
```
splits/[prompt-name-without-extension]/
  ├── 01-PARALLEL.md  (or 01-[label].md for complex plans)
  ├── 02-PARALLEL.md
  ├── ...
  ├── [NN]-FINAL-SOLO.md
  └── README.md
```

Note the **full absolute path** to the splits folder. You'll need it in Step 6.

**When done: Clear the chat and open a new one.**

---

## STEP 6 — Initialize the Sequential Runner

**What this does:** Scans the splits folder from Step 5, builds a state-tracking file, and auto-generates a runner command (`/111111111111`) you'll use to execute each step one by one in fresh chats.

**Command to type:**

```
/seq-init [absolute-path-to-splits-folder]
```

Example:

```
/seq-init "C:\Users\Alexb\My Drive\website-outreach-system\splits\subject-refined"
```

(Replace `subject-refined` with the actual folder name created in Step 5.)

**Output files it creates:**
- `splits/[folder]/seq-state.json` — tracks which steps are done/pending
- `.claude/commands/111111111111.md` — your runner command (auto-generated, no edits needed)

**When done: Clear the chat and open a new one.**

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

## Quick Reference — Commands in Order

| Step | Command | Argument |
|------|---------|----------|
| 2 | `/intent:analyze-draft` | Paste brain dump text (or file path) |
| 3 | `/intent:build-prompt` | `[vision.md] [profile.json]` |
| 4 | `/refinep` | `[exec-slug.md]` |
| 5 | `/split-orc-v2` | `[subject_refined.md]` |
| 6 | `/seq-init` | `[absolute path to splits folder]` |
| 7 | `/111111111111` | (no argument — repeat in fresh chats) |

---

## Rules

- **Every step runs in its own fresh chat.** Never chain steps in the same chat.
- **Always note output file paths** before clearing a chat — you'll need them in the next step.
- **Step 7 repeats** until all batches are complete. The state file tracks your progress automatically so you never lose your place.
- If you want to pick up where you left off on a previous run, just open a fresh chat and type `/111111111111` — it reads the state file and resumes.
