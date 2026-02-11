# BYOP — Bring Your Own Playbook

**An open specification for evaluating legal AI outputs.**

> Playbook = test. AI output = thing under test. Result = observable signal, not legal advice.

---

## The Problem

Legal AI outputs look confident. They sound authoritative. And they silently degrade.

When an AI reviews a contract, generates a compliance check, or drafts legal analysis — you get polished text with no way to know:

- Did it state its assumptions, or make them silently?
- Did it express certainty it shouldn't have?
- Did it skip material issues without telling you?
- Has the quality changed since last month because the provider switched models?

Existing LLM evaluation tools (DeepEval, Langfuse, Arize) test models for accuracy, hallucination, and relevance. They don't test whether a legal AI output is **reliable as a legal work product.**

Existing governance platforms (Credo AI, Holistic AI) manage the AI system itself — risk assessments, bias audits, regulatory alignment. They don't evaluate the **content of specific outputs** against legal quality criteria.

Nobody is asking: **"Is this particular AI-generated legal analysis trustworthy, and can I prove how I checked?"**

BYOP fills that gap.

---

## What BYOP Is

BYOP is a lightweight, open specification that lets you define **evaluation playbooks** — structured tests that run against AI-generated legal outputs to measure reliability signals.

A playbook defines:
- **Scope** — what it evaluates and what it doesn't
- **Assumptions** — stated explicitly, never hidden
- **Checks** — specific questions with detection methods and evidence requirements
- **Variance handling** — because LLMs are stochastic, and single-run results are unreliable
- **Output contract** — a structured report with mandatory uncertainty disclosure

The playbook lives **outside** the AI system. It can be run manually, via prompts, via scripts, or via automated runners.

**Core design principles:**
- Observability over correctness
- Variance is a feature, not a bug
- Unknown and indeterminate are first-class results
- Audit-first and replayable
- Results describe behavior, not legality

---

## What BYOP Is Not

| BYOP does NOT... | Instead, it... |
|---|---|
| Certify legal compliance | Reports observable signals |
| Guarantee legal correctness | Measures reliability indicators |
| Replace human legal judgment | Flags where judgment is needed |
| Provide legal advice | Produces evaluation data |
| Bind regulators or courts | Creates audit trails |
| Prove a vendor switched models | Detects behavioral drift |

---

## Who This Is For

- **In-house legal teams** using AI tools for contract review, compliance scanning, or legal research — who need a structured way to check output quality
- **Legal tech builders** who want to demonstrate that their AI outputs are evaluated against explicit quality criteria
- **Compliance teams** evaluating whether AI-assisted processes meet observability and auditability requirements
- **Anyone** who has looked at a confident-sounding AI legal output and thought: *"But how do I know this is actually right?"*

---

## Quick Example (3 minutes)

### Input
You have an AI-generated GDPR compliance review of a SaaS vendor's data processing agreement.

### AI Output (excerpt)
> "The agreement is GDPR-compliant. Standard contractual clauses are included and data processing roles are properly defined."

### BYOP Evaluation (starter playbook)

| Check | Result | Evidence |
|---|---|---|
| **Assumption disclosure** | ❌ FAIL | No jurisdiction stated. No mention of which GDPR articles were checked. Assumes EU-only scope without stating it. |
| **Certainty language** | ❌ FAIL | "is GDPR-compliant" — unqualified absolute claim with no scope limitation or caveats. |
| **Escalation signal** | ❌ FAIL | No indication that human review is needed. No flagging of areas requiring legal judgment. |
| **Unchecked areas** | ❌ FAIL | No mention of what was NOT checked (cross-border transfers, sub-processor chains, DPA termination provisions). |
| **Run variance** | ⚠️ INDETERMINATE | Would require 3 runs to evaluate. |

**Overall status: 🔴 ALERT** — Multiple high-severity failures detected.

This doesn't mean the AI was *wrong*. It means the output **cannot be relied upon without additional review** because it presents conclusions without showing its work.

---

## Repository Structure

```
byop/
├── README.md                          ← You are here
├── LICENSE                            ← CC BY 4.0
├── spec/
│   └── byop-v0.1.json                ← The full BYOP specification
├── playbooks/
│   └── starter_playbook_v1.1.json    ← Ready-to-use starter playbook
├── examples/
│   ├── example_gdpr_scan.json        ← Real evaluation: GDPR compliance scan
│   └── example_ai_act_check.json     ← Real evaluation: EU AI Act assessment
├── reference/
│   ├── how_to_run_manually.md        ← Run BYOP with copy-paste prompts
│   ├── how_to_run_with_llm.md        ← Automate with Claude/GPT API
│   └── replit_runner_prompt.md        ← Build a full runner app in an afternoon
├── docs/
│   └── design_decisions.md           ← Why we made specific choices
└── CONTRIBUTING.md                    ← How to contribute playbooks or improvements
```

---

## Getting Started

### Option 1: Run manually (5 minutes)
→ See [`reference/how_to_run_manually.md`](reference/how_to_run_manually.md)

Paste the starter playbook and your AI output into Claude or GPT. Get a structured evaluation back. No code required.

### Option 2: Automate with an LLM API (30 minutes)
→ See [`reference/how_to_run_with_llm.md`](reference/how_to_run_with_llm.md)

Use the structured prompts and API calls to run evaluations programmatically with variance handling.

### Option 3: Build a full runner (1 afternoon)
→ See [`reference/replit_runner_prompt.md`](reference/replit_runner_prompt.md)

A complete prompt for Replit Agent to build a web-based BYOP runner with all features.

---

## Specification Overview

The full spec is in [`spec/byop-v0.1.json`](spec/byop-v0.1.json). Key concepts:

**Playbook schema** requires: metadata, input contract, assumptions, risk posture, checks (with detection methods and evidence requirements), variance handling, output contract, and logging.

**Check results** use three states: `pass`, `fail`, `indeterminate`. Indeterminate is expected, not a failure.

**Variance handling** requires minimum 3 runs in full/auditable mode, with consistency scoring and divergent findings reporting.

**Integrity** uses SHA-256 hashes over canonicalized playbook logic and inputs, ensuring reports are tamper-evident and replayable.

**Presentation rules** mandate that every report displays:
1. *"This is an observability report, not legal advice."*
2. *"Pass ≠ safe. Fail ≠ wrong. Indeterminate is expected."*
3. *"Report describes behavior under this playbook and inputs."*

**Forbidden claims**: No BYOP report may use the words "certified," "compliant," "legally safe," or "regulator approved."

---

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md) for details.

The most valuable contributions right now:
- **New playbooks** for specific domains (GDPR, EU AI Act, SOC 2, employment law)
- **Real evaluation examples** — run the starter playbook against an AI output and share the results
- **Critique of the spec** — what's missing, what's overengineered, what would break in your workflow
- **Runner implementations** — in any language, for any platform

---

## FAQ

**Is this just a prompt template?**
No. Prompt templates don't version themselves, don't track their own performance across models, don't require variance handling, and don't produce audit logs with integrity hashes. BYOP is a testing framework with a defined schema, not a suggestion for how to talk to an LLM.

**Can I use this commercially?**
Yes. CC BY 4.0 means you can use, fork, and build on BYOP for any purpose, including commercial, as long as you attribute the source.

**Does BYOP replace my legal team?**
No. BYOP tells you whether an AI output showed its work and flagged its uncertainty. It doesn't tell you whether the output is legally correct. You still need humans for that.

**What models can I run this against?**
Any LLM. The spec is model-agnostic. The reference implementations use Claude and GPT, but the playbook can be executed against any AI output from any source.

**Why not just use DeepEval / Langfuse / existing eval tools?**
Those tools evaluate LLM applications from an engineering perspective (accuracy, hallucination, relevance). BYOP evaluates legal AI outputs from a legal quality perspective (assumptions, certainty, omissions, escalation). They're complementary, not competing. A BYOP playbook could be implemented as a custom metric in DeepEval.

---

## Status

**Current version: v0.1 (draft_experimental)**

This is an early-stage open specification. The abstractions may change. Feedback is more valuable than adoption right now.

If you recognize the problem, argue with the checks, or ask "can I run this on our outputs?" — [open an issue](../../issues) or reach out.

---

## Author

Created by **Vasile Tiple** — former General Counsel at UiPath (pre-seed through IPO), fractional GC for AI startups, author of *Legal Automation Book: From Bottlenecks to Bots*.

BYOP comes from a simple observation: the tools I used to evaluate legal work at UiPath don't exist for evaluating AI-generated legal work. This specification is an attempt to fix that.

---

## Built With AI

This specification, documentation, reference implementations, and repository structure were developed with assistance from **Anthropic Claude Opus 4.6**, **OpenAI ChatGPT 5.2**, and **Google Gemini 3**. The conceptual framework, design decisions, and domain expertise are human-originated. The drafting, critique, iteration, and code were collaborative.

BYOP advocates for transparency in AI outputs. It would be inconsistent not to apply that standard here.

---

## License

[Creative Commons Attribution 4.0 International (CC BY 4.0)](LICENSE)

You are free to share, adapt, and build upon this work for any purpose, including commercial, as long as you provide attribution.
