# ADAP: A Data Annotation Platform - Design Documents

This repository contains the product and architectural design documents for the Appen Data Annotation Platform (ADAP). ADAP is an end-to-end SaaS platform designed for the creation, management, and annotation of high-quality data required for training and evaluating artificial intelligence (AI) and machine learning (ML) models.

These documents provide a comprehensive overview of the platform, from the high-level product vision to the detailed, low-level design of its core components.

## Document Overview

For a complete understanding of the platform, it is recommended to read the documents in the following order:

### 1. [Product Specification (`product_spec.md`)](./product_spec.md)

**Purpose:** This document answers the **"what"** and **"why"** of the ADAP platform. It is the best starting point for anyone new to the project.

**Contents:**
- **Core Functionalities:** Details the platform's capabilities, including its multi-modal annotation toolset, ML-assisted labeling features, and integrated quality control mechanisms.
- **Common Use Cases:** Describes how the platform is used for training Generative AI (e.g., RLHF), search relevance engines, computer vision models, and more.
- **Business Context:** Provides an executive summary and historical context for the platform.

---

### 2. [High-Level System Architecture (`architecture.md`)](./architecture.md)

**Purpose:** This document provides the **10,000-foot architectural blueprint** of the system. It's essential for understanding how the platform is structured at a macro level.

**Contents:**
- **Conceptual Data Model:** Defines all core data entities (e.g., `Jobs`, `Units`, `Judgments`, `Workflows`) and their relationships via an ERD.
- **Subsystem Breakdown:** Decomposes the platform into its major conceptual subsystems, such as Client Management, Job Lifecycle, and Task & Execution, outlining the responsibilities and data ownership of each.
- **High-Level Interactions:** Uses sequence diagrams to illustrate how these subsystems collaborate to fulfill key user stories, like launching a job or completing a task.

---

### 3. [Deep Dive: Task & Execution Subsystem (`task_execution_subsystem.md`)](./task_execution_subsystem.md)

**Purpose:** This document is a **detailed, low-level design** for the operational heart of the platform—the engine that manages and executes all annotation work. It is critical reading for engineers working on the core system.

**Contents:**
- **Detailed Component Design:** Provides an in-depth look at the internal services, owned data models, and key APIs.
- **Scalability & Performance:** Addresses critical engineering challenges, including database sharding, handling "hotspot" jobs, scaling reads with replicas, and using asynchronous processing to maintain responsiveness.
- **Distributed System Resilience:** Details patterns for handling failures, such as the Transactional Outbox and Inbox patterns to ensure data integrity and exactly-once processing in an event-driven architecture.


