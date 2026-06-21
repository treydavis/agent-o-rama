# Datasets, Evaluators, and Experiments

## Overview

Agent-o-rama provides a comprehensive evaluation framework for systematic LLM agent testing. The platform addresses key development challenges including subjective assessment, edge case discovery, regression detection, and performance measurement.

## Key Components

### Datasets

Structured collections of test examples with inputs, optional reference outputs, and tags. Datasets support:
- Manual UI creation with JSON schema validation
- Programmatic additions via Java/Clojure APIs
- Bulk import from JSONL files
- Integration with production runs through actions
- Immutable snapshots for reproducible experiments

### Evaluators

Functions measuring agent performance across three types:

**Regular Evaluators** - Assess individual runs using input, reference output, and run output to return score maps with numeric, boolean, or string values.

**Comparative Evaluators** - Compare multiple outputs side-by-side, useful for subjective tasks where ranking is easier than independent scoring.

**Summary Evaluators** - Calculate metrics across entire datasets, such as precision, recall, and F1 scores.

Built-in evaluators include LLM Judge, Conciseness checker, and F1 Score calculator. Custom evaluators can access declared agent objects through a fetcher interface.

### Experiments

Controlled tests running agents against datasets with evaluators. Regular experiments use one target with regular/summary evaluators. Comparative experiments evaluate multiple targets using only comparative evaluators.

**Input Configuration** - JSON Path templates map example inputs to function arguments, supporting nested substitution and literal escaping with "$$".

**Results Display** - Aggregate metrics show average/P99 latencies, token usage, and evaluator aggregations with trend charts across experiments.

## Remote Datasets

Cross-cluster or cross-module dataset proxies enable testing agents before production deployment without local dataset duplication.
