# Agent-o-rama vs LangGraph4j

## Similarities

| Feature | Summary |
|---------|---------|
| JVM-based | LangGraph4j has a Java API. Agent-o-rama has both Java and Clojure APIs. |
| Agent workflows | Both define agents as explicit graphs of functions. |
| Tool integration | Both allow exposing regular functions as tools callable by LLMs. |
| Control-flow patterns | Both support graph constructs such as branching, looping, and conditional logic. |
| Streaming | Both support streaming model responses. |

## Differences

| Area | Agent-o-rama | LangGraph4j |
|------|--------------|------------|
| Scope | Complete platform: runtime, storage, datasets, experiments, telemetry, UI. | Library for building agent graphs; no platform components. |
| Execution model | Distributed, parallel graph execution with no central coordination. | Executed inside one process with central state and limited ability to parallelize. |
| Threading model | Nodes run on virtual threads so code is easy and efficient to write in blocking style. | Uses traditional CompletableFuture-based async code, which is considerably more complex to write and harder to reason about. |
| Storage | Built-in, scalable, replicated storage (any data model) or external databases. | External databases only. |
| Datasets | Built-in versioned datasets for capturing inputs/outputs for use in experiments. | No dataset concept. |
| Experiments | First-class experiment runner to evaluate agent quality and performance with LLM or function evaluators. | No experiment runner. |
| Actions / online evaluation | Easy to set up custom hooks on production runs for online evaluation, adding to datasets, webhooks, and more. | No equivalent feature. |
| Telemetry | Built-in time-series telemetry for performance, latency, tokens, costs, and online evaluation. | No equivalent feature. |
| Tracing | Full node-level tracing with a UI for graph execution, inputs/outputs, model/tool calls, database calls. | No equivalent feature. |
| UI | Unified UI for traces, datasets, experiments, and telemetry. | No UI. |
| Deployment | Runs on a Rama cluster (in-process, single-node, or distributed). Rama is the only dependency. Deploying/updating/scaling agents are one-line CLI commands. | Embedded in your application; deployment/scaling left entirely to the user. |

## Missing Pieces You Must Build Yourself if Using LangGraph4j Alone

LangGraph4j provides graph construction and execution patterns, but everything resembling a full agent platform must be built by you:

### Runtime and execution

- Distributed or parallel agent execution across threads/machines
- Backpressure, retries, timeouts, and fault-tolerance across agent steps

### Storage

- One or more stores for:
  - agent state
  - datasets
  - experiments
  - traces
  - telemetry

### Deployment and operations

- Infrastructure for:
  - scaling
  - clustering
  - job scheduling
  - distributed state
  - durability
- Rolling updates of new agent versions
- Developer tooling for local runs, testing, and debugging

### Datasets and experiments

- Versioned datasets with reproducibility guarantees
- Experiment runner to measure agent/node performance and quality
- LLM or code-based evaluators

### Tracing

- A way to persist, query, and visualize structured traces:
  - every node/step
  - model calls
  - tool calls
  - retries/failures
- Time-series metrics (latency, tokens, error rates, online evaluation, etc.)
- Dashboards and alerts (Prometheus/Grafana/etc.)

### Telemetry

- Time-series metrics (latency, tokens, errors, costs)
- Dashboards, alerts, and visualizations
- Long-term storage and indexing of traces and metrics
