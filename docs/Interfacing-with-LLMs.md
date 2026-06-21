# Interfacing with LLMs

## Overview

Agent-o-rama integrates with LangChain4j for AI model interactions, automatically capturing traces and streaming model calls. However, the framework is fundamentally an orchestration tool where developers can integrate with any tools or services needed. As stated in the documentation: "using LangChain4j is completely optional."

## Key Features

**Automatic Tracing**
When LangChain4j chat models are declared as agent objects, the framework automatically wraps them to capture execution details including timing, token counts, request/response data, and model metadata.

**Streaming Model Support**
When declaring a `StreamingChatModel` as an agent object, Agent-o-rama automatically wraps it for token streaming. Notably, when fetching the model in an agent node, developers receive a `ChatModel` interface rather than `StreamingChatModel`, allowing streaming models to be used in a blocking style while clients can subscribe to real-time token streams.

**Structured Outputs**
LangChain4j supports structured outputs through JSON schema enforcement. However, this feature currently works only with non-streaming chat models due to LangChain4j limitations.

## Integration Flexibility

Developers using alternative LLM providers can manually record nested operations using the `recordNestedOp` method on `AgentNode`. Available operation types include `MODEL_CALL`, `TOOL_CALL`, `STORE_READ`, `STORE_WRITE`, `DB_READ`, `DB_WRITE`, `AGENT_CALL`, `HUMAN_INPUT`, and `OTHER`, enabling consistent tracing and telemetry across different tools.
