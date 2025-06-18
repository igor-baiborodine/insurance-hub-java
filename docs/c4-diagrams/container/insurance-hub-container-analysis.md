### Explanation of Containers and Flows

This document explains the containers and data flows shown in the C4 Container Diagram for the Insurance Hub system.

#### Containers (Internal Systems)

*   **Web Vue App**: The frontend single-page application (SPA) built with Vue.js. It provides the user interface for the Insurance Agent to manage and sell insurance policies.
*   **Agent Portal Gateway**: A Micronaut-based API Gateway that acts as the single entry point for all requests from the frontend. It routes traffic to the appropriate downstream microservices.
*   **Policy Service**: The core service responsible for the business logic of creating and managing insurance offers and policies. It acts as a central orchestrator for many business processes.
*   **Product Service**: Manages the complete insurance product catalog. It provides information about available products, their coverage options, and the questions required to generate an offer.
*   **Pricing Service**: Calculates the price for an insurance product based on a set of rules (tariffs) and the data provided in an offer.
*   **Policy Search Service**: Provides a powerful search capability over all policies. It maintains a denormalized read model of policy data to enable fast and complex queries.
*   **Payment Service**: Handles all financial aspects, including managing policyholder accounts, processing payments, and reconciling bank statements.
*   **Documents Service**: Responsible for generating, storing, and providing access to policy-related documents, such as the final policy PDF.
*   **Databases**: Each core service that owns data has its own dedicated database, following the database-per-service pattern.
    *   **Policy Database (RDBMS)**: Stores offers and policies for the `Policy Service`.
    *   **Product Database (MongoDB)**: Stores the product catalog for the `Product Service`.
    *   **Payment Database (PostgreSQL)**: Stores accounts and transactions for the `Payment Service`.
    *   **Search Index (Elasticsearch)**: Stores the denormalized policy data for the `Policy Search Service`.

#### External Systems & Services

*   **Insurance Agent (Person)**: The end-user of the system who interacts with the Web Vue App.
*   **Apache Kafka**: An event streaming platform used as a message broker for asynchronous communication, decoupling the services.
*   **JSReport Service**: An external service used by the `Documents Service` to generate PDF documents from templates and data.
*   **File System / Storage**: Several services interact with storage systems for specific files:
    *   **Tariff Rules Storage**: Stores MVEL rule scripts used by the `Pricing Service`.
    *   **Bank Statements Storage**: Stores CSV bank statements that are imported by the `Payment Service`.
    *   **Document Storage**: Stores the generated PDF documents managed by the `Documents Service`.

#### Flows

1.  **User Interaction Flow**:
    *   The `Insurance Agent` uses the `Web Vue App`.
    *   The `Web Vue App` communicates with the backend by sending requests to the `Agent Portal Gateway`.

2.  **Synchronous API Flow (Policy Creation)**:
    *   The `Gateway` routes a request to create an offer to the `Policy Service`.
    *   The `Policy Service` calls the `Product Service` to get details about the selected insurance product.
    *   The `Policy Service` then calls the `Pricing Service` with the offer data to get a price calculation.
    *   The `Pricing Service` reads tariff rules from its `Tariff Rules Storage` to perform the calculation.
    *   Once the offer is converted to a policy, the `Policy Service` saves the final policy to its `Policy Database`.

3.  **Asynchronous Event Flow (Policy Finalized)**:
    *   When a policy is created and saved, the `Policy Service` publishes a `PolicyCreated` event to an `Apache Kafka` topic.
    *   This event is consumed independently by several other services:
        *   The `Policy Search Service` consumes the event, transforms the data into a read model, and saves it to its `Search Index`.
        *   The `Payment Service` consumes the event and creates a new policy account in its `Payment Database`.
        *   The `Documents Service` consumes the event, orchestrates the generation of a policy PDF by calling the `JSReport Service`, and saves the result to `Document Storage`.

4.  **Other Flows**:
    *   **Search Flow**: A search query from the `Web Vue App` goes through the `Gateway` to the `Policy Search Service`, which queries its `Search Index` and returns the results.
    *   **Payment Flow**: The `Payment Service` periodically reads CSV bank statements from `Bank Statements Storage` to reconcile payments.
    *   **Document Download**: The user can request to download a document via the `Web Vue App`. The request goes through the `Gateway` to the `Documents Service`, which retrieves the file from `Document Storage`.