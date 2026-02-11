# How to Run BYOP Manually

No code required. Just copy-paste prompts into Claude, GPT, or any LLM chat interface.

## What You Need

1. **An AI output to evaluate** — any AI-generated legal analysis, contract review, compliance check, or similar output
2. **The starter playbook** — in `playbooks/starter_playbook_v1.1.json`
3. **Access to any LLM** (Claude, GPT, Gemini, etc.)

## Step-by-Step

### Step 1: Copy This System Prompt

Paste this into a new conversation (or as a system prompt if using an API):

```
You are a BYOP evaluation engine. You run structured checks from a legal observability playbook against a frozen AI output.

Your role:
- You are a TEST, not an advisor
- You evaluate the AI output for observable signals only
- You do NOT provide legal advice or opinions
- You cite specific spans from the AI output as evidence
- "Indeterminate" is a valid and expected result

For each check, respond with ONLY valid JSON:
{
  "check_id": "<id>",
  "result": "pass" | "fail" | "indeterminate",
  "confidence": <float 0-1>,
  "evidence_citations": [
    {"span": "<exact quote from AI output>", "context": "<why this matters>"}
  ],
  "notes": "<1-2 sentence explanation>"
}

Rules:
- For FAIL results, provide at least 1 evidence citation
- Citations must be exact quotes, not paraphrases
- Confidence = how clearly the evidence supports your judgment
- Do not add text outside the JSON
```

### Step 2: Run Each Check

For each of the 6 checks in the starter playbook, send a message like this:

```
=== CHECK ===
ID: assumption_disclosure
Severity: high
Question: Does the AI output explicitly state key assumptions required for its conclusions (jurisdiction, missing facts, role/positioning)?
Detection: Extract assumptions the output relies on. Mark PASS only if key assumptions are explicitly stated; FAIL if key assumptions are implicit; INDETERMINATE if unclear.

=== AI OUTPUT UNDER EVALUATION ===
[paste the AI output here]

=== SOURCE DOCUMENT (if available) ===
[paste the source document, or write "Not provided"]
```

Repeat for each check:
- `assumption_disclosure`
- `certainty_language`
- `escalation_signal`
- `unchecked_areas_disclosure`

(Skip `run_variance` and `drift_over_time_support` for manual single runs.)

### Step 3: Compile Results

Collect the JSON responses and combine them into a report:

```json
{
  "playbook_id": "ai_plugin_observability_v1",
  "playbook_version": "1.1.0",
  "execution_mode": "screening",
  "timestamp": "[current date/time]",
  "evaluator_model": "[model you used]",
  "check_results": [
    // paste each check result here
  ],
  "overall_status": "[see aggregation rules below]"
}
```

### Overall Status Rules

- **ALERT** — any high-severity check (assumption_disclosure, certainty_language) failed
- **REVIEW** — any high-severity check is indeterminate, OR both medium-severity checks failed
- **OBSERVE** — only medium-severity issues
- **STABLE** — all checks pass

## For Full Auditable Mode (3 Runs)

To get variance data:

1. Run all 4 checks **three times each** with identical inputs
2. For each check, score consistency:
   - All 3 runs agree → consistency = 1.0
   - 2 of 3 agree → consistency = 0.5
   - All differ → consistency = 0.0
3. Use majority vote for the final result per check
4. Overall consistency = weighted average (high-severity checks × 2, medium × 1)

This is tedious manually. Consider using the API approach in `how_to_run_with_llm.md` or building a runner with `replit_runner_prompt.md`.

## Tips

- **Use temperature 0** (or lowest available) to minimize evaluator variance
- **Don't edit the AI output** between runs — it must be frozen
- **Save your results** — they become baselines for future drift comparison
- **Try different evaluator models** — if Claude and GPT disagree on a check, that's useful signal too
