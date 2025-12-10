# OpenKM CE Architecture Analysis

## 1. High-Level Architecture Map

OpenKM CE (Community Edition) is a three-tier web application built on the Java EE stack.

*   **Presentation Layer (Frontend)**:
    *   **GWT (Google Web Toolkit)**: The main user interface is a Single Page Application (SPA) built with GWT. It communicates with the backend via GWT RPC (Remote Procedure Calls).
    *   **JSP/Servlets**: Used for specific non-GWT pages (login, error, download, mobile version, admin tools) and serving resources.
    *   **REST/SOAP/CMIS**: Exposes APIs for external integration.

*   **Business Logic Layer (Backend)**:
    *   **Core Modules**: Manages Documents, Folders, Mails, Security (Auth), and Utilities.
    *   **Services**: Interfaces (`com.openkm.module.*`) that define business operations.
    *   **Workflow Engine**: Integrated jBPM (JBoss Business Process Management) for document workflow.
    *   **Automation**: Rule engine for automatic actions based on events.
    *   **Search Engine**: Uses Apache Lucene (via Hibernate Search) for full-text indexing and retrieval.
    *   **Security**: Spring Security for authentication and authorization.

*   **Data Access Layer (Persistence)**:
    *   **Hibernate ORM**: Handles object-relational mapping to the database.
    *   **Database**: Stores metadata, configuration, and structural information (users, roles, folder hierarchy).
    *   **File System / Data Store**: Stores the actual binary content of documents (configurable, can be on disk or in DB).

## 2. Major Modules & Components

| Module | Description | Key Package |
| :--- | :--- | :--- |
| **Frontend (Web)** | The web-based user interface. Uses GWT for the desktop-like experience and JSP for simpler views. | `com.openkm.frontend` |
| **API Module** | The core service layer exposing business operations. Used by the frontend and external APIs. | `com.openkm.api`, `com.openkm.module` |
| **DAO Layer** | Data Access Objects using Hibernate to interact with the database. | `com.openkm.dao` |
| **Search Engine** | Handles indexing of document metadata and content (text extraction) using Lucene. | `com.openkm.index`, `com.openkm.module.db.DbSearchModule` |
| **Workflow** | Manages business processes and tasks assigned to users using jBPM. | `com.openkm.workflow` |
| **Automation** | Executes actions (like moving documents, sending emails) based on events and validation rules. | `com.openkm.automation` |
| **Security (Auth)** | Manages users, roles, and permissions (JAAS, Spring Security). | `com.openkm.principal`, `com.openkm.jaas` |
| **Rest/SOAP/CMIS** | External interfaces for third-party integration. | `com.openkm.rest`, `com.openkm.ws`, `com.openkm.cmis` |
| **Extensions** | Framework for extending OpenKM functionality. | `com.openkm.extension` |

## 3. Core Backend Workflow

The typical workflow for a request (e.g., "Create Document") is:

1.  **Request**: The user initiates an action in the GWT Frontend (e.g., clicking "Upload").
2.  **Service Call**: The GWT client calls a remote service (servlet) in `com.openkm.servlet.frontend`.
3.  **Module Interception**: The servlet delegates the call to the appropriate Module interface in `com.openkm.module` (e.g., `DocumentModule.create()`).
    *   These modules are typically implemented in `com.openkm.module.db` (e.g., `DbDocumentModule`).
4.  **Security Check**: The module checks if the current user has the required permissions via `AuthModule`.
5.  **Validation & Automation**:
    *   `AutomationManager` checks for pre-events.
    *   Validators run to ensure data integrity.
6.  **Core Processing**:
    *   The `DocumentModule` prepares the document metadata.
    *   Content is extracted for indexing.
7.  **Persistence**:
    *   `NodeDocumentDAO` (in `com.openkm.dao`) uses Hibernate to save the document metadata to the database.
    *   The binary file is stored (on disk or DB).
8.  **Indexing**: The `Indexer` (Lucene) is notified to index the new document for search.
9.  **Post-Processing**: Automation post-events are triggered (e.g., notify users).
10. **Response**: The updated document object is returned to the GWT client to update the UI.

## 4. Core Packages

*   `com.openkm.api`: High-level API classes (`OKMDocument`, `OKMFolder`) wrapping the core logic.
*   `com.openkm.bean`: POJOs (Plain Old Java Objects) representing data entities (e.g., `Document`, `Version`).
*   `com.openkm.core`: Core infrastructure, configuration (`Config`), and exceptions.
*   `com.openkm.dao`: Hibernate data access objects.
*   `com.openkm.frontend`: GWT client-side code and UI logic.
*   `com.openkm.module`: Interfaces for the main business modules.
*   `com.openkm.module.db`: Database implementation of the business modules.
*   `com.openkm.servlet`: Java Servlets handling HTTP requests (startup, download, admin).
*   `com.openkm.util`: Utility classes (file handling, formatting, etc.).

## 5. Main Entry Points

*   **Application Startup**: `com.openkm.servlet.RepositoryStartupServlet`. Defined in `web.xml` with `load-on-startup`, it initializes the repository, configuration, and background tasks.
*   **Web Interface (GWT)**: `com.openkm.frontend.client.Main.onModuleLoad()`. This is the entry point for the GWT client application.
*   **API Entry Points**:
    *   **REST**: Defined in `src/main/webapp/WEB-INF/rest.xml` (using Apache CXF).
    *   **SOAP**: Defined in `src/main/webapp/WEB-INF/soap.xml` (using Apache CXF).
    *   **CMIS**: `org.apache.chemistry.opencmis.server.impl.atompub.CmisAtomPubServlet`.
*   **Servlets**: `com.openkm.servlet.frontend.*` act as endpoints for GWT RPC calls.

## 6. Complex / Important Parts

*   **`com.openkm.module.db.*`**: This is where the "meat" of the application logic lives. Understanding `DbDocumentModule.java` and `DbFolderModule.java` is crucial for understanding how data is manipulated.
*   **Security Model**: The permission system is granular (ACLs). Understanding how `AuthModule` interacts with other modules is key.
*   **GWT Implementation**: The frontend is a large GWT codebase. The `com.openkm.frontend.client` package handles the complex UI logic (panels, widgets, event handling). Debugging UI issues requires understanding GWT's asynchronous nature.
*   **Automation & Workflow**: These subsystems (`com.openkm.automation`, `com.openkm.workflow`) add layers of logic on top of simple CRUD operations. They can intercept and modify standard behaviors.
