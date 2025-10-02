# Design an Online Bookstore

This collection of design documents are created to be a comprehensive learning resource that covers a wide spectrum of technical elements beyond just software system design. Using the real-world context of a modern online bookstore, they offer a practical, end-to-end exploration of software architecture, data engineering, and applied machine learning algorithms.

## Document Structure
Each design document in this collection follows a consistent, top-down structural pattern to ensure a clear and logical flow. Topics are typically introduced with a detailed set of business goals and functional/nonfunctional requirements. This is followed by a high-level architecture that outlines the main components and their interactions. Finally, each document provides deep dives into specific technical details, including data models, algorithms, sequence diagrams, and code examples.

## Summary of Documents

### 1\. [`bookstore_highlevel.md`](./bookstore_highlevel.md)

**Overview:** This document provides a high-level view of the overall website architecture and design principles.

  - It details the system's evolution from an initial 3-tier design to a modern, cloud-native architecture.
  - It outlines a microservices-based, event-driven system with key components like an API Gateway, a Single Page Application (SPA) frontend, and decoupled backend services with a database-per-service pattern.

### 2\. [`bookstore_checkout.md`](./bookstore_checkout.md)

**Overview:** This document details the architecture for the customer checkout and order management process.

  - It explains the **Saga Orchestration Pattern** used to manage the checkout workflow across multiple microservices (Orders, Inventory, Payments).
  - It covers how to ensure data consistency through state persistence and handle failures using compensating transactions.
  - It also refines the design into an **asynchronous flow** to provide a fast and responsive user experience during checkout.

### 3\. [`bookstore_payment.md`](./bookstore_payment.md)

**Overview:** This document provides a summary of the payment processing workflow and its integration with third-party gateways.

  - It explains the critical role of **tokenization** (using `Stripe.js`) to drastically reduce PCI DSS compliance scope by ensuring sensitive card data never touches the bookstore's servers.
  - It discusses the use of `PaymentMethod` and `Customer` objects to securely save customer card information for repeat purchases.

### 4\. [`bookstore_search.md`](./bookstore_search.md)

**Overview:** This document outlines the design of the bookstore's comprehensive search functionality using Elasticsearch.

  - It details essential keyword-based features like fuzzy matching, autocomplete, and faceted navigation.
  - It introduces advanced capabilities like **Semantic Search** using text embeddings and the `dense_vector` type.
  - It covers data modeling solutions, such as using **`nested` documents** to handle multiple book formats, and the design of a two-phase **Learning to Rank (LTR)** architecture.

### 5\. [`bookstore_searchranking.md`](./bookstore_searchranking.md)

**Overview:** This document provides a deep dive into the Learning to Rank (LTR) system, from business goals to the underlying ML theory.

  - It defines the business goals, user signals (book, user, interaction), and the formulation of the ranking problem using graded relevance labels.
  - It details the full, five-stage **MLOps lifecycle** for the LTR model, including data collection, feature generation, model training (XGBoost/LightGBM), A/B testing, and the monitoring feedback loop.
  - It explains the core algorithms, including how Gradient Boosted Trees are trained and how the **LambdaMART** objective function optimizes for ranking metrics like **nDCG**.

### 6\. [`bookstore_recommendations.md`](./bookstore_recommendations.md)

**Overview:** This document summarizes the design of the hybrid recommendation engine, which pre-computes recommendations offline for low-latency online serving.

  - It details the design of three core model types:
      - **Collaborative Filtering:** Using **Matrix Factorization (ALS)** to find users with similar tastes based on broad interaction data.
      - **Content-Based Filtering:** Using modern **text embeddings** to recommend thematically similar books.
      - **Co-purchase Model:** Using transaction data to power "Frequently Bought Together" features.
  - It also explains how text embedding models are trained or fine-tuned and how structured book data is transformed into natural language for model input.

### 7\. [`bookstore_analytics.md`](./bookstore_analytics.md)

**Overview:** This document details the architecture for the analytics platform that powers dashboards and ad-hoc analysis.

  - It designs a **Modern Data Stack** with a dual-path (Lambda) architecture:
      - A **Batch Path** using a Data Lake (S3), a Data Warehouse (e.g., Snowflake), and **dbt** for accurate, historical reporting.
      - A **Streaming Path** using a real-time analytics database like **Apache Druid** for live dashboards.
  - It explains how these paths are unified in a BI tool and discusses the evolution to a simpler **Kappa Architecture**.
  - It illustrates how **ML workloads** (for LTR and recommendations) are integrated, using the data platform for training and deploying models to online services.

-----

Each document can be referenced for in-depth details on its respective topic. This overview serves as a starting point for understanding the bookstore system's design and documentation structure.
