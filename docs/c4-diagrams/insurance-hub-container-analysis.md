### Explanation of Containers and Flows
This document describes the containers within the **Insurance Hub** system and the primary flows of information between them. The architecture is designed as a set of collaborating microservices that communicate both synchronously via REST APIs and asynchronously via an event bus.
#### 1. Systems & Actors External to the Hub
- **Insurance Agent (Person)**: The primary user of the system who interacts with the various services through a unified user interface that communicates with the **Agent Portal Gateway**.
- **JSReport Service**: A third-party, external system responsible for rendering HTML templates into PDF documents. It is used exclusively by the **Documents Service**.

#### 2. Containers within the Insurance Hub System
- **Agent Portal Gateway**:
    - **Role**: The single, unified entry point for all synchronous API calls originating from the client-side user interface.
    - **Function**: It handles cross-cutting concerns like authentication and routes incoming HTTP requests to the appropriate downstream microservice (, , , or ). `policy-service``policy-search-service``documents-service``payment-service`

- **Policy Service**:
    - **Role**: The core service responsible for the business logic of creating and managing insurance offers and policies.
    - **Function**: It implements a CQRS pattern, separating write-operations (creating offers, converting them to policies) from read-operations. It synchronously calls the **Pricing Service** to get price calculations. Upon successful creation of a policy, it publishes a `PolicyCreated` event to **Apache Kafka**.

- **Pricing Service**:
    - **Role**: A specialized service that provides on-demand price calculations for insurance products.
    - **Function**: It exposes a synchronous REST API used by the **Policy Service**. It calculates prices by executing complex business rules (tariffs) which are loaded from MVEL files stored on the **Tariff Rule Files** file system.

- **Policy Search Service**:
    - **Role**: Provides an optimized, fast full-text search capability over all policies.
    - **Function**: It asynchronously listens for `PolicyCreated` events on the **Apache Kafka** bus. When an event is received, it transforms the policy data into a denormalized read model and indexes it in **Elasticsearch**. It also exposes its own REST API for handling search queries from the gateway.

- **Documents Service**:
    - **Role**: Manages the asynchronous generation and lifecycle of policy documents (PDFs).
    - **Function**: It is triggered by `PolicyCreated` events from **Apache Kafka**. Upon receiving an event, it orchestrates the generation of a PDF by calling the external **JSReport Service**. The final document is then saved to the **Document Storage** and can be retrieved later via this service's API.

- **Payment Service**:
    - **Role**: Manages all financial aspects of a policy, including its account, balance, and payments.
    - **Function**: It listens for `PolicyCreated` events to automatically create a new financial account for the policy in the **Payment DB**. It also has a scheduled process for importing and reconciling payments from CSV files found in the **Bank Statements** file system.

- **Apache Kafka**:
    - **Role**: The central event bus for the entire system.
    - **Function**: It facilitates the event-driven architecture by decoupling services. The **Policy Service** publishes events, and multiple other services subscribe to these events to perform their work asynchronously and independently.

- **Data Stores (Policy DB, Payment DB, Elasticsearch, etc.)**:
    - **Role**: A set of dedicated, persistent storage containers.
    - **Function**: Each service owns and manages its own database, following the database-per-service pattern. This includes relational databases for transactional data, a search engine for query optimization, and various file system mounts for storing files like PDFs, pricing rules, and bank statements.


