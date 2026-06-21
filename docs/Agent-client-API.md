# Agent Client API

The Agent Client API is how you interact with deployed agents from your application code. It provides methods for invoking agents, tracking executions, and retrieving results.

## Table of Contents

1. Getting an Agent Client
2. Invoking Agents
3. Initiating Agent Executions
4. Invoking with Metadata
5. Getting Agent Results
6. Other Features

## Getting an Agent Client

Before you can invoke an agent, you need to get an `AgentClient` instance. This requires an `AgentManager`, which you create from a Rama cluster and module name.

### Cluster Managers

The `AgentManager` is created from a cluster manager, which is the interface to a Rama cluster. There are two types of cluster managers:

1. **InProcessCluster (IPC)**: For local development and testing. Runs a complete Rama cluster in a single JVM process.
2. **RamaClusterManager**: For production. Connects to a deployed Rama cluster.

Both implement the same `ClusterManagerBase` interface, so your agent code works the same way in development and production.

### Local Development with InProcessCluster

#### Java API

```java
import com.rpl.agentorama.*;
import com.rpl.rama.test.InProcessCluster;
import com.rpl.rama.test.LaunchConfig;

try (InProcessCluster ipc = InProcessCluster.create()) {
  MyAgentModule module = new MyAgentModule();
  ipc.launchModule(module, new LaunchConfig(4, 2));

  String moduleName = module.getModuleName();
  AgentManager manager = AgentManager.create(ipc, moduleName);

  AgentClient agent = manager.getAgentClient("MyAgent");

  Set<String> agentNames = manager.getAgentNames();
  System.out.println("Available agents: " + agentNames);
}
```

#### Clojure API

```clojure
(require '[com.rpl.agent-o-rama :as aor]
         '[com.rpl.rama :as rama]
         '[com.rpl.rama.test :as rtest])

(with-open [ipc (rtest/create-ipc)]
  (rtest/launch-module! ipc MyAgentModule {:tasks 4 :threads 2})

  (let [module-name (rama/get-module-name MyAgentModule)
        manager (aor/agent-manager ipc module-name)]

    (let [agent (aor/agent-client manager "MyAgent")]

      (println "Available agents:" (aor/agent-names manager)))))
```

### Production with RamaClusterManager

#### Java API

```java
import com.rpl.agentorama.*;
import com.rpl.rama.RamaClusterManager;

try (RamaClusterManager cluster = RamaClusterManager.open(Map.of("conductor.host", "1.2.3.4"))) {
  AgentManager manager = AgentManager.create(cluster, "MyModule");

  AgentClient agent = manager.getAgentClient("MyAgent");

  String result = agent.invoke("input data");
  System.out.println("Result: " + result);
}
```

#### Clojure API

```clojure
(require '[com.rpl.agent-o-rama :as aor]
         '[com.rpl.rama :as rama])

(with-open [cluster (rama/open-cluster {"conductor.host" "1.2.3.4"})]
  (let [manager (aor/agent-manager cluster "MyModule")]

    (let [agent (aor/agent-client manager "MyAgent")]

      (let [result (aor/agent-invoke agent "input data")]
        (println "Result:" result)))))
```

## Invoking Agents

### Synchronous Invocation

#### Java API

```java
// Single argument
String result = agent.invoke("Hello, world!");

// Multiple arguments
Map<String, Object> result = agent.invoke("query", "context", 42);
```

#### Clojure API

```clojure
;; Single argument
(def result (aor/agent-invoke agent "Hello, world!"))

;; Multiple arguments
(def result (aor/agent-invoke agent "query" "context" 42))
```

### Asynchronous Invocation

#### Java API

```java
import java.util.concurrent.CompletableFuture;

CompletableFuture<String> future = agent.invokeAsync("Hello, world!");

System.out.println("Agent is running...");

String result = future.get();
System.out.println("Result: " + result);

future.thenAccept(result -> {
  System.out.println("Agent completed with: " + result);
});
```

#### Clojure API

```clojure
(let [future (aor/agent-invoke-async agent "Hello, world!")]

  (println "Agent is running...")

  (let [result (.get future)]
    (println "Result:" result))

  (.thenAccept future
    (reify java.util.function.Consumer
      (accept [_ result]
        (println "Agent completed with:" result)))))
```

## Initiating Agent Executions

For more control over agent execution (e.g., for streaming or human input), use `initiate` to start an execution and get a handle for tracking it.

### Java API

```java
AgentInvoke invoke = agent.initiate("Hello, world!");

String result = agent.result(invoke);
```

### Clojure API

```clojure
(let [invoke (aor/agent-initiate agent "Hello, world!")]

  (let [result (aor/agent-result agent invoke)]
    (println "Result:" result)))
```

### Async Initiation

#### Java API

```java
CompletableFuture<AgentInvoke> future = agent.initiateAsync("Hello, world!");

future.thenAccept(invoke -> {
  String result = agent.result(invoke);
  System.out.println("Result: " + result);
});
```

#### Clojure API

```clojure
(let [future (aor/agent-initiate-async agent "Hello, world!")]
  (.thenAccept future
    (reify java.util.function.Consumer
      (accept [_ invoke]
        (let [result (aor/agent-result agent invoke)]
          (println "Result:" result))))))
```

## Invoking with Metadata

Metadata allows you to attach custom key-value data to agent executions. This is useful for:

- **Tracking**: User IDs, session IDs, request IDs for correlating agent executions
- **A/B Testing**: Model versions, feature flags, experimental configurations
- **Configuration**: Runtime parameters like model names that agents can access
- **Debugging**: Additional context for troubleshooting specific executions

Metadata keys must be strings, and values must be strings, numbers (int, long, float, double), or booleans.

### Creating Metadata Context

#### Java API

```java
import com.rpl.agentorama.AgentContext;

AgentContext context = AgentContext.metadata("user-id", "user-123")
                                   .metadata("model", "gpt-4");
```

#### Clojure API

```clojure
(def context {:metadata {"user-id" "user-123"
                         "model" "gpt-4"}})
```

### Invoking with Metadata

#### Java API

```java
AgentContext context = AgentContext.metadata("user-id", "user-123")
                                   .metadata("model", "gpt-4");

String result = agent.invokeWithContext(context, "Hello, world!");
System.out.println("Result: " + result);

CompletableFuture<String> future = agent.invokeWithContextAsync(context, "Hello, world!");
String result2 = future.get();
```

#### Clojure API

```clojure
(let [context {:metadata {"user-id" "user-123"
                          "model" "gpt-4"}}
      result (aor/agent-invoke-with-context agent context "Hello, world!")]
  (println "Result:" result))

(let [context {:metadata {"user-id" "user-123"
                          "model" "gpt-4"}}
      future (aor/agent-invoke-with-context-async agent context "Hello, world!")
      result (.get future)]
  (println "Result:" result))
```

### Initiating with Metadata

#### Java API

```java
AgentContext context = AgentContext.metadata("user-id", "user-123")
                                   .metadata("session-id", "session-456");

AgentInvoke invoke = agent.initiateWithContext(context, "Hello, world!");

agent.stream(invoke, "process", (allChunks, newChunks, reset, complete) -> {
  // Handle streaming...
});

String result = agent.result(invoke);

CompletableFuture<AgentInvoke> futureInvoke =
  agent.initiateWithContextAsync(context, "Hello, world!");
```

#### Clojure API

```clojure
(let [context {:metadata {"user-id" "user-123"
                          "session-id" "session-456"}}
      invoke (aor/agent-initiate-with-context agent context "Hello, world!")]

  (aor/agent-stream agent invoke "process"
    (fn [all-chunks new-chunks reset? complete?]
      ;; Handle streaming...
      ))

  (let [result (aor/agent-result agent invoke)]
    (println "Result:" result)))
```

## Getting Agent Results

### With invoke/invokeAsync

```java
// Java - result is returned
String result = agent.invoke("input");
```

```clojure
;; Clojure - result is returned
(def result (aor/agent-invoke agent "input"))
```

### With initiate

#### Java API

```java
AgentInvoke invoke = agent.initiate("input");

String result = agent.result(invoke);

CompletableFuture<String> futureResult = agent.resultAsync(invoke);
```

#### Clojure API

```clojure
(let [invoke (aor/agent-initiate agent "input")]

  (let [result (aor/agent-result agent invoke)]
    (println "Result:" result))

  (let [future-result (aor/agent-result-async agent invoke)]
    (.thenAccept future-result
      (reify java.util.function.Consumer
        (accept [_ result]
          (println "Result:" result))))))
```

### Checking Completion Status

#### Java API

```java
AgentInvoke invoke = agent.initiate("input");

if (agent.isAgentInvokeComplete(invoke)) {
  String result = agent.result(invoke);
  System.out.println("Already complete: " + result);
} else {
  System.out.println("Still running...");
}
```

#### Clojure API

```clojure
(let [invoke (aor/agent-initiate agent "input")]
  (if (aor/agent-invoke-complete? agent invoke)
    (let [result (aor/agent-result agent invoke)]
      (println "Already complete:" result))
    (println "Still running...")))
```

## Other Features

### Streaming

For real-time feedback as agents process data, use streaming. See the [Streaming](Streaming.md) documentation for details.

### Human Input

For human-in-the-loop patterns, agents can request human input during execution. See the [Human-in-the-loop](Human-in-the-loop.md) documentation for details.
