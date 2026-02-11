# How to Run BYOP with an LLM API

Automate BYOP evaluations using the Claude or OpenAI API. This enables full mode with variance handling.

## Architecture

```
Your script
  ├── Load playbook JSON
  ├── Load AI output (frozen artifact)
  ├── For each check:
  │     ├── Build prompt from check definition
  │     ├── Call LLM API (N times for variance)
  │     ├── Parse structured JSON response
  │     └── Score consistency
  ├── Compute overall status
  ├── Generate integrity hashes
  └── Output BYOP report JSON
```

## Python Example (Claude API)

```python
import anthropic
import json
import hashlib
from datetime import datetime

client = anthropic.Anthropic()  # uses ANTHROPIC_API_KEY env var

SYSTEM_PROMPT = """You are a BYOP evaluation engine. You run structured checks
from a legal observability playbook against a frozen AI output.

Your role:
- You are a TEST, not an advisor
- You evaluate the AI output for observable signals only
- You do NOT provide legal advice or opinions
- You cite specific spans from the AI output as evidence

Respond with ONLY valid JSON in this exact format:
{
  "result": "pass" | "fail" | "indeterminate",
  "confidence": <float 0-1>,
  "evidence_citations": [
    {"span": "<exact quote from AI output>", "context": "<why this matters>"}
  ],
  "notes": "<1-2 sentence explanation>"
}

Rules:
- "indeterminate" is valid and expected. Use it when evidence is genuinely ambiguous.
- For FAIL, provide at least 1 evidence citation.
- Citations must be exact quotes from the AI output.
- Do not add any text outside the JSON object."""


def run_check(check, ai_output, source_doc=None):
    """Run a single check against an AI output."""
    user_msg = f"""=== CHECK ===
ID: {check['id']}
Severity: {check['severity']}
Question: {check['question']}
Detection method: {check['detection_method']['instructions']}

=== AI OUTPUT UNDER EVALUATION ===
{ai_output}

=== SOURCE DOCUMENT ===
{source_doc or 'Not provided'}"""

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1000,
        temperature=0,
        system=SYSTEM_PROMPT,
        messages=[{"role": "user", "content": user_msg}]
    )

    text = response.content[0].text
    # Strip markdown fences if present
    text = text.replace("```json", "").replace("```", "").strip()
    return json.loads(text)


def compute_consistency(results):
    """Score consistency across multiple runs of the same check."""
    states = [r['result'] for r in results]
    if len(set(states)) == 1:
        return 1.0  # all agree
    from collections import Counter
    most_common = Counter(states).most_common(1)[0][1]
    if most_common >= 2:
        return 0.5  # majority agrees
    return 0.0  # all differ


def majority_vote(results):
    """Return the majority result state."""
    from collections import Counter
    states = [r['result'] for r in results]
    return Counter(states).most_common(1)[0][0]


def compute_hash(data):
    """SHA-256 hash of canonicalized JSON."""
    canonical = json.dumps(data, sort_keys=True, separators=(',', ':'))
    return f"sha256:{hashlib.sha256(canonical.encode()).hexdigest()}"


def run_playbook(playbook, ai_output, source_doc=None, mode="full"):
    """Run full BYOP evaluation."""
    num_runs = 3 if mode == "full" else 1
    eval_checks = [c for c in playbook['checks']
                   if c['id'] not in ('run_variance', 'drift_over_time_support')]

    all_results = []

    for check in eval_checks:
        print(f"Running check: {check['id']} ({num_runs} runs)...")
        runs = []
        for i in range(num_runs):
            try:
                result = run_check(check, ai_output, source_doc)
                result['check_id'] = check['id']
                runs.append(result)
            except Exception as e:
                runs.append({
                    'check_id': check['id'],
                    'result': 'indeterminate',
                    'confidence': 0.0,
                    'evidence_citations': [],
                    'notes': f'Evaluator error: {str(e)}'
                })

        consistency = compute_consistency(runs) if mode == "full" else None
        final_result = majority_vote(runs) if mode == "full" else runs[0]['result']

        all_results.append({
            'check_id': check['id'],
            'severity': check['severity'],
            'result': final_result,
            'per_check_confidence': sum(r['confidence'] for r in runs) / len(runs),
            'per_check_consistency': consistency,
            'evidence_citations': runs[0]['evidence_citations'],
            'notes': runs[0]['notes'],
            'raw_runs': runs if mode == "full" else None
        })

    # Compute overall status
    high_fails = any(r['result'] == 'fail' and r['severity'] == 'high'
                     for r in all_results)
    high_indeterminate = any(r['result'] == 'indeterminate' and r['severity'] == 'high'
                            for r in all_results)
    medium_fails = sum(1 for r in all_results
                       if r['result'] == 'fail' and r['severity'] == 'medium')

    if high_fails:
        overall = "ALERT"
    elif high_indeterminate or medium_fails >= 2:
        overall = "REVIEW"
    elif medium_fails > 0:
        overall = "OBSERVE"
    else:
        overall = "STABLE"

    # Compute overall consistency
    if mode == "full":
        weights = {'high': 2, 'medium': 1}
        total_weight = sum(weights[r['severity']] for r in all_results)
        weighted_sum = sum(r['per_check_consistency'] * weights[r['severity']]
                          for r in all_results)
        overall_consistency = weighted_sum / total_weight if total_weight > 0 else 0
    else:
        overall_consistency = None

    report = {
        'spec_version': '0.1',
        'playbook_id': playbook['metadata']['id'],
        'playbook_version': playbook['metadata']['version'],
        'execution_mode': mode,
        'timestamp': datetime.utcnow().isoformat() + 'Z',
        'summary': {
            'overall_status': overall,
            'key_risks': [r['check_id'] for r in all_results if r['result'] == 'fail'],
            'recommended_next_steps': ['Human review required' if overall == 'ALERT'
                                       else 'Monitor for drift']
        },
        'check_results': all_results,
        'variance_summary': {
            'num_runs': num_runs,
            'consistency_score': overall_consistency,
            'divergent_findings': [r['check_id'] for r in all_results
                                   if r.get('per_check_consistency', 1.0)
                                   and r.get('per_check_consistency', 1.0) < 0.85]
        } if mode == "full" else None,
        'integrity': {
            'playbook_logic_hash': compute_hash(playbook['checks']),
            'inputs_fingerprint': compute_hash({
                'ai_output': ai_output,
                'source_doc': source_doc or ''
            }),
            'runner_fingerprint': 'byop-python-reference-v0.1'
        },
        'presentation_rules': {
            'disclaimers': [
                'This is an observability report, not legal advice.',
                'Pass ≠ safe. Fail ≠ wrong. Indeterminate is expected.',
                'Report describes behavior under this playbook and inputs.'
            ]
        }
    }

    return report


# === Usage ===
if __name__ == "__main__":
    with open('playbooks/starter_playbook_v1.1.json') as f:
        playbook = json.load(f)

    ai_output = """[Paste the AI-generated legal output you want to evaluate here]"""

    report = run_playbook(playbook, ai_output, mode="full")

    with open('byop_report.json', 'w') as f:
        json.dump(report, f, indent=2)

    print(f"\nOverall status: {report['summary']['overall_status']}")
    print(f"Consistency: {report['variance_summary']['consistency_score']:.2f}")
    for r in report['check_results']:
        print(f"  {r['check_id']}: {r['result']} (confidence: {r['per_check_confidence']:.2f})")
```

## Requirements

```
pip install anthropic
```

Set your API key:
```
export ANTHROPIC_API_KEY=sk-ant-...
```

## Adapting for OpenAI

Replace the `run_check` function's API call:

```python
from openai import OpenAI
client = OpenAI()

response = client.chat.completions.create(
    model="gpt-4o",
    temperature=0,
    messages=[
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": user_msg}
    ]
)
text = response.choices[0].message.content
```

Everything else stays the same.

## Cost Estimate

Full mode (3 runs × 4 checks = 12 API calls):
- Claude Sonnet: ~$0.50–2.00 per evaluation depending on input size
- GPT-4o: ~$0.30–1.50 per evaluation

Screening mode (1 run × 4 checks = 4 API calls):
- Roughly 1/3 of the above

## Next Steps

- Save reports as baselines for drift detection
- Run the same evaluation weekly to track model behavior changes
- Compare results across different evaluator models
- Build a CI/CD integration to run BYOP on every release
