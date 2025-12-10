# OpenKM CE Frontend Migration Plan

## Executive Summary
This document outlines the strategy for migrating the OpenKM Community Edition frontend from **GWT (Google Web Toolkit)** to a modern JavaScript framework (e.g., **React, Vue.js, or Angular**).

## 1. Current State Analysis
*   **Technology**: GWT 2.8.2.
*   **Architecture**: Client-side Java compiled to JavaScript. Tightly coupled with the backend via **GWT-RPC**.
*   **Communication**: `com.openkm.servlet.frontend` contains servlets that extend `RemoteServiceServlet`. These servlets not only handle data transport but also contain significant **view logic** (e.g., formatting data for the UI, handling specific view hierarchies like Thesaurus/Metadata, PDF merging logic).
*   **Authentication**: Session-based (JSESSIONID), managed by Spring Security.

## 2. Migration Strategy: "API-First"
Directly replacing the frontend is impossible without first decoupling the backend logic from GWT-RPC. The existing REST API (`com.openkm.rest`) is robust but incomplete regarding specific UI-supporting features found in the GWT servlets.

### Phase 1: Gap Analysis & API Extension (Backend)
**Goal**: Create a backend that fully supports a generic frontend without GWT dependencies.

1.  **Identify Logic Gaps**: Compare `com.openkm.servlet.frontend.*` with `com.openkm.rest.endpoint.*`.
    *   *Example Gap*: `DocumentServlet.getChilds` handles Thesaurus/Metadata virtual paths. `DocumentService.getChildren` (REST) does not.
    *   *Example Gap*: `DocumentServlet.convertToPdf` handles caching and logic. `DocumentService` does not expose this.
    *   *Example Gap*: `WorkspaceServlet` loads initial state (user config, stamps, etc.) in one go. REST requires multiple calls.
2.  **Extend REST API**:
    *   Create new REST endpoints (e.g., `v2` or `frontend-api`) to expose the missing logic.
    *   **Crucial**: Ensure these new endpoints return standard JSON (using Jackson/Gson) instead of GWT-specific objects.
    *   Refactor logic from `DocumentServlet` into reusable Service components (if not already in `com.openkm.module`) so both GWT (legacy) and REST (new) can share it.

### Phase 2: Authentication & Session
**Goal**: Establish a secure connection for the new SPA (Single Page Application).

1.  **Authentication**: Continue using Spring Security.
    *   The new SPA will perform a POST to the login endpoint (e.g., `/j_spring_security_check` or a custom JSON login endpoint).
    *   Server returns `JSESSIONID` cookie.
    *   Browser handles the cookie automatically for subsequent API calls.
2.  **CSRF Protection**: Ensure the REST API is protected against CSRF if using session cookies.

### Phase 3: Frontend Development (Modern Framework)
**Goal**: Build the new UI.

1.  **Technology Choice**: React or Vue.js are recommended for their component-based architecture which maps well to OpenKM's panel layout.
2.  **Scaffolding**: Create a new project (e.g., `openkm-ui`) outside or inside the Maven structure (e.g., `src/main/frontend`).
3.  **Component Mapping**:
    *   `Main.java` -> App Entry / Layout (Sidebar, Topbar).
    *   `Workspace.java` -> Dashboard / FileBrowser views.
    *   `FileBrowser` -> Components for Tree, List, Properties, Preview.
4.  **Development Workflow**: Use a proxy (e.g., Vite/Webpack dev server) to forward API requests to the running Tomcat/OpenKM instance.

### Phase 4: Integration & Deployment
**Goal**: Serve the new frontend from the OpenKM WAR.

1.  **Build Process**: Configure Maven (using `frontend-maven-plugin`) to build the React/Vue app during the `package` phase.
2.  **Artifacts**: Copy the production build (HTML/CSS/JS) to `src/main/webapp` (e.g., under `/ui`).
3.  **Routing**: Configure `web.xml` or a Servlet Filter to serve the `index.html` for the new UI routes.

## 3. Components to Modify

| Component | Action | Details |
| :--- | :--- | :--- |
| `com.openkm.servlet.frontend.*` | **Read/Refactor** | Extract business logic (Thesaurus navigation, specialized filtering) into `com.openkm.module` or helper classes so it can be reused by REST. |
| `com.openkm.rest.endpoint.*` | **Modify/Create** | Add missing endpoints identified in Phase 1. Add `v2` endpoints for UI-specific aggregations (to reduce network chatter). |
| `src/main/webapp/WEB-INF/web.xml` | **Modify** | Add configuration to serve the new static assets. |
| `pom.xml` | **Modify** | Add build steps for the new frontend (Node.js/NPM integration). |

## 4. Risks & Mitigations
*   **Risk**: Feature Parity. GWT has been developed for years.
    *   *Mitigation*: Don't aim for a "Big Bang" release. Release the new UI as a "Beta" accessible via a different URL (e.g., `/openkm/new`) while keeping the GWT UI default.
*   **Risk**: Custom Extensions. Users might have GWT-based extensions.
    *   *Mitigation*: This is a breaking change. Extensions will need to be rewritten in the new framework. Provide a plugin architecture in the new UI.
