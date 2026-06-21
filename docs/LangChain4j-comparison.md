# Agent-o-rama vs LangChain4j

## Similarities

| Feature | Summary |
|---------|---------|
| JVM-first | Both target JVM developers with first-class Java APIs (Agent-o-rama also has a Clojure API). |
| Tool integration | Both allow exposing regular Java methods as LLM-callable tools. |
| Streaming | Both support streaming LLM responses back to the client. |
| RAG | Both can integrate with vector stores and databases for RAG workflows. |

## Differences

| Area | Agent-o-rama | LangChain4j |
|------|--------------|------------|
| Scope | End-to-end agent platform: runtime, orchestration, storage, tracing, datasets, experiments, telemetry, UI. | Library only: provides LLMs, tools, embeddings, and basic agent patterns; all surrounding infra is user-built. |
| Execution model | Agents run as distributed, parallel graphs on a Rama cluster with built-in scaling. | Runs inside a single JVM process; no built-in distribution or parallel graph execution. |
| Storage | Built-in, high-performance, scalable, replicated storage (any data model) or external databases | No storage; you must run external databases yourself. |
| Tracing | Full structured tracing for every agent and node run, with tokens, latencies, DB calls, and model calls. | No tracing system |
| Datasets | Built-in versioned datasets for capturing inputs/outputs for use in experiments | No dataset concept |
| Experiments | First-class experiment runner to evaluate agent quality and performance with LLM or function evaluators | No experiment runner |
| Online evaluation / actions | Easy to set up custom hooks on production runs for online evaluation, adding to datasets, webhooks, and more. | No equivalent feature |
| Telemetry | Built-in time-series telemetry for agent performance, latency, token usage, model costs, online evaluation | No built-in telemetry |
| Agent model | Explicit graphs of Java/Clojure functions with parallel node execution | Agent control-flow is ad hoc in code via AiServices + program logic |
| Deployment | Runs on a Rama cluster (in-process, single-node, or distributed). Rama is the only dependency. Deploying/updating/scaling agents are one-line CLI commands. | Embeds in your application; all infra design (deployment, scaling, orchestration, monitoring) is your responsibility. |
| Integration | Integrates with LangChain4j for model access and adds a full platform on top. | Standalone; provides no equivalent observability, storage, evaluation, or orchestration layers. |

## Missing Pieces You Must Build Yourself if Using LangChain4j Alone

LangChain4j is intentionally a **library**, not a platform. Here is what JVM teams must engineer from scratch if using LangChain4j without Agent-o-rama:

### Runtime & Execution

* Distributed or parallel agent execution across threads/machines
* Backpressure, retries, timeouts, and fault-tolerance across agent steps
* Pause/resume mechanics for human-in-the-loop

### Storage

* One or more stores for:
    * agent state
    * datasets
    * experiments
    * traces
    * telemetry

### Deployment & Operations

* Infrastructure for:
    * scaling
    * clustering
    * job scheduling
    * distributed state
    * durability
* Rolling updates of new agent versions
* Developer tooling for local runs, testing, and debugging

### Tracing

* A way to persist, query, and visualize structured traces:
    * every node/step
    * model calls
    * tool calls
    * retries/failures
* Time-series metrics (latency, tokens, error rates, online evaluation, etc.)
* Dashboards and alerts (Prometheus/Grafana/etc.)

### Datasets and Experiments

* Versioned datasets with reproducibility guarantees
* Experiment runner to measure agent/node performance and quality
* LLM or code-based evaluators
