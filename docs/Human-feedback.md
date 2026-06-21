# Human Feedback

Human feedback is essential for evaluating and improving LLM agents. While automated evaluators can measure many aspects of agent performance, some assessments require human judgment—especially for subjective qualities like helpfulness, tone, or appropriateness. Agent-o-rama provides a comprehensive system for collecting, managing, and analyzing human feedback on agent executions.

This page explains Agent-o-rama's human feedback features: **Human Metrics** define the criteria by which humans evaluate agent performance. **Human Feedback Queues** organize collections of agent runs for systematic human review. You can provide structured feedback on individual agent executions and manage that feedback through the UI. The system also tracks human feedback metrics over time in telemetry.

## Table of Contents

- [Human Metrics](#human-metrics)
- [Human Feedback Queues](#human-feedback-queues)
- [Providing Feedback](#providing-feedback)
- [Managing Feedback](#managing-feedback)
- [Adding to Feedback Queues from Traces](#adding-to-feedback-queues-from-traces)
- [Human Feedback Telemetry](#human-feedback-telemetry)

## Human Metrics

Human metrics define the criteria by which humans evaluate agent performance. Each metric specifies what aspect of the agent's output is being assessed and how that assessment should be structured.

There are two types of human metrics: **Categorical Metrics** evaluate outputs using predefined categories (e.g., "helpful" vs "not helpful", "factual" vs "opinion" vs "nonsense"). **Numeric Metrics** evaluate outputs using numeric scores within a specified range (e.g., 1-10 for quality).

### Creating Human Metrics

Human metrics are created through the "Human metrics" page in the module side panel. You can add new metrics or delete existing ones. They can also be created via the API.

When adding a metric, you first select whether it's categorical or numeric:

#### Categorical Metrics

For categorical metrics, you specify one or more category names (non-empty strings). At least one category must be provided. These categories become the options available when providing feedback.

**Java API:**

```java
AgentManager manager = AgentManager.create(cluster, moduleName);

manager.createCategoricalHumanMetric(
    "helpfulness",
    "description of metric",
    Set.of("very helpful", "somewhat helpful", "not helpful")
);
```

**Clojure API:**

```clojure
(def manager (aor/agent-manager cluster module-name))

(aor/create-categorical-human-metric! manager
  "helpfulness"
  "description of metric"
  #{"very helpful" "somewhat helpful" "not helpful"})
```

#### Numeric Metrics

For numeric metrics, you specify a minimum and maximum value. Both must be integers, and the maximum must be greater than the minimum. These bounds define the valid range for feedback scores.

**Java API:**

```java
manager.createNumericHumanMetric(
    "quality-score",
    "description of metric",
    1,  // min
    10  // max
);
```

**Clojure API:**

```clojure
(def metric-id
  (aor/create-numeric-human-metric! manager
    "quality-score"
    "description of metric"
    1   ; min
    10)) ; max
```

### Managing Human Metrics

The Human metrics page displays all defined metrics in a paginated list. You can search for metrics by name using the search box at the top.

To delete a metric, click the delete button next to it. Note that deleting a metric will remove it from any human feedback queues that reference it, but existing feedback using that metric will remain.

**Java API:**

```java
manager.removeHumanMetric(metricId);
```

**Clojure API:**

```clojure
(aor/remove-human-metric! manager metric-id)
```

## Human Feedback Queues

Human feedback queues organize collections of agent runs for systematic human review. Each queue defines which metrics should be evaluated and which are required versus optional. Queues enable structured workflows for reviewing agent performance, whether for quality assurance, training data collection, or continuous improvement. Agent runs in queues can either be root agent runs or individual node runs.

### Creating Human Feedback Queues

Human feedback queues are created through the "Human feedback queues" page in the module side panel.

When creating a queue, you specify a **name** (a descriptive name for the queue), an optional **description** (text describing the purpose or criteria for this queue), and **rubrics** (a list of human metrics to evaluate). Each rubric includes the **metric** to use (selected from a dropdown with search) and a **required** checkbox indicating whether this metric must be provided when reviewing items.

### Managing Human Feedback Queues

The Human feedback queues page displays all queues in a paginated list. You can search for queues by name using the search box at the top.

To delete a queue, click the delete button next to it. This will also remove all items in the queue.

### Viewing a Human Feedback Queue

Clicking on a human feedback queue opens its detail page, which shows **queue information** (name, description, and the list of rubrics configured for this queue – rubrics that reference deleted metrics are automatically filtered out) and a **paginated list of queue items** (agent or node runs waiting for review).

#### Editing a Queue

You can edit a queue's description and rubrics by clicking the "Edit" button. This brings up a form similar to the creation form, allowing you to modify the description and add, remove, or change the required status of rubrics.

## Providing Feedback

### Reviewing Queue Items

To review an item in a human feedback queue, click on it from the queue items list. This opens the evaluation form.

The evaluation form displays the **input** that was provided to the agent or node, the **output** produced by the agent or node, and a **feedback form** to provide structured feedback. There's also a link to the full trace for the run. For each metric in the queue's rubrics, the form provides a component to collect feedback. The form also includes a required **reviewer name** field and an optional **comment** text area for additional notes.

The form includes navigation controls: **Previous/Next buttons** to navigate to adjacent items in the queue, and a **Dismiss button** to remove the item from the queue without providing feedback.

When you submit feedback, the system records the feedback on the trace and automatically opens the evaluation form for the next item in the queue.

This workflow enables efficient batch review of queue items, automatically advancing through the queue as you provide feedback.

## Managing Feedback

### Adding Feedback Manually

You can add human feedback directly to any agent execution or node run through the feedback panel in the trace view. The feedback panel for agent roots and nodes includes an "Add feedback" button.

Clicking "Add feedback" opens a form that allows you to dynamically select any number of human metrics using a dropdown selector with search. As you select each metric, the appropriate form component appears (dropdown for categorical, text input for numeric).

### Editing Feedback

All human feedback can be edited. In the feedback panel, click the edit button next to any human feedback entry. This opens the same form used for adding feedback, pre-populated with the existing values.

You can modify any aspect of the feedback: change metric values, update the comment, or even change which metrics are included.

### Deleting Feedback

You can delete human feedback by clicking the delete button in the feedback panel. This permanently removes the feedback from the system.

## Adding to Feedback Queues from Traces

Just like the "Add to dataset" buttons for agents and nodes, there are also "Add to human feedback queue" buttons in the trace view.

Clicking this button brings up a dropdown selector with search for choosing which human feedback queue to add the run to. This makes it easy to collect interesting examples from production runs for later review.

### Automatic Queue Addition via Actions

You can also automatically add runs to human feedback queues using actions.

This enables continuous collection of runs for human review, ensuring important examples don't get missed.

## Human Feedback Telemetry

The telemetry section for agents is expanded to include charts for each human metric defined in the system. These charts show time-series data of human feedback scores, enabling you to track how human-assessed quality metrics change over time.

Human feedback telemetry complements automated evaluator telemetry, giving you visibility into both objective and subjective aspects of agent performance. Together, they provide a complete picture of how your agents are performing in production.
