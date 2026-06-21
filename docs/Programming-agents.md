# Programming Agents

Agent-o-rama is a library for building LLM agents as directed graphs. Nodes are the fundamental computation units in agent graphs. Each node is a plain Java or Clojure function that receives data, processes it, and either passes it along to other nodes or returns a final result. This is the basic building block that enables all other agent patterns. Agent-o-rama executes all nodes on [virtual threads](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html), which means node functions can be long-running and written in a blocking style without wasting thread resources.

Agent-o-rama captures all inputs, nested operations (e.g. model calls or database operations), and outputs from each node for viewing in the web UI. This information is also used to produce and display aggregated analytics about individual agent executions and time-series analytics for all agent executions.

Besides tracing, nodes are also the granularity at which streaming is consumed by agent clients. Things like calls to [Langchain4j](https://docs.langchain4j.dev/) models are automatically streamed for the node, and node functions can explicitly stream chunks back as well. This is discussed more on the [agent client](Agent-client-API.md) page.

## Key Components

* **AgentGraph**: The builder interface for defining agent execution graphs
* **AgentNode**: The interface for interacting with the agent execution environment from within nodes
* **AgentTopology**: The interface for defining agents, stores, and objects
* **AgentClient**: The interface for invoking agents and managing executions

## Understanding the Flow

Every agent execution starts with an invocation that provides input data to the first node. From there, data flows through the graph via `emit()` calls, which send data to downstream nodes. The execution continues until a node calls `result()`, which terminates the agent and returns the final output.

The `outputNodesSpec` parameter when defining nodes is crucial - it declares which nodes can receive data from this node. This creates a contract that the runtime enforces, preventing errors from emitting to undeclared nodes.

## Simple Example: Greeting Pipeline

### Java API

```java
import com.rpl.agentorama.*;

public class BasicAgentModule extends AgentModule {
  @Override
  protected void defineAgents(AgentTopology topology) {
    topology.newAgent("BasicAgent")
            .node("start", "process", (AgentNode agentNode, String input) -> {
                agentNode.emit("process", "Hello " + input);
            })
            .node("process", null, (AgentNode agentNode, String data) -> {
                agentNode.result("Processed: " + data);
            });
  }
}
```

### Clojure API

```clojure
(require '[com.rpl.agent-o-rama :as aor])

(aor/defagentmodule BasicAgentModule
  [topology]
  (-> (aor/new-agent topology "BasicAgent")
      (aor/node
       "start"
       "process"
       (fn [agent-node input]
         (aor/emit! agent-node "process" (str "Hello " input))))
      (aor/node
       "process"
       nil
       (fn [agent-node data]
         (aor/result! agent-node (str "Processed: " data))))))
```

## Key Concepts

* **emit()**: Sends data to another node in the agent graph
* **result()**: Sets the final result of the agent execution (first-one-wins)
* **outputNodesSpec**: Declares which nodes can receive emissions from this node. This is either a single node name string, a list of node names, or null to indicate a terminal node.

## Routing in Agent Graphs

While simple linear pipelines are useful, real-world agents often need complex control flow. Agent graphs support loops, conditional routing, and multiple execution paths that can reconverge. This enables sophisticated decision-making and parallel processing within a single agent.

### Conditional Routing Example

#### Java API

```java
public class RouterAgentModule extends AgentModule {
  @Override
  protected void defineAgents(AgentTopology topology) {
    topology.newAgent("RouterAgent")
            .node("route", new String[]{"handle-urgent", "handle-default"},
                  (AgentNode agentNode, String message) -> {
              if (message.startsWith("urgent:")) {
                  agentNode.emit("handle-urgent", message);
              } else {
                  agentNode.emit("handle-default", message);
              }
            })
            .node("handle-urgent", "finalize", (AgentNode agentNode, String message) -> {
              String content = message.substring(7);
              agentNode.emit("finalize", Map.of("priority", "HIGH", "message", content));
            })
            .node("handle-default", "finalize", (AgentNode agentNode, String message) -> {
              agentNode.emit("finalize", Map.of("priority", "NORMAL", "message", message));
            })
            .node("finalize", null, (AgentNode agentNode, Map<String, String> data) -> {
              String result = String.format("[%s] %s", data.get("priority"), data.get("message"));
              agentNode.result(result);
            });
  }
}
```

#### Clojure API

```clojure
(aor/defagentmodule RouterAgentModule
  [topology]
  (-> (aor/new-agent topology "RouterAgent")
      (aor/node
       "route"
       ["handle-urgent" "handle-default"]
       (fn [agent-node message]
         (if (str/starts-with? message "urgent:")
           (aor/emit! agent-node "handle-urgent" message)
           (aor/emit! agent-node "handle-default" message))))
      (aor/node
       "handle-urgent"
       "finalize"
       (fn [agent-node message]
         (aor/emit! agent-node "finalize" {"priority" "HIGH" "message" (subs message 7)})))
      (aor/node
       "handle-default"
       "finalize"
       (fn [agent-node message]
         (aor/emit! agent-node "finalize" {"priority" "NORMAL" "message" message})))
      (aor/node
       "finalize"
       nil
       (fn [agent-node {:strs [priority message]}]
         (aor/result! agent-node (format "[%s] %s" priority message))))))
```

### Emitting Multiple Times

When a node emits multiple times, the first emit runs on the same node/thread, but subsequent emits will run in parallel on other threads or even other nodes. This means agent graphs automatically parallelize and distribute execution. A node can emit any number of times to any number of downstream nodes.

If multiple nodes call `result()`, only the first one wins – subsequent results are ignored.

#### Java API

```java
public class MultiEmitAgentModule extends AgentModule {
  @Override
  protected void defineAgents(AgentTopology topology) {
    topology.newAgent("MultiEmitAgent")
            .node("start", new String[]{"process-a", "process-b"}, (AgentNode agentNode, String input) -> {
              agentNode.emit("process-a", input + "-A1");
              agentNode.emit("process-b", input + "-B");
              agentNode.emit("process-a", input + "-A2");
            })
            .node("process-a", "finalize", (AgentNode agentNode, String data) -> {
              Thread.sleep(100);
              agentNode.emit("finalize", "Result A: " + data);
            })
            .node("process-b", "finalize", (AgentNode agentNode, String data) -> {
              Thread.sleep(50);
              agentNode.emit("finalize", "Result B: " + data);
            })
            .node("finalize", null, (AgentNode agentNode, String result) -> {
              agentNode.result(result);
            });
  }
}
```

#### Clojure API

```clojure
(aor/defagentmodule MultiEmitAgentModule
  [topology]
  (-> (aor/new-agent topology "MultiEmitAgent")
      (aor/node
       "start"
       ["process-a" "process-b"]
       (fn [agent-node input]
         (aor/emit! agent-node "process-a" (str input "-A"))
         (aor/emit! agent-node "process-b" (str input "-B"))
         (aor/emit! agent-node "process-a" (str input "-A"))))
      (aor/node
       "process-a"
       "finalize"
       (fn [agent-node data]
         (Thread/sleep 100)
         (aor/emit! agent-node "finalize" (str "Result A: " data))))
      (aor/node
       "process-b"
       "finalize"
       (fn [agent-node data]
         (Thread/sleep 50)
         (aor/emit! agent-node "finalize" (str "Result B: " data))))
      (aor/node
       "finalize"
       nil
       (fn [agent-node result]
         (aor/result! agent-node result)))))
```

## Aggregation Subgraphs

Aggregation subgraphs enable fan-out/fan-in patterns where work is distributed to multiple parallel nodes and results are collected and combined. This is essential for handling multiple concurrent operations, like making multiple LLM calls in parallel and then combining the results.

### Basic Aggregation Example

#### Java API

```java
public class AggregationAgentModule extends AgentModule {
  @Override
  protected void defineAgents(AgentTopology topology) {
    topology.newAgent("AggregationAgent")
            .aggStartNode("distribute-work", "process-item", (AgentNode agentNode, List<String> items) -> {
              for (String item: items) {
                  agentNode.emit("process-item", item);
              }
              return null;
            })
            .node("process-item", "collect-results", (AgentNode agentNode, String item) -> {
              String processed = "Processed: " + item.toUpperCase();
              agentNode.emit("collect-results", processed);
            })
            .aggNode("collect-results", null, BuiltIn.LIST_AGG,
                     (AgentNode agentNode, List<String> results, Object nodeStartRes) -> {
              agentNode.result(results);
            });
  }
}
```

#### Clojure API

```clojure
(aor/defagentmodule AggregationAgentModule
  [topology]
  (-> (aor/new-agent topology "AggregationAgent")
      (aor/agg-start-node
       "distribute-work"
       "process-item"
       (fn [agent-node items]
         (doseq [item items]
           (aor/emit! agent-node "process-item" item))))
      (aor/node
       "process-item"
       "collect-results"
       (fn [agent-node item]
         (let [processed (str "Processed: " (str/upper-case item))]
           (aor/emit! agent-node "collect-results" processed))))
      (aor/agg-node
       "collect-results"
       nil
       aggs/+vec-agg
       (fn [agent-node results _]
         (aor/result! agent-node results)))))
```

### Aggregation Scope

Aggregation subgraphs can be nested. Agg start nodes are the only nodes that have return values. The return value is passed as the last argument to the corresponding agg node, allowing you to pass non-aggregated information through the aggregation.

#### Java API

```java
public class NestedAggregationModule extends AgentModule {
  @Override
  protected void defineAgents(AgentTopology topology) {
    topology.newAgent("NestedAggregationAgent")
            .aggStartNode("distribute-docs", "analyze-doc", (AgentNode agentNode, List<String> docs) -> {
              for (String doc : docs) {
                agentNode.emit("analyze-doc", doc);
              }
              return docs.size();
            })
            .aggStartNode("analyze-doc", "analyze-method", (AgentNode agentNode, String doc) -> {
              agentNode.emit("analyze-method", doc, "sentiment");
              agentNode.emit("analyze-method", doc, "keywords");
              agentNode.emit("analyze-method", doc, "summary");
              return doc;
            })
            .node("analyze-method", "combine-analysis", (AgentNode agentNode, String doc, String method) -> {
              String result = method + " analysis of: " + doc;
              agentNode.emit("combine-analysis", Map.of("method", method, "result", result));
            })
            .aggNode("combine-analysis", "collect-docs", BuiltIn.LIST_AGG,
                     (AgentNode agentNode, List<Map<String, String>> analyses, String originalDoc) -> {
              agentNode.emit("collect-docs", Map.of("doc", originalDoc, "analyses", analyses));
            })
            .aggNode("collect-docs", null, BuiltIn.LIST_AGG,
                     (AgentNode agentNode, List<Map<String, Object>> allResults, Integer totalDocs) -> {
              agentNode.result(Map.of("total-docs", totalDocs, "results", allResults));
            });
  }
}
```

#### Clojure API

```clojure
(aor/defagentmodule NestedAggregationModule
  [topology]
  (-> (aor/new-agent topology "NestedAggregationAgent")
      (aor/agg-start-node
       "distribute-docs"
       "analyze-doc"
       (fn [agent-node docs]
         (doseq [doc docs]
           (aor/emit! agent-node "analyze-doc" doc))
         (count docs)))
      (aor/agg-start-node
       "analyze-doc"
       "analyze-method"
       (fn [agent-node doc]
         (aor/emit! agent-node "analyze-method" doc "sentiment")
         (aor/emit! agent-node "analyze-method" doc "keywords")
         (aor/emit! agent-node "analyze-method" doc "summary")
         doc))
      (aor/node
       "analyze-method"
       "combine-analysis"
       (fn [agent-node doc method]
         (let [result (str method " analysis of: " doc)]
           (aor/emit! agent-node "combine-analysis" {:method method :result result}))))
      (aor/agg-node
       "combine-analysis"
       "collect-docs"
       aggs/+vec-agg
       (fn [agent-node analyses original-doc]
         (aor/emit! agent-node "collect-docs" {:doc original-doc :analyses analyses})))
      (aor/agg-node
       "collect-docs"
       nil
       aggs/+vec-agg
       (fn [agent-node all-results total-docs]
         (aor/result! agent-node {:total-docs total-docs :results all-results})))))
```

### Custom Aggregators

Built-in aggregators handle most use cases, but sometimes you need custom logic. Agent-o-rama also has a special aggregator type called "multi aggregator" which can process different kinds of inputs. When using multi-aggregators, aggregation inputs specify which "target" to run by including a tag as the first argument to `emit()`.

#### Java API

```java
public class MultiAggAgentModule extends AgentModule {
  @Override
  protected void defineAgents(AgentTopology topology) {
    topology.newAgent("MultiAggAgent")
            .aggStartNode("distribute-data", Arrays.asList("process-numbers", "process-text"),
                          (AgentNode agentNode, Map<String, Object> data) -> {
              List<Integer> numbers = (List<Integer>) data.get("numbers");
              List<String> text = (List<String>) data.get("text");

              for (Integer num : numbers) {
                  agentNode.emit("process-numbers", num);
              }
              for (String txt : text) {
                  agentNode.emit("process-text", txt);
              }
              return null;
            })
            .node("process-numbers", "combine-results", (AgentNode agentNode, Integer number) -> {
              agentNode.emit("combine-results", "number", number);
            })
            .node("process-text", "combine-results", (AgentNode agentNode, String text) -> {
              agentNode.emit("combine-results", "text", text);
            })
            .aggNode("combine-results", null,
                     MultiAgg.init(() -> {
                         Map<String, Object> state = new HashMap<>();
                         state.put("number-sum", 0);
                         state.put("text", "");
                         return state;
                     })
                     .on("number", (Map<String, Object> state, Integer num) -> {
                         state.put("number-sum", (Integer) state.get("number-sum") + num);
                         return state;
                     })
                     .on("text", (Map<String, Object> state, String txt) -> {
                         state.put("text", state.get("text") + txt + " ");
                         return state;
                     }),
                     (AgentNode agentNode, Map<String, Object> state, Object _) -> {
              agentNode.result(state);
            });
  }
}
```

#### Clojure API

```clojure
(aor/defagentmodule MultiAggAgentModule
  [topology]
  (-> (aor/new-agent topology "MultiAggAgent")
      (aor/agg-start-node
       "distribute-data"
       ["process-numbers" "process-text"]
       (fn [agent-node {:strs [numbers text]}]
         (doseq [num numbers] (aor/emit! agent-node "process-numbers" num))
         (doseq [txt text] (aor/emit! agent-node "process-text" txt))))
      (aor/node
       "process-numbers"
       "combine-results"
       (fn [agent-node number]
         (aor/emit! agent-node "combine-results" "number" number)))
      (aor/node
       "process-text"
       "combine-results"
       (fn [agent-node text]
         (aor/emit! agent-node "combine-results" "text" text)))
      (aor/agg-node
       "combine-results"
       nil
       (aor/multi-agg
        (init [] {"number-sum" 0 "text" ""})
        (on "number" [state num] (update state "number-sum" + num))
        (on "text" [state txt] (update state "text" str txt " ")))
       (fn [agent-node state _]
         (aor/result! agent-node state)))))
```

### Early Aggregation Return

Aggregators can be written to return early, causing aggregation to immediately finish before all incoming data has been processed.

#### Java API

```java
import com.rpl.agentorama.FinishedAgg;
import com.rpl.rama.ops.RamaAccumulatorAgg1;

public class SumUntil100 implements RamaAccumulatorAgg1<Integer, Integer> {
  @Override
  public Integer initVal() {
    return 0;
  }

  @Override
  public Integer accumulate(Integer curr, Integer value) {
    Integer newSum = curr + value;
    if (newSum > 100) {
      return new FinishedAgg(newSum);
    }
    return newSum;
  }
}
```

#### Clojure API

```clojure
(def +sum-until-100
  (accumulator
   (fn [v]
     (term (fn [curr]
             (let [ret (+ curr v)]
               (if (> ret 100)
                 (reduced ret)
                 ret
               ))
           )))
   :init-fn
   (constantly 0)))
```

## Metadata

Metadata allows you to attach custom key-value data to agent executions. Metadata is set when invoking an agent and can be accessed from any node within the agent execution.

Common use cases:
* **Tracking**: User IDs, session IDs, request IDs
* **A/B Testing**: Feature flags, model versions, experimental configurations
* **Configuration**: Runtime parameters like model names
* **Debugging**: Additional context for troubleshooting specific executions

### Setting Metadata

#### Java API

```java
import com.rpl.agentorama.AgentContext;

AgentContext context = AgentContext.metadata("user-id", "user-123")
                                   .metadata("model", "gpt-4");

String result = agent.invokeWithContext(context, "Hello, world!");
```

#### Clojure API

```clojure
(let [context {:metadata {"user-id" "user-123"
                          "model" "gpt-4"}}]

  (let [result (aor/agent-invoke-with-context agent context "Hello, world!")]
    (println "Result:" result)))
```

### Accessing Metadata in Agents

#### Java API

```java
public class MetadataAgentModule extends AgentModule {
  @Override
  protected void defineAgents(AgentTopology topology) {
    topology.newAgent("MetadataAgent")
            .node("process", null, (AgentNode agentNode) -> {
              Map<String, Object> metadata = agentNode.getMetadata();

              String userId = (String) metadata.get("user-id");
              String model = (String) metadata.get("model");

              System.out.println("Processing for user: " + userId);
              System.out.println("Using model: " + model);

              agentNode.result("Processed for " + userId);
            });
  }
}
```

#### Clojure API

```clojure
(aor/defagentmodule MetadataAgentModule
  [topology]
  (-> (aor/new-agent topology "MetadataAgent")
      (aor/node
       "process"
       nil
       (fn [agent-node]
         (let [metadata (aor/get-metadata agent-node)
               user-id (get metadata "user-id")
               model (get metadata "model")]

           (println "Processing for user:" user-id)
           (println "Using model:" model)

           (aor/result! agent-node (str "Processed for " user-id)))))))
```

## Fault-tolerance and Retries

Agent-o-rama has built-in fault-tolerance for agents. If a node fails, like due to an exception making an API call or a hardware failure on a cluster node, it will retry. By default, an agent can have at most two retries, and this is configurable in the web UI in the config page for the agent on the `max.retries` config.

## Agent Objects

Agent objects are shared resources like AI models, database connections, or API clients that agents can access during execution. Many resources like AI models and database connections are expensive to create and maintain persistent connections. Agent object builders allow you to create these resources once and reuse them across multiple agent invocations.

### Thread Safety and Pooling

1. **Thread-safe objects**: When declared with `threadSafe()`, one object is built for the entire process and reused across all node invokes on all threads.
2. **Pooled objects**: By default, a pool of objects is maintained, and nodes get exclusive access to an instance during execution. Pool size defaults to 100, configurable via `workerObjectLimit(amt)`.

### Static and Dynamic Objects

#### Java API

```java
public class AgentObjectsModule extends AgentModule {
  @Override
  protected void defineAgents(AgentTopology topology) {
    topology.declareAgentObject("openai-api-key", System.getenv("OPENAI_API_KEY"));

    topology.declareAgentObjectBuilder("openai-model", setup -> {
      String apiKey = setup.getAgentObject("openai-api-key");
      return OpenAiStreamingChatModel.builder()
                                    .apiKey(apiKey)
                                    .modelName("gpt-4o-mini")
                                    .build();
      },
      AgentObjectOptions.workerObjectLimit(200));

    topology.newAgent("AgentWithObjects")
            .node("process", null, (AgentNode agentNode, String input) -> {
              ChatModel model = agentNode.getAgentObject("openai-model");
              String response = model.chat(input);
              agentNode.result(response);
            });
  }
}
```

#### Clojure API

```clojure
(aor/defagentmodule AgentObjectsModule
  [topology]
  (aor/declare-agent-object topology "openai-api-key" (System/getenv "OPENAI_API_KEY"))

  (aor/declare-agent-object-builder
   topology
   "openai-model"
   (fn [setup]
     (-> (OpenAiStreamingChatModel/builder)
         (.apiKey (aor/get-agent-object setup "openai-api-key"))
         (.modelName "gpt-4o-mini")
         .build))
    {:worker-object-limit 200})

  (-> (aor/new-agent topology "AgentWithObjects")
      (aor/node
       "process"
       nil
       (fn [agent-node input]
         (let [model (aor/get-agent-object agent-node "openai-model")]
           (aor/result! agent-node (lc4j/basic-chat model input)))))))
```

### Streaming Chat Models

When you declare a `StreamingChatModel` as an agent object, Agent-o-rama automatically captures the stream and forwards chunks to the node. However, when you fetch the object in a node, you always get a `ChatModel` interface. If you don't want streaming behavior, declare the object as a non-streaming `ChatModel`.

## Stores

Agent-o-rama stores provide persistent data access for agents, enabling them to maintain state across invocations and share data between different agent executions. Stores are built-in and are high-performance, durable, scalable, and replicated.

There are three types of stores: key-value store, document store, and [PState](https://redplanetlabs.com/docs/~/pstates.html) store. Store names always begin with `$$`.

### Key-Value Store Example

#### Java API

```java
public class KeyValueStoreModule extends AgentModule {
  @Override
  protected void defineAgents(AgentTopology topology) {
    topology.declareKeyValueStore("$$counters", String.class, Integer.class);

    topology.newAgent("KeyValueStoreAgent")
            .node("manage-counter", null, (AgentNode agentNode, String counterName, String operation) -> {
              KeyValueStore<String, Integer> store = agentNode.getStore("$$counters");
              switch (operation) {
                  case "get":
                      Integer value = store.get(counterName);
                      agentNode.result(Map.of("counter", counterName, "value", value));
                      break;
                  case "increment":
                      Integer currentValue = store.get(counterName);
                      if (currentValue == null) currentValue = 0;
                      store.put(counterName, currentValue + 1);
                      agentNode.result(Map.of("counter", counterName, "new-value", currentValue + 1));
                      break;
              }
            });
  }
}
```

#### Clojure API

```clojure
(aor/defagentmodule KeyValueStoreModule
  [topology]
  (aor/declare-key-value-store topology "$$counters" String Long)

  (-> (aor/new-agent topology "KeyValueStoreAgent")
      (aor/node
       "manage-counter"
       nil
       (fn [agent-node counter-name operation]
         (let [store (aor/get-store agent-node "$$counters")]
           (case operation
             "get"
             (aor/result! agent-node {:counter counter-name :value (store/get store counter-name)})
             "increment"
             (let [current-value (or (store/get store counter-name) 0)
                   new-value (inc current-value)]
               (store/put! store counter-name new-value)
               (aor/result! agent-node {:counter counter-name :new-value new-value}))))))))
```

### Document Store Example

Document stores are key-value stores where the values are maps with their own schema for each field. You can perform operations on individual nested values without reading or writing the entire document.

#### Java API

```java
public class DocumentStoreModule extends AgentModule {
  @Override
  protected void defineAgents(AgentTopology topology) {
    topology.declareDocumentStore("$$user-profiles", String.class,
                                   "name", String.class,
                                   "age", Long.class);

    topology.newAgent("DocumentStoreAgent")
            .node("update-profile", "read-profile", (AgentNode agentNode, Map<String, Object> data) -> {
              DocumentStore store = agentNode.getStore("$$user-profiles");
              String userId = (String) data.get("user-id");
              Map<String, Object> updates = (Map<String, Object>) data.get("updates");

              if (updates.containsKey("name")) {
                store.putDocumentField(userId, "name", updates.get("name"));
              }
              if (updates.containsKey("age")) {
                store.putDocumentField(userId, "age", updates.get("age"));
              }

              agentNode.emit("read-profile", userId);
            })
            .node("read-profile", null, (AgentNode agentNode, String userId) -> {
              DocumentStore store = agentNode.getStore("$$user-profiles");
              String name = store.getDocumentField(userId, "name");
              Long age = store.getDocumentField(userId, "age");
              agentNode.result(Map.of("user-id", userId, "name", name, "age", age));
            });
  }
}
```

#### Clojure API

```clojure
(aor/defagentmodule DocumentStoreModule
  [topology]
  (aor/declare-document-store topology "$$user-profiles" String
                              "name" String
                              "age" Long)

  (-> (aor/new-agent topology "DocumentStoreAgent")
      (aor/node
       "update-profile"
       "read-profile"
       (fn [agent-node {:strs [user-id updates]}]
         (let [store (aor/get-store agent-node "$$user-profiles")]
           (when (contains? updates "name") (store/put-document-field! store user-id "name" (get updates "name")))
           (when (contains? updates "age") (store/put-document-field! store user-id "age" (get updates "age")))
           (aor/emit! agent-node "read-profile" user-id))))
      (aor/node
       "read-profile"
       nil
       (fn [agent-node user-id]
         (let [store (aor/get-store agent-node "$$user-profiles")
               name (store/get-document-field store user-id "name")
               age (store/get-document-field store user-id "age")]
           (aor/result! agent-node {:user-id user-id :name name :age age}))))))
```

### PState Store Example

PState stores provide direct access to Rama's [PStates](https://redplanetlabs.com/docs/~/pstates.html). They are used when you need more sophisticated structures than key-value or document stores provide, such as nested maps, lists with subindexing, or complex hierarchical data.

#### Java API

```java
import com.rpl.rama.Path;
import com.rpl.rama.PState;

public class PStateStoreModule extends AgentModule {
  @Override
  protected void defineAgents(AgentTopology topology) {
    topology.declarePStateStore(
      "$$user-data",
      PState.mapSchema(
        String.class,
        PState.fixedKeysSchema(
          "age", Integer.class,
          "memories", PState.listSchema(String.class).subindexed())));

    topology.newAgent("PStateStoreAgent")
            .node("update-user", "read-user", (AgentNode agentNode, Map<String, Object> data) -> {
              PStateStore store = agentNode.getStore("$$user-data");
              String userId = (String) data.get("user-id");
              Integer age = (Integer) data.get("age");
              String memory = (String) data.get("memory");

              if (age != null) {
                store.transform(userId, Path.key(userId, "age").termVal(age));
              }

              if (memory != null) {
                store.transform(userId, Path.key(userId, "memories").afterElem().termVal(memory));
              }

              agentNode.emit("read-user", userId);
            })
            .node("read-user", null, (AgentNode agentNode, String userId) -> {
              PStateStore store = agentNode.getStore("$$user-data");

              Integer age = (Integer) store.selectOne(Path.key(userId, "age"));

              List<String> memories = store.select(Path.key(userId, "memories").all());

              agentNode.result(Map.of("user-id", userId, "age", age, "memories", memories));
            });
  }
}
```

#### Clojure API

```clojure
(require '[com.rpl.rama.path :as path])

(aor/defagentmodule PStateStoreModule
  [topology]
  (aor/declare-pstate-store
   topology
   "$$user-data"
   {String (fixed-keys-schema
            {:age Long
             :memories (vector-schema String {:subindex? true})})})

  (-> topology
      (aor/new-agent "PStateStoreAgent")
      (aor/node
       "update-user"
       "read-user"
       (fn [agent-node {:strs [user-id age memory]}]
         (let [store (aor/get-store agent-node "$$user-data")]
           (when age
             (store/pstate-transform!
              [(path/keypath user-id :age) (path/termval age)]
              store
              user-id))

           (when memory
             (store/pstate-transform!
              [(path/keypath user-id :memories) AFTER-ELEM (path/termval memory)]
              store
              user-id))

           (aor/emit! agent-node "read-user" user-id))))
      (aor/node
       "read-user"
       nil
       (fn [agent-node user-id]
         (let [store (aor/get-store agent-node "$$user-data")
               age (store/pstate-select-one (path/keypath user-id :age) store user-id)
               memories (store/pstate-select [(path/keypath user-id :memories) ALL] store user-id)]
           (aor/result! agent-node {:user-id user-id :age age :memories memories}))))))
```

## Subagents and Recursion

Agents can call other agents within the same module or across modules, including recursively and mutually recursively.

### Calling Agents in the Same Module

#### Java API

```java
public class SubagentModule extends AgentModule {
  @Override
  protected void defineAgents(AgentTopology topology) {
    topology.newAgent("TextProcessor")
            .node("process", null, (AgentNode agentNode, String text) -> {
              String processed = text.toUpperCase();
              agentNode.result(processed);
            });

    topology.newAgent("MainAgent")
            .node("orchestrate", null, (AgentNode agentNode, String input) -> {
              AgentClient processor = agentNode.getAgentClient("TextProcessor");
              String result = processor.invoke(input);
              agentNode.result("Processed: " + result);
            });
  }
}
```

#### Clojure API

```clojure
(aor/defagentmodule SubagentModule
  [topology]
  (-> topology
      (aor/new-agent "TextProcessor")
      (aor/node
       "process"
       nil
       (fn [agent-node text]
         (aor/result! agent-node (str/upper-case text)))))

  (-> topology
      (aor/new-agent "MainAgent")
      (aor/node
       "orchestrate"
       nil
       (fn [agent-node input]
         (let [processor (aor/agent-client agent-node "TextProcessor")
               result (aor/agent-invoke processor input)]
           (aor/result! agent-node (str "Processed: " result)))))))
```

### Recursive Agent Invocation

#### Java API

```java
public class RecursiveModule extends AgentModule {
  @Override
  protected void defineAgents(AgentTopology topology) {
    topology.newAgent("Factorial")
            .node("compute", null, (AgentNode agentNode, Integer n) -> {
              if (n <= 1) {
                agentNode.result(1);
              } else {
                AgentClient self = agentNode.getAgentClient("Factorial");
                Integer subResult = (Integer) self.invoke(n - 1);
                agentNode.result(n * subResult);
              }
            });
  }
}
```

#### Clojure API

```clojure
(aor/defagentmodule RecursiveModule
  [topology]
  (-> topology
      (aor/new-agent "Factorial")
      (aor/node
       "compute"
       nil
       (fn [agent-node n]
         (if (<= n 1)
           (aor/result! agent-node 1)
           (let [self (aor/agent-client agent-node "Factorial")
                 sub-result (aor/agent-invoke self (dec n))]
             (aor/result! agent-node (* n sub-result))))))))
```

### Cross-Module Agent Calls

#### Java API

```java
// Module 1: Greeter agent
public class GreeterModule extends AgentModule {
  @Override
  protected void defineAgents(AgentTopology topology) {
    topology.newAgent("Greeter")
            .node("greet", null, (AgentNode agentNode, String name) -> {
              agentNode.result("Hello, " + name + "!");
            });
  }
}

// Module 2: Mirror agent that calls Greeter
public class MirrorModule extends AgentModule {
  private static final String GREETER_MODULE_NAME = new GreeterModule().getModuleName();

  @Override
  protected void defineAgents(AgentTopology topology) {
    topology.newAgent("MirrorAgent")
            .node("process", null, (AgentNode agentNode, String name) -> {
                AgentClient greeterClient = agentNode.getMirrorAgentClient(GREETER_MODULE_NAME, "Greeter");
                String greeting = (String) greeterClient.invoke(name);
                agentNode.result("Mirror says: " + greeting);
            });
  }
}
```

#### Clojure API

```clojure
(aor/defagentmodule GreeterModule
  [topology]
  (-> topology
      (aor/new-agent "Greeter")
      (aor/node
       "greet"
       nil
       (fn [agent-node name]
         (aor/result! agent-node (str "Hello, " name "!"))))))

(aor/defagentmodule MirrorModule
 [topology]
 (-> topology
     (aor/new-agent "MirrorAgent")
     (aor/node
      "process"
      nil
      (fn [agent-node name]
        (let [greeter-client (aor/mirror-agent-client agent-node (get-module-name GreeterModule) "Greeter")
              greeting (aor/agent-invoke greeter-client name)]
          (aor/result! agent-node (str "Mirror says: " greeting)))))))
```

## Serialization

Agent-o-rama needs to know how to serialize any objects sent to agents as arguments, used as results, or passed between agents in emits. Most commonly used types are already supported, and it's easy to add serializers for your own types. See Rama's serialization documentation [for Java](https://redplanetlabs.com/docs/~/serialization.html) and [for Clojure](https://redplanetlabs.com/docs/~/clj-serialization.html) to learn how.

## Deploying Modules

Once you've defined your agents, you need to deploy them to a Rama cluster.

### Local Development

For local development and testing, you can use `InProcessCluster` (IPC) which runs everything in a single JVM process. You can also start the full UI with IPC, which by default will launch at `http://localhost:1974`.

#### Java API

```java
import com.rpl.rama.test.*;

public class Main {
  public static void main(String[] args) throws Exception {
    try (InProcessCluster ipc = InProcessCluster.create()) {
      try(AutoCloseable ui = UI.start(ipc)) {
        MyAgentModule module = new MyAgentModule();
        ipc.launchModule(module, new LaunchConfig(4, 2));

        String moduleName = module.getModuleName();
        AgentManager manager = AgentManager.create(ipc, moduleName);
        AgentClient agent = manager.getAgentClient("MyAgent");

        Object result = agent.invoke("input data");
        System.out.println("Result: " + result);
      }
    }
  }
}
```

#### Clojure API

```clojure
(require '[com.rpl.rama.test :as rtest])

(with-open [ipc (rtest/create-ipc)
            ui (aor/start-ui ipc)]
  (rtest/launch-module! ipc MyAgentModule {:tasks 4 :threads 2})

  (let [module-name (rama/get-module-name MyAgentModule)
        manager (aor/agent-manager ipc module-name)
        agent (aor/agent-client manager "MyAgent")]

    (let [result (aor/agent-invoke agent "input data")]
      (println "Result:" result))))
```

### Testing with Remote Datasets

When developing a new version of an agent, you may want to test it locally against real data before deploying it to an actual cluster. Agent-o-rama supports this by letting you create "remote datasets" in IPC and then running experiments against that. See [this section](Datasets-evaluators-and-experiments.md#remote-datasets) for the details.

### Deploying to a Cluster

#### Building the JAR

```
# Maven
mvn clean package

# Leiningen
lein uberjar
```

#### Deploying with Rama CLI

```
rama deploy \
  --action launch
  --jar target/my-agents.jar \
  --module com.mycompany.MyAgentModule \
  --tasks 32 \
  --threads 8 \
  --workers 4
```

## Updating Modules

```
rama deploy \
  --action update \
  --jar target/my-agents-v2.jar \
  --module com.mycompany.MyAgentModule
```

### Update Modes

Set the update mode on the agent definition:

#### Java API

```java
topology.newAgent("MyAgent")
        .setUpdateMode(UpdateMode.CONTINUE)  // or RESTART or DROP
        .node("process", null, (AgentNode agentNode, String input) -> {
          agentNode.result("processed: " + input);
        });
```

#### Clojure API

```clojure
(-> (aor/new-agent topology "MyAgent")
    (aor/set-update-mode :continue)  ; or :restart or :drop
    (aor/node
     "process"
     nil
     (fn [agent-node input]
       (aor/result! agent-node (str "processed: " input)))))
```

#### 1. CONTINUE Mode (Default)

In-flight executions continue where they left off with the new agent definition. Use this when you're making incremental changes compatible with in-flight execution state.

#### 2. RESTART Mode

In-flight executions restart from the beginning with the new agent definition. Use this when the new code has significant changes that make continuing problematic.

#### 3. DROP Mode

In-flight executions are terminated and not restarted. Use this when the agent's work is no longer needed or you're deprecating functionality.

### Scaling Modules

```
rama scaleExecutors \
  --module com.mycompany.MyAgentModule \
  --threads 32
  --workers 16
```

## Learn Next

* [Agent clients](Agent-client-API.md)
* [Human-in-the-loop](Human-in-the-loop.md)
* [Streaming](Streaming.md)
* [Tools agent](Tools.md)
