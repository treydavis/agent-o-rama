# Agent-o-rama vs Embabel

## Similarities

| Feature | Summary |
|---------|---------|
| JVM-based | Both run on the JVM. Embabel has Java and Kotlin templates; Agent-o-rama has first-class Java and Clojure APIs. |
| Agent workflows | Both support multi-step agent workflows, though with completely different approaches. |
| Tool integration | Both allow exposing regular functions as tools for LLMs. |
| RAG | Both integrate with database and vector stores to support RAG-style retrieval workflows. |

## Differences

| Area | Agent-o-rama | Embabel |
|------|--------------|---------|
| Scope | Complete platform: runtime, storage, datasets, experiments, telemetry, UI. | Agent framework on top of Spring AI; focuses on agent logic Spring integration. |
| Control model | Agents are explicit graphs of functions with named nodes/edges and parallel execution. | Agents are non-deterministic and modeled with actions, goals, conditions. A planner decides which actions to run and in what order. |
| Execution model | Distributed, parallel graph execution with built-in scaling on a Rama cluster. | Runs inside your process (typically a Spring Boot app); any clustering or horizontal scaling is up to you. |
| Human-in-the-loop | Built-in pause/resume API for requesting human input during execution. | No equivalent feature, must be built manually. |
| Storage | Built-in, high-performance, scalable, replicated storage (any data model) or external databases. | No general-purpose storage engine; applications rely on Spring data sources or separate databases that you operate. |
| Datasets | Built-in versioned datasets for capturing inputs/outputs for use in experiments. | No equivalent feature. |
| Experiments | Built-in experiment runner for evaluating whole agents or individual nodes with LLM or function evaluators. | No experiment runner. |
| Actions / online evaluation | Easy to set up custom hooks on production runs for online evaluation, adding to datasets, webhooks, and more. | No equivalent feature. |
| Telemetry | Built-in time-series telemetry for agent performance, latency, token usage, model costs, and online evaluation. | Provides telemetry integrations (e.g. OpenTelemetry) but no built-in time-series storage or dashboards. No online evaluation. |
| UI | Includes UI for traces, datasets, experiments, telemetry. | No general-purpose evaluation/observability UI. |
| Deployment | Runs on a Rama cluster (in-process, single-node, or distributed). Rama is the only dependency. Deploying/updating/scaling agents are one-line CLI commands. | Runs inside Spring-based applications or other JVM apps; deployment, scaling, and orchestration are your responsibility. |

## Missing Pieces You Must Build Yourself If Using Embabel Alone

Embabel gives you a planning-based agent framework on the JVM, but you must supply the platform around it.

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
- Time-series metrics (latency, tokens, errors, costs) can be emitted, but you must choose and operate the backend yourself
- Dashboards, alerting, and visualizations must be built using external tools (Prometheus, Grafana, Langfuse, OpenTelemetry backends, etc.)
- Long-term storage and querying of traces and metrics must also be set up and maintained by your team
