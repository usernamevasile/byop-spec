# Contributing to BYOP

BYOP is an open specification. Contributions are welcome and valued.

## What's Most Useful Right Now

### 1. New Playbooks
Create evaluation playbooks for specific domains:
- GDPR data processing reviews
- EU AI Act compliance assessments
- SOC 2 control evaluations
- Employment law screening
- Contract review quality checks
- Any legal/compliance domain where AI outputs need evaluation

A good playbook follows the schema in `spec/byop-v0.1.json` and includes:
- Clear scope and assumptions
- Checks with detection methods and evidence requirements
- Realistic severity levels
- All three result states (pass, fail, indeterminate)

### 2. Real Evaluation Examples
Run the starter playbook (or your own) against a real AI output and share the results:
- Anonymize any confidential content
- Include the AI output, the playbook used, and the evaluation results
- Note which model generated the output and which model ran the evaluation
- Include variance data if you ran multiple times

### 3. Spec Critique
Open an issue if:
- Something in the spec is unclear or contradictory
- A required field doesn't make sense in practice
- You tried to implement it and something broke
- You think a concept is missing or overengineered

### 4. Runner Implementations
Build a runner in any language or platform:
- Python script
- Node.js CLI
- Web app
- Notebook
- Integration with existing eval tools (DeepEval, Langfuse, etc.)

## How to Contribute

1. **Open an issue** to discuss your idea before building
2. **Fork the repo** and create a branch
3. **Follow the existing file structure** (playbooks go in `playbooks/`, examples in `examples/`)
4. **Submit a pull request** with a clear description

## Guidelines

- Keep playbooks focused — one domain, one use case
- Be explicit about assumptions — don't hide them
- Use all three result states — indeterminate is not a cop-out
- Include real examples where possible (anonymized if needed)
- Don't claim legal correctness — BYOP evaluates behavior, not legality

## Code of Conduct

Be constructive. Argue about the checks, not the people. If you think the spec is wrong, explain why with examples. If you think a playbook is bad, suggest a better one.

## License

By contributing, you agree that your contributions will be licensed under CC BY 4.0, the same license as the rest of the project.
