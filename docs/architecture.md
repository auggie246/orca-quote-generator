# Architecture Document: Orca Quote Generator (v1.0)

### 1. High Level Architecture

* **Technical Summary**
    This architecture describes a containerized monolith application. The system consists of a single service, built on a Rust/Python foundation, that acts as both the web server and the backend logic engine. It is responsible for rendering and serving a simple HTML user interface using the Jinja2 templating engine, orchestrating the OrcaSlicer CLI, calculating prices, and dispatching Telegram notifications. The entire system is designed for deployment as a single Docker container.

* **High Level Overview**
    The project will be built as a Containerized Monolith, managed within a Monorepo. Project management will be handled via GitHub Issues/Projects, with technical workspace management handled by `uv`.

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
    * **Dependency Injection & Configuration Provider:** To be established in **Epic 1**. A dedicated `/packages/config` directory will be used for shared configuration.
    * **Repository Pattern:** To be formally implemented in **Epic 3**.
    * **Graceful Degradation:** The UI will inform the user if the backend is unavailable.
    * **Centralized Logging:** The service will output structured logs to be aggregated by the container orchestrator.

### 2. Tech Stack

| Category | Technology | Version | Purpose | Rationale |
| :--- | :--- | :--- | :--- | :--- |
| **Language & Runtime** | Python | 3.12+ | Primary backend language for orchestration. | Modern, robust, and specified in the PRD. |
| | Rust | 1.79+ | Core logic implementation for performance. | A key stakeholder requirement for the project. |
| **Language Bridge** | PyO3 / Maturin | Latest | Bridge between Python and Rust. | Essential for the mixed-language architecture. |
| **Backend Framework**| FastAPI | 0.111+ | Web framework and API server. | High-performance and handles all web requests. |
| **Templating Engine**| Jinja2 | 3.1+ | Renders server-side HTML for the UI. | Standard templating engine for FastAPI; simple and powerful. |
| **Data Validation** | Pydantic | 2.8+ | Data validation and settings management. | The industry standard for data validation in modern Python; integrates perfectly with FastAPI. |
| **Package Management**| uv | 0.1.40+ | Managing Python dependencies. | A core requirement, chosen for its speed. |
| **Code Quality** | Ruff / pre-commit | Latest | Linter, formatter, and Git hook framework. | A core requirement for automated code quality. |
| **Testing** | Pytest | 8.2+ | Framework for writing and running Python unit tests. | The standard for Python testing. |
| **Integrations** | python-telegram-bot| 21.1+ | Library for communicating with the Telegram API. | A core requirement specified in the PRD. |
| | OrcaSlicer CLI | 2.1+ | External tool for 3D model slicing. | The core engine for the quoting logic. |
| **Deployment** | Docker | 26.1+ | Containerization for the monolith service. | A core requirement for consistent, portable deployments. |
| | Traefik | 3.0+ | Reverse proxy. | Leverages the existing reverse proxy on the host server. |

### 3. Data Models

* **`QuoteRequest`**
    * **Purpose:** The core data model that represents a single, complete quote job.
    * **Key Attributes:** `id`, `customer_name`, `contact_method`, `customer_contact`, `status`, `failure_reason`, `original_filename`, `stored_filename`, `file_path`, `file_size_bytes`, `material`, `quality`, `estimated_print_time_minutes`, `filament_required_grams`, `pricing_formula_version`, `calculated_price`, `created_at`, `updated_at`.
    * **Future Considerations:** The architecture should be designed to easily accommodate multi-file support in a future version.

### 4. Components

* **`Application_Service`**
    * **Responsibility:** As a monolith, this single service is responsible for all system logic: UI rendering, handling uploads, security scans, orchestrating the slicer, calculating prices, and sending notifications.
    * **Key Interfaces:** Exposes HTML-serving endpoints, a `GET /healthz` endpoint for monitoring, and interacts with the file system, OrcaSlicer CLI, and Telegram API.

### 5. External APIs

* **OrcaSlicer CLI**
    * **Integration Notes:** The application will use a configurable path (`ORCA_SLICER_PATH`) and include a specific version of the executable within its Docker container. Slicer profiles will be version-controlled within the project repository.
* **Telegram Bot API**
    * **Integration Notes:** The application will interact with an internal `NotificationService` which acts as an abstraction layer over the `python-telegram-bot` library. The library version will be pinned.

### 6. Core Workflows
This diagram shows the end-to-end process for a successful quote request. The automated workflow concludes when the final quote is delivered to the administrator. The final communication with the end-user is a manual process handled by the administrator.

```
sequenceDiagram
    participant User
    participant AppService as Application Service (FastAPI)
    participant Admin
    participant OrcaSlicer as OrcaSlicer CLI
    participant Telegram as Telegram API

    User->>+AppService: POST /quote (with form data + file)
    AppService->>AppService: Validate input & file
    AppService->>+Telegram: sendMessage ("New Request for Slicing", Inline Keyboard)
    Telegram-->>-Admin: Shows request with "Start Slicing" button

    Admin->>+Telegram: Clicks "Start Slicing"
    Telegram->>-AppService: Webhook with "start_slice" command

    AppService->>+OrcaSlicer: Execute slice command
    OrcaSlicer-->>-AppService: Returns success
    AppService->>AppService: Parse output, calculate price

    AppService->>+Telegram: sendMessage ("Quote Ready", Inline Keyboard)
    Telegram-->>-Admin: Shows quote with "Approve" button
    
    Admin->>+Telegram: Clicks "Approve"
    Telegram-->>-AppService: Webhook with "quote_approved" event
    
    AppService->>+Telegram: sendMessage (Final quote details to Admin)
    Telegram-->>-Admin: Delivers final quote for manual forwarding
```

This diagram shows what happens if the OrcaSlicer CLI fails to process a file.
```
sequenceDiagram
    participant AppService as Application Service (FastAPI)
    participant Admin
    participant OrcaSlicer as OrcaSlicer CLI
    participant Telegram as Telegram API

    Admin->>+Telegram: Clicks "Start Slicing"
    Telegram->>-AppService: Webhook with "start_slice" command

    AppService->>+OrcaSlicer: Execute slice command
    OrcaSlicer-->>-AppService: Returns ERROR with failure reason
    
    AppService->>AppService: Catch error and log details
    AppService->>+Telegram: sendMessage ("SLICING FAILED: [Error Details]")
    Telegram-->>-Admin: Delivers failure notification
```

### 7. REST API Specification
Security & Implementation Notes

- Protection: The endpoint must be protected against Cross-Site Request Forgery (CSRF) and be appropriately rate-limited to prevent abuse.
- Idempotency: The endpoint must support an Idempotency-Key header to prevent duplicate request processing for the MVP.
- File Handling: The application must stream file uploads directly to disk and not buffer them in memory.

```
openapi: 3.0.1
info:
  title: "Orca Quote Generator API"
  version: "1.0.0"
  description: "API for submitting 3D models to generate a print quote."

servers:
  - url: "/api/v1"
    description: "API Version 1"

paths:
  /quote-request:
    post:
      summary: "Submit a new quote request"
      parameters:
        - in: header
          name: Idempotency-Key
          schema:
            type: string
            format: uuid
          required: true
          description: "A unique key to prevent duplicate submissions."
      requestBody:
        required: true
        content:
          multipart/form-data:
            schema:
              type: object
              required:
                - name
                - contact_method
                - customer_contact
                - material
                - quality
                - file
              properties:
                name:
                  type: string
                contact_method:
                  type: string
                  enum: [TELEGRAM, EMAIL]
                customer_contact:
                  type: string
                material:
                  type: string
                  enum: [PLA, PETG, ASA]
                quality:
                  type: string
                  enum: [Standard, High]
                file:
                  type: string
                  format: binary
      responses:
        '202':
          description: "Accepted. The request has been successfully queued."
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SuccessResponse'
        '400':
          description: "Bad Request. Invalid input."
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '413':
          description: "Payload Too Large. The uploaded file exceeds the 50MB limit."
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

components:
  schemas:
    SuccessResponse:
      type: object
      properties:
        request_id:
          type: string
          format: uuid
        message:
          type: string
          example: "Your request has been submitted successfully."
    ErrorResponse:
      type: object
      properties:
        error:
          type: object
          properties:
            code:
              type: string
              example: "VALIDATION_ERROR"
            message:
              type: string
              example: "File type not supported."
```

