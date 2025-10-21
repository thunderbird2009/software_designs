# Deep Dive: Workflow Orchestration Engine

This document provides a detailed architectural overview of the Workflow Orchestration Engine, which is a core component of the Task & Execution Subsystem.

- [Deep Dive: Workflow Orchestration Engine](#deep-dive-workflow-orchestration-engine)
  - [**1. Overview and Core Principles**](#1-overview-and-core-principles)
    - [**1.1. Purpose**](#11-purpose)
    - [**1.2. Core Principle: Unified Workflow Model**](#12-core-principle-unified-workflow-model)
    - [**1.3. Core Principle: Immutable Unit Chaining**](#13-core-principle-immutable-unit-chaining)
  - [**2. Engine Architecture: Specialized Orchestrator**](#2-engine-architecture-specialized-orchestrator)
    - [**2.1. Comparison: General-Purpose vs. Specialized**](#21-comparison-general-purpose-vs-specialized)
  - [**3. Execution Modes**](#3-execution-modes)
    - [**3.1. Streaming Mode**](#31-streaming-mode)
    - [**3.2. Batch (Blocking) Mode**](#32-batch-blocking-mode)
  - [**4. Data Models for Workflow**](#4-data-models-for-workflow)
    - [**4.1. Static Definitions**](#41-static-definitions)
    - [**4.2. Runtime State**](#42-runtime-state)
  - [**5. Detailed Logic and Scenarios**](#5-detailed-logic-and-scenarios)
    - [**5.1. Scenario: Unit Routing (Streaming)**](#51-scenario-unit-routing-streaming)
    - [**5.2. Scenario: Batch Synchronization**](#52-scenario-batch-synchronization)

---

## **1. Overview and Core Principles**

### **1.1. Purpose**
The Workflow Engine is the heart of the Task & Execution Subsystem. It is responsible for orchestrating the end-to-end lifecycle of work on the platform by interpreting user-defined workflows, routing units of work between different steps, and managing the state of the entire process.

### **1.2. Core Principle: Unified Workflow Model**
To eliminate complexity and dual execution paths, we have adopted a unifying principle: **every task run on the platform is a workflow.**

*   **Concept:** A simple "job" is treated as a single-step workflow. This removes the distinction between "jobs" and "workflows" at the execution level.
*   **Implication:** The `WorkflowEngine` is the sole mechanism for task execution. The concept of a `JobDefinition` is simplified to that of a reusable **"Task Template"** which is instantiated by a `WorkflowStep`.
*   **Benefit:** This provides a single, consistent model for all work, simplifies the execution engine, and allows users to seamlessly scale a simple task into a multi-step process without migration.

### **1.3. Core Principle: Immutable Unit Chaining**
To simplify state management and create a clear data pipeline, we will instantiate a new `Unit` for each step in a workflow.

*   **Concept:** When a `Unit` is completed at `Step A`, the `WorkflowEngine` creates a *new* `Unit` for `Step B`. This forms a chain of units linked by a common identifier (e.g., `root_unit_id` or `source_data_id`).
*   **Implication:** This model eliminates the need for a complex, centralized `UnitWorkflowState` entity. Each `Unit` has a simple, self-contained lifecycle. The output of one unit (its `aggregated_result`) becomes part of the input for the next unit in the chain.
*   **Benefit:** Greatly simplifies state management, promotes data isolation between steps, and creates a clear, traceable "input/output" data flow that is easy to reason about and debug.

---

## **2. Engine Architecture: Specialized Orchestrator**
Our `Workflow Engine` is a highly specialized orchestrator, not a general-purpose one like Apache Airflow or AWS Step Functions. The difference lies in the specific requirements of a human-in-the-loop data annotation platform.

### **2.1. Comparison: General-Purpose vs. Specialized**

| Feature | General-Purpose Engine | ADAP's Specialized Orchestrator |
| :--- | :--- | :--- |
| **Focus** | Automating machine tasks | Orchestrating human annotation |
| **Key Trigger** | Time / System Event | Human task completion |
| **Routing Logic** | Success/Failure | Annotation content & quality metrics |
| **Primary Actor** | System / Service | Human Contributor / Reviewer |
| **State** | `succeeded`, `failed` | `judgments_collected`, `pending_review` |

This specialization is why building a dedicated `Workflow Engine` is necessary. A general-purpose engine is not equipped to handle the unique, human-centric, and quality-driven logic that is the cornerstone of a data annotation platform.

---

## **3. Execution Modes**
The engine must support two distinct execution modes for routing units between steps, providing flexibility for different types of tasks.

### **3.1. Streaming Mode**
*   **Behavior:** As soon as a single `Unit` is finalized at `Step A`, the `Workflow Engine` immediately creates the corresponding `Unit` for `Step B` and makes it available for work. This is the default behavior.
*   **Use Case:** Standard annotation pipelines where tasks are independent. This maximizes throughput and minimizes the time a contributor has to wait for work.

### **3.2. Batch (Blocking) Mode**
*   **Behavior:** The `Workflow Engine` will wait until **all** `Units` associated with a specific batch have been successfully finalized at `Step A`. Only then will it create the `Units` for `Step B`.
*   **Use Case:**
    *   **Reporting/Aggregation Steps:** A step that needs to calculate statistics over the entire dataset.
    *   **Model Training Steps:** A step that uses the full set of annotated data to train a model.
    *   **Bulk Data Operations:** A step that performs a bulk export or validation on the entire completed batch.

---

## **4. Data Models for Workflow**

### **4.1. Static Definitions**
These entities collectively form the static blueprint for any workflow.

*   **`WorkflowDefinition`:** The top-level container for the workflow.
*   **`WorkflowStep`:** A single stage in the workflow, which references a `JobDefinition` (Task Template) and contains step-specific configuration.
*   **`WorkflowRoute`:** Defines a directed path from a source step to a target step.
    *   **`execution_mode` (ENUM: `'STREAMING'`, `'BATCH'`):** A critical field on this entity that specifies whether the transition is streaming or blocking.
*   **`WorkflowRule`:** Encapsulates the specific conditional logic (e.g., based on annotation data) that must be met for a `WorkflowRoute` to be taken.

### **4.2. Runtime State**
This entity is required to manage the Batch execution mode.

*   **`BatchState`:**
    *   `batch_id` (PK)
    *   `workflow_id`
    *   `source_step_id`
    *   `total_units_expected` (The total number of units in the batch)
    *   `units_completed` (An atomically updated counter)
    *   `status` (`COLLECTING`, `COMPLETED`)

---

## **5. Detailed Logic and Scenarios**

### **5.1. Scenario: Unit Routing (Streaming)**
*(A sequence diagram will be added here showing a single unit being finalized at Step A, and the engine immediately creating a new unit for Step B.)*

### **5.2. Scenario: Batch Synchronization**
*(A sequence diagram will be added here showing multiple units arriving at a batch-mode route. The engine updates the `BatchState` counter for each one. Only when the counter is full does the engine trigger the creation of units for the next step.)*
