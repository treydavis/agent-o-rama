# Human-in-the-loop

## Overview

Human-in-the-loop is a critical pattern for AI agents that need human judgment, approval, or clarification during execution. Agent-o-rama provides built-in support for agents to pause execution and request input from humans, then resume once the input is provided.

## Table of Contents

1. Requesting Human Input in Agents
2. Handling Human Input from Clients
3. Viewing and Providing Input in the UI

## Requesting Human Input in Agents

Inside an agent node, you can request human input at any point using `getHumanInput` (Java) or `get-human-input` (Clojure). This method pauses the agent execution until a human provides a response. Since agent nodes run on virtual threads, this doesn't actually block any actual threads.

The method takes a prompt string and returns the human's response as a string. Your agent code can then use that response to make decisions or continue processing.

### Java API

```java
import com.rpl.agentorama.*;

public class HumanInputAgentModule extends AgentModule {
  @Override
  protected void defineAgents(AgentTopology topology) {
    topology.newAgent("ApprovalAgent")
            .node("process-request", null, (AgentNode agentNode, String itemName, Double cost) -> {
              // Request human approval
              String response = agentNode.getHumanInput(
                String.format("Approve purchase of %s for $%.2f? (yes/no): ", itemName, cost)
              );

              if ("yes".equalsIgnoreCase(response)) {
                agentNode.result(Map.of("status", "approved", "item", itemName));
              } else {
                agentNode.result(Map.of("status", "rejected", "item", itemName));
              }
            });
  }
}
```

### Clojure API

```clojure
(require '[com.rpl.agent-o-rama :as aor])

(aor/defagentmodule HumanInputAgentModule
  [topology]
  (-> (aor/new-agent topology "ApprovalAgent")
      (aor/node
       "process-request"
       nil
       (fn [agent-node item cost]
         ;; Request human approval
         (let [response (aor/get-human-input
                         agent-node
                         (format "Approve purchase of %s for $%.2f? (yes/no): " item cost))]
           (if (= "yes" (clojure.string/lower-case response))
             (aor/result! agent-node {:status "approved" :item item})
             (aor/result! agent-node {:status "rejected" :item item})))))))
```

### Multiple Human Input Requests

An agent can request human input multiple times during execution. Each call to `getHumanInput` / `get-human-input` will pause execution until that specific request is answered. Additionally, if multiple nodes are running in parallel there can be multiple pending human input requests.

```java
// Java - Multiple requests
String name = agentNode.getHumanInput("What is your name? ");
String email = agentNode.getHumanInput("What is your email? ");
String confirmation = agentNode.getHumanInput(
  String.format("Confirm: Name=%s, Email=%s. Is this correct? (yes/no): ", name, email)
);
```

```clojure
;; Clojure - Multiple requests
(let [name (aor/get-human-input agent-node "What is your name? ")
      email (aor/get-human-input agent-node "What is your email? ")
      confirmation (aor/get-human-input
                    agent-node
                    (format "Confirm: Name=%s, Email=%s. Is this correct? (yes/no): " name email))]
  ...)
```

## Handling Human Input from Clients

When an agent requests human input, the execution pauses. From the client side, you have two main approaches for handling these requests:

1. **Step-by-step with `nextStep`**: Get the next execution step, check if it's a human input request, provide input, and continue
2. **Batch processing with `pendingHumanInputs`**: Get all pending requests at once and provide responses

### Using nextStep

The `nextStep` method returns the next step in the agent execution, which is either:

- A **human input request** (check with `instanceof HumanInputRequest` in Java or `human-input-request?` in Clojure)
- The **final result** (an `AgentComplete` object in Java, or a map with `:result` key in Clojure)

This approach is ideal for interactive command-line tools or UIs where you handle requests one at a time. If there are multiple human input requests pending, `nextStep` will return whichever one was requested first.

#### Java API

```java
import com.rpl.agentorama.*;
import java.util.Scanner;

public class InteractiveClient {
  public static void main(String[] args) throws Exception {
    try (InProcessCluster ipc = InProcessCluster.create();
         Scanner scanner = new Scanner(System.in)) {

      HumanInputAgentModule module = new HumanInputAgentModule();
      ipc.launchModule(module, new LaunchConfig(4, 2));

      AgentManager manager = AgentManager.create(ipc, module.getModuleName());
      AgentClient agent = manager.getAgentClient("ApprovalAgent");

      AgentInvoke invoke = agent.initiate(Map.of("item", "laptop", "cost", 1200.0));

      AgentStep step = agent.nextStep(invoke);
      while (step instanceof HumanInputRequest) {
        HumanInputRequest request = (HumanInputRequest) step;

        System.out.println(request.getPrompt());
        System.out.print(">> ");
        String response = scanner.nextLine();

        agent.provideHumanInput(request, response);

        step = agent.nextStep(invoke);
      }

      System.out.println("Final result: " + ((AgentComplete) step).getResult());
    }
  }
}
```

#### Clojure API

```clojure
(require '[com.rpl.agent-o-rama :as aor]
         '[com.rpl.rama.test :as rtest])

(with-open [ipc (rtest/create-ipc)]
  (rtest/launch-module! ipc HumanInputAgentModule {:tasks 4 :threads 2})
  (let [manager (aor/agent-manager ipc (rama/get-module-name HumanInputAgentModule))
        agent (aor/agent-client manager "ApprovalAgent")]

    (let [invoke (aor/agent-initiate agent {:item "laptop" :cost 1200.0})]

      (loop [step (aor/agent-next-step agent invoke)]
        (if (aor/human-input-request? step)
          (do
            (println (:prompt step))
            (print ">> ")
            (flush)
            (let [response (read-line)]
              (aor/provide-human-input agent step response)
              (recur (aor/agent-next-step agent invoke))))
          (let [result (:result step)]
            (println "Final result:" result)))))))
```

### Using pendingHumanInputs

The `pendingHumanInputs` method returns all human input requests that are currently waiting for responses. This is useful when:

- You want to see all pending requests at once
- You're building a UI that displays multiple pending requests
- You want to batch process multiple requests

Each request object has:

- **Java**: `.getPrompt()` for the prompt text, `.getNode()` for the node name
- **Clojure**: `:prompt` for the prompt text, `:node` for the node name

#### Java API

```java
AgentInvoke invoke = agent.initiate(Map.of("item", "laptop", "cost", 1200.0));

Thread.sleep(100);

List<HumanInputRequest> pending = agent.pendingHumanInputs(invoke);
System.out.println("Pending requests: " + pending.size());

for (HumanInputRequest request : pending) {
  System.out.println("Node: " + request.getNode());
  System.out.println("Prompt: " + request.getPrompt());

  agent.provideHumanInput(request, "yes");
}

if (agent.isAgentInvokeComplete(invoke)) {
  Object result = agent.result(invoke);
  System.out.println("Result: " + result);
}
```

#### Clojure API

```clojure
(let [agent (aor/agent-client manager "ApprovalAgent")
      invoke (aor/agent-initiate agent {:item "laptop" :cost 1200.0})]

  (Thread/sleep 100)

  (let [pending (aor/pending-human-inputs agent invoke)]
    (println "Pending requests:" (count pending))

    (doseq [request pending]
      (println "Node:" (:node request))
      (println "Prompt:" (:prompt request))

      (aor/provide-human-input agent request "yes")))

  (when (aor/agent-invoke-complete? agent invoke)
    (let [result (aor/agent-result agent invoke)]
      (println "Result:" result))))
```

## Viewing and Providing Input in the UI

The Agent-o-rama web UI provides a visual interface for viewing and responding to human input requests. In an agent invoke trace, nodes with pending human input have a human icon on them, and clicking on the node will show the request below with an input box to provide a response.
