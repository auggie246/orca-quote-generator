# Architecture Document: Orca Quote Generator (v1.0)

### 1. High Level Architecture

* **Technical Summary**
    This architecture describes a containerized monolith application. The system consists of a single service, built on a Rust/Python foundation, that acts as both the web server and the backend logic engine. It is responsible for rendering and serving a simple HTML user interface using the Jinja2 templating engine, orchestrating the OrcaSlicer CLI, calculating prices, and dispatching Telegram notifications. The entire system is designed for deployment as a single Docker container.

* **High Level Overview**
    The project will be built as a Containerized Monolith, managed within a Monorepo. Project management will be handled via GitHub Issues/Projects, with technical package management handled by `uv`.

* **High Level Project Diagram**
    ```mermaid
    graph TD
        subgraph User's Device
            A[User's Browser]
        end

        subgraph "Local Server (Docker Environment)"
            C[Traefik Reverse Proxy]
            subgraph "Application Container"
                D[Application Service <br><i>(FastAPI + Jinja2 + Python/Rust)</i>]
            end
            E[OrcaSlicer CLI]
            F[File System <br><i>(Temp Storage)</i>]
        end

        subgraph External Services
            G[Telegram API]
        end

        A -- HTTP/S Request --> C
        C -- Routes Traffic --> D
        D -- Serves HTML UI --> A
        D -- Handles File Upload --> A
        D -- Writes/Reads File --> F
        D -- Executes --> E
        E -- Reads File --> F
        D -- Sends Notification --> G
    ```

* **Architectural and Design Patterns**
    * **Dependency Injection & Configuration Provider:** To be established in **Epic 1** to ensure the core engine is testable and configurable. A dedicated `/packages/config` directory will be used for shared configuration.
    * **Repository Pattern:** To be formally implemented in **Epic 3** when we introduce the database logic for storing quotes.

### 2. Tech Stack

| Category | Technology | Version | Purpose | Rationale |
| :--- | :--- | :--- | :--- | :--- |
| **Language & Runtime** | Python | 3.12 | Primary backend language for orchestration. | Modern, robust, and specified in the PRD. |
| | Rust | 1.88 | Core logic implementation for performance. | A key stakeholder requirement for the project. |
| **Language Bridge** | PyO3 | 0.25.1 | Rust bindings for the Python interpreter. | The essential library for Rust/Python interoperability. |
| | Maturin | 1.7+ | Build tool for creating Rust-powered Python packages. | Manages the build process for our mixed-language library. |
| **Backend Framework**| FastAPI | 0.111+ | Web framework and API server. | High-performance and handles all web requests. |
| **Templating Engine**| Jinja2 | 3.1+ | Renders server-side HTML for the UI. | Standard templating engine for FastAPI. |
| **Data Validation** | Pydantic | 2.8+ | Data validation and settings management. | The industry standard for data validation in modern Python. |
| **Package Management**| uv | 0.1.40+ | Managing Python dependencies. | A core requirement, chosen for its speed. |
| **Code Quality** | Ruff / pre-commit | Latest | Linter, formatter, and Git hook framework. | A core requirement for automated code quality. |
| **Testing** | Pytest | 8.2+ | Framework for writing and running Python unit tests. | The standard for Python testing. |
| | factory-boy | 3.3+ | A library for creating test data fixtures. | Provides a clean, factory-based approach to test data management. |
| **Integrations** | python-telegram-bot| 21.1+ | Library for communicating with the Telegram API. | A core requirement specified in the PRD. |
| | OrcaSlicer CLI | 2.1+ | External tool for 3D model slicing. | The core engine for the quoting logic. |
| **Database** | PostgreSQL | 16+ | Primary relational database for storing quote data. | Powerful, open-source, and works excellently with the Python stack. |
| **Object Storage**| MinIO | Latest | S3-compatible storage for 3D model files. | Scalable, robust, and decouples file storage from the application server. |
| **Storage Client** | minio (Python) | 7.2+ | Python client library for interacting with MinIO. | Official and well-supported library for object storage operations. |
| **Deployment** | Docker | 26.1+ | Containerization for the monolith service. | A core requirement for consistent, portable deployments. |
| | Traefik | 3.0+ | Reverse proxy. | Leverages the existing reverse proxy on the host server. |

### 3. Data Models

* **`QuoteRequest`**
    * **Purpose:** The core data model representing a single quote job.
    * **Key Attributes:** `id`, `customer_name`, `contact_method`, `customer_contact`, `status`, `failure_reason`, `storage_object_key`, `original_filename`, `file_size_bytes`, `pricing_formula_version`, `calculated_price`, `created_at`, `updated_at`.
    * **Relationships:** A `QuoteRequest` has a many-to-one relationship with `Material` and `Quality`.
* **`Material` (New)**
    * **Purpose:** A lookup table for valid print materials.
    * **Key Attributes:** `id`, `name`, `is_active`.
* **`Quality` (New)**
    * **Purpose:** A lookup table for valid print qualities.
    * **Key Attributes:** `id`, `name`, `is_active`.

### 4. Components

* **`Application_Service`**
    * **Responsibility:** As a monolith, this single service is responsible for all system logic: UI rendering, handling uploads, security scans, orchestrating the slicer, calculating prices, and sending notifications.
    * **Key Interfaces:** Exposes HTML-serving endpoints, a `GET /healthz` endpoint for monitoring, and interacts with the file system, OrcaSlicer CLI, and Telegram API.

### 5. External APIs

* **OrcaSlicer CLI**
    * **Integration Notes:** The application will use a configurable path (`ORCA_SLICER_PATH`) and the executable will be included in the Docker container. Slicer profiles will be version-controlled within the project repository.
* **Telegram Bot API**
    * **Integration Notes:** The application will interact with an internal `NotificationService` which acts as an abstraction layer over the `python-telegram-bot` library. The library version will be pinned.

### 6. Core Workflows
*(This section contains the two Mermaid sequence diagrams for the "Happy Path" and "Failure Path" as previously defined).*

### 7. REST API Specification
*(This section contains the finalized OpenAPI 3.0 specification, including CSRF protection, idempotency, and standardized error responses as previously defined).*

### 8. Database Schema
*(This section contains the finalized PostgreSQL DDL, including the three tables (`quote_requests`, `materials`, `qualities`) and the `updated_at` trigger as previously defined).*

### 9. Source Tree
*(This section contains the finalized flat directory structure as requested by the user).*

### 10. Infrastructure and Deployment
*(This section describes the final deployment strategy using a CI/CD pipeline with GitHub Actions to build and push the Docker image to GHCR, and clarifies the use of bind mounts for data persistence).*

### 11. Error Handling Strategy
*(This section describes the final strategy using Loguru, `X-Request-ID`, and proactive alerting as defined).*

### 12. Coding Standards
*(This section contains the finalized, strict standards for code, testing, and Git workflow as defined).*

### 13. Test Strategy
*(This section contains the final strategy detailing the quality-first philosophy, manual integration tests, use of factory-boy, and the single `lint-test.yaml` CI workflow).*

### 14. Security
*(This section contains the final, consolidated security plan including dependency scanning, Docker hardening, and explicit data protection rules).*

### 15. Checklist Results Report
* **Result:** **PASS**. The document is comprehensive, robust, and ready for development.

### 16. Next Steps
* **Handoff:** The PRD and this Architecture Document are the final blueprints for the MVP. Development can now begin.