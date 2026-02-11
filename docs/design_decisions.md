# Design Decisions

This document explains key choices in the BYOP v0.1 specification and why alternatives were rejected.

## Why "observability" and not "compliance"?

Compliance implies a standard has been met. BYOP doesn't know what standard you're measuring against — you define that in your playbook. BYOP tells you what the AI output *did* (stated assumptions, used qualified language, flagged uncertainty) and what it *didn't* (omitted issues, expressed false certainty). Whether that behavior constitutes "compliance" with any particular regulation is a judgment call for humans.

Calling it observability keeps expectations honest.

## Why require 3 runs minimum?

LLMs are stochastic. The same input produces different outputs. A check that passes once and fails twice is unreliable — but without multiple runs, you'd never know. The 3-run minimum is the lowest number that allows a majority vote while catching obvious variance. It's a pragmatic floor, not a scientific sample size.

Full mode (3 runs) is labeled "auditable." Screening mode (1 run) is explicitly labeled "non-auditable, not comparable over time." This forces users to acknowledge the trade-off rather than pretending a single run is definitive.

## Why "indeterminate" as a first-class result?

Binary pass/fail creates false precision. Many legal AI outputs are genuinely ambiguous — the check can't tell whether assumptions were stated because the output is too short, or because the question doesn't apply to this type of document. Forcing a pass or fail in these cases corrupts the signal.

In practice, indeterminate will be the most common result for several checks, especially on first use. This is by design. A report with three passes and three indeterminates tells you more than a report with six passes from a system that couldn't distinguish "checked and fine" from "couldn't check."

## Why SHA-256 hashes?

Two reasons:

1. **Tamper evidence.** If someone modifies a BYOP report after the fact, the hashes won't match. This matters for audit trails.

2. **Replayability.** The inputs fingerprint lets you confirm that a report was generated against specific inputs. The playbook logic hash lets you confirm which version of the playbook was used. Together, they make the evaluation reproducible.

We chose SHA-256 over simpler checksums because it's standard, widely supported, and collision-resistant enough for this use case.

## Why not score legal correctness?

Because we can't. An LLM evaluating another LLM's legal analysis has no ground truth to compare against. It can detect surface signals (certainty language, missing qualifiers, absent disclosures) but it cannot determine whether a legal conclusion is actually correct.

BYOP measures *how* the AI presented its analysis, not *whether* the analysis is right. This is a deliberate limitation that prevents the most dangerous failure mode: users treating a BYOP "pass" as confirmation that the legal content is correct.

## Why per-check confidence instead of a global confidence score?

A global confidence score is uninterpretable. "Confidence: 0.72" tells you nothing about which checks were strong and which were weak. Per-check confidence lets users see that the evaluator was very confident about certainty language detection (0.95) but uncertain about assumption disclosure (0.5) — which is actionable information.

Confidence is defined as: "how clearly the evidence supports this judgment." This is narrower than "how likely is this result to be correct" — which would require calibration data we don't have.

## Why not use existing eval frameworks?

We considered building BYOP as a plugin for DeepEval or Langfuse. The problem: those frameworks evaluate LLM applications from an engineering perspective (accuracy, hallucination, relevance). Their abstractions don't map well to legal quality criteria.

A DeepEval custom metric could implement a BYOP check, but the surrounding framework (test cases, expected outputs, scoring) assumes you have ground truth. BYOP explicitly does not. The variance handling, integrity hashing, and presentation rules also have no equivalent in existing frameworks.

BYOP playbooks *could* be implemented inside these frameworks in the future. The spec is designed to be framework-agnostic so that this remains possible.

## Why CC BY 4.0?

We want maximum adoption with minimum friction. CC BY 4.0 allows anyone to use, fork, and commercialize BYOP-derived work. The only requirement is attribution — which is how authorship gravity works. If someone builds a product on BYOP, they credit the spec. If they don't, the license gives us a basis to request it.

We considered MIT (too focused on software), Apache 2.0 (includes patent provisions we don't need), and CC BY-SA (share-alike would discourage commercial adoption). CC BY 4.0 is the cleanest fit for a specification that isn't software.

## Why "forbidden claims"?

The biggest risk with any evaluation framework is that users will over-interpret results. A BYOP report that says "STABLE" will inevitably appear in a pitch deck or compliance filing as evidence that the AI tool is "safe" or "compliant."

The forbidden claims list ("certified," "compliant," "legally safe," "regulator approved") creates an explicit boundary. It won't prevent misuse entirely, but it establishes a clear standard for what constitutes misuse. If someone uses a BYOP report to claim compliance, the spec itself says that's not a valid claim — which matters in any subsequent dispute.
