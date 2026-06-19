---
name: analyze-draft
description: "Meta-prompt processor. Takes any raw, unrefined input — voice-to-text, stream-of-consciousness, fragmented notes — and extracts maximum meaning using 8 analysis techniques: explicit intent, implicit intent, psychoanalysis, goal decomposition, constraint mapping, dependency analysis, pattern recognition, and vision reconstruction. Always outputs exactly two files: [slug]-vision.md (reorganized human-readable vision) and [slug]-profile.json (comprehensive 14-key context profile). The 95% confidence score is self-referential — it measures how completely this skill applied its own rules, not how certain it is about the user's intent."
argument-hint: "[optional: ({path/to/chat-history-file})] [raw unrefined prompt, voice note, or stream-of-consciousness text]"
allowed-tools: Write, Read, AskUserQuestion
---

You are executing the `intent:analyze-draft` command. Your job is to take a messy, unrefined prompt and extract maximum meaning from it — then produce two canonical output files every time, with the same schema regardless of what the input contains.

---

## PHASE 0 — SETUP

**Parse arguments for optional chat history:**

Check whether `$ARGUMENTS` begins with `({` (the optional chat history prefix):

- **If `$ARGUMENTS` starts with `({`:** Extract the content between `({` and the first `})` as `CHAT_HISTORY_PATH`. Set `RAW_INPUT` = everything after the closing `})`, trimmed of leading whitespace. Then read the file at `CHAT_HISTORY_PATH` using the Read tool and store the result as `CHAT_HISTORY_CONTEXT`. If the file cannot be read (path invalid, file missing, permission error), set `CHAT_HISTORY_CONTEXT = null` and note: *"Chat history file could not be read — proceeding without it."* Do not fail.
- **If `$ARGUMENTS` does not start with `({`:** Set `RAW_INPUT = $ARGUMENTS` and `CHAT_HISTORY_CONTEXT = null`. Proceed exactly as before — no change to behavior.

```
CHAT_HISTORY_PATH = extracted from ({...}) prefix, or null
CHAT_HISTORY_CONTEXT = file contents at CHAT_HISTORY_PATH, or null
RAW_INPUT = $ARGUMENTS with ({...}) prefix removed, or $ARGUMENTS as-is
```

If `RAW_INPUT` is empty or whitespace-only after parsing, use AskUserQuestion:
> "Paste the prompt, voice note, or raw idea you want me to analyze."

Then proceed. Do NOT ask clarifying questions about the content — the entire point of this skill is to work with whatever is given.

**Derive the SLUG** from `RAW_INPUT` using this algorithm:
1. Extract the 3–5 most semantically dense words (skip stop words: a, the, I, my, me, want, need, to, for, and, or, but, with, that, this, just, like, some, it, its, be, is, are, was, have, how, what, so)
2. If `RAW_INPUT` contains a proper noun (product name, project name, company name, person's name), make it the first word
3. Lowercase all words, join with hyphens, strip all punctuation except hyphens
4. Truncate to 40 characters maximum
5. If the result is still too generic (e.g., "help-stuff-better"), fall back to the first 3 content words of the input
6. Collision rule: if `[slug]-vision.md` already exists in the current directory, append `-2`, then `-3`, etc.

**Define output paths:**
- `VISION_FILE` = `[current working directory]/[slug]-vision.md`
- `PROFILE_FILE` = `[current working directory]/[slug]-profile.json`

---

## PHASE 1 — SIGNAL EXTRACTION (internal only, no output)

Before touching the schema, do one internal read-through of `RAW_INPUT` and mentally annotate:

- **Voice-to-text artifacts**: filler words (um, like, so, you know), repeated phrases, run-on clauses, incomplete sentences — note them; clean without losing meaning
- **Emotional register**: is the speaker frustrated? excited? uncertain? urgent? calm? overwhelmed?
- **Domain vocabulary**: technical terms, jargon, product names — these signal expertise level and domain
- **Contradictions**: places where the input says two things that conflict — flag each one
- **Implied context**: sentence fragments that assume the listener knows something — each one is an implicit intent candidate
- **Desire signals**: words like "always", "never", "every time", "automatically", "without having to" signal systematic/repeatable intent
- **Fear signals**: words like "don't want to", "tired of", "hate", "frustrated", "keep having to" signal pain points driving the request

**If `CHAT_HISTORY_CONTEXT` is not null, do a second signal extraction pass on it:**

- **Prior goals**: What was the user working toward in the prior session?
- **Decisions made**: Key choices that were reached — options chosen, approaches selected, things ruled out
- **Artifacts produced**: Files created, code written, configs set, documents drafted
- **Open threads**: Unresolved questions, next steps that were identified but not executed, things left mid-stream
- **Established context**: Background the prior session established — project name, codebase, stack, constraints, personas involved

Hold these as `HISTORY_SIGNALS`. Merge them with `RAW_INPUT` signals when populating Phase 2 fields — treat history signals as implicit context that enriches (but does not override) what the raw prompt says.

This phase produces no output. It is internal preparation for Phase 2.

---

## PHASE 2 — SCHEMA POPULATION

Populate all 14 top-level JSON keys in this strict order. For each key:
1. First extract what is **explicitly stated** (directly in `RAW_INPUT`)
2. Then **infer** what is implicitly supported (signal-based, not invented)
3. Mark what **cannot be determined** as `null` with a sibling `[fieldname]_confidence` value of `0.0`
4. **Never fabricate.** If a field has no evidence base, it is null. Inference is allowed only when supported by signals in Phase 1.

**Key population order:**

### 1. `intent` — The anchor. All other keys are shaped by this one.
- `explicit`: The literal request, verbatim reconstructed and cleaned of artifacts
- `implicit`: What they meant but didn't say — the real need behind the stated request
- `primary_goal`: The single most important thing they want accomplished (one sentence)
- `secondary_goals`: Supporting objectives, ordered by priority (array of strings)
- `tertiary_goals`: Nice-to-haves, soft objectives (array of strings)
- `anti_goals`: What they explicitly or implicitly do NOT want (array of strings)
- `trigger_event`: What caused this prompt — deadline, frustration, opportunity, curiosity, external pressure
- `urgency_signal`: `"immediate"` | `"short_term"` | `"long_term"` | `"undefined"`

### 2. `domain` — The subject matter frame
- `primary_domain`: e.g., "software development", "content creation", "business strategy", "personal productivity"
- `sub_domain`: e.g., "React frontend", "YouTube scripts", "pricing strategy", "Claude Code skills"
- `domain_maturity`: `"novice"` | `"practitioner"` | `"expert"` — inferred from vocabulary and framing
- `domain_jargon_used`: Array of technical/domain-specific terms they used (signals their familiarity level)
- `industry_context`: SaaS, e-commerce, agency, creator, enterprise, personal, consulting, etc.
- `cross_domain_dependencies`: Other domains this touches (e.g., ["legal", "design", "devops"])

### 3. `persona` — Who is speaking and what role they occupy
- `operator_role`: Their actual role — founder, developer, marketer, writer, consultant, student, etc.
- `operator_experience_level`: Inferred experience with THIS specific task (not their general expertise)
- `team_context`: `"solo"` | `"small_team"` | `"large_team"` | `"enterprise"` | `"undefined"`
- `decision_authority`: Can they decide alone, or do they need approval from others?
- `technical_fluency`: `"non_technical"` | `"semi_technical"` | `"technical"` | `"expert"`
- `communication_style`: `"formal"` | `"casual"` | `"technical"` | `"stream_of_consciousness"` | `"voice_dictation"`

### 4. `psychoanalysis` — Underlying motivations beneath the stated request
Read against `intent` and `persona` to derive these.
- `surface_motivation`: What they think they want
- `deeper_motivation`: Why they actually want it (status, security, autonomy, efficiency, recognition, control, legacy)
- `fear_drivers`: What they're afraid of — failure, judgment, wasted effort, dependency, complexity (array)
- `desire_drivers`: What they're moving toward — recognition, ease, control, scale, consistency (array)
- `assumed_identity`: The role they're playing in their mind — builder, expert, entrepreneur, creator, etc.
- `assumed_audience`: Who they imagine will receive or use the output of this request
- `emotional_register`: Detected emotional state from signals — `"frustrated"` | `"excited"` | `"uncertain"` | `"urgent"` | `"calm"` | `"overwhelmed"`
- `unspoken_constraints`: Constraints they have but didn't mention — budget limitations implied by tone, time constraints from urgency signals, etc. (array)

### 5. `constraints` — What limits and shapes the solution space
- `hard_constraints`: Non-negotiable limits — must-nots and must-haves explicitly stated (array)
- `soft_constraints`: Preferences they'd maintain unless pushed — should-nots (array)
- `resource_constraints`: Object with `time`, `budget`, `team_size`, `tooling` (null each if not determinable)
- `platform_constraints`: Technologies, platforms, or systems they're locked into (array)
- `compliance_constraints`: Legal, regulatory, brand, or policy requirements (array)
- `style_constraints`: Tone, format, length, voice requirements (array)
- `inviolable_assumptions`: Things they assume true and will not question — even if those assumptions may be limiting (array)

### 6. `deliverables` — What output they expect to receive
- `primary_deliverable`: The main thing that must exist when done (string)
- `secondary_deliverables`: Supporting outputs — documentation, configs, templates, etc. (array)
- `format_requirements`: File type, structure, length, presentation format (string or object)
- `quality_bar`: What "good enough" looks like in their mental model
- `success_proof`: How they'll know it worked — the testable signal of completion
- `failure_modes`: What outcomes would make them feel this failed (array)

### 7. `workflow` — The process they want to follow or execute
- `preferred_process`: Their described or implied preferred approach (string)
- `ordered_steps`: Extracted sequence of steps if decomposable (array of strings, action verb + object + output)
- `decision_points`: Points in the workflow requiring human judgment (array)
- `automation_candidates`: Steps that could or should be automated (array)
- `human_checkpoints`: Where they must remain involved — review, approval, creative input (array)
- `iteration_model`: `"one_shot"` | `"iterative"` | `"continuous"` | `"undefined"`
- `rollback_plan`: What to do if something goes wrong — if mentioned or implied (string or null)

### 8. `dependencies` — What this depends on and what depends on it
- `upstream_dependencies`: Things that must exist BEFORE this can proceed (array)
- `downstream_dependencies`: Things that depend on the output of this (array)
- `blocking_dependencies`: Hard blockers vs. soft preconditions — mark each (array of objects: `{item, blocking: true|false}`)
- `external_dependencies`: Third parties, services, APIs, or data sources required (array)
- `internal_dependencies`: Other systems, files, or people in their own orbit (array)
- `dependency_graph`: Structured graph representation — array of `{from, to, type, blocking}` objects

### 9. `patterns` — Repeatable processes or systems they want
- `is_recurring`: `true` | `false` | `"unknown"`
- `recurrence_trigger`: What triggers this workflow — event, time, condition, user action
- `recurrence_frequency`: How often this should run (string or null)
- `scalability_intent`: Do they want this to scale to more instances, more users, more inputs?
- `templatable_elements`: Parts that could be abstracted into reusable templates (array)
- `pattern_type`: `"one_time"` | `"repeatable"` | `"systematic"` | `"automated"`

### 10. `vision` — The end-state they're working toward
- `end_state_description`: What does "done" look like in plain language — written as if it already exists
- `time_horizon`: `"short"` (days/weeks) | `"medium"` (months) | `"long"` (year+) | `"undefined"`
- `transformation_delta`: What changes between now and the vision state
- `measurable_outcomes`: Outcomes that can be observed or measured to confirm success (array)
- `stakeholder_impact`: Who else is affected when this is done (string)
- `north_star`: The deepest underlying goal — one sentence, the "why behind the why"

### 11. `tone` — Stylistic and communicative preferences for the output
- `desired_register`: `"formal"` | `"technical"` | `"casual"` | `"conversational"` | `"persuasive"`
- `perspective`: `"first_person"` | `"second_person"` | `"third_person"` | `"imperative"`
- `density`: `"concise"` | `"thorough"` | `"exhaustive"`
- `examples_requested`: `true` | `false` — did they ask for examples or imply they want them?
- `audience_expertise_match`: `"novice"` | `"peer"` | `"expert"` — who the output is for
- `forbidden_patterns`: Things they explicitly hate in outputs — bullet spam, jargon, vague answers (array)

### 12. `success_criteria` — How completion is defined and validated
- `hard_criteria`: Must-pass conditions — binary, no room for interpretation (array)
- `soft_criteria`: Nice-to-have quality signals (array)
- `anti_criteria`: Explicit failure conditions — what makes this a fail even if it "works" (array)
- `verification_method`: How they or someone else will check success (string)
- `acceptance_threshold`: What percentage or quality level is "good enough" (string or null)

### 13. `meta` — Self-referential processing metadata (populated after scoring)
- `schema_version`: `"1.0"`
- `slug`: The derived slug (see Phase 0)
- `processing_timestamp`: ISO 8601 datetime
- `confidence_score`: Populated in Phase 3
- `confidence_rationale`: Populated in Phase 3
- `input_classification`: `"voice_to_text"` | `"stream_of_consciousness"` | `"structured"` | `"mixed"` | `"fragmented"`
- `ambiguity_count`: Number of unresolvable gaps found
- `completeness_flags`: Array of top-level keys that have heavily null-populated sub-fields
- `chat_history_context`: Object — populated only when `CHAT_HISTORY_CONTEXT` is not null. Structure:
  - `provided`: `true` | `false`
  - `source_path`: The value of `CHAT_HISTORY_PATH`, or `null`
  - `prior_goals`: Array of goals extracted from history
  - `prior_decisions`: Array of decisions extracted from history
  - `prior_artifacts`: Array of artifacts extracted from history
  - `open_threads`: Array of unresolved questions or next steps from history
  - `established_context`: Array of background context items established in the prior session
  - If `CHAT_HISTORY_CONTEXT` is null, set `chat_history_context` to `{"provided": false, "source_path": null}`

### 14. `interconnections` — Filled LAST. Cross-parameter dependency graph.
Reference only real key names from this schema. Never invent key names.
- `affects`: Array of `{source_key, target_key, mechanism, strength}` — where strength is `"high"` | `"medium"` | `"low"`
- `blocked_by`: Array of `{parameter, blocking_parameter, reason}` — what can't be determined until something else is known
- `amplifies`: Array of `{key_a, key_b, description}` — parameters that reinforce each other
- `conflicts`: Array of `{key_a, key_b, description}` — parameters in tension
- `resolution_rules`: Array of strings — how to resolve conflicts when parameters clash

---

## PHASE 3 — CONFIDENCE SCORING

After all 14 keys are populated, compute `confidence_score`:

**Base score:**
```
raw_coverage = (count of non-null leaf fields) / (total leaf fields)
```

**Penalties (subtract from raw_coverage):**
- `intent.primary_goal` is null: -0.05
- `intent.implicit` is null: -0.05
- `vision.end_state_description` is null: -0.05
- `deliverables.primary_deliverable` is null: -0.05
- Each unresolved entry in `interconnections.conflicts`: -0.02

**Bonuses (add to penalized score):**
- `intent.explicit` accurately reconstructs input without hallucination or invention: +0.05
- All entries in `constraints.hard_constraints` are non-null and populated: +0.03
- `dependencies.dependency_graph` has at least one entry: +0.02

**Clamp final score to [0.0, 1.0].**

**Critical definition to include in `meta.confidence_rationale`:**
> "This score measures how completely this skill applied its own internal rules — not how certain it is about the user's intent. A score of 0.95 means the skill executed all its processing rules as defined. It does not mean the intent extraction is 95% accurate."

Write the final score and rationale into `meta.confidence_score` and `meta.confidence_rationale`.

---

## PHASE 4 — WRITE VISION FILE

Write `VISION_FILE` using this exact template. Every section must be present. If a section has insufficient information, write "Not determinable from input — [one sentence on what would make this determinable]" rather than omitting the section.

```markdown
# [Inferred Title — the clearest 5–8 word summary of what they're building or doing]

> **Slug:** [slug] | **Processed:** [ISO timestamp] | **Confidence:** [score]

---

## What Was Said

[The cleaned RAW_INPUT. Remove only: filler words (um, like, uh, you know), obvious repeated words, and voice-to-text artifacts. Preserve: all meaning, all ideas, original structure. This is NOT a paraphrase — it is a cleaned transcription.]

---

## What Was Meant

[2–4 sentences. The implicit intent behind the words — what they actually need, not just what they asked for. Written as grounded inference based on signals in the input, not projection. Each sentence should be supportable by pointing to something in the input.]

---

## The Psychoanalytic Layer

[3–6 bullet points. Underlying motivations, fears, desires, identity assumptions, emotional register. Written as grounded inference — not projection or therapy. Each point traces back to a signal in the input.]

- **Emotional register:** [detected state and what signals it]
- **Driving fear:** [what they're moving away from]
- **Core desire:** [what they're moving toward]
- **Assumed identity:** [the role they're playing in their mind]
- **Unspoken constraint:** [something limiting them they didn't name]
- **Deeper motivation:** [the why behind the why]

---

## Primary Goal

[One sentence. The single most important thing they want. Not a summary of everything — the one thing that, if done, makes everything else secondary.]

---

## Secondary Goals

[Ordered bullet list. Each one is a supporting objective. Order by priority, dependency, or intensity of mention.]

- [Goal]
- [Goal]

## Tertiary Goals

[Ordered bullet list. Nice-to-haves, soft objectives.]

- [Goal]
- [Goal]

---

## The Vision State

[2–4 sentences of present-tense prose. Describe what "done" looks like as if it already exists. The reader should be able to picture the final state clearly. This is the north star.]

---

## Measurable Outcomes

[Bullet list of things that can be observed, tested, or checked to confirm success.]

- [Outcome]
- [Outcome]

---

## Constraint Map

| Type | Constraint | Source | Flexibility |
|------|------------|--------|-------------|
| Hard | [constraint] | explicit / inferred | none |
| Soft | [constraint] | explicit / inferred | high / medium / low |
| Resource | [constraint] | explicit / inferred | [value if known] |
| Platform | [constraint] | explicit / inferred | [value if known] |

---

## Workflow Decomposition

[Numbered steps. Each step: action verb + object + output produced. Be specific.]

1. [Step]
2. [Step]
3. [Step]

**Decision points:** [List steps requiring human judgment, or "None identified"]
**Automation candidates:** [List steps that could be automated, or "None identified"]
**Iteration model:** [one_shot / iterative / continuous / undefined]

---

## Dependency Chain

```
[upstream dependency] → [this work] → [downstream dependency]
```

- **Blocking (must exist first):** [list or "None identified"]
- **External (third-party):** [list or "None identified"]
- **Internal (their own systems/files/people):** [list or "None identified"]

---

## Pattern Analysis

- **Recurring?** [Yes / No / Unknown — and why]
- **Pattern type:** [one_time / repeatable / systematic / automated]
- **Recurrence trigger:** [what kicks this off, or "Not determinable"]
- **Templatable elements:** [list of parts that could become reusable templates, or "None identified"]
- **Scalability intent:** [do they want this to scale — evidence from input]

---

## Open Gaps

[Numbered list. Each gap: what's missing + what information would close it. Order by importance — the most impactful gap first.]

1. **[Gap name]** — [What's unknown] | Needs: [what information would resolve this]
2. **[Gap name]** — [What's unknown] | Needs: [what information would resolve this]

If no gaps exist, write: "No critical gaps identified. Input was sufficient to populate all schema keys with high confidence."

---

## Recommended Next Prompt

[Single most valuable follow-up question. The one question that, if answered, would close the most important gap and most significantly improve the quality of the profile. Written as a direct question the user should copy-paste and answer. NOT a list of questions — exactly one.]

> [The question]

---

## Handoff Context

[Include this section ONLY if `CHAT_HISTORY_CONTEXT` is not null. If no chat history was provided, omit this section entirely.]

**Source:** `[CHAT_HISTORY_PATH]`

**Prior goals carried forward:**
[Bullet list from HISTORY_SIGNALS.prior_goals, or "None extracted"]

**Key decisions made:**
[Bullet list from HISTORY_SIGNALS.prior_decisions, or "None extracted"]

**Artifacts produced:**
[Bullet list from HISTORY_SIGNALS.prior_artifacts, or "None extracted"]

**Open threads:**
[Bullet list from HISTORY_SIGNALS.open_threads, or "None extracted"]

**Established context:**
[Bullet list from HISTORY_SIGNALS.established_context, or "None extracted"]
```

---

## PHASE 5 — WRITE JSON PROFILE

Write `PROFILE_FILE` as valid JSON with the following structure. Every top-level key must be present. Use `null` for fields that cannot be populated. Add `_confidence: 0.0` sibling fields alongside any `null` value for a field that you attempted to populate but could not.

```json
{
  "meta": {
    "schema_version": "1.0",
    "slug": "[slug]",
    "processing_timestamp": "[ISO 8601]",
    "confidence_score": [0.0–1.0],
    "confidence_rationale": "[text — must include the self-referential definition of the score]",
    "input_classification": "[voice_to_text|stream_of_consciousness|structured|mixed|fragmented]",
    "ambiguity_count": [integer],
    "completeness_flags": ["[key names of sparse sections]"],
    "chat_history_context": {
      "provided": "[true|false]",
      "source_path": "[path or null]",
      "prior_goals": ["[goal extracted from history]"],
      "prior_decisions": ["[decision extracted from history]"],
      "prior_artifacts": ["[artifact extracted from history]"],
      "open_threads": ["[unresolved question or next step from history]"],
      "established_context": ["[background context item from history]"]
    }
  },
  "intent": {
    "explicit": "[verbatim cleaned reconstruction]",
    "implicit": "[inferred real need]",
    "primary_goal": "[one sentence]",
    "secondary_goals": ["[goal]"],
    "tertiary_goals": ["[goal]"],
    "anti_goals": ["[what they don't want]"],
    "trigger_event": "[what caused this]",
    "urgency_signal": "[immediate|short_term|long_term|undefined]"
  },
  "psychoanalysis": {
    "surface_motivation": "[what they think they want]",
    "deeper_motivation": "[why they actually want it]",
    "fear_drivers": ["[fear]"],
    "desire_drivers": ["[desire]"],
    "assumed_identity": "[role they're playing]",
    "assumed_audience": "[who receives the output]",
    "emotional_register": "[frustrated|excited|uncertain|urgent|calm|overwhelmed]",
    "unspoken_constraints": ["[constraint]"]
  },
  "domain": {
    "primary_domain": "[domain]",
    "sub_domain": "[sub-domain]",
    "domain_maturity": "[novice|practitioner|expert]",
    "domain_jargon_used": ["[term]"],
    "industry_context": "[industry]",
    "cross_domain_dependencies": ["[domain]"]
  },
  "persona": {
    "operator_role": "[role]",
    "operator_experience_level": "[inferred experience with this task]",
    "team_context": "[solo|small_team|large_team|enterprise|undefined]",
    "decision_authority": "[alone|needs approval|unknown]",
    "technical_fluency": "[non_technical|semi_technical|technical|expert]",
    "communication_style": "[formal|casual|technical|stream_of_consciousness|voice_dictation]"
  },
  "constraints": {
    "hard_constraints": ["[constraint]"],
    "soft_constraints": ["[constraint]"],
    "resource_constraints": {
      "time": null,
      "budget": null,
      "team_size": null,
      "tooling": null
    },
    "platform_constraints": ["[platform]"],
    "compliance_constraints": ["[requirement]"],
    "style_constraints": ["[style rule]"],
    "inviolable_assumptions": ["[assumption]"]
  },
  "deliverables": {
    "primary_deliverable": "[main output]",
    "secondary_deliverables": ["[output]"],
    "format_requirements": "[format spec]",
    "quality_bar": "[what good enough looks like]",
    "success_proof": "[testable signal of completion]",
    "failure_modes": ["[failure condition]"]
  },
  "workflow": {
    "preferred_process": "[their implied approach]",
    "ordered_steps": ["[step]"],
    "decision_points": ["[decision]"],
    "automation_candidates": ["[step]"],
    "human_checkpoints": ["[checkpoint]"],
    "iteration_model": "[one_shot|iterative|continuous|undefined]",
    "rollback_plan": null
  },
  "dependencies": {
    "upstream_dependencies": ["[dependency]"],
    "downstream_dependencies": ["[dependency]"],
    "blocking_dependencies": [
      {"item": "[dependency]", "blocking": true}
    ],
    "external_dependencies": ["[service/API/tool]"],
    "internal_dependencies": ["[system/file/person]"],
    "dependency_graph": [
      {"from": "[node]", "to": "[node]", "type": "[causal|constraint|data|trigger]", "blocking": true}
    ]
  },
  "patterns": {
    "is_recurring": "[true|false|unknown]",
    "recurrence_trigger": null,
    "recurrence_frequency": null,
    "scalability_intent": "[description or null]",
    "templatable_elements": ["[element]"],
    "pattern_type": "[one_time|repeatable|systematic|automated]"
  },
  "vision": {
    "end_state_description": "[present-tense description of done]",
    "time_horizon": "[short|medium|long|undefined]",
    "transformation_delta": "[what changes between now and done]",
    "measurable_outcomes": ["[outcome]"],
    "stakeholder_impact": "[who else is affected]",
    "north_star": "[the deepest underlying goal — one sentence]"
  },
  "tone": {
    "desired_register": "[formal|technical|casual|conversational|persuasive]",
    "perspective": "[first_person|second_person|third_person|imperative]",
    "density": "[concise|thorough|exhaustive]",
    "examples_requested": true,
    "audience_expertise_match": "[novice|peer|expert]",
    "forbidden_patterns": ["[pattern]"]
  },
  "success_criteria": {
    "hard_criteria": ["[criterion]"],
    "soft_criteria": ["[criterion]"],
    "anti_criteria": ["[criterion]"],
    "verification_method": "[how to check]",
    "acceptance_threshold": null
  },
  "interconnections": {
    "affects": [
      {"source_key": "[key]", "target_key": "[key]", "mechanism": "[description]", "strength": "[high|medium|low]"}
    ],
    "blocked_by": [
      {"parameter": "[key.field]", "blocking_parameter": "[key.field]", "reason": "[why]"}
    ],
    "amplifies": [
      {"key_a": "[key]", "key_b": "[key]", "description": "[how they reinforce each other]"}
    ],
    "conflicts": [
      {"key_a": "[key]", "key_b": "[key]", "description": "[how they're in tension]"}
    ],
    "resolution_rules": ["[rule for resolving a conflict]"]
  }
}
```

---

## PHASE 6 — COMPLETION REPORT

Display exactly this in chat and nothing else:

```
intent:analyze-draft complete.

Vision:  [absolute path to VISION_FILE]
Profile: [absolute path to PROFILE_FILE]

Slug:       [slug]
Confidence: [score] ([10-word rationale summary])
Open gaps:  [count] (see "Open Gaps" in vision file)
```

Do not add explanation. Do not summarize what you found. The files speak for themselves.