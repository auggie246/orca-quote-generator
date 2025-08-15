# Product Requirements Document: Automated 3D Printing Quoting Web Application (v1.7)

### 1. Goals and Background Context

#### Change Log

| Date | Version | Description | Author |
| :--- | :--- | :--- | :--- |
| 2025-07-30 | 1.7 | Major architectural pivot to a single-container monolith with a server-rendered Jinja2 UI. PRD updated to reflect simplified architecture and tech stack. | Winston, Architect |
| 2025-07-28 | 1.6 | Finalized epic structure based on risk-first approach. Clarified pricing config is file-based for MVP. | John, PM |
| 2025-07-28 | 1.5 | Finalized technical assumptions: Monorepo, Monolith architecture. Clarified that integration tests are required but manually executed by lead dev. | John, PM |
| 2025-07-28 | 1.4 | Finalized UI Goals with progress bar, confirmation modal, and branding definition task. | John, PM |
| 2025-07-28 | 1.3 | Finalized MVP requirements: limited contact to Telegram only, made quality selection mandatory, and deferred business metric tracking. | John, PM |
| 2025-07-28 | 1.2 | Redefined workflow to include admin verification before slicing. Updated goals and requirements accordingly. | John, PM |
| 2025-07-28 | 1.1 | Revised goals to include admin approval workflow and measurable targets. | John, PM |
| 2025-07-28 | 1.0 | Initial PRD draft | John, PM |


#### Goals

* To achieve a 98% reduction in the manual work required to generate a quote.
* To deliver a final, calculated quote to the administrator **within 90 seconds of the administrator initiating the slice command**.
* To enable the administrator to approve a quote and forward it to the customer with a single action.

#### Background Context

The current quoting process is a manual, inefficient workflow. This project aims to build a web application that validates and queues new quote requests for administrator approval. Once an admin verifies a request, the system integrates an open-source slicer (OrcaSlicer) to fully automate the price calculation, empowering the administrator to respond to potential customers with accurate quotes.

### 2. Requirements

#### Functional Requirements

1.  **FR1:** The system must allow a user to input their Name and their **Telegram contact**.
2.  **FR2:** The system must allow a user to upload one or more 3D model files.
3.  **FR3:** The system must enforce a **maximum file upload size of 50MB** per file.
4.  **FR4:** The system must validate uploaded files and only accept `.stl`, `.obj`, `.step`, and `.3mf` formats.
5.  **FR5:** The system must perform a security scan (e.g., virus scan) on all uploaded files before they are stored or processed further.
6.  **FR6:** The system must securely store the scanned files in a temporary local directory.
7.  **FR7:** The system must provide the user with a choice of printing materials (PLA, PETG, ASA).
8.  **FR8:** The system must require the user to **choose a print quality** (e.g., Standard, High), with "Standard" selected by default.
9.  **FR9:** The system must send a "New Request for Slicing" notification to the administrator via Telegram, including user info, file names, and a "Start Slicing" button.
10. **FR10:** The administrator must be able to initiate the slicing process by interacting with the Telegram notification.
11. **FR11:** **Upon admin initiation**, the system must call the OrcaSlicer CLI with the correct profiles based on the user's selections.
12. **FR12:** If the slicing process fails, the system must **notify the administrator via Telegram with the slicer's error output** and cancel the quoting process for that request.
13. **FR13:** The system must parse the slicer output to extract the estimated print time and required filament (in grams).
14. **FR14:** The system must calculate a price based on the parsed slicer output using the externally configured formula.
15. **FR15:** The system must enforce a minimum price of $5.00.
16. **FR16:** The system must send a notification to the administrator via a Telegram bot, containing the final quote.
17. **FR17:** The system must provide a mechanism for the administrator to approve a generated quote to be sent to the customer.

#### Non-Functional Requirements

1.  **NFR1:** The backend logic must use Rust bindings to the Python interpreter via PyO3 and `maturin`. **As much as possible, most logic should be implemented in Rust and called from a thin Python layer**.
2.  **NFR2:** All package and project management must use `uv` within a `venv` environment.
3.  **NFR3:** All code must be linted and formatted using `ruff`, adhering to PEP 8 with a 120-character line limit.
4.  **NFR4:** All functions must include Google Style docstrings and type hints. Type hints must adhere to the best practices defined in **PEP 484** and related typing PEPs.
5.  **NFR5:** Every function must have a corresponding unit test that validates code logic only.
6.  **NFR6:** A strict Git workflow (feature branches, pre-commit checks, PRs to `dev`) must be followed.
7.  **NFR7:** The pricing formula variables must be configurable in an external file (e.g., YAML or JSON) that can be modified by an administrator and reloaded by the application without requiring a new code deployment.
8.  **NFR8:** The Telegram bot integration must use the `python-telegram-bot` package with credentials stored in an `.env` file.
9.  **NFR9:** The system must deliver the calculated quote to the administrator within 90 seconds of the administrator initiating the slice command.

### 3. User Interface Design Goals

#### Overall UX Vision
The user experience should be simple, fast, and clear. The primary goal is to guide the user through the form submission with zero friction. The interface must feel modern, trustworthy, and professional.

#### Key Interaction Paradigms
The application will be a **server-rendered web application**. The FastAPI backend will use the Jinja2 templating engine to generate and serve HTML pages directly to the user.

#### Core Screens and Views
The user journey will be a simple, linear flow:
1.  **Quote Form Page:** A server-rendered HTML page containing the input form (Name, Contact), file upload zone, and material/quality selectors.
2.  **Submission Response Page:** After submission, the server will respond with a new page that either confirms the request was successfully submitted for admin review or displays a clear error message if validation or the security scan failed.

#### Legal & Compliance Considerations
* **PDPA (Singapore):** Personal data and files will only be processed after the user clicks the final submit button.

#### Accessibility: WCAG AA
The application must meet WCAG 2.1 AA standards.

#### Branding
A brand identity (logo, color scheme) for the service is required. The aesthetic should be modern and professional.

#### Target Device and Platforms: Web Responsive
The application must be fully responsive for desktop and mobile browsers.

### 4. Technical Assumptions

#### Repository Structure
A **Monorepo** structure will be used.

#### Service Architecture
A **Containerized Monolith** architecture will be implemented. A single service will handle all backend logic and UI rendering.

#### Testing Requirements
* Unit tests are required for every function (code logic only).
* Integration tests are required but will be executed manually by the lead developer.

#### Additional Technical Assumptions and Requests
* **Backend Technology:** Core logic in Rust, called from a thin Python layer using PyO3.
* **Package Management:** `uv` will be used.
* **Code Quality:** `ruff` for linting/formatting. Google Style docstrings and type hints are required.
* **Version Control:** A strict Git workflow with feature branches must be followed.
* **Deployment:** Self-hosted on a local server using Docker and Traefik.

### 5. Epic List

1.  **Epic 1: Foundational Slice & Technical Spike**
    * **Goal:** Establish the core project foundation, a containerized monolith service, and implement a minimal end-to-end "tracer bullet" workflow that can take a hardcoded 3D model, slice it, **apply the externally configurable pricing formula**, and log the result.
2.  **Epic 2: User Interface & Request Submission**
    * **Goal:** Develop the full user-facing, **server-rendered HTML interface using Jinja2**, allowing users to upload their own files and submit quote requests.
3.  **Epic 3: Full Admin Workflow & Final Quoting**
    * **Goal:** Implement the complete administrator notification/approval workflow via Telegram and the final delivery of the quote to the customer.

### 6. Checklist Results Report
* **Result:** **PASS**. The document is comprehensive, internally consistent, and ready for the architecture phase.

### 7. Next Steps
* **Architect Handoff:** This PRD is now ready to be handed off to the Architect.
* **Architect Prompt:** "Please review this completed Product Requirements Document. Your task is to create the full technical architecture for this containerized monolith service, respecting all non-functional requirements and technical constraints."