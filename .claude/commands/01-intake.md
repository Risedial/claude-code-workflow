---
name: intake
description: "Step 1 of 3 in the enhanced intake pipeline. Takes any raw brain dump, voice note, or idea → researches the domain (4–8 web searches + deep fetches) → extracts intent signals → asks 4–10 targeted multiple-choice clarifying questions → writes a structured intake document capturing all findings explicitly. No interpretation happens here — only gathering. Feed the output to /extract-profile."
argument-hint: "[optional: ({path/to/chat-history-file})] [raw idea, brain dump, or voice transcript — paste directly or pass file:path]"
allowed-tools: WebSearch, WebFetch, AskUserQuestion, Write, Read, Edit
---

You are executing the `/intake` command. Your single job is to gather: research the domain, extract signals from the raw input, ask clarifying questions, and write everything you learn into a structured document. You do not interpret, infer, or produce a vision document here. You capture.

---

## PHASE 0 — SETUP

**Parse arguments:**
- If $ARGUMENTS starts with `({`: extract content between `({` and `})` as CHAT_HISTORY_PATH. Set RAW_INPUT = everything after `})`, trimmed. Read CHAT_HISTORY_PATH with the Read tool and store as CHAT_HISTORY_CONTEXT. If the file cannot be read, set CHAT_HISTORY_CONTEXT = null and continue.
- If $ARGUMENTS starts with `file:`: read the file at the path that follows and set RAW_INPUT = file contents.
- Otherwise: RAW_INPUT = $ARGUMENTS as-is.
- If RAW_INPUT is empty or whitespace after parsing: use AskUserQuestion to ask "Paste your raw idea, brain dump, voice note, or transcript."

**Derive SLUG from RAW_INPUT:**
1. Extract the 3–5 most semantically dense words (skip stop words: a, the, I, my, me, want, need, to, for, and, or, but, with, that, this, just, like, some, it, its, be, is, are, was, have, how, what, so)
2. If RAW_INPUT contains a proper noun (product name, project name, company name), make it the first word
3. Lowercase, join with hyphens, strip all punctuation except hyphens
4. Truncate to 40 characters maximum
5. Collision rule: if [slug]-intake.md already exists, append -2, -3, etc.

**Define output path:**
INTAKE_FILE = [current working directory]/[slug]-intake.md

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

## PHASE 2 — DOMAIN RESEARCH (silent — no output shown to user)

**Step 2.1 — Identify search queries:**
Based on RAW_INPUT and SIGNALS, derive 4–8 search queries that would return the most useful context. Prioritize:
- What already exists that is similar to this idea?
- What are standard implementation approaches in this domain?
- What do practitioners commonly struggle with?
- What terminology does this domain use?
- What constraints or compliance requirements apply?

**Step 2.2 — Run searches:**
Execute 4–8 WebSearch calls. For complex or specialized domains, use 6–8 searches. For clearly defined straightforward ideas, 4 searches is sufficient.

**Step 2.3 — Deep fetch:**
For the 2–3 highest-signal search results, use WebFetch to extract page content.

**Step 2.4 — Organize findings internally as RESEARCH:**
- EXISTING_SOLUTIONS: tools, products, or approaches that already address this
- STANDARD_APPROACHES: most common or established implementation patterns
- KNOWN_CHALLENGES: what practitioners in this domain commonly struggle with
- DOMAIN_VOCABULARY: technical terms to use accurately in questions and outputs
- MISSING_SIGNALS: things the user likely doesn't know they should specify, surfaced by research

If any search returns no useful results: note as "limited prior art in [area]" and continue. Do not block.

---

## PHASE 3 — ROUND 1 CLARIFYING QUESTIONS

Ask 4–6 questions using AskUserQuestion (one tool call per question). Use SIGNALS + RESEARCH findings to make each question sharp and each option meaningful.

Each question must follow these rules:
- question field: a complete, unambiguous sentence ending with ?
- header field: 1–3 word topic label
- options field: 2–4 options, each with label and description that communicates a full cluster of implications (what choosing this means about their values, constraints, and intent)
- multiSelect: false

Cover as many of these as apply:
- Primary goal: what does success look like in concrete terms?
- Scope: what is in and what is out?
- Audience: who is this for?
- Priority: what matters most — speed, depth, quality, simplicity, scale?
- Top constraint: what is the most limiting factor?
- Feared outcome: what would make this a failure?

Use DOMAIN_VOCABULARY from RESEARCH to make options specific to this domain.
Store all questions asked and answers received as QA_ROUND_1.

---

## PHASE 4 — ROUND 2 CLARIFYING QUESTIONS (conditional)

Run this phase ONLY IF any of these are true after analyzing Round 1 answers:
- A Round 1 answer introduced a new unknown not present in the original idea
- A Round 1 answer contradicted the original idea or another Round 1 answer
- A Round 1 answer revealed a critical decision point still unresolved
- The combination of answers leaves a dimension ambiguous that would affect output

If none are true: skip Phase 4 and proceed to Phase 5.

Ask 2–4 targeted questions using AskUserQuestion. Each question resolves exactly one specific unresolved item. Do not ask about anything already clarified in Round 1.
Store as QA_ROUND_2.

---

## PHASE 5 — WRITE INTAKE DOCUMENT

Write INTAKE_FILE to disk. Create it blank first using the Write tool with empty content, then populate in sections using Edit tool calls.

The intake document must use exactly this structure (write it verbatim with all section headers):

---

# Intake: [slug]

> Generated: [ISO 8601 timestamp]
> Source: intake-v1

---

## Original Input

[Write the full RAW_INPUT cleaned of voice artifacts (filler words, repetition, fragments) but preserving 100% of intent, every stated detail, nuance, and constraint. Prose paragraphs. Do not add ideas not in the original.]

---

## Chat History Context

[If CHAT_HISTORY_CONTEXT is not null: write a structured summary with subsections:
### Prior Goals, ### Decisions Made, ### Artifacts Produced, ### Open Threads, ### Established Context
If null: write "None provided."]

---

## Research Findings

### Existing Solutions
[Bulleted list of tools, products, approaches that already address this domain or problem]

### Standard Approaches
[Bulleted list of the most common or established ways to implement/achieve this]

### Known Challenges
[Bulleted list of what practitioners in this domain commonly struggle with]

### Domain Vocabulary
[Bulleted list of technical terms relevant to this domain, with brief definitions where non-obvious]

### Missing Context Signals
[Bulleted list of things the user likely doesn't know to specify — surfaced by research as important but unmentioned in the input]

---

## Signal Extraction

### Explicit Statements
[Bulleted list — things directly and unambiguously stated in the original input. Labeled: EXPLICIT.]

### Implied Intent
[Bulleted list — things meant but not said directly. Each item labeled: INFERRED — [evidence basis].]

### Contradictions
[Numbered list — each contradiction as: "States X in [location] but also states Y in [location]". If none: "None identified."]

### Desire Signals
[Bulleted list — exact phrases from the input that signal systematic or recurring intent: "always", "automatically", "never having to", etc. Quote the phrase, then state what it implies.]

### Fear Signals
[Bulleted list — exact phrases signaling pain points. Quote the phrase, then state what fear it reveals.]

### Emotional Register
[Single value from: frustrated / excited / uncertain / urgent / calm / overwhelmed. One sentence of evidence.]

---

## Q&A Session

[For each question asked in Phase 3 and Phase 4, write this exact format:]

### Q[N]: [header topic from the question]
**Question:** [full question text]
**Options:**
- A) [label] — [full description as presented]
- B) [label] — [full description as presented]
- C) [label] — [full description as presented]
[additional options if any]
**Selected:** [letter]) [label] — [full description]

[Write all questions from Round 1 first, then Round 2 if it ran. If Round 2 did not run, write: "Round 2: Not triggered — all critical dimensions resolved in Round 1."]

---

## Synthesis

### Confirmed Facts
[Numbered list. Each item is something now known with certainty — either directly stated or explicitly confirmed by a Q&A answer. Format: "[N]. [Fact statement] (Source: explicit statement / Q[N] answer)"]

### High-Confidence Inferences
[Numbered list. Each item is something strongly implied but not explicitly confirmed. Format: "[N]. [Inference] (Evidence: [signal or research finding that supports this])"]

### Gaps and Open Questions
[Numbered list. Things that remain unknown or ambiguous after research and Q&A. Format: "[N]. [What is unknown] — [impact if assumed incorrectly]"]

### Urgency and Trigger
[Prose, 1–3 sentences. What caused this request, what the timeline appears to be, and any urgency signals detected.]

---

## PHASE 6 — COMPLETION REPORT

Display to user:
- Intake file path
- Slug
- Number of web searches run
- Number of Q&A questions asked
- Number of confirmed facts
- Number of gaps remaining
- Next step: "/02-extract-profile [intake-file-path]"

---

## COMMAND NOTES

- Do not display research results or Q&A options in the chat beyond the AskUserQuestion popups
- Do not produce a vision document or JSON profile — those are Command 2's job
- Do not interpret or synthesize beyond what is explicitly captured in the Synthesis section
- If a Q&A answer directly contradicts the original input, record both in Contradictions and note the conflict in Gaps
