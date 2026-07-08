# Implementation Plan: Sales Lead Management Platform

## Overview

Phased delivery across 6 phases: Phase 1 (Foundation/MVP), Phase 2 (Meeting Reports & Actions), Phase 3 (Opportunity Pipeline), Phase 4 (AI/pgvector), Phase 5 (Mobile PWA), Phase 6 (Production Hardening). The backend is FastAPI + Python, the frontend is React + TypeScript + Vite. All tasks build incrementally toward a fully wired production system.

## Tasks

### Phase 1: Foundation & MVP

- [ ] 1. Set up backend project structure and infrastructure scaffolding
  - Create FastAPI project layout: `app/`, `app/api/`, `app/models/`, `app/schemas/`, `app/services/`, `app/db/`, `app/core/`
  - Add `pyproject.toml` with pinned dependencies: `fastapi==0.111.*`, `uvicorn==0.30.*`, `sqlalchemy==2.0.*`, `asyncpg==0.29.*`, `alembic==1.13.*`, `pydantic==2.7.*`, `python-jose==3.3.*`, `boto3==1.34.*`, `hypothesis==6.100.*`, `pytest-asyncio==0.23.*`, `httpx==0.27.*`
  - Configure `app/core/config.py` to load all secrets from AWS Secrets Manager (never env vars)
  - Set up `app/db/session.py` with `asyncpg` + `SQLAlchemy AsyncSession` connection pool (10–20 connections)
  - Configure structured JSON logging middleware emitting `request_id`, `user_id`, `endpoint`, `status_code`, `duration_ms` to CloudWatch
  - _Requirements: 21.2, 22.1_

- [ ] 2. Create initial database migrations (Phase 1 schema)
  - [ ] 2.1 Write Alembic migration for `roles`, `teams`, and `users` tables
    - Include `cognito_sub`, `email`, `full_name`, `role_id`, `team_id`, `is_active` columns per design schema
    - _Requirements: 1.1, 2.1_
  - [ ] 2.2 Write Alembic migration for `accounts` and `contacts` tables
    - Add cascade-delete FK from contacts → accounts
    - Include `role_in_decision` column with check constraint accepting `Decision Maker`, `Influencer`, `User`, `Gatekeeper`
    - _Requirements: 3.5, 3.6, 3.7_
  - [ ] 2.3 Write Alembic migration for `leads` table and indexes
    - Include `idx_leads_status`, `idx_leads_owner_id`, `idx_leads_account_id`
    - _Requirements: 4.1, 4.2, 19.1_
  - [ ] 2.4 Write Alembic migration for `audit_logs` table
    - Include `idx_audit_entity` index; table must be insert-only (no update/delete permissions granted)
    - _Requirements: 15.1, 15.2, 15.4_
  - [ ] 2.5 Write Alembic migration for `notifications` table
    - _Requirements: 17.1, 17.2_

- [ ] 3. Implement Cognito JWT authentication middleware
  - [ ] 3.1 Implement `get_current_user` dependency in `app/core/auth.py`
    - Fetch Cognito JWKS, validate JWT signature and expiry, extract `sub`, `email`, `custom:role`, `team_id`
    - Return `AuthContext`; raise HTTP 401 on missing/expired token
    - _Requirements: 1.2, 1.3, 1.4, 22.6_
  - [ ] 3.2 Implement `require_role(*allowed_roles)` dependency factory
    - Raise HTTP 403 with `{"detail": "Access denied"}` when role not permitted
    - Role MUST be derived exclusively from JWT claim, never from request body
    - _Requirements: 2.2, 2.3_
  - [ ] 3.3 Implement `/auth/me`, `/auth/logout`, `/auth/refresh-token` endpoints
    - _Requirements: 1.5, 1.6, 1.7_
  - [ ]* 3.4 Write unit tests for auth middleware
    - Test valid JWT, expired JWT (401), missing JWT (401), wrong role (403)
    - Test each of the 7 defined roles resolves correctly
    - _Requirements: 1.2, 1.3, 2.1, 2.2_

- [ ] 4. Implement Account and Contact CRUD API
  - [ ] 4.1 Create `Account` SQLAlchemy model and `AccountCreate`/`AccountResponse` Pydantic schemas
    - _Requirements: 3.1, 3.2_
  - [ ] 4.2 Implement `POST /accounts`, `GET /accounts`, `GET /accounts/{id}`, `PUT /accounts/{id}` route handlers
    - `GET /accounts/{id}` must return account + all associated contacts
    - Guard `POST`/`PUT` with `require_role(sales_executive, sales_manager, sales_head, admin)`
    - Update `updated_at` on every write
    - _Requirements: 3.1, 3.2, 3.3_
  - [ ] 4.3 Create `Contact` SQLAlchemy model and `ContactCreate`/`ContactResponse` schemas
    - Validate `role_in_decision` accepts only: `Decision Maker`, `Influencer`, `User`, `Gatekeeper`
    - _Requirements: 3.6, 3.7_
  - [ ] 4.4 Implement `POST /accounts/{id}/contacts`, `PATCH /contacts/{id}` route handlers
    - Cascade delete of contacts when account is deleted (enforced at DB level)
    - _Requirements: 3.4, 3.5_
  - [ ]* 4.5 Write unit tests for account and contact validation
    - Test invalid `role_in_decision` value returns 422; valid values succeed
    - Test cascade delete behavior
    - _Requirements: 3.5, 3.7_

- [ ] 5. Implement Lead CRUD and status lifecycle
  - [ ] 5.1 Create `Lead` SQLAlchemy model and `LeadCreate`/`LeadUpdate`/`LeadResponse` schemas
    - Validate `estimated_value >= 0` at schema level; reject with HTTP 422 if negative
    - _Requirements: 4.1, 4.2, 4.3_
  - [ ] 5.2 Implement `POST /leads`, `GET /leads`, `GET /leads/{id}`, `PUT /leads/{id}`, `DELETE /leads/{id}` handlers
    - `POST /leads` sets initial status to `new`, inserts audit log entry on creation
    - `DELETE /leads/{id}` is soft-delete, admin only; cascade soft-delete child entities
    - Apply RBAC data scoping: `sales_executive` sees only owned leads, `sales_manager` sees team leads
    - _Requirements: 4.1, 4.4, 4.5, 4.6, 2.4, 2.5_
  - [ ] 5.3 Implement `PATCH /leads/{id}/status` handler with state machine validation
    - Implement `validate_status_transition` using `LEAD_STATUS_TRANSITIONS` dict from design
    - On valid transition: update status, set `updated_at`, insert audit log with `before_state`/`after_state`
    - Return HTTP 400 with allowed transitions on invalid request
    - Only lead owner or `sales_manager`/`sales_head`/`admin` may perform transition
    - _Requirements: 5.3, 5.4, 5.5, 5.7, 22.5_
  - [ ]* 5.4 Write property test for lead status transition completeness
    - **Property 2: Status transitions must be valid — every (current, new) pair either passes or raises HTTP 400, no unhandled exceptions**
    - **Validates: Requirements 5.2, 5.3, 5.4**
    - Use `@given(current=valid_statuses, new=valid_statuses)` with `hypothesis`
    - _Requirements: 5.2, 5.3_
  - [ ]* 5.5 Write unit tests for lead lifecycle
    - Test all valid transitions pass; all invalid transitions return 400
    - Test `estimated_value < 0` returns 422
    - Test audit log entry created on lead creation and status change
    - _Requirements: 4.3, 4.4, 5.3, 5.4, 5.5_

- [ ] 6. Implement RBAC data scoping and competitive intelligence
  - [ ] 6.1 Implement data-scoping helpers in `app/core/rbac.py`
    - `sales_executive`: leads filtered to `owner_id == auth.user_id`
    - `sales_manager`: leads filtered to team members' leads
    - `admin`/`sales_head`: no filter
    - _Requirements: 2.4, 2.5, 2.6_
  - [ ] 6.2 Implement competitors CRUD: `POST`, `GET`, `DELETE` under `/leads/{id}/competitors`
    - Store `competitor_name`, `strengths`, `weaknesses`, `our_differentiators`
    - Include competitor records in lead detail response
    - Cascade delete when parent lead is deleted
    - _Requirements: 16.1, 16.2, 16.3_
  - [ ]* 6.3 Write unit tests for RBAC scoping
    - Test `sales_executive` cannot see another executive's lead (403)
    - Test `sales_manager` sees only team leads
    - _Requirements: 2.4, 2.5_

- [ ] 7. Implement basic Dashboard API
  - [ ] 7.1 Implement `GET /dashboard/summary` returning lead count by status, count of overdue action items, total weighted pipeline value
    - Scope metrics to role: `sales_executive` → own data; `sales_manager` → team data; `admin`/`sales_head` → all
    - _Requirements: 18.1, 18.3_
  - [ ]* 7.2 Write unit tests for dashboard scoping
    - Test each role returns correctly scoped metrics
    - _Requirements: 18.1, 18.3_

- [ ] 8. Set up frontend project structure
  - [ ] 8.1 Initialize Vite + React + TypeScript project with `package.json` pinned dependencies
    - Include: `react@^18.3`, `typescript@^5.4`, `vite@^5.2`, `tailwindcss@^3.4`, `@tanstack/react-query@^5.40`, `react-hook-form@^7.51`, `zod@^3.23`, `axios@^1.7`, `react-router-dom@^6.23`, `date-fns@^3.6`
    - Configure Tailwind CSS, path aliases, and Vite PWA plugin stub
    - _Requirements: 20.1_
  - [ ] 8.2 Implement Cognito auth flow: login page, token storage, Axios interceptor for JWT injection, `/auth/me` call on app load
    - _Requirements: 1.1, 1.2_
  - [ ] 8.3 Implement React Router layout with protected routes (redirect to login if no token), role-aware navigation sidebar
    - _Requirements: 2.1, 2.2_

- [ ] 9. Implement Lead, Account, and Contact UI components
  - [ ] 9.1 Build `LeadListPage`: filterable/sortable table with cursor-based pagination (`after` + `limit`), filter by `status`, `owner_id`, `account_id`
    - _Requirements: 19.1, 19.2, 19.3_
  - [ ] 9.2 Build `LeadDetailPage`: display lead info, status badge, competitive intelligence list; include `LeadStatusTransitionModal` for status changes with duplicate-detection warning modal
    - _Requirements: 4.7, 4.8, 5.3, 5.4_
  - [ ] 9.3 Build `LeadCreateForm` and `LeadEditForm` using `react-hook-form` + `zod`
    - _Requirements: 4.1, 4.2, 4.3_
  - [ ] 9.4 Build `AccountListPage`, `AccountDetailPage` (with embedded contacts list), `AccountCreateForm`
    - _Requirements: 3.1, 3.2, 3.3_

- [ ] 10. Phase 1 checkpoint — Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

### Phase 2: Meeting Reports & Actions

- [ ] 11. Create Phase 2 database migrations
  - [ ] 11.1 Write Alembic migration for `meetings`, `meeting_attendees`, `meeting_reports` tables
    - _Requirements: 6.1, 6.3_
  - [ ] 11.2 Write Alembic migration for `action_items` table and indexes
    - Include `idx_action_items_owner`, `idx_action_items_due_date`, `idx_action_items_status`
    - Include partial index: `CREATE INDEX idx_open_actions ON action_items(owner_id, due_date) WHERE status != 'completed'`
    - _Requirements: 7.1, 7.5_
  - [ ] 11.3 Write Alembic migration for `documents` and `notes` tables
    - _Requirements: 9.4, 10.1_

- [ ] 12. Implement Meeting Report API
  - [ ] 12.1 Create `Meeting`, `MeetingAttendee`, `MeetingReport` SQLAlchemy models and Pydantic schemas
    - Include `MeetingType` enum, `AttendeeInput`, `MeetingReportCreate`, `MeetingReportResponse`
    - _Requirements: 6.3, 6.4, 6.5_
  - [ ] 12.2 Implement `POST /meetings` route handler with atomic transaction
    - Atomically INSERT into `meetings`, `meeting_attendees`, `meeting_reports`, `action_items`, `audit_logs` in a single DB transaction
    - Require at least 1 attendee; validate `defence_aerospace` requires `security_classification` + `department_directorate`
    - Rollback entire transaction and return HTTP 500 if any step fails
    - After commit, asynchronously enqueue embedding for `requirement_discussion + internal_notes`
    - Verify user has `defence_clearance` Cognito attribute when meeting type is `defence_aerospace`
    - _Requirements: 6.1, 6.2, 6.4, 6.6, 6.7, 6.8, 2.7, 21.6_
  - [ ] 12.3 Implement `GET /meetings`, `GET /meetings/{id}`, `PUT /meetings/{id}`, `GET /meetings/{id}/report`, `GET /leads/{id}/meetings` handlers
    - _Requirements: 6.1, 6.3_
  - [ ]* 12.4 Write unit tests for meeting report creation
    - Test `defence_aerospace` missing `security_classification` returns 422
    - Test `defence_aerospace` missing `department_directorate` returns 422
    - Test missing attendees returns 422
    - Test transaction rollback on DB failure returns 500
    - **Property 6: Every meeting has at least one attendee**
    - **Property 7: Defence meetings always have security_classification**
    - **Validates: Requirements 6.4, 6.6**
    - _Requirements: 6.2, 6.4, 6.6_

- [ ] 13. Implement Action Item API and overdue logic
  - [ ] 13.1 Create `ActionItem` SQLAlchemy model and `ActionItemCreate`/`ActionItemResponse` schemas
    - Compute `is_overdue` dynamically: `due_date < today AND status NOT IN (completed, cancelled)`
    - Validate `due_date` is not in the past at creation time; reject with HTTP 422 if it is
    - _Requirements: 7.1, 7.2, 7.3, 7.4_
  - [ ] 13.2 Implement `GET /action-items`, `POST /action-items`, `PATCH /action-items/{id}`, `GET /action-items/overdue` handlers
    - `PATCH` status to `completed` only permitted for item owner or `admin`
    - Filter list by `owner_id`, `lead_id`, `status` per query params
    - _Requirements: 7.8, 19.4_
  - [ ]* 13.3 Write property test for overdue action item invariant
    - **Property 3: No overdue flag for completed/cancelled items — `is_overdue` is never true when status is completed or cancelled**
    - **Validates: Requirements 7.3, 7.4**
    - Use `hypothesis` to generate action items with arbitrary due dates and statuses
    - _Requirements: 7.3, 7.4_
  - [ ]* 13.4 Write unit tests for action item edge cases
    - Test past due date at creation returns 422
    - Test `is_overdue=true` when `due_date < today` and status is `open`
    - Test `is_overdue=false` when status is `completed`, regardless of due date
    - _Requirements: 7.2, 7.3, 7.4_

- [ ] 14. Implement EventBridge daily overdue notification job
  - [ ] 14.1 Implement `POST /internal/check-overdue-actions` internal handler
    - Query `action_items WHERE due_date < NOW() AND status NOT IN ('completed', 'cancelled')`
    - For each overdue item: INSERT notification record, call SES `send_email` for email digest, call SNS `publish` for push
    - _Requirements: 7.5, 7.6, 7.7_
  - [ ] 14.2 Wire EventBridge rule targeting `POST /internal/check-overdue-actions` with daily cron expression
    - Configure in CDK/CloudFormation IaC
    - _Requirements: 7.5_
  - [ ]* 14.3 Write integration tests for overdue notification flow
    - Seed overdue action items, call handler, verify notifications inserted and mock SES/SNS called
    - _Requirements: 7.5, 7.6, 7.7_

- [ ] 15. Implement Document and Notes API
  - [ ] 15.1 Implement `POST /documents/upload-url` generating S3 pre-signed URL, `GET /documents` filtered by entity
    - Validate MIME type server-side (not from client header); reject disallowed types with HTTP 415
    - Reject files > 25 MB with HTTP 413
    - Store `entity_type`, `entity_id`, `file_name`, `s3_key`, `mime_type`, `uploaded_by`
    - Use pre-signed URL scoped to specific object key with limited expiry
    - _Requirements: 9.1, 9.2, 9.3, 9.4, 9.5, 21.3, 21.5_
  - [ ] 15.2 Implement `POST /notes`, `GET /notes` filtered by entity type and id
    - After note save, asynchronously enqueue embedding generation for the note content
    - _Requirements: 10.1, 10.2, 10.3_
  - [ ]* 15.3 Write unit tests for document upload validation
    - Test MIME type rejection returns 415
    - Test size > 25MB returns 413
    - _Requirements: 9.2, 9.3_

- [ ] 16. Implement PDF export for meeting reports
  - [ ] 16.1 Add PDF rendering library (e.g., `reportlab` or `weasyprint`) and implement `GET /meetings/{id}/export-pdf` handler
    - Render all meeting fields, attendees list, and action items into a downloadable PDF
    - _Requirements: 6.9_

- [ ] 17. Build Meeting Report and Action Item UI
  - [ ] 17.1 Build `MeetingCreateForm`: dynamic form switching between `commercial_industrial` and `defence_aerospace` templates; attendee management sub-form; embedded action items sub-form
    - _Requirements: 6.3, 6.4, 6.5, 6.6_
  - [ ] 17.2 Build `MeetingDetailPage`: display full report, attendees, action items, PDF export button
    - _Requirements: 6.9_
  - [ ] 17.3 Build `ActionItemListPage`: filterable by `owner_id`, `lead_id`, `status`; overdue badge; pagination
    - _Requirements: 7.3, 19.4_
  - [ ] 17.4 Build in-app `NotificationBell` component: fetch unread notifications for current user, mark-as-read action
    - _Requirements: 17.1, 17.3, 17.4_

- [ ] 18. Phase 2 checkpoint — Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

### Phase 3: Opportunity Pipeline

- [ ] 19. Create Phase 3 database migrations
  - [ ] 19.1 Write Alembic migration for `opportunities` table
    - Add `CHECK (probability BETWEEN 0 AND 100)` constraint
    - _Requirements: 8.1, 8.2, 8.3_

- [ ] 20. Implement Opportunity Pipeline API
  - [ ] 20.1 Create `Opportunity` SQLAlchemy model and `OpportunityCreate`/`OpportunityPatch`/`OpportunityResponse` schemas
    - Validate `probability` is in [0, 100]; reject with HTTP 422 if outside range
    - _Requirements: 8.1, 8.2, 8.3_
  - [ ] 20.2 Implement `POST /opportunities`, `GET /opportunities`, `GET /opportunities/{id}`, `PATCH /opportunities/{id}` handlers
    - On `CLOSED_WON`/`CLOSED_LOST` stage: update parent lead status to `won`/`lost` respectively and insert audit log
    - Apply RBAC: `sales_executive` sees own opportunities; `sales_manager` sees team opportunities
    - _Requirements: 5.6, 8.4, 8.6, 8.7_
  - [ ] 20.3 Implement `GET /opportunities/forecast` returning weighted pipeline per stage
    - Calculate `estimated_deal_value * probability / 100` per opportunity; aggregate by stage
    - Restrict to team for `sales_manager`; restrict to user for `sales_executive`
    - _Requirements: 8.5, 8.6, 8.7_
  - [ ]* 20.4 Write property test for opportunity probability constraint
    - **Property 4: Opportunity probability is always in [0, 100] — out-of-range values are always rejected**
    - **Validates: Requirements 8.2**
    - Use `@given(probability=st.integers())` with `hypothesis`
    - _Requirements: 8.2_
  - [ ]* 20.5 Write unit tests for pipeline forecast aggregation
    - Test weighted value calculation per stage
    - Test `sales_manager` scoping filters to team
    - Test `sales_executive` scoping filters to user
    - _Requirements: 8.5, 8.6, 8.7_

- [ ] 21. Build Opportunity Pipeline UI
  - [ ] 21.1 Build `PipelineKanbanPage` with drag-and-drop columns per stage using `@dnd-kit/core`
    - On card drop: call `PATCH /opportunities/{id}` with new stage
    - _Requirements: 8.3_
  - [ ] 21.2 Build `OpportunityDetailPage`: probability slider, deal value input, qualification checklist fields
    - _Requirements: 8.1_
  - [ ] 21.3 Build `ForecastChartPage` using `recharts` bar/funnel chart showing weighted deal value per stage
    - _Requirements: 18.2_

- [ ] 22. Phase 3 checkpoint — Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

### Phase 4: AI / pgvector Intelligence

- [ ] 23. Set up pgvector and embedding infrastructure
  - [ ] 23.1 Write Alembic migration enabling `pgvector` extension and creating `notes_embeddings` table
    - Add HNSW index: `USING hnsw (embedding vector_cosine_ops) WITH (m=16, ef_construction=64)`
    - Add `idx_embeddings_entity` index on `(entity_type, entity_id)`
    - _Requirements: 11.2, 11.7_
  - [ ] 23.2 Implement `EmbedClient` service in `app/services/embed_client.py`
    - Support both OpenAI `text-embedding-3-small` and AWS Bedrock Titan via config toggle
    - Implement exponential backoff retry (3 attempts) on failure; log ERROR to CloudWatch on all retries exhausted
    - Never raise exception to caller on permanent failure; emit `embedding_failure_count` CloudWatch metric
    - _Requirements: 11.1, 11.4, 11.5, 22.3_
  - [ ] 23.3 Implement `generate_and_store_embedding` service function
    - Upsert logic: if `(entity_type, entity_id)` embedding already exists, update rather than insert duplicate
    - Store `entity_type`, `entity_id`, `content`, `embedding` (1536-dim), `model_version`
    - _Requirements: 11.2, 11.3, 11.7_
  - [ ]* 23.4 Write property test for embedding dimension invariant
    - **Property 5: Every embedding has exactly 1536 dimensions**
    - **Validates: Requirements 11.7**
    - Mock embed client to return vectors of arbitrary length; verify only 1536-dim vectors are stored
    - _Requirements: 11.7_
  - [ ]* 23.5 Write unit tests for embedding retry logic
    - Test 3 retries with exponential backoff on API failure
    - Test permanent failure logs ERROR and does not raise
    - _Requirements: 11.4, 11.5_

- [ ] 24. Implement Semantic Search and Similar Leads APIs
  - [ ] 24.1 Implement `POST /ai/semantic-search` handler
    - Embed query string, run pgvector `<=>` cosine distance query with `entity_type` filter
    - Validate `top_k` in [1, 100] (HTTP 400 if not); reject empty/whitespace query (HTTP 400)
    - Return `entity_type`, `entity_id`, `content_snippet` (max 300 chars), `similarity_score` (4 decimal places)
    - On `EmbedClient` unavailability, degrade gracefully (keyword fallback or empty list, no 5xx)
    - _Requirements: 12.1, 12.2, 12.3, 12.4, 12.5, 12.6, 12.7_
  - [ ] 24.2 Implement `GET /ai/similar-leads/{id}` returning top-5 similar leads by cosine similarity
    - _Requirements: 13.3_
  - [ ]* 24.3 Write unit tests for semantic search validation
    - Test empty query returns 400; whitespace-only query returns 400
    - Test `top_k=0` returns 400; `top_k=101` returns 400
    - Test entity_type filter restricts results
    - _Requirements: 12.3, 12.4, 12.6_

- [ ] 25. Implement Duplicate Lead Detection
  - [ ] 25.1 Integrate `check_duplicate_leads` into `POST /leads` handler
    - Embed new lead name, query `notes_embeddings` for leads with cosine similarity > 0.92
    - If duplicates found: return HTTP 409 with list of `{lead_id, lead_name, status, similarity_score}`
    - If no duplicates: proceed with normal lead creation
    - _Requirements: 4.7, 4.8, 13.1, 13.2, 13.4_
  - [ ]* 25.2 Write unit tests for duplicate detection
    - Test similarity > 0.92 returns 409 with duplicate list
    - Test similarity <= 0.92 proceeds to create lead (201)
    - _Requirements: 4.7, 4.8, 13.1, 13.4_

- [ ] 26. Implement EventBridge embedding retry job
  - [ ] 26.1 Implement `POST /internal/retry-missing-embeddings` handler
    - Query all `leads`, `meeting_reports`, `notes` that have no corresponding `notes_embeddings` row
    - Re-enqueue embedding generation for each missing entity
    - _Requirements: 11.6_
  - [ ] 26.2 Wire EventBridge rule for daily embedding retry cron job targeting the handler
    - _Requirements: 11.6_

- [ ] 27. Implement AI Summarization and Next-Action Recommendation APIs
  - [ ] 27.1 Implement `POST /ai/summarize-meeting/{id}` handler (role: `sales_manager`+)
    - Send `requirement_discussion` + `internal_notes` as context to LLM; return generated summary
    - Return HTTP 503 on LLM failure
    - _Requirements: 14.1, 14.3_
  - [ ] 27.2 Implement `GET /ai/recommend-next-action/{lead_id}` handler (role: `sales_executive`+)
    - Send lead's recent meeting notes and current status to LLM; return recommended action string
    - Return HTTP 503 on LLM failure
    - _Requirements: 14.2, 14.3_
  - [ ]* 27.3 Write unit tests for AI endpoints
    - Test LLM failure returns 503
    - Test role restriction: `viewer` role returns 403 on summarize endpoint
    - _Requirements: 14.1, 14.3_

- [ ] 28. Build AI Search and Insights UI
  - [ ] 28.1 Build `SemanticSearchPage`: debounced search input (500 ms), display `SearchResultCard` per result with entity type, snippet, similarity score
    - _Requirements: 12.1, 12.5_
  - [ ] 28.2 Add "Similar Leads" panel to `LeadDetailPage` showing top-5 similar leads with similarity scores
    - _Requirements: 13.3_
  - [ ] 28.3 Add "AI Summary" button to `MeetingDetailPage` and "Recommend Next Action" button to `LeadDetailPage`
    - _Requirements: 14.1, 14.2_

- [ ] 29. Phase 4 checkpoint — Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

### Phase 5: Mobile PWA Enhancement

- [ ] 30. Implement PWA manifest and service worker
  - [ ] 30.1 Create `public/manifest.json` with `name`, `short_name`, `icons`, `start_url`, `display: standalone`, `theme_color`
    - _Requirements: 20.1_
  - [ ] 30.2 Configure Vite PWA plugin (`vite-plugin-pwa`) to generate service worker with Workbox
    - Cache static assets and API GET responses for offline access
    - _Requirements: 20.2_

- [ ] 31. Implement offline draft queue with IndexedDB
  - [ ] 31.1 Implement `offlineDraftStore` in `src/lib/offlineDrafts.ts` using `idb` or `dexie` library
    - Store incomplete meeting report form state to IndexedDB on every field change
    - _Requirements: 20.2_
  - [ ] 31.2 Implement sync-on-reconnect: listen to `navigator.onLine` / `online` event, iterate IndexedDB drafts, POST each to `/meetings`, remove on success
    - Show sync status toast to user
    - _Requirements: 20.2_

- [ ] 32. Implement push notification support
  - [ ] 32.1 Implement service worker `push` event handler to display notification using `self.registration.showNotification`
    - Display notification even when the PWA is in background
    - _Requirements: 20.3, 20.4_
  - [ ] 32.2 Implement `POST /internal/send-push-notification` backend handler using AWS SNS `publish` for mobile push
    - Called by the overdue action item job to trigger push notifications
    - _Requirements: 20.3, 7.7_

- [ ] 33. Phase 5 checkpoint — Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

### Phase 6: Production Hardening

- [ ] 34. Implement audit log completeness and admin query API
  - [ ] 34.1 Implement `GET /audit-logs` handler (admin only) accepting `entity_type` + `entity_id` filters
    - Return all audit entries for requested entity; table is read-only (no update/delete mutations)
    - _Requirements: 15.1, 15.4, 15.5_
  - [ ]* 34.2 Write property test for audit log completeness
    - **Property 8: Audit log is complete — every create, update, delete, status_change mutation on Lead, Opportunity, Meeting_Report, Action_Item, Account, Contact produces an audit log entry**
    - **Validates: Requirements 15.1, 15.3**
    - Generate random sequences of mutations and verify each has a corresponding audit log row
    - _Requirements: 15.1, 15.3_
  - [ ]* 34.3 Write unit tests for audit log immutability
    - Test no UPDATE or DELETE SQL is issued against `audit_logs` table
    - Test `before_state` / `after_state` captured correctly on status change
    - _Requirements: 15.2, 15.3, 15.4_

- [ ] 35. Implement pagination on all list endpoints
  - [ ] 35.1 Add cursor-based pagination (`after=<uuid>`, `limit`) to all list endpoints: `/leads`, `/accounts`, `/meetings`, `/action-items`, `/opportunities`, `/notes`, `/documents`
    - Default `limit=50` when not supplied
    - _Requirements: 19.1, 19.2_
  - [ ]* 35.2 Write integration tests for pagination
    - Test `limit` defaults to 50; test `after` cursor returns correct page
    - _Requirements: 19.1, 19.2_

- [ ] 36. Implement security hardening
  - [ ] 36.1 Audit all SQLAlchemy queries; ensure all are parameterized (no string interpolation)
    - _Requirements: 21.4_
  - [ ] 36.2 Add global exception handler middleware returning HTTP 500 with generic message + full stack trace logged to CloudWatch at ERROR level
    - _Requirements: 22.2_
  - [ ] 36.3 Verify TLS enforcement: ALB listener redirects HTTP → HTTPS; RDS uses `sslmode=require`; S3 bucket policy denies non-HTTPS requests
    - _Requirements: 21.1_
  - [ ]* 36.4 Write integration tests for security controls
    - Test requests without JWT return 401; requests with wrong role return 403
    - Test S3 pre-signed URL is scoped to a specific key
    - _Requirements: 21.5, 22.6_

- [ ] 37. Implement CloudWatch observability
  - [ ] 37.1 Configure CloudWatch alarms in IaC: HTTP 5xx rate alarm; ECS CPU > 80% alarm
    - _Requirements: 22.4_
  - [ ] 37.2 Add `embedding_failure_count` CloudWatch custom metric emission in `EmbedClient` on permanent failure after 3 retries
    - _Requirements: 22.3_
  - [ ] 37.3 Verify structured request logging middleware emits all required fields: `request_id`, `user_id`, `endpoint`, `status_code`, `duration_ms`
    - _Requirements: 22.1_

- [ ] 38. Performance optimizations
  - [ ] 38.1 Add partial index for open action items: `CREATE INDEX idx_open_actions ON action_items(owner_id, due_date) WHERE status != 'completed'`
    - (Ensure Alembic migration includes this if not already present from Phase 2)
    - _Requirements: 19.1_
  - [ ] 38.2 Configure TanStack Query stale times in frontend: lead lists 60 s, account lists 60 s, opportunity pipeline 60 s
    - _Requirements: 19.1_
  - [ ] 38.3 Verify asyncpg connection pool is configured with min 10 / max 20 connections per ECS task in `app/db/session.py`
    - _Requirements: 22.1_

- [ ] 39. Final integration tests for lead validity invariant
  - [ ]* 39.1 Write property test for lead status validity
    - **Property 1: Every lead must always have a valid status — no lead row in the DB can have a status value outside the defined LeadStatus enum**
    - **Validates: Requirements 5.1, 5.2**
    - Use `hypothesis` to attempt to write arbitrary status strings and verify only valid enum values are accepted
    - _Requirements: 5.1, 5.2_

- [ ] 40. Final checkpoint — Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP delivery
- Each task references specific requirements from `requirements.md` for full traceability
- The design document uses Python (FastAPI backend) and TypeScript (React frontend) — no language selection needed
- Property-based tests use the `hypothesis` library (Python); 8 correctness properties from the design are covered across tasks 5.4, 13.3, 20.4, 23.4, 34.2, and 39.1
- All secrets must be loaded from AWS Secrets Manager — never environment variables or source code
- Defence aerospace meeting reports require `defence_clearance` Cognito attribute verification (Req 2.7, 21.6)
- Checkpoints at the end of each phase ensure incremental validation before the next phase begins

### Phase 7: Outlook Email Integration

- [ ] 41. Database migration for Outlook integration tables
  - [ ] 41.1 Write Alembic migration for `outlook_connections` table with all columns per design schema
    - _Requirements: 23, 24, 27_
  - [ ] 41.2 Write Alembic migration for `emails` table with `UNIQUE(lead_id, outlook_message_id)` constraint and indexes `idx_emails_lead_id`, `idx_emails_sent_at`
    - _Requirements: 23, 24, 27_
  - [ ] 41.3 Write Alembic migration for `email_action_item_suggestions` table with partial index `idx_suggestions_lead_pending WHERE status='pending'`
    - _Requirements: 23, 24, 27_

- [ ] 42. Implement Outlook OAuth2 connection flow
  - [ ] 42.1 Add `msal==1.28.*` and `cryptography==42.0.*` to `pyproject.toml`
    - _Requirements: 23.1_
  - [ ] 42.2 Implement `GET /outlook/connect` — generate Microsoft OAuth2 authorization URL with `Mail.ReadWrite` and `offline_access` scopes, include CSRF state parameter
    - _Requirements: 23.1, 23.2_
  - [ ] 42.3 Implement `GET /outlook/callback` — exchange authorization code for tokens, store encrypted in AWS Secrets Manager at key `outlook/{user_id}`, register Graph webhook subscription, update `outlook_connections` record to `connected`
    - _Requirements: 23.2, 23.3, 23.4_
  - [ ] 42.4 Implement `store_outlook_tokens()` and `get_outlook_tokens()` service functions using Secrets Manager — tokens NEVER written to DB columns, logs, or API responses
    - _Requirements: 23.3, 23.6_
  - [ ] 42.5 Implement proactive token refresh: if access token expires within 5 minutes, call MSAL refresh silently
    - _Requirements: 23.4_

- [ ] 43. Implement webhook receiver and email sync
  - [ ] 43.1 Implement `POST /webhooks/outlook` handler — validate `clientState` HMAC secret, acknowledge Graph with 202 immediately, enqueue background task for email fetch
    - _Requirements: 24.1, 24.2, 25.6_
  - [ ] 43.2 Implement `fetch_and_sync_email(message_id, user_id)` background task — call Graph API `GET /messages/{id}`, extract subject/sender/recipients/body, call `match_email_to_leads()`, insert into `emails` table with encrypted `body_full`, idempotent on duplicate `outlook_message_id`
    - _Requirements: 24.3, 24.4, 24.5, 24.7_
  - [ ] 43.3 Implement `match_email_to_leads()` service function per design algorithm — match email addresses against `contacts.email`, return list of `lead_ids`, handle multi-lead fan-out
    - _Requirements: 24.3, 24.4_
  - [ ] 43.4 After email insert, asynchronously enqueue embedding generation for `body_preview` content (`entity_type='email'`)
    - _Requirements: 25.2_

- [ ] 44. Implement webhook subscription management
  - [ ] 44.1 Implement `register_webhook_subscription(user_id)` — POST to Graph `/subscriptions` for inbox and sentItems folders, store `subscription_id` and expiry in `outlook_connections`
    - _Requirements: 25.1, 25.3_
  - [ ] 44.2 Implement `POST /internal/renew-outlook-subscriptions` handler — query connections where `subscription_expiry < NOW() + 24h`, renew each with 3-retry exponential backoff per design algorithm, set `sync_error` on permanent failure
    - _Requirements: 25.4, 25.5_
  - [ ] 44.3 Wire EventBridge rule to trigger renewal endpoint every 24 hours
    - _Requirements: 25.3_

- [ ] 45. Implement historical email backfill
  - [ ] 45.1 Implement `POST /internal/backfill-outlook/{user_id}` handler — set status to `backfill_in_progress`, paginate through Graph API `/messages` for last 90 days, apply contact-matching and insert logic, set status to `connected` on completion
    - _Requirements: 26.1, 26.2, 26.3, 26.4_
  - [ ] 45.2 Trigger backfill job automatically from OAuth callback on first connect
    - _Requirements: 26.5_
  - [ ] 45.3 Implement idempotent insertion — skip emails where `(lead_id, outlook_message_id)` already exists
    - _Requirements: 26.6_

- [ ] 46. Implement AI action item extraction from emails
  - [ ] 46.1 Implement `extract_action_items_from_email(email_id, body, lead_id)` async task per design algorithm — call LLM with prompt, parse JSON array response, insert `email_action_item_suggestions` with `status=pending`
    - _Requirements: 27.1, 27.2_
  - [ ] 46.2 Implement `POST /email-suggestions/{id}/accept` — create linked `Action_Item`, set suggestion status to `accepted`, store `accepted_action_item_id`
    - _Requirements: 27.4, 27.5_
  - [ ] 46.3 Implement `POST /email-suggestions/{id}/dismiss` — set suggestion status to `dismissed`, no `Action_Item` created
    - _Requirements: 27.6, 27.7_
  - [ ] 46.4 Ensure LLM failure logs WARN and does not block or delay email sync
    - _Requirements: 27.7_

- [ ] 47. Implement email list and detail APIs
  - [ ] 47.1 Implement `GET /leads/{id}/emails` — paginated list of emails for a lead, return `id`, `subject`, `sender_email`, `recipient_emails`, `sent_at`, `body_preview` (NOT `body_full`)
    - _Requirements: 24.6, 28.1_
  - [ ] 47.2 Implement `GET /emails/{id}` — decrypt and return `body_full`, enforce that requesting user is the `synced_by_user_id` or admin
    - _Requirements: 28.2, 28.3_
  - [ ] 47.3 Implement `GET /leads/{id}/email-suggestions` — return all pending suggestions for the lead
    - _Requirements: 27.3_
  - [ ] 47.4 Implement `GET /outlook/status` — return connection status and `last_synced_at` for the requesting user
    - _Requirements: 28.4_
  - [ ] 47.5 Implement `DELETE /outlook/disconnect` — delete SM tokens, cancel Graph subscription, set status to `disconnected`, retain all email records
    - _Requirements: 28.5, 28.6_
  - [ ] 47.6 Implement `GET /admin/outlook-connections` (admin only) — return `user_id`, `email`, `status`, `connected_at`, `last_synced_at` for all users (no email body content)
    - _Requirements: 28.7, 28.8_

- [ ] 48. Build Emails tab on Lead Detail page (Frontend)
  - [ ] 48.1 Add "Emails" tab to `LeadDetailPage` — fetch emails via `GET /leads/{id}/emails` with TanStack Query, display timeline of `EmailCard` components
    - _Requirements: 24.6, 28.4_
  - [ ] 48.2 Build `EmailCard` component — show subject, sender, recipients, `sent_at`, `body_preview`, expandable "View Full Email" section (fetches `GET /emails/{id}` on expand)
    - _Requirements: 24.6, 28.1, 28.2_
  - [ ] 48.3 Build AI Suggestions panel — display pending suggestions from `GET /leads/{id}/email-suggestions`, show "Add to Lead" and "Dismiss" buttons
    - _Requirements: 27.3_
  - [ ] 48.4 Show Outlook sync status badge in Emails tab header ("Connected — last synced X min ago" / "Disconnected" / "Sync Error")
    - _Requirements: 28.4, 28.5_

- [ ] 49. Build Outlook settings in User Profile page (Frontend)
  - [ ] 49.1 Add "Integrations" tab to User Profile page
    - _Requirements: 23.7_
  - [ ] 49.2 Build `OutlookConnectionCard` component — show connected state with `last_synced_at`, disconnected state with "Connect Outlook Account" button that calls `GET /outlook/connect` and redirects
    - _Requirements: 23.7, 28.1, 28.4_
  - [ ] 49.3 Implement disconnect confirmation modal — warn that sync will stop but existing emails remain
    - _Requirements: 28.5_

- [ ] 50. Property-based and integration tests for Outlook integration
  - [ ]* 50.1 Write property test — every inserted email record has a non-null `lead_id`
    - _Requirements: 24.3_
  - [ ]* 50.2 Write property test — `match_email_to_leads` never returns duplicate `lead_ids` for the same email
    - _Requirements: 24.7_
  - [ ]* 50.3 Write integration test — full OAuth callback flow: mock MS token endpoint, verify tokens in Secrets Manager, verify `outlook_connections` record updated
    - _Requirements: 23.2, 23.6_
  - [ ]* 50.4 Write integration test — webhook receiver: valid `clientState` → email inserted; invalid `clientState` → 400 returned, no insert
    - _Requirements: 25.6_
  - [ ]* 50.5 Write unit test — `extract_action_items_from_email`: LLM failure logs WARN and does not raise exception
    - _Requirements: 27.6_
  - [ ]* 50.6 Write unit test — accept suggestion creates `Action_Item`; dismiss does not
    - _Requirements: 27.6_

- [ ] 51. Phase 7 checkpoint — Ensure all tests pass
  - Ensure all Outlook integration tests pass, ask user if questions arise.

### Phase 8: Product / Solution Catalogue

- [ ] 52. Database migrations for Product Catalogue
  - [ ] 52.1 Write Alembic migration for `product_categories` table
  - [ ] 52.2 Write Alembic migration for `products` table with unique constraint on `code`, indexes idx_products_category, idx_products_active, idx_products_code
  - [ ] 52.3 Write Alembic migration for `product_leads` table with UNIQUE(lead_id, product_id)
  - [ ] 52.4 Write Alembic migration for `product_opportunities` table with GENERATED ALWAYS AS line_value column, CHECK constraints, UNIQUE(opportunity_id, product_id)
  - _Requirements: 29.1, 30.1, 31.1_

- [ ] 53. Implement Product and Category CRUD APIs
  - [ ] 53.1 Create `Product`, `ProductCategory` SQLAlchemy models and Pydantic schemas (`ProductCreate`, `ProductResponse`, `ProductCategoryCreate`)
    - `ProductResponse` includes computed `is_admin` field based on requesting user role
    - _Requirements: 29.1, 29.5, 29.6_
  - [ ] 53.2 Implement `POST /product-categories`, `GET /product-categories` handlers (admin create, all read)
    - _Requirements: 29.5_
  - [ ] 53.3 Implement `POST /products`, `GET /products`, `GET /products/{id}`, `PUT /products/{id}`, `PATCH /products/{id}/deactivate` handlers
    - `POST`/`PUT`/`PATCH deactivate`: admin only (HTTP 403 otherwise)
    - `GET /products`: default filter `is_active=true`; support `?category_id=`, `?q=` keyword search (name/code/description, case-insensitive)
    - `PATCH /products/{id}/deactivate`: set `is_active=false`, do NOT delete any associations
    - Return HTTP 409 on duplicate `code`
    - _Requirements: 29.1, 29.2, 29.3, 29.4, 29.6, 29.7, 33.1, 33.2, 33.3, 33.4, 33.5, 33.6_
  - [ ]* 53.4 Write unit tests for product catalogue management
    - Test duplicate `code` returns 409
    - Test deactivation does not delete existing Product_Lead or Product_Opportunity records
    - Test non-admin create returns 403
    - Test keyword search is case-insensitive and matches name, code, description
    - _Requirements: 29.2, 29.4, 29.7, 33.3_

- [ ] 54. Implement Product-Lead association APIs
  - [ ] 54.1 Implement `POST /leads/{id}/products` — create Product_Lead; reject if `product.is_active == false` (HTTP 422)
    - _Requirements: 30.1, 30.2_
  - [ ] 54.2 Implement `DELETE /leads/{id}/products/{product_id}` — delete Product_Lead record; return HTTP 204
    - _Requirements: 30.4_
  - [ ] 54.3 Extend `GET /leads/{id}` response to include associated products list (show all, regardless of is_active)
    - _Requirements: 30.3_
  - [ ]* 54.4 Write unit tests for product-lead association
    - Test inactive product association returns 422
    - Test lead detail includes products regardless of is_active
    - Test cascade delete on lead deletion
    - _Requirements: 30.2, 30.3, 30.6_

- [ ] 55. Implement Product-Opportunity line item APIs
  - [ ] 55.1 Create `ProductOpportunity` SQLAlchemy model and `ProductOpportunityCreate`/`ProductOpportunityResponse` schemas
    - Server-computes `line_value`; never accept client-supplied `line_value`
    - _Requirements: 31.1, 31.2_
  - [ ] 55.2 Implement `POST /opportunities/{id}/line-items`, `PUT /opportunities/{id}/line-items/{item_id}`, `DELETE /opportunities/{id}/line-items/{item_id}` handlers
    - Reject inactive product (HTTP 422), quantity < 1 (HTTP 422), unit_price < 0 (HTTP 422)
    - _Requirements: 31.3, 31.4_
  - [ ] 55.3 Extend `GET /opportunities/{id}` response to include `total_line_value` and `line_items` list
    - `total_line_value` is null when no line items exist; equals sum of line_value when items exist
    - _Requirements: 31.5, 31.6_
  - [ ]* 55.4 Write property test for line_value correctness
    - **Property: line_value always equals quantity × unit_price for every Product_Opportunity record**
    - Use hypothesis to generate arbitrary quantity/unit_price pairs and verify computed line_value
    - _Requirements: 31.2_
  - [ ]* 55.5 Write unit tests for line item validation
    - Test quantity=0 returns 422; quantity=1 succeeds
    - Test unit_price < 0 returns 422; unit_price=0 succeeds
    - Test total_line_value correctly sums all line items
    - Test null total_line_value when no line items
    - _Requirements: 31.2, 31.4, 31.5, 31.6_

- [ ] 56. Implement Pipeline by Product report API
  - [ ] 56.1 Implement `GET /pipeline/by-product` handler
    - Execute aggregation query per design algorithm grouping by product and category
    - Apply RBAC scoping: executive → own opps; manager → team opps; admin/head → all
    - Return `product_rows` and `category_summary`
    - HTTP 403 for viewer role
    - _Requirements: 32.1, 32.2, 32.3, 32.4, 32.5, 32.6, 32.7_
  - [ ]* 56.2 Write unit tests for pipeline by product
    - Test product rows weighted values are computed correctly per product
    - Test executive scope restriction
    - Test manager scope restriction
    - Test viewer role returns 403
    - Test empty result when no opportunities in scope
    - _Requirements: 32.1, 32.2, 32.3, 32.6, 32.7_

- [ ] 57. Build Product Catalogue UI (Frontend)
  - [ ] 57.1 Add "Catalogue" nav item to sidebar; build `CataloguePage` with product cards grouped by category, keyword search input, category filter dropdown
    - _Requirements: 33.1, 33.2, 33.3_
  - [ ] 57.2 Build `ProductCard` component: show code, name, category badge, base_price, description; show Edit/Deactivate buttons only when `is_admin=true` in response
    - _Requirements: 33.4, 33.5_
  - [ ] 57.3 Build `ProductCreateForm` modal (admin only): fields for code, name, description, category, base_price, unit; submit to `POST /products`
    - _Requirements: 29.1_
  - [ ] 57.4 Add product tags panel to `LeadDetailPage` Overview tab: show associated products as chips, "+ Add Product" button opens searchable dropdown of active products, × removes association
    - _Requirements: 30.1, 30.3, 30.4_
  - [ ] 57.5 Add Line Items table to `OpportunityDetailPage`: show product, qty, unit_price, line_value columns; total row; "+ Add Product" inline form; edit/delete per row
    - _Requirements: 31.1, 31.5, 31.6_
  - [ ] 57.6 Add "By Product" view toggle to Pipeline/Forecast page: table showing product rows with opportunity_count, total_estimated_value, total_weighted_value; category grouping
    - _Requirements: 32.1, 32.4_

- [ ] 58. Phase 8 checkpoint — Ensure all tests pass
  - Ensure all product catalogue tests pass, ask user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP delivery
- Each task references specific requirements from `requirements.md` for full traceability
- The design document uses Python (FastAPI backend) and TypeScript (React frontend) — no language selection needed
- Property-based tests use the `hypothesis` library (Python); 8 correctness properties from the design are covered across tasks 5.4, 13.3, 20.4, 23.4, 34.2, and 39.1
- All secrets must be loaded from AWS Secrets Manager — never environment variables or source code
- Defence aerospace meeting reports require `defence_clearance` Cognito attribute verification (Req 2.7, 21.6)
- Checkpoints at the end of each phase ensure incremental validation before the next phase begins
- Phase 7 adds Outlook Email Integration: OAuth2 via MSAL, Graph API webhooks, encrypted email storage, AI action item extraction, and frontend Emails tab/settings
- Phase 8 adds Product/Solution Catalogue: product categories, lead/opportunity associations, line-item deal value, pipeline-by-product reporting
- Phase 9 adds Auto-Tag Meeting Notes, Forecasting Trends, Sales Velocity Dashboard, Win/Loss Review Workflow, Account Health Score, and Offline-First Meeting Reports

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["2.1", "2.2", "2.3", "2.4", "2.5", "8.1"] },
    { "id": 1, "tasks": ["3.1", "3.2", "3.3"] },
    { "id": 2, "tasks": ["3.4", "4.1", "4.3"] },
    { "id": 3, "tasks": ["4.2", "4.4", "5.1"] },
    { "id": 4, "tasks": ["4.5", "5.2", "5.3", "8.2"] },
    { "id": 5, "tasks": ["5.4", "5.5", "6.1", "8.3"] },
    { "id": 6, "tasks": ["6.2", "6.3", "7.1", "9.1", "9.2", "9.3", "9.4"] },
    { "id": 7, "tasks": ["7.2", "11.1", "11.2", "11.3"] },
    { "id": 8, "tasks": ["12.1", "13.1"] },
    { "id": 9, "tasks": ["12.2", "12.3", "13.2", "15.1", "15.2"] },
    { "id": 10, "tasks": ["12.4", "13.3", "13.4", "14.1", "14.2", "15.3", "16.1"] },
    { "id": 11, "tasks": ["14.3", "17.1", "17.2", "17.3", "17.4"] },
    { "id": 12, "tasks": ["19.1"] },
    { "id": 13, "tasks": ["20.1", "20.2", "20.3"] },
    { "id": 14, "tasks": ["20.4", "20.5", "21.1", "21.2", "21.3"] },
    { "id": 15, "tasks": ["23.1"] },
    { "id": 16, "tasks": ["23.2", "23.3"] },
    { "id": 17, "tasks": ["23.4", "23.5", "24.1", "24.2", "25.1", "26.1", "26.2"] },
    { "id": 18, "tasks": ["24.3", "25.2", "27.1", "27.2"] },
    { "id": 19, "tasks": ["27.3", "28.1", "28.2", "28.3"] },
    { "id": 20, "tasks": ["30.1", "30.2"] },
    { "id": 21, "tasks": ["31.1", "32.1", "32.2"] },
    { "id": 22, "tasks": ["31.2"] },
    { "id": 23, "tasks": ["34.1", "35.1", "36.1", "36.2", "36.3", "37.1", "37.2", "37.3", "38.1", "38.2", "38.3"] },
    { "id": 24, "tasks": ["34.2", "34.3", "35.2", "36.4", "39.1"] },
    { "id": 25, "tasks": ["41.1", "41.2", "41.3"] },
    { "id": 26, "tasks": ["42.1"] },
    { "id": 27, "tasks": ["42.2", "42.3", "42.4"] },
    { "id": 28, "tasks": ["42.5", "43.1", "44.1"] },
    { "id": 29, "tasks": ["43.2", "43.3", "44.2", "44.3"] },
    { "id": 30, "tasks": ["43.4", "45.1", "46.1"] },
    { "id": 31, "tasks": ["45.2", "45.3", "46.2", "46.3", "46.4"] },
    { "id": 32, "tasks": ["47.1", "47.2", "47.3", "47.4", "47.5", "47.6"] },
    { "id": 33, "tasks": ["48.1", "48.2", "49.1", "49.2"] },
    { "id": 34, "tasks": ["48.3", "48.4", "49.3"] },
    { "id": 35, "tasks": ["50.1", "50.2", "50.3", "50.4", "50.5", "50.6"] },
    { "id": 36, "tasks": ["52.1", "52.2", "52.3", "52.4"] },
    { "id": 37, "tasks": ["53.1", "53.2"] },
    { "id": 38, "tasks": ["53.3", "54.1", "54.2"] },
    { "id": 39, "tasks": ["53.4", "54.3", "55.1"] },
    { "id": 40, "tasks": ["54.4", "55.2", "55.3", "56.1"] },
    { "id": 41, "tasks": ["55.4", "55.5", "56.2"] },
    { "id": 42, "tasks": ["57.1", "57.2", "57.3"] },
    { "id": 43, "tasks": ["57.4", "57.5", "57.6"] },
    { "id": 44, "tasks": ["58"] },
    { "id": 45, "tasks": ["59.1", "59.2", "59.3", "59.4", "59.5"] },
    { "id": 46, "tasks": ["60.1", "61.1", "62.1", "63.1"] },
    { "id": 47, "tasks": ["60.2", "60.3", "61.2", "61.3", "63.2", "64.1"] },
    { "id": 48, "tasks": ["60.4", "60.5", "60.6", "61.4", "63.3", "63.4", "64.2"] },
    { "id": 49, "tasks": ["60.7", "61.5", "62.2", "63.5", "63.6", "64.3", "64.4"] },
    { "id": 50, "tasks": ["64.5", "65.1"] },
    { "id": 51, "tasks": ["65.2", "65.3"] },
    { "id": 52, "tasks": ["65.4", "65.5", "65.6", "65.7"] },
    { "id": 53, "tasks": ["65.8", "66.1", "66.2", "66.3"] },
    { "id": 54, "tasks": ["66.4", "66.5", "66.6"] },
    { "id": 55, "tasks": ["67"] }
  ]
}
```

### Phase 9: Auto-Tagging, Forecasting Trends, Sales Velocity, Win/Loss Review, Account Health Score, Offline-First PWA

- [ ] 59. Database migrations for Phase 9 features
  - [ ] 59.1 Write Alembic migration for `meeting_tags` table with CHECK constraint on tag values and UNIQUE(meeting_report_id, tag) — partial index `idx_meeting_tags_tag`
    - _Requirements: 34.2, 34.3_
  - [ ] 59.2 Write Alembic migration for `forecast_snapshots` table with UNIQUE(month, stage) constraint
    - _Requirements: 35.1, 35.2_
  - [ ] 59.3 Write Alembic migration for `win_actuals` table
    - _Requirements: 35.3_
  - [ ] 59.4 Write Alembic migration for `win_loss_reviews` table with `win_loss_reason` ENUM type
    - _Requirements: 37.2, 37.3_
  - [ ] 59.5 Write Alembic migration for `account_health_scores` table with GENERATED ALWAYS AS `at_risk` column and CHECK constraints on all component scores
    - _Requirements: 38.1, 38.6, 38.7_

- [ ] 60. Implement AI Auto-Tagger for Meeting Reports
  - [ ] 60.1 Implement `tag_meeting_report(meeting_report_id, content, db, llm_client)` async service per design algorithm — call LLM, filter to TAG_TAXONOMY, skip manual-override tags, upsert via ON CONFLICT DO NOTHING
    - _Requirements: 34.1, 34.2, 34.3, 34.8, 34.10_
  - [ ] 60.2 Wire auto-tagger into `POST /meetings` handler: enqueue `tag_meeting_report` asynchronously after successful DB commit — meeting save must complete regardless of tag outcome
    - _Requirements: 34.1, 34.8_
  - [ ] 60.3 Implement `GET /meetings/{id}/tags` returning tag array with source field
    - _Requirements: 34.4_
  - [ ] 60.4 Implement `PUT /meetings/{id}/tags` for manual override — replace all tags, set source='manual'; manual tags are never overwritten by subsequent AI runs
    - _Requirements: 34.9, 34.10_
  - [ ] 60.5 Implement `GET /admin/tag-frequency` (admin only) — return each Tag_Taxonomy value and count of meeting_reports carrying it, ordered by count descending
    - _Requirements: 34.6_
  - [ ] 60.6 Extend `POST /ai/semantic-search` to accept optional `tags` filter — restrict results to meeting_reports with at least one matching meeting_tag
    - _Requirements: 34.5_
  - [ ]* 60.7 Write unit tests for auto-tagger
    - Test non-taxonomy tags are discarded
    - Test manual override tags are preserved after AI re-run
    - Test LLM failure logs WARN and does not raise
    - _Requirements: 34.3, 34.8, 34.10_

- [ ] 61. Implement Forecasting with Historical Trends
  - [ ] 61.1 Implement `POST /internal/capture-forecast-snapshot` handler — aggregate active opportunities by stage, insert Forecast_Snapshot rows per stage; UNIQUE constraint prevents duplicate month+stage rows (no updates)
    - _Requirements: 35.1, 35.2_
  - [ ] 61.2 Wire EventBridge monthly cron rule targeting `/internal/capture-forecast-snapshot` (last day of each month at 23:55)
    - _Requirements: 35.1_
  - [ ] 61.3 Extend `PATCH /leads/{id}/status` to insert a `win_actuals` row when status transitions to `won` — store `actual_value` from linked opportunity's `estimated_deal_value` and `close_month` as current YYYY-MM
    - _Requirements: 35.3_
  - [ ] 61.4 Implement `GET /forecast/trends` — return 12-month array per design algorithm with `total_weighted_value`, `total_actual_won_value`, `forecast_accuracy_pct` (null when denominator=0, capped at 100)
    - HTTP 403 for `sales_executive`
    - _Requirements: 35.4, 35.5, 35.6, 35.7, 35.8_
  - [ ]* 61.5 Write unit tests for forecast trends
    - Test snapshot immutability (no UPDATE/DELETE issued)
    - Test `forecast_accuracy_pct` is null when weighted_value=0
    - Test accuracy is capped at 100 when actual > forecast
    - Test executive role returns 403
    - _Requirements: 35.2, 35.6, 35.7, 35.8_

- [ ] 62. Implement Sales Velocity Dashboard
  - [ ] 62.1 Implement `GET /analytics/sales-velocity` handler — compute average stage dwell times from audit_logs, deal cycle time, dropout rates; apply RBAC scoping; identify bottleneck stage
    - Serve from 5-minute server-side cache; recompute on miss
    - _Requirements: 36.1, 36.2, 36.3, 36.4, 36.5, 36.6, 36.7_
  - [ ]* 62.2 Write unit tests for sales velocity
    - Test stage dwell days are always >= 0
    - Test dropout rates are always in [0.0, 1.0]
    - Test bottleneck is the stage with highest dwell
    - Test executive sees only own leads
    - _Requirements: 36.2, 36.4, 36.5, 36.6_

- [ ] 63. Implement Win/Loss Review Workflow
  - [ ] 63.1 Extend `PATCH /leads/{id}/status` response to include `requires_win_loss_review: true` when new status is `won` or `lost`
    - _Requirements: 37.1_
  - [ ] 63.2 Implement `POST /win-loss-reviews` — validate `primary_reason` is in enum, verify lead is in `won`/`lost` status (HTTP 422 otherwise), insert immutable review record
    - _Requirements: 37.2, 37.3, 37.4, 37.5_
  - [ ] 63.3 After review submission: asynchronously enqueue embedding for `key_decision_factor + lessons_learned` (entity_type='win_loss_review')
    - _Requirements: 37.6_
  - [ ] 63.4 Implement `GET /win-loss-reviews` with RBAC scoping: admin/sales_head → all; sales_manager → team; sales_executive → own
    - _Requirements: 37.8_
  - [ ] 63.5 Implement EventBridge daily job that checks for leads in `won`/`lost` status with no Win_Loss_Review after 7 days and sends reminder notification via existing notification mechanism
    - _Requirements: 37.7_
  - [ ]* 63.6 Write unit tests for win/loss review workflow
    - Test `primary_reason` outside enum returns 422
    - Test lead not in terminal status returns 422
    - Test review is immutable (no UPDATE/DELETE permitted)
    - Test embedding is enqueued after submission
    - _Requirements: 37.3, 37.4, 37.5, 37.6_

- [ ] 64. Implement Account Health Score
  - [ ] 64.1 Implement `compute_account_health_score(account_id, db, llm_client)` service per design algorithm — compute all four components, sum to total, set at_risk flag, upsert into `account_health_scores`
    - _Requirements: 38.1, 38.2, 38.3, 38.4, 38.5, 38.6, 38.7_
  - [ ] 64.2 Implement `POST /internal/compute-account-health` handler — iterate all active Accounts, call `compute_account_health_score` for each, upsert results; wire EventBridge daily cron rule
    - _Requirements: 38.8_
  - [ ] 64.3 Implement `GET /accounts/{id}/health` — return score, all four component values, `at_risk` flag, `computed_at` timestamp
    - _Requirements: 38.9_
  - [ ] 64.4 Extend `GET /dashboard/summary` to include at-risk accounts count for managers+; implement `GET /dashboard/at-risk-accounts` returning all Accounts with `at_risk=true` in scope
    - _Requirements: 38.10_
  - [ ]* 64.5 Write property tests for account health score
    - **Property: total_score = sum of four components, each in [0,25], total in [0,100]**
    - **Property: at_risk is always true iff total_score < 40**
    - _Requirements: 38.1, 38.7_

- [ ] 65. Build offline-first full meeting report PWA
  - [ ] 65.1 Replace existing `offlineDraftStore` (draft-only) with full `OfflineQueueStore` in `src/lib/offlineQueue.ts` using `dexie` — supports `upsert`, `listPending`, `markSynced`, `remove`, `count` per design interface
    - _Requirements: 39.1, 39.6_
  - [ ] 65.2 Modify `MeetingCreateForm` to persist entire multi-step form state (all 4 steps) to `OfflineQueueStore` on every field change with `status='draft'`
    - _Requirements: 39.1_
  - [ ] 65.3 Modify form submit handler: when `navigator.onLine === false`, set queue item to `status='pending'` and show offline confirmation toast instead of POSTing; when online, POST normally
    - _Requirements: 39.2_
  - [ ] 65.4 Implement `SyncService` in `src/lib/syncService.ts` — listen to `window.addEventListener('online')`, iterate `listPending()`, POST each to `/meetings`; on 201: `markSynced` + success toast; on 404: show `ConflictResolutionModal`; on other errors: leave pending
    - _Requirements: 39.3, 39.4_
  - [ ] 65.5 Build `PendingSyncBanner` component — show "X reports pending sync" banner when `Offline_Queue.count('pending') > 0`
    - _Requirements: 39.5_
  - [ ] 65.6 Build `ConflictResolutionModal` — when sync fails with 404 (lead not found), offer "Reassign to different lead" or "Discard report" options; discard calls `OfflineQueueStore.remove(id)`
    - _Requirements: 39.4_
  - [ ] 65.7 Implement edit-in-queue: when user edits a `pending` queue item, call `upsert` with updated payload (not create new); ensure no duplicate entries per meeting
    - _Requirements: 39.7_
  - [ ]* 65.8 Write unit tests for offline queue
    - Test `listPending()` never returns items with status='synced'
    - Test markSynced → item status becomes 'synced' not removed
    - Test upsert on existing id updates payload, does not duplicate
    - _Requirements: 39.8, 39.7_

- [ ] 66. Build Phase 9 UI components
  - [ ] 66.1 Add Meeting_Tag chips to `MeetingDetailPage` — coloured chips per tag, show edit icon for creator/admin that opens tag override modal
    - _Requirements: 34.4, 34.9_
  - [ ] 66.2 Build `ForecastTrendsPage` — line chart (recharts) showing `total_weighted_value` vs `total_actual_won_value` per month; accuracy % bar below; 12-month rolling window
    - _Requirements: 35.4, 35.5_
  - [ ] 66.3 Build `SalesVelocityPage` — bar chart of average stage dwell days, deal cycle time card, dropout rate table per stage, bottleneck stage highlight
    - _Requirements: 36.1, 36.3, 36.4, 36.5_
  - [ ] 66.4 Build `WinLossReviewModal` — triggered after lead status transitions to won/lost; fields for primary_reason (dropdown), key_decision_factor, competitor (optional dropdown), lessons_learned; submit to `POST /win-loss-reviews`
    - _Requirements: 37.1, 37.2_
  - [ ] 66.5 Add Account Health Score widget to `AccountDetailPage` — circular gauge 0–100, four component bars, `at_risk` banner
    - _Requirements: 38.9_
  - [ ] 66.6 Add "At-Risk Accounts" widget to Dashboard — card list of accounts with `at_risk=true`, showing score, account name, primary weakness component
    - _Requirements: 38.10_

- [ ] 67. Phase 9 checkpoint — Ensure all tests pass
  - Ensure all Phase 9 tests pass, ask user if questions arise.
