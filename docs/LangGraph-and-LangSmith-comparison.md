# Agent-o-rama vs LangGraph and LangSmith

## Similarities

Both platforms share key features including:

- **Agent definitions**: Both define agents as explicit graphs of regular functions
- Streaming capabilities for LLM outputs
- Forking previous executions for testing variations
- Structured tracing with node-level details
- Versioned dataset support
- Experiment testing with evaluators
- Online evaluation hooks and webhooks
- Human feedback collection mechanisms
- Human input pause/resume functionality
- Performance telemetry and metrics

## Key Differences

| Area | Agent-o-rama | LangGraph/LangSmith |
|------|--------------|-------------------|
| **Runtime** | JVM with Java/Clojure APIs | Python-based |
| **Execution** | Distributed, parallel execution with no central coordinator | Single Python process with centralized state |
| **Threading** | Virtual threads for efficiency | Standard threads |
| **Storage** | Built-in scalable storage | External databases only |
| **Human input** | Function call suspension model | Exception-based breakpoints |
| **Deployment** | Self-hosted Rama cluster (free tier available) | SaaS or enterprise self-hosted |
| **Feature gaps** | Missing few-shot examples | (none listed) |

The comparison emphasizes Agent-o-rama's distributed architecture and deployment simplicity versus LangGraph's Python ecosystem focus.
