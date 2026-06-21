---
name: extract-profile
description: "Step 2 of the cc-workflow pipeline. Receives a file path to step-01-intake.json (written by the intake subagent). Reads the intake .md file referenced within, maps its content to a JSON profile (14-key schema) and a vision document (5-section format). All inter-step data travels via files on disk — never via parent context. Writes step-02-profile.json as output signal to parent."
argument-hint: "[path to step-01-intake.json produced by /intake subagent]"
allowed-tools: Read, Write, Edit
---

You are executing the `/extract-profile` command as a subagent in the cc-workflow pipeline. Your job: read the intake document, map its content to the JSON profile schema and vision document format, and write all output files to disk.

---

## PHASE 0 — SETUP

**Parse arguments:**
- If $ARGUMENTS is a path ending in .json: Read the file. Parse as JSON. Extract `intake_file` field — this is the path to the intake markdown. Extract `clarification_mode_file` field.
- If $ARGUMENTS is a path ending in .md: treat it as the intake file directly (legacy invocation).
- If $ARGUMENTS is empty: output "Missing argument. Provide path to step-01-intake.json" and stop.

**Read clarification mode:**
Read the clarification_mode_file path. If file exists, parse and extract `clarification_mode`. If missing, set CLARIFICATION_MODE = "yes" (safe default — allows questions if needed).
Do NOT call AskUserQuestion if CLARIFICATION_MODE = "no".

**Read intake document:**
Read the intake .md file at the extracted path. Store full contents as INTAKE.

**Validate intake document:**
Check that INTAKE contains all of these section headers:
- "## Original Input"
- "## Research Findings"
- "## Signal Extraction"
- "## Q&A Session"
- "## Synthesis"

If any section is missing: output "Intake document is missing required sections. Re-run /01-intake to regenerate." and stop.

**Extract slug:**
From the first line of INTAKE: "# Intake: [slug]" — extract the slug value.

**Define output paths:**
PROFILE_FILE = [same directory as INTAKE]/[slug]-profile.json
VISION_FILE = [same directory as INTAKE]/[slug]-vision.md
OUTPUT_JSON_FILE = [same directory as INTAKE]/step-02-profile.json

---

## PHASE 1 — JSON PROFILE EXTRACTION

Populate all 14 keys of the JSON schema. For every field, follow this strict source priority:
1. EXPLICIT: A directly stated fact or a Q&A answer from the ## Q&A Session section (highest trust)
2. CONFIRMED: A "Confirmed Facts" item from ## Synthesis
3. INFERRED: A "High-Confidence Inferences" item from ## Synthesis (label with _confidence < 0.9)
4. RESEARCH: A finding from ## Research Findings
5. null: If no source provides this value (never fabricate)

For each null field, add a sibling field named [fieldname]_confidence with value 0.0.

**Field mappings — extract from INTAKE:**

`intent` key: explicit, implicit, primary_goal, secondary_goals, tertiary_goals, anti_goals, trigger_event, urgency_signal
`domain` key: primary_domain, sub_domain, domain_maturity, domain_jargon_used, industry_context, cross_domain_dependencies
`persona` key: operator_role, operator_experience_level, team_context, decision_authority, technical_fluency, communication_style
`psychoanalysis` key: surface_motivation, deeper_motivation, fear_drivers, desire_drivers, assumed_identity, assumed_audience, emotional_register, unspoken_constraints
`constraints` key: hard_constraints, soft_constraints, resource_constraints, platform_constraints, compliance_constraints, style_constraints, inviolable_assumptions
`deliverables` key: primary_deliverable, secondary_deliverables, format, quality_bar, success_proof, failure_modes
`workflow` key: preferred_process, ordered_steps, decision_points, automation_candidates, human_checkpoints, iteration_model, rollback
`dependencies` key: upstream_dependencies, downstream_dependencies, blocking_dependencies, external_dependencies, internal_dependencies, dependency_graph
`patterns` key: is_recurring, trigger, frequency, scalability, templatable_elements, pattern_type
`vision` key: end_state_description, time_horizon, transformation_delta, measurable_outcomes, stakeholder_impact, north_star
`tone` key: desired_register, perspective, density, examples, audience_expertise_match, forbidden_patterns
`success_criteria` key: hard_criteria, soft_criteria, anti_criteria, verification_method, acceptance_threshold
`meta` key: schema_version ("1.0.0"), slug, timestamp (ISO8601), confidence_score, confidence_rationale, input_classification ("intake_v1_output"), ambiguity_count, completeness_flags, chat_history_context
`interconnections` key: affects, blocked_by, amplifies, conflicts, resolution_rules

**Confidence score:** Base = (non-null fields) / (total fields). Penalty = -0.03 per critical null (intent.primary_goal, deliverables.primary_deliverable, constraints.hard_constraints, success_criteria.hard_criteria). Clamp to [0.0, 1.0].

---

## PHASE 2 — VISION DOCUMENT EXTRACTION

Generate [slug]-vision.md using the 5-section format. Populate every section from INTAKE content only — do not add information not present in the intake document.

**Section 0:** Re-Articulated Original Input — source: ## Original Input. Clean prose, preserve all intent.
**Section 1:** Summary — source: Synthesis + Confirmed Facts + Q&A. Narrative prose 400-600 words. Covers: core problem, proposed solution, key workflow, primary goals, dependencies.
**Section 2:** Key Points — source: Confirmed Facts + Q&A + Research. 5-8 ## categories, 20-35 bullets using • character. Specific to this idea, not generic.
**Section 3:** Structured Summary — source: All sections. 8-12 ## headers, 2-5 ### subsections each, 2-5 bullets per subsection.
**Section 4:** Complete Vision Expansion — all 6 required ### subsections: Explicit Non-Goals, Edge Case Handling, Key Design Principles, Component or Document Relationships, Success Criteria, Open Questions and Assumptions.

---

## PHASE 3 — WRITE OUTPUT FILES

**Write profile JSON:**
1. Create PROFILE_FILE blank using Write tool
2. Write complete JSON using Edit tool
3. Do not display JSON in chat

**Write vision document:**
1. Create VISION_FILE blank using Write tool
2. Write document in sections using Edit tool (one Edit call per section)
3. Do not display vision document in chat

**Write step-02 output JSON:**
Write OUTPUT_JSON_FILE:
```json
{
  "step": "02-extract-profile",
  "slug": "[slug]",
  "profile_file": "[absolute path to PROFILE_FILE]",
  "vision_file": "[absolute path to VISION_FILE]",
  "confidence_score": [N],
  "completeness_flags": [...],
  "completed_at": "[ISO8601 timestamp]"
}
```

---

## PHASE 4 — COMPLETION REPORT

Display to user:
- Profile JSON path + confidence score
- Vision document path
- Number of null fields
- step-02-profile.json path
- Next step (handled by parent agent)