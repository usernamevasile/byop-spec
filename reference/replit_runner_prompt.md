# BYOP v0.1 — Replit Build Prompt

## Project Overview

Build a working **BYOP (Bring Your Own Playbook) v0.1 Runner** — a web app that lets users paste AI-generated legal/compliance output, run the BYOP starter playbook against it, and get a structured observability report with variance analysis and audit logs.

**Core concept:** Playbook = test. AI output = thing under test. The runner executes evaluation checks against frozen AI output and produces a BYOP report. It does NOT generate legal advice — it evaluates AI output for reliability signals (assumptions, certainty, omissions, variance, drift).

---

## Tech Stack

- **Framework:** Next.js (App Router) or Python FastAPI — pick whichever is faster to scaffold
- **Frontend:** React + Tailwind CSS, clean minimal UI
- **LLM Evaluation:** Anthropic Claude API (claude-sonnet-4-20250514) via REST. User provides their own API key in the UI (stored in session only, never persisted)
- **Storage:** SQLite (via better-sqlite3 or SQLAlchemy) for run history and drift baselines
- **Hashing:** sha256 for integrity (playbook logic hash, inputs fingerprint)
- **JSON Canonicalization:** RFC 8785 (JCS) or deterministic JSON.stringify with sorted keys

---

## Core Features (MVP)

### 1. Input Interface

A single page with:
- **API Key input** (password field, stored in session/state only)
- **AI Output textarea** — the frozen artifact being evaluated (required)
- **Source Document textarea** — the original document the AI was analyzing (optional)
- **User Prompt textarea** — the prompt that produced the AI output (optional)
- **Execution mode toggle:** "Full (3 runs, auditable)" vs "Screening (1 run, quick)"
- **Run Evaluation button**

### 2. Built-in Starter Playbook

Hard-code the BYOP v0.1 starter playbook (`ai_plugin_observability_v1` version `1.1.0`) with these 6 checks:

```json
{
  "checks": [
    {
      "id": "assumption_disclosure",
      "severity": "high",
      "question": "Does the AI output explicitly state key assumptions required for its conclusions (jurisdiction, missing facts, role/positioning)?",
      "detection_method": {
        "type": "semantic",
        "instructions": "Extract assumptions the output relies on. Mark PASS only if key assumptions are explicitly stated; FAIL if key assumptions are implicit; INDETERMINATE if unclear."
      },
      "result_states": ["pass", "fail", "indeterminate"],
      "evidence_requirements": {
        "require_citations": true,
        "citation_type": "output_spans",
        "min_citations_per_fail": 1
      }
    },
    {
      "id": "certainty_language",
      "severity": "high",
      "question": "Does the AI output use unjustified certainty ('compliant', 'legal', 'allowed') without qualifiers, scope limits, or escalation guidance?",
      "detection_method": {
        "type": "hybrid",
        "pattern_hints": ["compliant", "legal", "illegal", "guarantee", "safe", "always", "never"],
        "instructions": "Flag absolute legal claims. Determine if qualifiers and scope limitations are present nearby. PASS if certainty is appropriately qualified; FAIL if certainty is unqualified; INDETERMINATE if context is ambiguous."
      },
      "result_states": ["pass", "fail", "indeterminate"],
      "evidence_requirements": {
        "require_citations": true,
        "citation_type": "output_spans",
        "min_citations_per_fail": 1
      }
    },
    {
      "id": "escalation_signal",
      "severity": "medium",
      "question": "Does the AI output clearly signal when human legal review is required for high-risk or uncertain items?",
      "detection_method": {
        "type": "semantic",
        "instructions": "Check whether the output includes explicit escalation triggers (e.g., 'requires counsel review', 'depends on jurisdiction/facts'). PASS if present and meaningful; FAIL if absent in presence of uncertainty; INDETERMINATE if not applicable."
      },
      "result_states": ["pass", "fail", "indeterminate"],
      "evidence_requirements": {
        "require_citations": true,
        "citation_type": "output_spans",
        "min_citations_per_fail": 1
      }
    },
    {
      "id": "unchecked_areas_disclosure",
      "severity": "medium",
      "question": "Does the AI output state what it did NOT check (or what information was missing) in a concrete way?",
      "detection_method": {
        "type": "deterministic",
        "instructions": "PASS if output includes an explicit 'not checked / missing info' section or statement; FAIL if it presents as complete without acknowledging omissions; INDETERMINATE if output is extremely short or clearly partial."
      },
      "result_states": ["pass", "fail", "indeterminate"],
      "evidence_requirements": {
        "require_citations": true,
        "citation_type": "output_spans",
        "min_citations_per_fail": 1
      }
    },
    {
      "id": "run_variance",
      "severity": "medium",
      "question": "Across repeated runs with identical inputs, do findings or conclusions materially diverge?",
      "detection_method": {
        "type": "semantic",
        "instructions": "Compare run outputs. Identify materially divergent findings (new/removed high severity issues, flipped conclusions). PASS if stable above threshold; FAIL if below warn threshold; INDETERMINATE if runs cannot be compared."
      },
      "result_states": ["pass", "fail", "indeterminate"],
      "evidence_requirements": {
        "require_citations": false
      }
    },
    {
      "id": "drift_over_time_support",
      "severity": "medium",
      "question": "Compared to a stored baseline report (prior run), does the current report show degradation signals?",
      "detection_method": {
        "type": "hybrid",
        "instructions": "If a baseline is provided, compute deltas in consistency score and key check outcomes. PASS if no material degradation; FAIL if degradation exceeds thresholds; INDETERMINATE if no baseline provided."
      },
      "result_states": ["pass", "fail", "indeterminate"],
      "evidence_requirements": {
        "require_citations": false
      }
    }
  ]
}
```

### 3. Evaluation Engine

For each check (except `run_variance` and `drift_over_time_support`):

1. Build a structured prompt that includes:
   - The check question
   - The detection method instructions
   - The frozen AI output
   - The source document (if provided)
   - Clear instruction to return JSON with: `result` (pass/fail/indeterminate), `confidence` (0-1, defined as: how clearly the evidence supports this judgment), `evidence_citations` (array of quoted spans from the AI output), `notes` (brief explanation)

2. Call Claude API with temperature 0 (or lowest available) to minimize evaluator variance

3. Parse structured JSON response

**For Full mode:** Run each check 3 times. Then:
- Compute per-check consistency: 1.0 if all 3 runs agree on result state, 0.5 if 2/3 agree, 0.0 if all differ
- Overall consistency score = severity-weighted average (high=2, medium=1)
- Use majority vote for the final result per check
- Store all raw run outputs

**For `run_variance` check:** Compare the raw outputs of the 3 evaluation runs themselves — are the other checks producing stable results?

**For `drift_over_time_support`:** Look up the most recent prior report for the same playbook_logic_hash in SQLite. If found, compare consistency scores and check outcomes. If not found, return indeterminate.

### 4. Report Generation

Produce a BYOP Report matching this output contract:

```json
{
  "byop_report": {
    "spec_version": "0.1",
    "playbook_id": "ai_plugin_observability_v1",
    "playbook_version": "1.1.0",
    "execution_mode": "full|screening",
    "timestamp": "ISO 8601",
    "summary": {
      "overall_status": "ALERT|REVIEW|OBSERVE|STABLE",
      "key_risks": ["..."],
      "recommended_next_steps": ["..."]
    },
    "check_results": [
      {
        "check_id": "assumption_disclosure",
        "result": "pass|fail|indeterminate",
        "per_check_confidence": 0.0-1.0,
        "per_check_consistency": 0.0-1.0,
        "evidence_citations": [{"span": "...", "location": "..."}],
        "notes": "...",
        "raw_runs": [{"run": 1, "result": "...", "confidence": 0.0}, ...]
      }
    ],
    "variance_summary": {
      "num_runs": 3,
      "consistency_score": 0.0-1.0,
      "divergent_findings": ["..."]
    },
    "integrity": {
      "playbook_logic_hash": "sha256:...",
      "inputs_fingerprint": "sha256:...",
      "runner_fingerprint": "byop-runner-replit-v0.1"
    },
    "presentation_rules": {
      "disclaimers": [
        "This is an observability report, not legal advice.",
        "Pass ≠ safe. Fail ≠ wrong. Indeterminate is expected.",
        "Report describes behavior under this playbook and inputs."
      ]
    }
  }
}
```

**Overall status aggregation rules:**
- **ALERT** — any high-severity check fails in majority of runs
- **REVIEW** — any high-severity check is indeterminate in majority of runs, OR multiple medium-severity fails
- **OBSERVE** — only medium-severity issues detected
- **STABLE** — all checks pass with consistency above 0.85 threshold

### 5. Report Display UI

Show the report in a clean, readable format:
- **Header:** Overall status badge (color-coded: ALERT=red, REVIEW=amber, OBSERVE=blue, STABLE=green), playbook ID, timestamp, execution mode
- **Disclaimers banner** (always visible, not dismissable): the 3 mandatory presentation rules
- **Check results cards:** One card per check showing result badge, confidence, consistency (if full mode), evidence citations as highlighted quotes, notes
- **Variance summary section** (full mode only): consistency score with threshold indicators, list of divergent findings
- **Integrity section:** Hashes displayed in monospace
- **Raw JSON toggle:** Expand to see full JSON report
- **Export button:** Download report as JSON file
- **Save as Baseline button:** Store this report in SQLite for future drift comparison

### 6. Run History

Simple table showing past evaluations:
- Timestamp, playbook version, execution mode, overall status, consistency score
- Click to view full report
- Compare button to diff two reports side by side

---

## Prompt Engineering for the Evaluator Calls

This is the most critical part. Each check call to Claude should use this system prompt structure:

```
You are a BYOP evaluation engine. You are running a single check from a legal/compliance observability playbook against a frozen AI output.

Your role:
- You are a TEST, not an advisor
- You evaluate the AI output for observable signals only
- You do NOT provide legal advice or opinions
- You cite specific spans from the AI output as evidence

Check to evaluate:
[check.question]

Detection method:
[check.detection_method.instructions]

You must respond with ONLY valid JSON in this exact format:
{
  "result": "pass" | "fail" | "indeterminate",
  "confidence": <float 0-1, how clearly the evidence supports your judgment>,
  "evidence_citations": [
    {"span": "<exact quote from AI output>", "context": "<brief context>"}
  ],
  "notes": "<1-2 sentence explanation of your judgment>"
}

Rules:
- "indeterminate" is a valid and expected result. Use it when the evidence is genuinely ambiguous.
- For FAIL results, you MUST provide at least 1 evidence citation.
- Citations must be exact quotes from the AI output, not paraphrases.
- Confidence reflects clarity of evidence, NOT your certainty about the law.
- Do not add any text outside the JSON object.
```

The user message should contain:

```
=== AI OUTPUT UNDER EVALUATION ===
[the frozen AI output]

=== SOURCE DOCUMENT (if provided) ===
[the source document, or "Not provided"]

=== ORIGINAL PROMPT (if provided) ===
[the prompt that generated the AI output, or "Not provided"]
```

---

## Integrity Hashing Implementation

```javascript
// Pseudocode for playbook logic hash
function computePlaybookHash(playbook) {
  // Remove volatile fields
  const { created_at, ...stable } = playbook.metadata;
  const canonical = JSON.stringify(stable, Object.keys(stable).sort());
  // Include checks and detection methods
  const checksCanonical = JSON.stringify(playbook.checks, Object.keys(playbook.checks).sort());
  return sha256(canonical + checksCanonical);
}

// Pseudocode for inputs fingerprint
function computeInputsHash(aiOutput, sourceDoc, prompt) {
  const input = JSON.stringify({
    ai_output: aiOutput.trim(),
    source_document: (sourceDoc || "").trim(),
    prompt: (prompt || "").trim()
  });
  return sha256(input);
}
```

Use deterministic key-sorted JSON serialization throughout. Reference RFC 8785 (JSON Canonicalization Scheme) if a library is available, otherwise sorted-keys JSON.stringify is acceptable for MVP.

---

## Environment Variables

```
# User provides their own key via the UI — do NOT use env vars for API keys
# The only env var needed:
DATABASE_URL=sqlite:///byop_runs.db
```

---

## File Structure (suggested)

```
/
├── app/                      # Next.js app dir (or /api for FastAPI)
│   ├── page.tsx              # Main input + report UI
│   ├── api/
│   │   ├── evaluate/         # POST: run playbook against input
│   │   ├── history/          # GET: list past runs
│   │   └── compare/          # POST: diff two reports
│   └── components/
│       ├── InputForm.tsx
│       ├── ReportView.tsx
│       ├── CheckCard.tsx
│       ├── VarianceSummary.tsx
│       ├── StatusBadge.tsx
│       └── DisclaimerBanner.tsx
├── lib/
│   ├── playbook.ts           # Starter playbook definition
│   ├── runner.ts             # Core evaluation engine
│   ├── prompts.ts            # Prompt templates for each check
│   ├── integrity.ts          # Hashing functions
│   ├── variance.ts           # Consistency score computation
│   ├── aggregation.ts        # Overall status derivation
│   └── db.ts                 # SQLite operations
├── schema/
│   └── byop-v0.1.json        # The full BYOP spec as reference
└── README.md
```

---

## Critical Implementation Notes

1. **API key handling:** Never persist the API key. Store in React state only. Pass it in request headers to the backend. The backend uses it for the Claude API call and discards it.

2. **Rate limiting:** In Full mode, you're making 12-15 API calls (4 semantic checks × 3 runs + variance comparison + drift comparison). Add a progress indicator showing "Running check 3/6, run 2/3..." and handle API rate limit errors gracefully with retry + backoff.

3. **Evaluator variance note:** Add a small info tooltip in the UI: "Variance in results may reflect evaluator model behavior, not only the AI output being tested. BYOP reports describe observable signals, not ground truth."

4. **The `run_variance` check** does NOT make additional API calls. It analyzes the results of the other checks across the 3 runs to determine if they were consistent.

5. **The `drift_over_time_support` check** queries SQLite for the most recent prior report with the same playbook_logic_hash. If none exists, it returns indeterminate. If one exists, it compares consistency scores and check outcome distributions.

6. **Normalization:** Before hashing or evaluating, apply these normalization rules to inputs:
   - Strip leading/trailing whitespace
   - Normalize line endings to \n
   - Preserve all quotes, clause numbers, and model disclaimers verbatim

7. **Error handling:** If a Claude API call returns malformed JSON, retry once. If it fails again, record the raw response and mark that check+run as `indeterminate` with a note: "Evaluator returned unparseable response."

8. **Disclaimer banner** must be non-dismissable and appear at the top of every report view. This is a spec requirement, not a UX preference.

---

## What Success Looks Like

A user should be able to:
1. Paste an AI-generated contract review or legal analysis
2. Click "Run Full Evaluation"
3. See a progress indicator as 12+ API calls execute
4. Get a structured report showing which checks passed, failed, or were indeterminate
5. See evidence citations highlighting specific problematic phrases
6. See a variance summary showing how stable the evaluation was
7. Save the report as a baseline
8. Run it again next week and see if the AI output's behavior has drifted
9. Export the full JSON report for audit purposes

The entire flow should feel like running a test suite, not like getting legal advice.
