---
name: build-prompt
description: "Step 3 of the cc-workflow pipeline. Receives a file path to step-02-profile.json (written by the extract-profile subagent). Reads the profile JSON and vision document referenced within, produces a self-contained execution prompt file on disk. Writes step-03-prompt.json as output signal to parent. All inter-step data travels via files — never via parent context."
argument-hint: "[path to step-02-profile.json produced by /extract-profile subagent]"
allowed-tools: Read, Write
---

You are executing the `/build-prompt` command as a subagent in the cc-workflow pipeline. Your job: read the profile JSON and vision document, produce a self-contained execution prompt file, write it to disk.

---

## PHASE 0 — SETUP

**Parse arguments:**
- Read the file at $ARGUMENTS. Parse as JSON.
- Extract `profile_file` — absolute path to the [slug]-profile.json
- Extract `vision_file` — absolute path to the [slug]-vision.md
- If either path is missing or files cannot be read: output "Missing or unreadable source files. Verify step-02-profile.json is valid." and stop.

**Read and validate:**
1. Read the vision file. Store as VISION.
2. Read the profile JSON. Parse and store as PROFILE.
3. Extract from PROFILE: meta.slug, meta.confidence_score, meta.completeness_flags, intent.primary_goal, intent.anti_goals, deliverables.primary_deliverable, deliverables.secondary_deliverables, deliverables.success_proof, deliverables.failure_modes, constraints.hard_constraints, workflow.ordered_steps, workflow.decision_points, workflow.human_checkpoints, success_criteria.hard_criteria, success_criteria.anti_criteria, dependencies.blocking_dependencies, vision.north_star
4. If meta.confidence_score < 0.70: note — will surface as warning in output prompt.

**Define output paths:**
OUTPUT_PROMPT_FILE = [same directory as vision file]/exec-[slug].md
OUTPUT_JSON_FILE = [same directory as vision file]/step-03-prompt.json

---

## PHASE 1 — GENERATE OUTPUT PROMPT

Write the following content to OUTPUT_PROMPT_FILE. Substitute every [FIELD] with actual extracted values. If a field value is null, write "Not specified — use your best judgment based on context."

Output prompt content:

```
# Execution Brief — [meta.slug]

> **Source vision:** `[absolute path to vision file]`
> **Source profile:** `[absolute path to profile JSON]`
> **Extraction confidence:** [meta.confidence_score] — [if >= 0.85: "High — proceed with confidence." | if 0.70-0.84: "Moderate — flag ambiguities as you go." | if < 0.70: "LOW — review completeness flags before starting: [meta.completeness_flags]"]

---

## Read These Files First

Before doing anything else, read both source files completely:
1. Read the vision document at: `[absolute path to vision file]`
2. Read the JSON profile at: `[absolute path to profile JSON]`

---

## What You Are Building

**Primary deliverable:** [deliverables.primary_deliverable]
**The goal in one sentence:** [intent.primary_goal]
**The north star:** [vision.north_star]

**Secondary deliverables:**
[each item in deliverables.secondary_deliverables as a bullet]

---

## How to Use the JSON Profile

- `intent.explicit` — The verbatim reconstructed request. Your anchor.
- `intent.implicit` — The real need behind the stated request.
- `intent.primary_goal` — The single most important objective.
- `intent.secondary_goals` — Supporting objectives, priority-ordered.
- `intent.tertiary_goals` — Nice-to-haves, include only if no added complexity.
- `intent.anti_goals` — What NOT to build. Treat as hard constraints.
- `constraints.hard_constraints` — Non-negotiable limits.
- `constraints.soft_constraints` — Preferences to maintain unless strong reason not to.
- `workflow.ordered_steps` — Execute in this order.
- `workflow.decision_points` — State decision before proceeding.
- `workflow.human_checkpoints` — Pause and ask for input.
- `success_criteria.hard_criteria` — Must-pass conditions.
- `success_criteria.anti_criteria` — Conditions that make output a failure.
- `psychoanalysis.fear_drivers` — What the user fears. Avoid triggering these.
- `psychoanalysis.desire_drivers` — What the user wants. Favor these.

---

## Execution Sequence

1. Read both source files completely.
2. Verify blocking dependencies from `dependencies.blocking_dependencies`.
3. Review Open Gaps from the vision document.
4. Execute `workflow.ordered_steps` in sequence.
5. At each `workflow.decision_points` entry: state your decision before executing.
6. At each `workflow.human_checkpoints` entry: pause and ask for input.
7. Produce all secondary deliverables.
8. Verify against `success_criteria.hard_criteria`.
9. Check `deliverables.failure_modes` and `success_criteria.anti_criteria`.
10. Apply `deliverables.success_proof`.

---

## Hard Constraints

[each item in constraints.hard_constraints as a bullet with "MUST:" prefix]

---

## Anti-Goals

[each item in intent.anti_goals as a bullet with "DO NOT:" prefix]

---

## Failure Conditions

[each item in deliverables.failure_modes as a bullet]
[each item in success_criteria.anti_criteria as a bullet]

---

## You Are Done When

[deliverables.success_proof]

And all of these pass:
[each item in success_criteria.hard_criteria as a numbered item]
```

---

## PHASE 2 — WRITE OUTPUT FILES

1. Write OUTPUT_PROMPT_FILE using the Write tool with the populated prompt content.
2. Do not print the prompt content to chat.
3. Write OUTPUT_JSON_FILE:
```json
{
  "step": "03-build-prompt",
  "slug": "[slug]",
  "prompt_file": "[absolute path to OUTPUT_PROMPT_FILE]",
  "completed_at": "[ISO8601 timestamp]"
}
```

---

## PHASE 3 — COMPLETION REPORT

Display to user:
- Output prompt file path
- Confidence score
- step-03-prompt.json path
- Next step (handled by parent agent)