---
name: are-reasoning
description: "Use when: you need templates, validation checklists, and utility workflows for ARE reasoning tasks. Includes hypothesis templates, discrimination checklists, and tool integrations for code execution and log analysis."
---

# ARE Reasoning Skill

This skill provides templates, checklists, and tool integration patterns for the Adversarial Reasoning Engine.

## Part 1: Hypothesis Template

Copy and fill this for any new hypothesis:

```markdown
## Hypothesis: [Clear, concise title]

**Assumption Chain:**
- A1: [Explicit assumption the solution rests on]
- A2: [What must be environmentally true]
- A3: [What must be true about system state]

**Reasoning:**

Starting from the assumptions:
1. [Implication of A1]
2. [Implication of A2]
3. [Combined implication of A1 + A2]
4. [Why this leads to the root cause]
5. [Therefore, the problem is...]

**Evidence Supporting This:**
- [Observation from logs/code/metrics]
- [Observation from logs/code/metrics]
- [Observation from logs/code/metrics]

**Proposed Solution:**

1. Step 1: [Concrete action]
2. Step 2: [Concrete action]
3. Step 3: [Concrete action]

**Success Criteria:**
- [How will we know the fix worked?]
- [What metrics to monitor?]
- [What tests to run?]

**Initial Confidence: 0.XX**

**Why this score:**
- Aligns with X symptoms: Yes/Partial/No
- Contradicts any evidence: Yes/No
- Requires unproven assumptions: [List which ones]
- Domain experts would agree: Likely/Maybe/Unlikely
```

## Part 2: Discrimination Checklist

Use this when attacking a hypothesis:

```markdown
## Discriminating [Hypothesis Title]

### Assumption Validation

- [ ] Is each assumption explicit?
- [ ] Is each assumption testable?
- [ ] Can I prove any assumption wrong?
- [ ] What happens if assumption X is false?
- [ ] Do assumptions contradict each other?

### Logical Flow Check

- [ ] Does step 1→2 follow logically?
- [ ] Does step 2→3 follow logically?
- [ ] Are there unstated steps?
- [ ] Is the conclusion actually reached?
- [ ] Could a different conclusion also fit?

### Evidence Contradiction Check

- [ ] Does hypothesis explain all observed symptoms?
- [ ] Does hypothesis contradict any evidence?
- [ ] List symptoms it explains: [...]
- [ ] List symptoms it doesn't explain: [...]

### Edge Case Exploration

- [ ] What if input is at boundary? (0, max, null, empty)
- [ ] What if concurrent access happens?
- [ ] What if resource is exhausted?
- [ ] What if environment variable is not set?
- [ ] What if this is running in container/k8s/cloud?

### Counterexample Hunt

- [ ] Can I contrive a scenario where hypothesis is wrong?
- [ ] Can I find a real counterexample in similar systems?
- [ ] What domain-specific edge case breaks this?

### Solution Robustness

- [ ] Does the proposed fix actually solve the diagnosed problem?
- [ ] Could the fix cause new problems?
- [ ] What if the fix partially works?
- [ ] Is there a better way to solve it?

### Severity Assessment

- [ ] Critical flaw (breaks hypothesis entirely): Yes/No
- [ ] Major flaw (weakens hypothesis significantly): Yes/No
- [ ] Minor flaw (caution flag): Yes/No
- [ ] Robustness score: 0.XX
```

## Part 3: Refinement Integration

Use this after discrimination to strengthen hypothesis:

```markdown
## Refining [Hypothesis Title]

### Addressing Each Critique

**Critique 1: [What discriminator said]**
- Original assumption: [State it]
- Evidence gathered: [What did we find?]
- Resolution: [Updated reasoning / admitted valid flaw]
- New assumption status: ✅ Validated / ⚠️ Supported / ❌ Weakened

**Critique 2: [What discriminator said]**
- [Same process...]

### Assumption Validation Matrix

| Assumption | Originally | Evidence | Now | Status |
|-----------|-----------|----------|-----|--------|
| A1: ... | Unverified | [Found X in logs] | Validated | ✅ |
| A2: ... | Unverified | [Never verified] | Weakened | ⚠️ |

### Updated Confidence

- Before refinement: 0.XX
- Changes: [+0.1 for validated X, −0.05 for weakened Y]
- After refinement: 0.XX

**Reason for change**: [Honest assessment of improvement]
```

## Part 4: Judgment Scorecard

Use for final ranking:

```markdown
## Comparing [H1 vs H2 vs H3]

### Scoring Matrix

| Criterion | H1 | H2 | H3 | Weight | Notes |
|-----------|----|----|----|----|--------|
| Alignment (explains symptoms) | 0.9 | 0.7 | 0.6 | 30% | H1 covers all 5 symptoms |
| Reasoning quality (logic clear) | 0.85 | 0.8 | 0.68 | 25% | H1 has evidence for each step |
| Robustness (survived attack) | 0.80 | 0.75 | 0.65 | 25% | H1 handled edge cases; H2 didn't |
| Actionability (can implement) | 0.9 | 0.85 | 0.80 | 15% | All clear; H1 is simplest |
| **TOTAL** | **0.87** | **0.75** | **0.67** | 100% | |

### Risk Analysis

| Hypothesis | Risk Level | If Wrong, Impact | Mitigation |
|-----------|-----------|--------|----------|
| H1 | Low | Retry logic doesn't help; need different approach | Monitor; have fallback ready |
| H2 | Medium | Increase pool size but problem persists | Have H1 checked as backup |
| H3 | High | Changes core logic; could break other flows | Add comprehensive tests first |

### Final Ranking

1. **H1: Event loop blocked (0.87 confidence)**
   - Clear winner; explained all symptoms
   - Then if H1 is wrong, try H2 (0.75)

### Next Steps

1. Implement H1's solution
2. Monitor key metrics: [X, Y, Z]
3. If metrics don't improve in 1 hour, investigate H2
4. Have H3 researched as longer-term alternative
```

## Part 5: Tool Integration Patterns

### Code Execution Tool

Use when you need to validate hypothesis with code:

```python
# Pseudo-code for execution within ARE

import subprocess
import json

def run_diagnostic(problem_context):
    """
    Run code to test hypothesis against real system
    Example: Check if event loop is actually blocked
    """
    script = """
    // Node.js: Check event loop latency
    const start = Date.now();
    setImmediate(() => {
        const lag = Date.now() - start;
        console.log(JSON.stringify({
            event_loop_lag_ms: lag,
            blocked: lag > 50,
            timestamp: new Date().toISOString()
        }));
    });
    """
    
    result = subprocess.run(['node', '-e', script], capture_output=True)
    data = json.loads(result.stdout)
    
    return {
        "supports_hypothesis": data['blocked'],
        "confidence_adjustment": +0.2 if data['blocked'] else -0.3,
        "evidence": data
    }
```

### Log Parser Tool

Use when analyzing logs to validate assumptions:

```python
def parse_logs_for_patterns(log_file, hypothesis):
    """
    Parse logs to find evidence supporting/contradicting hypothesis
    Example: Hunt for timeout patterns every Nth request
    """
    patterns_found = {
        "timeout_frequency": extract_timeout_frequency(log_file),
        "correlations": find_correlation_with_events(log_file),
        "gaps": find_unexplained_events(log_file)
    }
    
    confidence_adjustment = 0
    
    # If pattern matches hypothesis prediction
    if patterns_found['timeout_frequency'] == "every_10th_request":
        confidence_adjustment += 0.25
    
    # If pattern contradicts hypothesis
    if patterns_found['gaps']:
        confidence_adjustment -= 0.1
    
    return {
        "patterns": patterns_found,
        "confidence_adjustment": confidence_adjustment
    }
```

### Metrics Query Tool

Use to validate assumptions about system state:

```python
def check_system_metrics(hypothesis, time_window="1h"):
    """
    Query monitoring system to validate assumptions
    Example: Check if database is actually overwhelmed
    """
    
    metrics = {
        "db_cpu": fetch_metric("db.cpu.percent", time_window),
        "db_connections": fetch_metric("db.connections.active", time_window),
        "queue_length": fetch_metric("service.queue.length", time_window),
    }
    
    # Hypothesis: DB is overwhelmed
    # Prediction: DB CPU should be 90+%
    # Reality: DB CPU is 30%
    
    contradiction = metrics['db_cpu'] < 0.9
    
    return {
        "metrics": metrics,
        "contradicts_hypothesis": contradiction,
        "confidence_adjustment": -0.3 if contradiction else +0.2
    }
```

## Part 6: Common Pitfalls & Fixes

| Pitfall | What Happens | How to Fix |
|---------|--------------|-----------|
| **Confirmation bias** | Generator proposes X; discriminator is too nice | Force discriminator to find 3+ flaws per hypothesis |
| **Unverified assumptions** | Assume X is true; never checked logs | Require discriminator to cite evidence for/against each assumption |
| **Oversimplification** | Solution is "just increase timeout" | Have refiner stress-test solution against edge cases |
| **Vague ranking** | Can't decide between H1 and H2 | Use scoring matrix; weight criteria explicitly |
| **Analysis paralysis** | Can't converge on answer; keep iterating | Stop at iteration 3; pick best available candidate |
| **Missing context** | Ignore constraint given in problem | Have judge explicitly check problem constraints |

## Part 7: Workflow Hooks

### Before Starting Reasoning

```markdown
- [ ] Problem statement is clear (not ambiguous)?
- [ ] Have I identified all constraints?
- [ ] Is there relevant context (logs, code, metrics)?
- [ ] What's the cost of being wrong?
- [ ] What's the cost of delay?
```

### Between Generator and Discriminator

```markdown
- [ ] Are hypotheses diverse or just variations?
- [ ] Are assumptions explicit?
- [ ] Is reasoning chain clear?
- [ ] Could a domain expert verify each one?
```

### Between Discriminator and Refiner

```markdown
- [ ] Did discriminator find real flaws or strawmen?
- [ ] Are refinements based on evidence or speculation?
- [ ] Should we collect more data before refining?
```

### Before Final Judgment

```markdown
- [ ] Can we rank hypotheses fairly?
- [ ] Are we hiding uncertainty?
- [ ] Are next steps clear?
- [ ] Do we know how to validate the winner?
```

## Part 8: Quick Reference

### Confidence Thresholds

- **0.8+**: High confidence; implement immediately
- **0.7–0.8**: Good confidence; implement with monitoring
- **0.6–0.7**: Moderate confidence; have fallback plan
- **0.5–0.6**: Low confidence; collect more data; iterate
- **<0.5**: Very low confidence; probably need new hypotheses

### Iteration Guidance

- **Iteration 1**: Generator proposes; discriminator attacks
- **Iteration 2**: Refined hypotheses; discriminator looks for NEW flaws
- **Iteration 3**: Last iteration; confidence must be ≥0.6 to proceed
- **Stop**: If confidence stalls or isn't improving; pick best available

### Success Metrics

- Hypothesis explains most observed symptoms
- Assumptions are explicit and testable
- Solution is concrete and actionable
- Remaining risks are documented
- Next steps for validation are clear
