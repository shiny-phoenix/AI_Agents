---
name: are-tools
description: "Tool integration utilities for ARE reasoning. Code runners, log parsers, metrics collectors."
---

# ARE Tool Integrations

This document describes how to integrate ARE with external tools for validation, code execution, and log analysis.

## 1. Code Execution Tool Integration

### Purpose

Allow ARE agents to run code snippets against real systems to validate hypotheses.

### Supported Scenarios

- **Performance Testing**: Measure event loop latency, GC paaks, etc.
- **Environmental Checks**: Verify assumptions about configuration
- **Live Debugging**: Run diagnostic code in production (with caution)
- **Concurrency Testing**: Reproduce race conditions

### Usage Pattern

```yaml
# Inside an agent response:

**Hypothesis Validation via Code Execution:**

I need to verify my assumption about event loop blocking. Let me run a diagnostic:

[CODE_EXECUTION]
language: node
timeout: 5s
---
const measurements = [];

// Run 100 times to capture patterns
for (let i = 0; i < 100; i++) {
  const start = Date.now();
  setImmediate(() => {
    const lag = Date.now() - start;
    measurements.push(lag);
  });
  
  // Simulate the problematic workload
  await fetch('http://localhost:8000/api/slow');
}

// Analyze results
const avgLag = measurements.reduce((a, b) => a + b) / measurements.length;
const maxLag = Math.max(...measurements);
const outliers = measurements.filter(m => m > 100).length;

console.log(JSON.stringify({
  average_lag_ms: avgLag,
  max_lag_ms: maxLag,
  outliers_over_100ms: outliers,
  hypothesis_supported: avgLag > 50 && outliers > 20
}));
---
[END_CODE_EXECUTION]

**Result Analysis:**
- Average lag: 127ms (confirms hypothesis)
- Outliers: 43 out of 100 (strong pattern)
- Confidence adjustment: **+0.25**
```

### Execution Model

1. Agent requests code execution
2. System sandboxes the code (container/isolated VM)
3. Captures stdout/stderr + execution time
4. Returns JSON results
5. Agent uses results to adjust confidence

### Safety Constraints

```python
# Built-in safeguards for code execution

MAX_EXECUTION_TIME = 30  # seconds
MAX_MEMORY = 512  # MB
MAX_NETWORK_CALLS = 100
ALLOWED_LANGUAGES = ['python', 'node', 'go', 'java', 'bash']
BLOCKED_OPERATIONS = [
    'chmod',           # File permission changes
    'rm -rf',          # Destructive deletions
    'DROP TABLE',      # Database destruction
    ':(){:|:&};:',     # Fork bombs
]

def validate_code(code, language):
    """Block dangerous patterns"""
    for blocked in BLOCKED_OPERATIONS:
        if blocked in code:
            raise SecurityError(f"Blocked operation: {blocked}")
    return True
```

### Example: Validating Timeout Hypothesis

**Python Example**

```python
# Hypothesis: Network timeout is exactly 30s, suggesting hard limit

import socket
import time

def test_timeout_theory():
    results = []
    
    for attempt in range(10):
        start = time.time()
        try:
            sock = socket.socket()
            sock.settimeout(None)  # Will hang
            sock.connect(('192.0.2.0', 80))  # Non-routable IP
        except socket.timeout:
            elapsed = time.time() - start
            results.append(elapsed)
        except Exception as e:
            results.append(str(e))
    
    import json
    print(json.dumps({
        "timeouts_captured": len([r for r in results if isinstance(r, float)]),
        "timeout_values_seconds": [r for r in results if isinstance(r, float)],
        "average_timeout": sum([r for r in results if isinstance(r, float)]) / len([r for r in results if isinstance(r, float)]),
        "theory_confirmed": all(29 < r < 31 for r in results if isinstance(r, float))
    }))

test_timeout_theory()
```

## 2. Log Parser Tool Integration

### Purpose

Automatically parse logs to find patterns, correlations, and contradictions.

### Log Formats Supported

- JSON logs
- Structured logs (ELK, CloudWatch, Stackdriver)
- Unstructured text logs
- Binary/compressed logs (.gz, .bz2)

### Query Patterns

```yaml
# Inside discriminator/refiner response:

**Hypothesis Test Against Logs:**

[LOG_QUERY]
source: cloudwatch
log_group: /aws/lambda/payment-api
time_window: 1h
query: |
  fields @timestamp, @duration, @error
  | filter @error like /timeout/
  | stats count() as timeout_count by bin(5m)
  | sort timeout_count desc
---
[END_LOG_QUERY]

**Result:**
- Timeouts per 5-min bucket: [8, 12, 3, 15, 7, 4, ...]
- Pattern identified: Not random; spikes at specific times
- Confidence adjustment: **+0.2** (aligns with hypothesis of periodic issue)
```

### Parser Implementation

```python
class LogParser:
    def __init__(self, log_source):
        self.source = log_source
    
    def find_pattern(self, hypothesis, time_window='1h'):
        """
        Search logs for patterns matching hypothesis predictions
        Example: If hypothesis says "every 10th request times out",
        look for requests in log and verify frequency
        """
        logs = self.fetch_logs(time_window)
        
        # Extract request IDs and durations
        requests = self.parse_timing_data(logs)
        
        # Check if timeouts match pattern (every Nth request)
        timeout_positions = [i for i, r in enumerate(requests) 
                            if r['duration'] > 30000]
        
        spacing = [timeout_positions[i+1] - timeout_positions[i] 
                  for i in range(len(timeout_positions)-1)]
        
        consistent_spacing = len(set(spacing)) == 1
        
        return {
            'pattern_found': consistent_spacing,
            'spacing': spacing[0] if spacing else None,
            'hypothesis_match': abs(spacing[0] - 10) < 2 if spacing else False,
            'confidence_adjustment': 0.25 if consistent_spacing else -0.1
        }
```

### Example Queries

**Find Correlated Failures**

```sql
-- CloudWatch Logs Insights

fields @timestamp, @duration, @error, user_id
| filter @duration > 30000
| stats count() as failures by user_id
| sort failures desc
| limit 20

-- Results: Specific users hit timeout more (not random)
-- Confidence: +0.15 (suggests user-specific state corruption)
```

**Find Timing Anomalies**

```sql
-- New Relic / DataDog

SELECT count(*), avg(duration), max(duration) 
FROM traces 
WHERE service = 'payment-api'
GROUP BY bin(time, 5m)
ORDER BY avg(duration) DESC

-- Results: 5-min buckets show 100-300x variation
-- Confidence: +0.1 (suggests external dependency load)
```

## 3. Metrics Collection Tool

### Purpose

Query live metrics to validate system state assumptions.

### Metric Sources

- Prometheus
- CloudWatch
- DataDog
- New Relic
- Custom endpoints

### Query Pattern

```yaml
# Inside refiner response:

**System Metrics Validation:**

[METRICS_QUERY]
source: cloudwatch
namespace: AWS/RDS
metrics:
  - CPUUtilization
  - DatabaseConnections
  - ReadLatency
  - QueryCount
time_range: 1h
---
[END_METRICS_QUERY]

**Results:**
- Database CPU: Average 24%, Max 67% (NOT overwhelmed)
- Connections: 45/100 (healthy)
- Read latency: 2ms average
- Confidence adjustment: **-0.3** (contradicts "DB overwhelmed" hypothesis)
```

### Implementation

```python
class MetricsCollector:
    def __init__(self, source='cloudwatch'):
        self.source = source
    
    def validate_assumption(self, assumption, time_window='1h'):
        """
        Example: Assumption is "database is overwhelmed"
        Collect metrics to disprove or support
        """
        
        metrics = {
            'cpu': self.query('AWS/RDS', 'CPUUtilization', time_window),
            'connections': self.query('AWS/RDS', 'DatabaseConnections', time_window),
            'latency': self.query('AWS/RDS', 'ReadLatency', time_window),
        }
        
        # Assumption: DB CPU should be 90+%
        # Reality: DB CPU avg 24%, max 67%
        cpu_high = metrics['cpu']['avg'] > 80
        connections_maxed = metrics['connections']['current'] > 95
        latency_high = metrics['latency']['avg'] > 100  # ms
        
        return {
            'metrics': metrics,
            'assumption_likely': (cpu_high and latency_high),
            'confidence_adjustment': -0.25 if not cpu_high else 0.2
        }
```

## 4. Integration Hook: Pre-Execution Checklist

Before running any diagnostic code:

```
- [ ] Is the code safe to execute in production?
- [ ] Will it generate excessive logs/metrics?
- [ ] Does it require special permissions?
- [ ] Is there a rollback plan if it causes issues?
- [ ] Have I set a reasonable timeout?
```

## 5. Integration Hook: Post-Execution Analysis

After receiving results:

```
- [ ] Do results support or contradict the hypothesis?
- [ ] Are results statistically significant?
- [ ] Could results be explained by other factors?
- [ ] What confidence adjustment is justified?
```

## 6. Example Workflow: Integrated Reasoning

### Problem

"Why does the payment API timeout on 5–10% of requests?"

### Generator Phase (with optional code execution)

```markdown
**Hypothesis 1: Event Loop Blocking**

[CODE_EXECUTION run event loop latency test...]
Result: Average lag 50ms, max 200ms
Confidence: 0.6 → 0.75 (code confirms blocking pattern)
```

### Discriminator Phase (with logs)

```markdown
**Attacking Hypothesis 1:**

[LOG_QUERY find timeout patterns...]
Result: Timeouts are random, not periodic
Analysis: If event loop blocked, timeouts should cluster
Confidence: 0.75 → 0.5 (logs contradict hypothesis)
```

### Refiner Phase (with metrics)

```markdown
**Refining after Discrimination:**

[METRICS_QUERY check CPU, memory, network...]
Result: Network latency spikes 300-500ms at timeout times
Analysis: Suggests external dependency issue, not local
Updated hypothesis: "Downstream API is slow"
Confidence: 0.5 → 0.7 (metrics explain contradiction)
```

### Judge Phase

```markdown
**Final Ranking:**

H2 (Downstream slow): 0.78 confidence
- Supported by: Network metrics, timeout random pattern
- Attacked but defended: Logs show no local blocking

H1 (Local blocking): 0.45 confidence
- Contradicted by: Random timeout pattern, low CPU
```

## 7. Safety & Compliance

### Data Privacy

- No logs containing PII should be parsed without consent
- Use log redaction patterns
- Restrict metrics queries to non-sensitive namespaces

### Resource Guards

```python
execution_limits = {
    'cloudwatch_queries': 10,           # per minute
    'code_execution': 5,                # concurrent
    'log_retention': '30 days',         # cleanup old logs
}
```

### Audit Trail

```
All tool invocations are logged:
- Tool name, timestamp, parameters
- Results (sanitized if sensitive)
- Confidence adjustment applied
- Agent that invoked it
```

## 8. CLI Examples

```bash
# Query logs for pattern
are-tool logs query \
  --source cloudwatch \
  --group /aws/lambda/payment-api \
  --pattern "timeout" \
  --time-window 1h

# Run diagnostic code
are-tool code run \
  --language node \
  --file ./diagnostics/event-loop-test.js \
  --timeout 30s

# Check metrics
are-tool metrics query \
  --source cloudwatch \
  --namespace AWS/RDS \
  --metrics CPUUtilization,DatabaseConnections \
  --time-window 1h
```

---

**End of Tool Integration Guide**
