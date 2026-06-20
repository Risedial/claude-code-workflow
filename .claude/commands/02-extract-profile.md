---
name: extract-profile
description: "Step 2 of 3 in the enhanced intake pipeline. Reads a [slug]-intake.md produced by /intake and maps its explicitly-captured content to two output files: [slug]-profile.json (14-key schema) and [slug]-vision.md (5-section format). No research, no Q&A — pure mapping from structured intake source to structured targets. Feed both outputs to /intent:build-prompt."
argument-hint: "[path to [slug]-intake.md produced by /intake]"
allowed-tools: Read, Write, Edit
---

You are executing the `/extract-profile` command. Your single job is to extract: read the structured intake document, map its content to the JSON profile schema and the vision document format, and write both files. You do not research, ask questions, or infer. Every value you write must come from the intake document.

---

## PHASE 0 — SETUP

**Parse arguments:**
- If $ARGUMENTS is a path ending in .md: Read the file. Store full contents as INTAKE.
- If $ARGUMENTS is empty: use AskUserQuestion to ask "Provide the path to your [slug]-intake.md file."

**Validate intake document:**
Check that INTAKE contains all of these section headers:
- "## Original Input"
- "## Research Findings"
- "## Signal Extraction"
- "## Q&A Session"
- "## Synthesis"

If any section is missing: output "Intake document is missing required sections. Re-run /intake to regenerate." and stop.

**Extract slug:**
From the first line of INTAKE: "# Intake: [slug]" — extract the slug value.

**Define output paths:**
PROFILE_FILE = [same directory as INTAKE]/[slug]-profile.json
VISION_FILE = [same directory as INTAKE]/[slug]-vision.md

---

## PHASE 1 — JSON PROFILE EXTRACTION

Populate all 14 keys of the JSON schema. For every field, follow this strict source priority:
1. EXPLICIT: A directly stated fact or a Q&A answer from the ## Q&A Session section (highest trust)
2. CONFIRMED: A "Confirmed Facts" item from ## Synthesis
3. INFERRED: A "High-Confidence Inferences" item from ## Synthesis (label it with _confidence < 0.9)
4. RESEARCH: A finding from ## Research Findings
5. null: If no source provides this value (never fabricate)

For each null field, add a sibling field named [fieldname]_confidence with value 0.0.

**Field mappings — extract these values from INTAKE:**

`intent` key:
- explicit: from "## Original Input" — the cleaned restatement of the stated request
- implicit: from Implied Intent bullets in Signal Extraction — what was meant but not said
- primary_goal: from Q&A answer to the "Primary goal" or "Success vision" question, or from Confirmed Facts item that states the core outcome
- secondary_goals: from Q&A answers or Confirmed Facts that describe supporting objectives (array)
- tertiary_goals: from High-Confidence Inferences about nice-to-haves (array)
- anti_goals: read every Q&A answer's implied non-choices (what was rejected by selecting a different option) + any explicit statements in Original Input about what is not wanted (array)
- trigger_event: from ## Urgency and Trigger section
- urgency_signal: from ## Urgency and Trigger — map to "immediate" | "short_term" | "long_term" | "undefined"

`domain` key:
- primary_domain: from Confirmed Facts or Q&A answer about scope/audience
- sub_domain: from Domain Vocabulary section + Q&A answers about scope
- domain_maturity: infer from Emotional Register + how technically the Q&A options were phrased — "novice" | "practitioner" | "expert"
- domain_jargon_used: all terms listed in Domain Vocabulary that appeared in the original input (not just research-surfaced terms)
- industry_context: from Q&A audience answer or Original Input
- cross_domain_dependencies: from Known Challenges and Existing Solutions where adjacent domains are mentioned

`persona` key:
- operator_role: from Q&A audience answer or Confirmed Facts
- operator_experience_level: from domain_maturity inference + Q&A answer sophistication
- team_context: from Q&A constraint answer — "solo" | "small_team" | "large_team" | "enterprise" | "undefined"
- decision_authority: from Q&A constraint answer — can they decide alone?
- technical_fluency: from language in Original Input + Q&A answer choices selected — "non_technical" | "semi_technical" | "technical" | "expert"
- communication_style: from Emotional Register + Original Input tone — "formal" | "casual" | "technical" | "stream_of_consciousness" | "voice_dictation"

`psychoanalysis` key:
- surface_motivation: from Confirmed Facts — what they stated as the goal
- deeper_motivation: from Fear Signals + Desire Signals — what those signals reveal about deeper purpose
- fear_drivers: from Fear Signals section — each signal becomes a fear driver (array)
- desire_drivers: from Desire Signals section — each signal becomes a desire driver (array)
- assumed_identity: the role the user imagines themselves in, inferred from language and Q&A choices
- assumed_audience: from Q&A audience question answer
- emotional_register: directly from Signal Extraction → Emotional Register value
- unspoken_constraints: from Missing Context Signals + Gaps and Open Questions items that are implicitly constrained

`constraints` key:
- hard_constraints: from Q&A answers about constraints + Explicit Statements about non-negotiable requirements (array)
- soft_constraints: from High-Confidence Inferences about preferences with some flexibility (array)
- resource_constraints: from Q&A constraint answer — time, budget, team size
- platform_constraints: from Q&A answers or Explicit Statements about technology requirements
- compliance_constraints: from Known Challenges related to legal/regulatory/policy, if any
- style_constraints: from Q&A priority answer about quality vs speed vs simplicity
- inviolable_assumptions: from Confirmed Facts items that, if wrong, would entirely change the approach (array)

`deliverables` key:
- primary_deliverable: from Q&A primary goal answer + Confirmed Facts — the main thing being produced
- secondary_deliverables: from Q&A answers or Confirmed Facts about supporting outputs (array)
- format: the expected form (app / document / script / service / API / workflow / etc.)
- quality_bar: from Q&A priority answer — what level of quality is acceptable
- success_proof: how would the user know it worked — from Q&A "feared outcome" answer (the inverse)
- failure_modes: from Known Challenges + Q&A "feared outcome" answer directly

`workflow` key:
- preferred_process: from Q&A priority answer — iterative / big-bang / phased / prototype-first
- ordered_steps: any sequential process described in Original Input or implied by Q&A answers (array)
- decision_points: from Contradictions and Gaps — places where a choice must be made
- automation_candidates: from Desire Signals with "automatically" or "always" signals (array)
- human_checkpoints: places where human review is implied or required
- iteration_model: from Q&A priority answer — continuous / milestone-based / one-shot
- rollback: from Known Challenges — what happens if a step fails

`dependencies` key:
- upstream_dependencies: inputs this system requires from elsewhere
- downstream_dependencies: what depends on this system's output
- blocking_dependencies: from Q&A constraint answer or Confirmed Facts — must exist before this can work
- external_dependencies: from Existing Solutions and Standard Approaches that this integrates with (array)
- internal_dependencies: internal teams or systems mentioned in Original Input or Chat History Context
- dependency_graph: a brief text map of the major relationships described

`patterns` key:
- is_recurring: from Desire Signals with "always", "every time", "automatically" — true if present
- trigger: what causes this to run — from Desire Signals or Original Input
- frequency: from Original Input or Q&A answers if time-based recurrence is described
- scalability: growth expectations from Q&A or Original Input
- templatable_elements: components that repeat across instances, from Standard Approaches
- pattern_type: "one_time" | "recurring" | "event_driven" | "continuous"

`vision` key:
- end_state_description: from Q&A primary goal answer — what the world looks like when this is done
- time_horizon: from Q&A constraint answer or Urgency and Trigger
- transformation_delta: the change from current state to end state — derived from Fear Signals (current pain) + Desire Signals (target state)
- measurable_outcomes: from Confirmed Facts about what success looks like in concrete terms
- stakeholder_impact: who benefits and how — from Q&A audience answer
- north_star: a single sentence capturing the essence of success — synthesize from Q&A primary goal + Confirmed Facts

`tone` key:
- desired_register: from Q&A priority answer or Original Input communication style
- perspective: first-person / instructional / technical / narrative
- density: concise vs comprehensive — from Q&A priority answer
- examples: whether examples are needed — from Q&A or Original Input
- audience_expertise_match: from Q&A audience answer
- forbidden_patterns: from anti_goals that are style-related

`success_criteria` key:
- hard_criteria: measurable, testable conditions from Q&A "success vision" answer + Confirmed Facts (array — phrase each as an assertion: "X is done when Y")
- soft_criteria: quality expectations from Q&A priority answer (array)
- anti_criteria: conditions that would indicate failure — from Q&A "feared outcome" answer + Fear Signals (array)
- verification_method: how to test or observe that success conditions are met
- acceptance_threshold: minimum viable success — from Q&A or Confirmed Facts

`meta` key:
- schema_version: "1.0.0"
- slug: [slug from INTAKE header]
- timestamp: ISO 8601 current timestamp
- confidence_score: calculate as follows:
  Base = (number of non-null fields) / (total fields across all 14 keys)
  Penalty = -0.03 for each critical null (intent.primary_goal, deliverables.primary_deliverable, constraints.hard_constraints, success_criteria.hard_criteria)
  Clamp to [0.0, 1.0]
- confidence_rationale: "Populated from /intake output. Primary sources: explicit Q&A answers and confirmed facts. Secondary sources: research findings and inferences."
- input_classification: "intake_v1_output"
- ambiguity_count: count of items in ## Gaps and Open Questions from INTAKE
- completeness_flags: array of field names that are null (so build-prompt knows where to hedge)
- chat_history_context: { provided: true/false, summary: [first 200 chars of Chat History Context section if provided, else null] }

`interconnections` key:
- affects: components this idea changes or produces outputs for
- blocked_by: dependencies that must exist before this can run
- amplifies: other tools or systems this works well alongside
- conflicts: from Contradictions section — unresolved conflicts
- resolution_rules: from Q&A answers that resolved a conflict, or from Confirmed Facts

**Confidence score calculation example:** If 95 out of 110 total fields are non-null, base = 0.864. If 2 critical fields are null, penalty = 0.06. Final = 0.804.

---

## PHASE 2 — VISION DOCUMENT EXTRACTION

Generate [slug]-vision.md using the 5-section format. Populate every section from INTAKE content — do not add information not in the intake document.

**Section 0: Re-Articulated Original Input**
Source: ## Original Input section from INTAKE. Write as clean prose — preserve all intent, remove remaining artifacts.

**Section 1: Summary**
Source: Synthesis → Confirmed Facts + Implied Intent + Q&A answers. Write as narrative prose (400–600 words for complex ideas, 150–300 for simple ones). Must cover: the core problem, proposed solution, key workflow, primary goals, integrations or dependencies mentioned. No bullet points.

**Section 2: Key Points**
Source: Confirmed Facts + Q&A answers + Research Findings. Organize into 5–8 categories (## headers). 20–35 bullets total using • character. Each bullet must be specific to this idea — no generic statements. Category names should reflect this specific idea's content.

**Section 3: Structured Summary**
Source: All sections of INTAKE. 8–12 major sections (## headers), 2–5 subsections each (### headers), 2–5 bullets per subsection. Coverage must include: purpose, philosophy, audience, input handling, core processes, outputs, technical requirements, constraints, edge cases, integrations, quality standards. Use Research → Known Challenges to add non-obvious angles.

**Section 4: Complete Vision Expansion**
All 6 required subsections (### headers):

### Explicit Non-Goals
Source: intent.anti_goals + Q&A answers where options were rejected + Missing Context Signals about common misinterpretations. Minimum 5 items. Format: "[System] does not [action]."

### Edge Case Handling
Source: Known Challenges + Gaps and Open Questions + Q&A "feared outcome" answer. Minimum 5 scenarios. Format: **Bold scenario name** followed by how the system handles it.

### Key Design Principles
Source: Confirmed Facts about non-negotiable requirements + Q&A priority answers. Minimum 4 principles. Format: numbered list, **bold name**, explanation.

### Component or Document Relationships
Source: Dependency graph from dependencies key + any workflow described. Use ASCII, indented lists, or → arrows. Every major component mentioned in Sections 2 and 3 must appear.

### Success Criteria
Source: success_criteria.hard_criteria. Format: "• [X] is successful when [measurable condition]."

### Open Questions and Assumptions
Source: Gaps and Open Questions from INTAKE + meta.completeness_flags. Minimum 5 items. Format: numbered list. For each: (1) what is unknown or assumed, (2) the assumption being made, (3) flag if wrong assumption significantly changes output.

---

## PHASE 3 — WRITE OUTPUT FILES

**Write profile JSON:**
1. Create PROFILE_FILE blank using Write tool
2. Write complete JSON using Edit tool
3. Do not display JSON in chat

**Write vision document:**
1. Create VISION_FILE blank using Write tool
2. Write document in sections using Edit tool (one Edit call per section, never exceed 20,000 tokens per call)
   - Edit 1: Title + --- + Section 0
   - Edit 2: --- + Section 1
   - Edit 3: --- + Section 2
   - Edit 4: --- + Section 3
   - Edit 5: --- + Section 4
3. Do not display vision document in chat

---

## PHASE 4 — COMPLETION REPORT

Display to user:
- Profile JSON path + confidence score with rationale
- Vision document path
- Number of null fields (completeness_flags count)
- Any critical gaps to be aware of (completeness_flags for intent, deliverables, constraints, success_criteria keys)
- Next step: "/03-build-prompt [vision-file-path] [profile-file-path]"
