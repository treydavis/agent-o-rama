# Agent-o-rama vs Spring AI

## Similarities

| Feature | Summary |
|---------|---------|
| JVM-based | Spring AI is Spring Boot–native; Agent-o-rama provides first-class Java and Clojure APIs. |
| Model abstraction | Both provide unified client APIs over multiple LLM providers. |
| Tool integration | Both allow exposing regular functions as tools for LLMs. |
| Streaming | Both support streaming model responses. |
| RAG | Both integrate with database and vector stores to support RAG-style retrieval workflows. |
| Evaluation utilities | Both provide LLM and code-based evaluators, but only Agent-o-rama includes the surrounding system for datasets, experiments, and online evaluation; Spring AI leaves all of that to the user. |

## Differences

| Area | Agent-o-rama | Spring AI |
|------|--------------|-----------|
| Scope | Complete platform: runtime, storage, datasets, experiments, telemetry, UI. | Library focused on AI integration inside Spring apps; platform pieces left to the user. |
| Agent model | Explicit graphs of Java/Clojure functions with parallel execution. | No agent graph runtime; control flow is ad hoc in application code. |
| Execution model | Distributed, parallel graph execution with built-in scaling on a Rama cluster. | Executes inside a Spring Boot service; no built-in distributed orchestration. |
| Human-in-the-loop | Built-in pause/resume API for requesting human input during execution. | No equivalent feature. |
| Storage | Built-in, scalable, replicated storage (any data model) or external databases. | No general-purpose storage engine; applications rely on Spring data sources or separate databases that you operate. |
| Datasets | Built-in versioned datasets for capturing inputs/outputs. | No equivalent feature. |
| Experiments | Built-in experiment runner for evaluating whole agents or individual nodes with LLM or function evaluators. | Provides evaluators but no experiment runner. |
| Actions / online evaluation | Easy to set up custom hooks on production runs for online evaluation, adding to datasets, webhooks, and more. | No equivalent feature. |
| Telemetry | Built-in time-series telemetry for agent performance, latency, token usage, model costs, and online evaluation. | Uses Spring observability integrations; no built-in dashboards or time-series store. |
| UI | Includes UI for traces, datasets, experiments, telemetry. | No agent-level UI; relies on generic Spring observability tools. |
| Deployment | Runs on a Rama cluster (in-process, single-node, or distributed). Rama is the only dependency. Deploying/updating/scaling agents are one-line CLI commands. | Embedded in Spring Boot apps; deployment and scaling rely on your own infrastructure and offer no built-in orchestration. |

## Missing Pieces You Must Build Yourself If Using Spring AI

Spring AI provides model clients, tools, RAG utilities, and basic evaluators, but everything resembling an agent platform must be built by you:

### Runtime and Execution

- Distributed or parallel agent execution across threads/machines
- Backpressure, retries, timeouts, and fault-tolerance across agent steps
- Pause/resume mechanics for human-in-the-loop

### Storage

- One or more stores for:
  - agent state
  - datasets
  - experiments
  - traces
  - telemetry

### Deployment and Operations

- Infrastructure for:
  - scaling
  - clustering
  - job scheduling
  - distributed state
  - durability
- Rolling updates of new agent versions
- Developer tooling for local runs, testing, and debugging

### Datasets and Experiments

- Versioned datasets with reproducibility guarantees
- Experiment runner to measure agent/node performance and quality
- LLM or code-based evaluators

### Telemetry and Tracing

- End-to-end tracing across workflow steps, database calls, and services, plus any UI for viewing and querying traces; Spring AI only emits data about its own model/tool calls
- Provides tracing hooks for model and tool calls, but no agent-level tracing or UI; all workflow, database, and cross-service tracing must be instrumented and maintained by the user
- Time-series metrics (latency, tokens, errors, costs) can be emitted, but you must choose and operate the backend yourself
- Dashboards, alerting, and visualizations must be built using external tools (Prometheus, Grafana, Langfuse, OpenTelemetry backends, etc.)
- Long-term storage and querying of traces and metrics must also be set up and maintained by your team
