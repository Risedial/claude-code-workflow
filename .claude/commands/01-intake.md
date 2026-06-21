---
name: intake
description: "Step 1 of the cc-workflow pipeline. Takes any raw brain dump, voice note, or idea. FIRST asks whether the user wants clarifying questions (clarification mode). If mode=no, runs silent automated intake with zero interactive pauses. If mode=yes, researches the domain (4-8 web searches), extracts intent signals, asks 4-10 targeted clarifying questions, and writes a structured intake document. Writes clarification_mode to step-00-clarification-mode.json for all downstream subagents."
argument-hint: "[optional: ({path/to/chat-history-file})] [raw idea, brain dump, or voice transcript — paste directly or pass file:path]"
allowed-tools: WebSearch, WebFetch, AskUserQuestion, Write, Read, Edit
---

You are executing the `/intake` command as a subagent in the cc-workflow pipeline. Your job: gather intent signals, optionally ask clarifying questions, and write a structured intake document plus a clarification mode state file.

---

## PHASE 0 — SETUP

**Parse arguments:**
- If $ARGUMENTS starts with `({`: extract content between `({` and `})` as CHAT_HISTORY_PATH. Set RAW_INPUT = everything after `})`, trimmed. Read CHAT_HISTORY_PATH with the Read tool and store as CHAT_HISTORY_CONTEXT. If the file cannot be read, set CHAT_HISTORY_CONTEXT = null and continue.
- If $ARGUMENTS starts with `file:`: read the file at the path that follows and set RAW_INPUT = file contents.
- Otherwise: RAW_INPUT = $ARGUMENTS as-is.
- If RAW_INPUT is empty or whitespace after parsing: use AskUserQuestion to ask "Paste your raw idea, brain dump, voice note, or transcript."

**Derive SLUG from RAW_INPUT:**
1. Extract the 3-5 most semantically dense words (skip stop words: a, the, I, my, me, want, need, to, for, and, or, but, with, that, this, just, like, some, it, its, be, is, are, was, have, how, what, so)
2. If RAW_INPUT contains a proper noun (product name, project name, company name), make it the first word
3. Lowercase, join with hyphens, strip all punctuation except hyphens
4. Truncate to 40 characters maximum
5. Collision rule: if [slug]-intake.md already exists, append -2, -3, etc.

**Define output paths:**
INTAKE_FILE = [current working directory]/[slug]-intake.md
CLARIFICATION_MODE_FILE = [current working directory]/step-00-clarification-mode.json
INTAKE_JSON_FILE = [current working directory]/step-01-intake.json

---

## PHASE 0.5 — CLARIFICATION MODE GATE (NEW — runs before all other phases)

Ask ONE question using AskUserQuestion:
- question: "Do you want me to ask clarifying questions during this workflow?"
- header: "Clarification Mode"
- options:
  - A) "Yes, ask me questions" — "I will run web research and ask 4-10 targeted questions before writing the intake document. Best for complex or ambiguous goals."
  - B) "No, run automatically" — "I will proceed with zero interactive pauses. Best when you want the full workflow to run hands-free. Output quality may be lower for ambiguous inputs."
- multiSelect: false

If user selects A: set CLARIFICATION_MODE = "yes"
If user selects B: set CLARIFICATION_MODE = "no"

Write CLARIFICATION_MODE_FILE immediately:
```json
{"clarification_mode": "[CLARIFICATION_MODE]", "set_at": "[ISO8601 timestamp]"}
```

All downstream subagents MUST read this file before calling AskUserQuestion. If CLARIFICATION_MODE = "no", all downstream subagents MUST NOT call AskUserQuestion under any circumstances.

---

## PHASE 1 — SIGNAL EXTRACTION (internal — no output)

Do one internal read-through of RAW_INPUT and annotate:

- **Voice-to-text artifacts**: filler words (um, like, so, you know), repeated phrases, run-on clauses, incomplete sentences
- **Explicit statements**: things stated directly, taken at face value
- **Implied intent**: sentence fragments that assume context — each one is an implicit intent candidate. Flag as INFERRED.
- **Contradictions**: places where the input says two conflicting things. Flag each pair.
- **Desire signals**: "always", "never", "every time", "automatically", "without having to" — signal systematic intent
- **Fear signals**: "don't want to", "tired of", "hate", "frustrated", "keep having to" — signal pain points
- **Emotional register**: frustrated / excited / uncertain / urgent / calm / overwhelmed
- **Domain vocabulary**: technical terms, jargon, product names

If CHAT_HISTORY_CONTEXT is not null, do a second extraction pass on it:
- Prior goals from the history session
- Decisions that were made (options chosen, things ruled out)
- Artifacts produced (files created, code written, configs set)
- Open threads (unresolved questions, next steps not yet executed)
- Established context (project name, codebase, stack, constraints)

Store all as SIGNALS. Hold internally for use in Phases 2, 3, and 4.

---

## PHASE 2 — DOMAIN RESEARCH

If CLARIFICATION_MODE = "no": run 3 silent web searches only. Store findings as RESEARCH. Do not display any results.
If CLARIFICATION_MODE = "yes": run full research per steps 2.1-2.4 below.

**Step 2.1 — Identify search queries:**
Based on RAW_INPUT and SIGNALS, derive 4-8 search queries. Prioritize:
- What already exists that is similar to this idea?
- What are standard implementation approaches in this domain?
- What do practitioners commonly struggle with?
- What terminology does this domain use?
- What constraints or compliance requirements apply?

**Step 2.2 — Run searches:**
Execute 4-8 WebSearch calls (3 if CLARIFICATION_MODE = "no").

**Step 2.3 — Deep fetch:**
If CLARIFICATION_MODE = "yes": for the 2-3 highest-signal results, use WebFetch to extract page content.

**Step 2.4 — Organize findings internally as RESEARCH:**
- EXISTING_SOLUTIONS
- STANDARD_APPROACHES
- KNOWN_CHALLENGES
- DOMAIN_VOCABULARY
- MISSING_SIGNALS

---

## PHASE 3 — ROUND 1 CLARIFYING QUESTIONS

If CLARIFICATION_MODE = "no": SKIP this phase entirely. Do not call AskUserQuestion.
If CLARIFICATION_MODE = "yes": ask 4-6 questions using AskUserQuestion (one tool call per question).

Each question must follow these rules:
- question field: a complete, unambiguous sentence ending with ?
- header field: 1-3 word topic label
- options field: 2-4 options, each with label and description
- multiSelect: false

Store all questions asked and answers received as QA_ROUND_1.

---

## PHASE 4 — ROUND 2 CLARIFYING QUESTIONS (conditional)

If CLARIFICATION_MODE = "no": SKIP this phase entirely.
If CLARIFICATION_MODE = "yes": run only if any of these are true after Round 1:
- A Round 1 answer introduced a new unknown
- A Round 1 answer contradicted the original idea
- A Round 1 answer revealed a critical unresolved decision point
- The combination of answers leaves a dimension ambiguous

If none are true: skip Phase 4 and proceed to Phase 5.
Ask 2-4 targeted questions. Store as QA_ROUND_2.

---

## PHASE 5 — WRITE INTAKE DOCUMENT

Write INTAKE_FILE to disk. Create blank first using Write tool, then populate using Edit.

If CLARIFICATION_MODE = "no" and the input is ambiguous: document the ambiguity in the Gaps and Open Questions section. Proceed with best-effort signal capture. Downstream subagents will adjust based on ambiguity flags.

The intake document uses exactly this structure:

---

# Intake: [slug]

> Generated: [ISO 8601 timestamp]
> Source: intake-v1
> Clarification Mode: [yes/no]

---

## Original Input

[Full RAW_INPUT cleaned of voice artifacts but preserving 100% of intent.]

---

## Chat History Context

[If CHAT_HISTORY_CONTEXT is not null: structured summary with subsections: ### Prior Goals, ### Decisions Made, ### Artifacts Produced, ### Open Threads, ### Established Context. If null: write "None provided."]

---

## Research Findings

### Existing Solutions
[Bulleted list]

### Standard Approaches
[Bulleted list]

### Known Challenges
[Bulleted list]

### Domain Vocabulary
[Bulleted list with brief definitions]

### Missing Context Signals
[Bulleted list of things not specified but important]

---

## Signal Extraction

### Explicit Statements
[Bulleted list labeled EXPLICIT.]

### Implied Intent
[Bulleted list labeled INFERRED — [evidence basis].]

### Contradictions
[Numbered list or "None identified."]

### Desire Signals
[Bulleted list — quote phrase, state implication.]

### Fear Signals
[Bulleted list — quote phrase, state fear.]

### Emotional Register
[Single value + one sentence of evidence.]

---

## Q&A Session

[If CLARIFICATION_MODE = "no": write "Clarification mode: no — Q&A skipped. All dimensions resolved from input signal and research only."]

[If CLARIFICATION_MODE = "yes": for each question asked in Phase 3 and Phase 4:]

### Q[N]: [header topic]
**Question:** [full question text]
**Options:**
- A) [label] — [full description]
- B) [label] — [full description]
**Selected:** [letter]) [label] — [full description]

[Round 2: Not triggered — OR list Round 2 questions in same format.]

---

## Synthesis

### Confirmed Facts
[Numbered list. Format: "[N]. [Fact] (Source: explicit statement / Q[N] answer)"]

### High-Confidence Inferences
[Numbered list. Format: "[N]. [Inference] (Evidence: [signal or research finding])"]

### Gaps and Open Questions
[Numbered list. Format: "[N]. [What is unknown] — [impact if assumed incorrectly]"]

### Urgency and Trigger
[Prose, 1-3 sentences.]

---

## PHASE 6 — WRITE STEP-01 OUTPUT JSON

Write INTAKE_JSON_FILE:
```json
{
  "step": "01-intake",
  "slug": "[slug]",
  "intake_file": "[absolute path to INTAKE_FILE]",
  "clarification_mode_file": "[absolute path to CLARIFICATION_MODE_FILE]",
  "clarification_mode": "[CLARIFICATION_MODE]",
  "confirmed_facts_count": [N],
  "gaps_count": [N],
  "completed_at": "[ISO8601 timestamp]"
}
```

---

## PHASE 7 — COMPLETION REPORT

Display to user:
- Intake file path
- Slug
- Clarification mode selected
- Number of web searches run
- Number of Q&A questions asked (0 if mode=no)
- Number of confirmed facts
- Number of gaps remaining
- step-01-intake.json path
- Next step (handled by parent agent)

---

## COMMAND NOTES

- CLARIFICATION_MODE must be written to step-00-clarification-mode.json before any other file
- If CLARIFICATION_MODE = "no": never call AskUserQuestion again after the initial mode question
- Do not produce a vision document or JSON profile — those are step 02's job
- If a Q&A answer directly contradicts the original input, record both in Contradictions and note in Gaps
