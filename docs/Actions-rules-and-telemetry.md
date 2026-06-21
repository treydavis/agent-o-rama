# Actions, Rules, and Telemetry

## Overview

Agent-o-rama's system enables automatic actions on production executions. Users define filtering criteria to select which runs trigger actions, executed asynchronously. Applications include continuous quality evaluation, webhook triggers, and custom logic execution.

## Rules and Actions

Rules determine when actions execute. Upon agent completion, the system evaluates active rules against that execution. Matching filters trigger action execution with access to inputs, outputs, metadata, and timing information. The system maintains per-task cursors ensuring actions run once per matching execution.

Rules can target either entire agent executions or individual node executions within an agent, providing granular control over processing scope.

### Creating Rules

Rule creation requires:
1. **Rule name** - Names the rule; referenced by dependent rules for online evaluation
2. **Scope** - Targets agent as a whole or particular node runs
3. **Status filter** - Executes on successful runs, failed runs, or all runs
4. **Sampling rate** - Percentage of runs (0-1.0) to execute
5. **Start time** - Backfill application point for historical rule application
6. **Action** - Selected action with configured parameters

Optional filtering uses input/output, run statistics, or online evaluation results. Filters compose with "and", "not", and "or" operators.

## Built-in Actions

### aor/eval
Runs evaluators on executions, attaching resulting feedback viewable in traces and agent analytics.

### aor/add-to-dataset
Automatically adds agent or node runs to datasets. Users specify target dataset and JSON path templates for transforming input/output to desired shapes.

### aor/webhook
Posts JSON payloads to external URLs. Configuration includes URL, timeout, HTTP headers, and payload templates with replaceable placeholders.

## Custom Actions

Custom action builders tailored to specific needs receive four arguments: fetcher object (access to declared agent objects), input, output, and run info object. Actions return maps with string keys recorded in action logs.

### Java API Example

```java
@Override
protected void defineAgents(AgentTopology topology) {
  topology.declareActionBuilder("print-action",
    "Prints info about the run to stdout",
    (Map<String, String> params) -> {
      String logLevel = params.get("logLevel");
      return (AgentObjectFetcher fetcher, Object input, Object output, RunInfo runInfo) -> {
        System.out.println(String.format("[%s] Agent: %s, Latency: %d ms",
          logLevel, runInfo.getAgentName(), runInfo.getLatencyMillis()));
        return new HashMap<String, Object>() {{
          put("logged", true);
        }};
      };
    },
    ActionBuilderOptions.params("logLevel", "Log level (info/debug)", "info"));
}
```

### Clojure API Example

```clojure
(aor/defagentmodule MyModule
  [topology]
  (aor/declare-action-builder
    topology
    "print-action"
    "Prints info about the run to stdout"
    (fn [params]
      (let [log-level (get params "logLevel")]
        (fn [fetcher input output run-info]
          (println (format "[%s] Agent: %s, Latency: %s ms"
                           log-level
                           (:agent-name run-info)
                           (:latency-millis run-info)))
          {"logged" true})))
    {:params {"logLevel" {:description "Log level (info/debug)" :default "info"}}})
  ;; ... rest of module definition ...
  )
```

## Action Logs

Every action execution logs detailed information including start/finish times, associated execution, success/failure status, and returned information. Logs capture error details with exception messages and stack traces. Users access logs from the rules list and navigate to associated agent run traces.

## Time-Series Telemetry

Analytics pages display time-series telemetry from all executions, aggregating metrics at multiple granularities:

- **Agent-level**: success rates, end-to-end latency, token counts, time-to-first-token
- **Model-level**: call counts, success rates, latencies for LLM interactions
- **Store/Database**: read and write latencies

Users customize granularity, time windows, and split charts by metadata keys. The system automatically aggregates per metadata value and time bucket. At most five metadata values track per bucket for optimal low-cardinality performance.

### Evaluator Telemetry

When rules execute evaluators on agent executions, resulting feedback scores automatically become additional telemetry metrics. Each evaluator rule generates time-series data for every score, enabling continuous quality metric monitoring alongside operational metrics.
