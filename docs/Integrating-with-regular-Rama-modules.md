# Integrating with Regular Rama Modules

Agent-o-rama can be integrated with regular Rama modules, allowing you to use agents alongside full Rama features like depots, stream processing, and query topologies. This page covers two main integration patterns:

1. **Accessing Rama objects from within agent nodes** - Using depots, PStates, and query topologies from agent node functions
2. **Adding agents to regular Rama modules** - Defining agents within a standard `RamaModule` implementation

## Accessing Rama objects from agent nodes

Within agent node functions, you can access Rama depots, PStates (via stores), and query topologies using methods on the `AgentNode` interface. This allows agents to interact with the broader Rama infrastructure.

### Accessing depots

Depots are Rama's append-only logs. You can access both local depots (from the same module) and mirror depots (from other modules).

**Local depot:**

```java
.node("process", null, (AgentNode agentNode, String input) -> {
    Depot depot = agentNode.getDepot("*events-depot");
    depot.append(input);
    // ...
})
```

**Mirror depot (from another module):**

```java
.node("process", null, (AgentNode agentNode, String input) -> {
    Depot mirrorDepot = agentNode.getMirrorDepot("com.mycompany.OtherModule", "*analytics-depot");
    mirrorDepot.append(input);
    // ...
})
```

### Accessing PStates via stores

PStates are accessed through the store interface. Agent-o-rama provides three types of stores that wrap PStates:

1. **KeyValueStore** - Simple key-value storage
2. **DocumentStore** - Schema-flexible nested data storage
3. **PStateStore** - Direct PState access

**Local store:**

```java
// Declare the store in the topology
topology.declareKeyValueStore("$$users", String.class, UserData.class);

// Use it in a node
topology.newAgent("myAgent")
        .node("process", null, (AgentNode agentNode, String userId) -> {
            KeyValueStore<String, UserData> store = agentNode.getStore("$$users");

            UserData user = store.get(userId);
            if (user == null) {
                user = fetchUserData(userId);
                store.put(userId, user);
            }

            agentNode.result(user);
        });
```

**Mirror store (read-only from another module):**

```java
.node("process", null, (AgentNode agentNode, String userId) -> {
    KeyValueStore<String, UserData> mirrorStore =
        agentNode.getMirrorStore("com.mycompany.UserModule", "$$user-db");

    UserData user = mirrorStore.get(userId);
    // ...
})
```

**PStateStore for direct PState access:**

PState stores can be fetched for PStates declared as part of the agent topology or for PStates declared outside the agent topology. PStates are read-only if declared in the module outside the agent topology.

```java
topology.newAgent("myAgent")
        .node("process", null, (AgentNode agentNode, String id) -> {
            PStateStore store = agentNode.getStore("$$myPState");
            Long value = store.selectOne(Path.key(id));
            // ...
        });
```

### Accessing query topologies

Query topologies allow you to invoke queries from within agent nodes.

**Local query topology:**

```java
.node("process", null, (AgentNode agentNode, String query) -> {
    QueryTopologyClient<Map> queryClient = agentNode.getQueryTopologyClient("search-query");
    Map results = queryClient.invoke(query, 100);
    // ...
})
```

**Mirror query topology (from another module):**

```java
.node("process", null, (AgentNode agentNode, String userId) -> {
    QueryTopologyClient<UserInfo> queryClient =
        agentNode.getMirrorQueryTopologyClient("com.mycompany.UserModule", "user-lookup");
    UserInfo user = queryClient.invoke(userId);
    // ...
})
```

## Adding agents to regular Rama modules

You can add agents to any regular Rama module by creating an `AgentTopology` manually and calling `define()` when done. This allows you to use agents alongside full Rama features like stream processing, custom depots, and other topologies.

### Basic pattern

Instead of extending `AgentModule`, implement `RamaModule` directly and create the agent topology manually:

```java
public class MyRamaModule implements RamaModule {
    @Override
    public void define(Setup setup, Topologies topologies) {
        // Declare regular Rama depots, topologies, PStates, etc.
        setup.declareDepot("*events-depot", Depot.random());

        // Create agent topology manually
        AgentTopology agentTopology = AgentTopology.create(setup, topologies);

        // Define agents
        agentTopology.declareKeyValueStore("$$cache", String.class, String.class);
        agentTopology.newAgent("myAgent")
            .node("process", null, (AgentNode agentNode, String input) -> {
                KeyValueStore<String, String> store = agentNode.getStore("$$cache");
                store.put("last-input", input);
                agentNode.result("Processed: " + input);
            });

        // Must call define() to finalize agent definitions
        agentTopology.define();
    }
}
```
