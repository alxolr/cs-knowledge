# API Design

**Track:** Design Concepts
**Difficulty Tier:** Beginner
**Prerequisites:** [SOLID Principles](../solid-principles/README.md)

## Concept Overview

API design is the practice of defining clear, consistent, and usable interfaces that allow software components to communicate. Whether you are building a REST API for a web service, a library SDK, or an internal module boundary, the quality of the interface determines how easily other developers can integrate with your system, how gracefully it evolves over time, and how reliably it behaves under unexpected conditions.

Good API design applies many of the same principles you learned in SOLID — single responsibility guides endpoint scoping, open/closed thinking shapes versioning strategy, and interface segregation keeps payloads focused. Beyond those foundations, API design introduces its own concerns: resource modeling, HTTP semantics, status code selection, pagination, authentication boundaries, and backward compatibility.

A poorly designed API forces consumers into workarounds, generates excessive support burden, and makes breaking changes inevitable. A well-designed API feels intuitive to use without reading documentation, communicates errors clearly, and can grow for years without forcing existing clients to rewrite their integrations. The problems in this module give you practice making these design decisions across realistic scenarios.

---

## Problem 1 — Designing a RESTful Resource for a Library Catalog

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Design a RESTful API for a public library's book catalog. The API should allow clients to list all books, retrieve a single book by identifier, add a new book, update book details, and remove a book from the catalog. Choose appropriate HTTP methods, URL paths, request/response shapes, and status codes for each operation.

### Scenario

**Context:** A city library is building a web application that lets patrons browse and search the catalog online. The backend team needs a clean API that the frontend and future mobile app can consume. The catalog contains roughly 50,000 books, each with a title, author, ISBN, genre, and availability status.

**Requirements:** Define the resource URL structure, HTTP methods, and expected status codes for the five CRUD operations. The design should follow REST conventions so that any developer familiar with REST can predict the API's behavior without reading extensive documentation.

**Expected Approach:** Model "book" as the primary resource. Map CRUD operations to standard HTTP verbs. Choose URL patterns that are noun-based and hierarchical. Select status codes that accurately communicate the outcome of each operation.

<details>
<summary>Hints</summary>

1. REST APIs model resources as nouns in the URL path — use `/books` for the collection and `/books/{id}` for an individual book.
2. Map operations to HTTP verbs: GET for reading, POST for creating, PUT or PATCH for updating, DELETE for removing.
3. Return 201 Created (with a Location header) for successful creation, 204 No Content for successful deletion, and 404 Not Found when a book ID does not exist.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Apply standard REST resource modeling to the book catalog.

1. **Resource URL:** `/books` represents the collection; `/books/{isbn}` represents a single book (ISBN is a natural unique identifier).
2. **List books:** `GET /books` → returns 200 with an array of book summaries. Support query parameters like `?genre=fiction&page=1&limit=20` for filtering and pagination.
3. **Get one book:** `GET /books/{isbn}` → returns 200 with the full book object, or 404 if the ISBN is not found.
4. **Add a book:** `POST /books` with a JSON body containing title, author, ISBN, genre, and availability. Returns 201 with a `Location: /books/{isbn}` header and the created resource in the body.
5. **Update a book:** `PUT /books/{isbn}` with the full updated book object → returns 200 with the updated resource, or 404 if not found. Alternatively use `PATCH /books/{isbn}` for partial updates.
6. **Remove a book:** `DELETE /books/{isbn}` → returns 204 on success, or 404 if not found.
7. Use plural nouns (`/books`, not `/book`), avoid verbs in URLs (`/books`, not `/getBooks`), and keep the hierarchy flat since books are the only resource in scope.

</details>

---

## Problem 2 — Designing Clear and Consistent Error Responses

**Difficulty:** Easy
**Estimated Time:** 30 minutes

### Problem Statement

Design a standardized error response format for an API that serves multiple client applications. The format should help clients programmatically handle errors, give developers enough detail to debug issues, and remain consistent across every endpoint in the API.

### Scenario

**Context:** A SaaS company's API currently returns errors in inconsistent formats — some endpoints return plain text messages, others return JSON with different field names, and a few return HTML error pages. Client developers complain that they cannot write a single error-handling function because every endpoint behaves differently.

**Requirements:** Define a single JSON error response structure that all endpoints will use. The structure must include a machine-readable error code, a human-readable message, and enough context to identify what went wrong. It should handle both single errors (e.g., resource not found) and validation errors (e.g., three fields failed validation simultaneously). The design must pair each error response with an appropriate HTTP status code.

**Expected Approach:** Identify the fields that every error response needs. Decide how to represent multiple validation errors in one response. Map common failure scenarios to HTTP status codes.

<details>
<summary>Hints</summary>

1. Include at minimum: an HTTP status code, an application-specific error code (string like `"RESOURCE_NOT_FOUND"`), a human-readable message, and an optional `details` array for field-level validation errors.
2. Validation errors (400) often involve multiple fields — use an array of objects, each with the field name and the specific violation.
3. Keep the top-level structure identical for all errors; only the contents change.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Define a uniform envelope that every error response follows.

1. **Error response structure:**
   ```json
   {
     "error": {
       "code": "VALIDATION_FAILED",
       "message": "One or more fields failed validation.",
       "details": [
         { "field": "email", "reason": "Must be a valid email address." },
         { "field": "age", "reason": "Must be a positive integer." }
       ]
     }
   }
   ```
2. **Required fields:** `code` (machine-readable string), `message` (human-readable summary). **Optional field:** `details` (array of objects providing granular context).
3. **Status code mapping:**
   - 400 Bad Request → validation failures, malformed JSON
   - 401 Unauthorized → missing or invalid authentication
   - 403 Forbidden → authenticated but insufficient permissions
   - 404 Not Found → resource does not exist
   - 409 Conflict → duplicate resource or state conflict
   - 500 Internal Server Error → unexpected server failure
4. Every endpoint returns this same structure on failure. Clients parse `error.code` to branch logic and display `error.message` to users.
5. For single-error cases (e.g., 404), the `details` array is omitted or empty, keeping the envelope consistent.

</details>

---

## Problem 3 — Designing Pagination for a Large Dataset

**Difficulty:** Easy
**Estimated Time:** 35 minutes

### Problem Statement

Design a pagination strategy for an API endpoint that returns a potentially large collection of records. The design should let clients efficiently navigate through results, handle the case where new records are added between page requests, and clearly communicate how many total results exist and how to fetch the next page.

### Scenario

**Context:** A job board platform has an endpoint `GET /jobs` that returns open job listings. The database contains over 200,000 active listings and grows by several thousand per day. The mobile app currently fetches all listings in a single response, causing timeouts and excessive memory usage on both server and client.

**Requirements:** Redesign the endpoint to return results in pages. Clients must be able to request a specific page size, navigate forward through results, and know when they have reached the last page. The design should minimize the impact of new listings being inserted while a client is paginating through results.

**Expected Approach:** Compare offset-based and cursor-based pagination. Choose the approach that best handles a rapidly growing dataset. Define the request parameters and response metadata.

<details>
<summary>Hints</summary>

1. Offset-based pagination (`?page=3&limit=25`) is simple but can skip or duplicate records when new items are inserted between requests.
2. Cursor-based pagination (`?after=<cursor>&limit=25`) uses an opaque token (often an encoded ID or timestamp) to mark the position, making it stable even when new records appear.
3. Include pagination metadata in the response — at minimum a `nextCursor` (or `nextPage`) field and a boolean or link indicating whether more results exist.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use cursor-based pagination to handle the rapidly growing dataset.

1. **Request:** `GET /jobs?limit=25&after=<cursor>` where `limit` controls page size (default 25, max 100) and `after` is an opaque cursor string. The first request omits `after`.
2. **Cursor implementation:** Encode the last item's sort key (e.g., creation timestamp + ID) as a base64 string. The server decodes it to build a `WHERE created_at < :ts OR (created_at = :ts AND id < :id)` query.
3. **Response structure:**
   ```json
   {
     "data": [ ... ],
     "pagination": {
       "limit": 25,
       "hasMore": true,
       "nextCursor": "eyJjcmVhdGVkX2F0IjoiMjAyNC..."
     }
   }
   ```
4. **Why cursor over offset:** When new jobs are inserted, offset-based pagination shifts all subsequent rows, causing clients to see duplicates or miss records. Cursor-based pagination is anchored to a specific position in the sorted result set, so inserts above the cursor do not affect pages below it.
5. **Last page:** When `hasMore` is `false`, the client knows there are no more results. Optionally include a `totalCount` field (computed via a separate count query) if the client needs to display "showing 1–25 of 203,417."
6. **Edge cases:** If the cursor is invalid or expired, return 400 with a clear error message instructing the client to restart pagination from the beginning.

</details>

---

## Problem 4 — Designing a Versioning Strategy for a Public API

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Design a versioning strategy for a public-facing API that has hundreds of active third-party integrations. The API needs to evolve — adding new fields, deprecating old endpoints, and occasionally making breaking changes — without disrupting existing clients. Define how versions are communicated, how long old versions are supported, and how clients migrate.

### Scenario

**Context:** A fintech company exposes a payments API used by 300+ merchant integrations. The team needs to restructure the transaction response object (renaming fields, nesting previously flat data, removing deprecated fields) to support new regulatory requirements. Deploying these changes without a versioning plan would break every existing integration simultaneously.

**Requirements:** Choose a versioning mechanism (URL path, query parameter, header, or content negotiation). Define a deprecation policy that gives clients time to migrate. Describe how the server handles requests that do not specify a version. Explain how the codebase supports multiple active versions without excessive duplication.

**Expected Approach:** Evaluate the trade-offs of each versioning mechanism. Design a lifecycle (active → deprecated → sunset) with concrete timelines. Propose a code-organization strategy that keeps version-specific logic manageable.

<details>
<summary>Hints</summary>

1. URL-path versioning (`/v1/transactions`, `/v2/transactions`) is the most visible and easiest for clients to understand, but it duplicates route definitions.
2. Header-based versioning (`Accept: application/vnd.company.v2+json`) keeps URLs clean but is less discoverable and harder to test in a browser.
3. Consider a "version lifecycle" with three states: active (fully supported), deprecated (functional but clients are warned via response headers), and sunset (returns 410 Gone).
4. Internally, use a shared service layer and version-specific adapter/transformer layers to avoid duplicating business logic.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Use URL-path versioning with a structured deprecation lifecycle and internal adapter layers.

1. **Versioning mechanism:** URL path — `/v1/transactions`, `/v2/transactions`. This is the most explicit approach and works well for a public API where discoverability matters.
2. **Default version:** Requests to `/transactions` (no version prefix) are redirected to the latest stable version via a 302 redirect, or return an error instructing the client to specify a version. Avoid silently defaulting to the latest version, as this can break clients unexpectedly.
3. **Version lifecycle:**
   - **Active:** Fully supported, receives bug fixes and security patches.
   - **Deprecated:** Still functional for 12 months after deprecation announcement. Responses include a `Sunset` header with the retirement date and a `Deprecation` header. Documentation is updated with migration guides.
   - **Sunset:** After the 12-month window, the version returns `410 Gone` with a body pointing to the migration guide.
4. **Communication:** Announce deprecations via changelog, email to registered developers, and a `Warning` response header on every deprecated-version response.
5. **Internal code organization:**
   - A shared service layer contains all business logic (transaction processing, validation, regulatory checks).
   - Each version has a thin adapter/transformer layer that maps between the version-specific request/response shapes and the internal domain model.
   - Example: `v1/TransactionAdapter` flattens the internal nested structure into the v1 flat format; `v2/TransactionAdapter` exposes the nested structure directly.
   - This avoids duplicating business logic across versions — only the serialization layer differs.
6. **Breaking vs. non-breaking changes:** Additive changes (new optional fields) are applied to the current version without a version bump. Breaking changes (field renames, removals, structural changes) require a new version.

</details>

---

## Problem 5 — Designing an API for a Multi-Tenant Task Management System

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Design a complete API for a multi-tenant task management system where organizations can create projects, assign tasks to team members, track task status through a workflow, and generate progress reports. The API must enforce tenant isolation, support role-based access control, handle concurrent task updates gracefully, and be designed for long-term evolvability.

### Scenario

**Context:** A startup is building a project management SaaS product. Each customer organization (tenant) has its own projects, users, and tasks. The API will be consumed by a web app, a mobile app, and third-party integrations via webhooks. The team expects rapid feature growth — comments on tasks, file attachments, and time tracking are on the six-month roadmap.

**Requirements:**
- Define resources, URL hierarchy, and HTTP methods for organizations, projects, tasks, and users.
- Tasks follow a workflow: `open → in_progress → review → done`. Only valid transitions should be allowed.
- Enforce tenant isolation so that Organization A can never access Organization B's data.
- Support role-based access: admin (full access), manager (manage projects and assign tasks), member (update own tasks only).
- Handle concurrent updates (two users editing the same task simultaneously) without silent data loss.
- Design the API so that future features (comments, attachments, time entries) can be added without breaking existing clients.

**Expected Approach:** Model the resource hierarchy carefully. Use sub-resources where ownership is clear. Apply state-machine thinking to task transitions. Choose a concurrency control mechanism. Plan for extensibility from the start.

<details>
<summary>Hints</summary>

1. Nest resources to express ownership: `/orgs/{orgId}/projects/{projectId}/tasks/{taskId}`. This naturally scopes every request to a tenant.
2. Model task state transitions as an explicit action endpoint (e.g., `POST /tasks/{id}/transitions` with a `targetState` field) rather than allowing arbitrary PATCH updates to the status field — this lets the server enforce valid transitions.
3. Use optimistic concurrency control with an `ETag` or `version` field. Clients send `If-Match: <etag>` on updates; the server returns 409 Conflict if the resource has changed since the client last read it.
4. For role-based access, include the user's role in the authentication token (JWT claim) and enforce permissions at the API gateway or middleware layer. Return 403 when a user attempts an action outside their role.
5. Design response payloads with an envelope that can accommodate new fields without breaking existing clients — clients should ignore unknown fields.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Combine hierarchical resource modeling, explicit state transitions, optimistic concurrency, and role-based middleware.

1. **Resource hierarchy and endpoints:**
   - `GET/POST /orgs/{orgId}/projects` — list and create projects within an organization.
   - `GET/PUT/DELETE /orgs/{orgId}/projects/{projectId}` — read, update, delete a project.
   - `GET/POST /orgs/{orgId}/projects/{projectId}/tasks` — list and create tasks within a project.
   - `GET/PUT/DELETE /orgs/{orgId}/projects/{projectId}/tasks/{taskId}` — read, update, delete a task.
   - `GET/POST /orgs/{orgId}/users` — list and invite users to the organization.
   - `POST /orgs/{orgId}/projects/{projectId}/tasks/{taskId}/transitions` — advance a task's workflow state.

2. **Tenant isolation:** Every URL is scoped under `/orgs/{orgId}`. Middleware extracts the `orgId` from the authenticated user's token and compares it to the URL parameter. If they do not match, return 403 immediately. Database queries always include an `org_id` filter.

3. **Task state machine:**
   - Valid transitions: `open → in_progress`, `in_progress → review`, `review → done`, `review → in_progress` (rework).
   - The `POST .../transitions` endpoint accepts `{ "targetState": "in_progress" }`. The server checks the current state, validates the transition, and returns 200 with the updated task or 422 Unprocessable Entity if the transition is invalid.
   - This keeps state logic on the server and prevents clients from setting arbitrary status values.

4. **Role-based access control:**
   - Roles: `admin`, `manager`, `member`.
   - Middleware reads the role from the JWT, then applies rules:
     - `admin`: all operations.
     - `manager`: create/update/delete projects and tasks, assign tasks to any user.
     - `member`: update only tasks assigned to themselves, read-only access to projects.
   - Unauthorized actions return 403 with an error body explaining the required role.

5. **Concurrency control:**
   - Every task response includes an `ETag` header (or a `version` field in the body) derived from a row version or hash.
   - Update requests must include `If-Match: <etag>`. The server compares the provided ETag with the current value. If they differ, return 409 Conflict with the current resource state so the client can merge or retry.
   - This prevents silent overwrites when two users edit the same task simultaneously.

6. **Extensibility:**
   - Future sub-resources (comments, attachments, time entries) fit naturally under the task: `/tasks/{taskId}/comments`, `/tasks/{taskId}/attachments`.
   - Response payloads use an additive strategy — new fields are added without removing existing ones. Clients are expected to ignore unknown fields.
   - Use URL-path versioning (`/v1/orgs/...`) from day one so breaking changes can be introduced in `/v2/` when necessary.

</details>
