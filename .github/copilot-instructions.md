# AI Adversarial Reasoning Engine (ARE)

This workspace includes a multi-agent reasoning system for validating ideas, hypotheses, and solutions before implementation.

## Quick Start

Ask Copilot in any of these ways:

- **Direct**: `/are-reasoning-engine "Your problem statement"`
- **Descriptive**: "Run adversarial reasoning on: [my issue]"
- **CLI** (future): `are-reason "Why is this failing?" --mode debug`

## What ARE Does

The system runs **4 phases internally** in a single agent:

1. **Generate** → Proposes 2–3 diverse hypotheses with explicit assumptions
2. **Discriminate** → Attacks each hypothesis; finds flaws, edge cases, contradictions
3. **Refine** → Strengthens winning hypotheses using feedback
4. **Judge** → Ranks refined hypotheses; assigns confidence scores

All phases execute in one conversation. You can ask follow-ups to drill deeper into any phase.

## When to Use ARE

- **Debugging**: "Why does my code fail on certain inputs?"
- **Design**: "Is this architecture defensible?"
- **Research**: "What are the tradeoffs in this approach?"
- **Validation**: "Have I considered all failure modes?"
- **Assumptions**: "What could be wrong with my reasoning?"

## Input Format (Optional)

```json
{
  "problem": "Why does the payment API timeout intermittently?",
  "mode": "debug | fix | research | design | optimize",
  "context": {
    "code": "...",
    "logs": "error logs here",
    "constraints": ["must be under 100ms", "no external dependencies"]
  },
  "expected_output": "Ranked list of root causes"
}
```

## Output Format

```
{
  "final_answer": "The most likely cause is...",
  "confidence": 0.85,
  "reasoning_chain": ["step 1", "step 2", ...],
  "alternatives": ["Alternative 1 (confidence: 0.7)", "Alternative 2 (confidence: 0.6)"],
  "key_assumptions": ["Assuming X", "Assuming Y"],
  "risks": ["Risk 1: ...", "Risk 2: ..."],
  "iterations": 2
}
```

## Tips

- **Be specific**: "Why does sorting fail on null values?" beats "sorting is broken"
- **Provide context**: Logs, code snippets, and constraints improve reasoning quality
- **Challenge results**: If confidence is low, ask "Why?" and iterate
- **Use alternatives**: The system often finds solutions you didn't consider

## Domain Support

- **Code**: Python, JavaScript, Java, .NET, Go, Rust, etc.
- **Infrastructure**: AWS, Azure, Kubernetes, Docker
- **Design**: Architecture, API design, data modeling
- **Non-code**: Business logic, research hypotheses, tradeoff analysis
