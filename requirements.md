# Requirements Document

## Introduction

The Sales Lead Management Platform is a centralized, full-stack application for capturing, tracking, qualifying, and progressing sales leads from first interaction to deal closure. It serves field sales executives, sales managers, pre-sales engineers, and administrators through a React/TypeScript web dashboard and a Progressive Web App (PWA) for mobile use.

The platform digitizes meeting report templates (Commercial/Industrial and Defence/Aerospace), enforces a structured lead lifecycle, tracks action items with owners and due dates, and layers AI-powered semantic search and recommendations via PostgreSQL with pgvector. The backend is FastAPI on AWS ECS Fargate, with RDS PostgreSQL, S3, Cognito for auth, and CloudWatch for observability.

---

## Glossary

- **API**: The FastAPI backend service running on AWS ECS Fargate
- **Lead**: A prospective sales opportunity linked to an account and owned by a sales user
- **Account**: A client or prospect organization tracked in the system
- **Contact**: An individual associated with an Account who participates in sales interactions
- **Opportunity**: A formal deal record linked to a Lead, tracking stage, probability, and estimated value
- **Meeting_Report**: A structured record of a customer meeting, capturing discussions, attendees, and action items
- **Action_Item**: A follow-up task from a meeting or lead, assigned to an owner with a due date
- **Lead_Status_Machine**: The defined set of valid lead statuses and their allowed transitions
- **Embedding_Service**: The external AI service (OpenAI or AWS Bedrock) that generates 1536-dimensional vector embeddings
- **Embedding**: A 1536-dimensional float vector stored in pgvector representing the semantic content of a note or report
- **Auth_Service**: AWS Cognito handling user authentication and JWT issuance
- **RBAC**: Role-Based Access Control enforced on every API endpoint
- **JWT**: JSON Web Token issued by AWS Cognito and validated on every request
- **pgvector**: PostgreSQL extension enabling cosine similarity search on Embedding columns
- **Audit_Log**: An immutable record of every create, update, delete, or status change mutation
- **Notification**: An in-app or email/push alert sent to a user regarding an overdue item, assignment, or reminder
- **PDF_Exporter**: The component responsible for rendering a Meeting_Report as a downloadable PDF
- **Pipeline_Forecast**: An aggregated view of weighted deal values across Opportunity stages
- **EventBridge_Scheduler**: The AWS EventBridge cron job that triggers daily overdue checks and embedding retries
- **Semantic_Search**: AI-powered search using cosine similarity between query embeddings and stored Embeddings
- **Duplicate_Detector**: The subsystem that compares new lead names against existing leads using cosine similarity
- **Outlook_Connector**: The Microsoft Graph API integration that syncs emails between Outlook and the platform
- **Email_Thread**: A chain of related emails between a sales user and a contact, stored against a Lead
- **Email_Action_Item**: An Action_Item automatically extracted from email content by AI analysis
- **Graph_API**: The Microsoft Graph API used to access Outlook mailbox data with delegated OAuth2 permissions
- **OAuth2_Token**: The Microsoft OAuth2 access/refresh token pair stored per user for Graph API access
- **Webhook_Subscription**: A Microsoft Graph change notification subscription that pushes new email events to the platform
- **Product**: A named offering in the company's catalogue (e.g., "Vision Inspection System", "Robotic Automation Line") that can be associated with Leads and Opportunities
- **Product_Category**: A grouping label for Products (e.g., "Vision Systems", "Robotics", "Software", "Services")
- **Product_Lead**: A join record linking a Product to a Lead, optionally capturing the relevant configuration or variant
- **Product_Opportunity**: A join record linking a Product to an Opportunity, capturing quantity, unit price, and line-item deal value
- **Pipeline_By_Product**: An aggregated pipeline view showing weighted deal value broken down by Product or Product_Category
- **Product_Catalogue**: The complete set of Products managed by admins and used by sales users to tag leads and opportunities
- **Meeting_Tag**: An AI-generated label attached to a Meeting_Report identifying a key theme (e.g., pricing_concern, poc_request, competitor_mention, technical_requirement, compliance_issue)
- **Tag_Taxonomy**: The predefined set of valid Meeting_Tag values the system recognises
- **Forecast_Snapshot**: A point-in-time record of the pipeline weighted value captured at the end of each calendar month, used for trend and accuracy analysis
- **Sales_Velocity**: A composite metric measuring how quickly leads move through the pipeline, including average stage dwell time, cycle time, and stage dropout rate
- **Win_Loss_Review**: A structured close-out record created when an Opportunity is marked won or lost, capturing reason, key decision factor, competitor chosen, and lessons learned
- **Account_Health_Score**: A computed numeric score (0–100) for an Account, derived from recency of meetings, open action items, AI-assessed sentiment, and pipeline value
- **Offline_Queue**: The IndexedDB store on the PWA client that holds complete meeting report form data for full offline submission, not just drafts

---

## Requirements

### Requirement 1: Authentication and Session Management

**User Story:** As a platform user, I want to authenticate securely using my corporate credentials, so that I can access the platform with appropriate permissions for my role.

#### Acceptance Criteria

1. WHEN a user submits valid Cognito credentials, THE Auth_Service SHALL issue a JWT containing the user's `sub`, `email`, `custom:role`, and `team_id` claims
2. WHEN an API request is received, THE API SHALL validate the JWT signature against the Cognito JWKS endpoint before processing the request
3. WHEN a JWT has expired, THE API SHALL reject the request with HTTP 401
4. WHEN a valid JWT is presented, THE API SHALL extract the `custom:role` claim and inject an `AuthContext` object into the route handler
5. WHEN a user requests a token refresh, THE Auth_Service SHALL issue a new JWT without requiring re-entry of credentials
6. WHEN a user logs out, THE API SHALL invalidate the user's session
7. THE API SHALL expose the `/auth/me` endpoint so that an authenticated user can retrieve their current profile

---

### Requirement 2: Role-Based Access Control

**User Story:** As a system administrator, I want access to be controlled by user roles, so that users can only perform actions appropriate to their position.

#### Acceptance Criteria

1. THE API SHALL enforce one of the following roles for every authenticated user: `admin`, `sales_head`, `sales_manager`, `sales_executive`, `pre_sales`, `delivery_team`, or `viewer`
2. WHEN a user's role is not in the set of allowed roles for an endpoint, THE API SHALL return HTTP 403 with a body indicating access is denied
3. THE API SHALL derive the user role exclusively from the JWT `custom:role` claim and never from the request body
4. WHILE a user has the `sales_executive` role, THE API SHALL restrict lead reads and writes to leads owned by that user
5. WHILE a user has the `sales_manager` role, THE API SHALL restrict lead reads to leads within that manager's team
6. WHILE a user has the `admin` role, THE API SHALL grant full read and write access to all entities
7. WHEN a `defence_aerospace` Meeting_Report is accessed, THE API SHALL verify the requesting user has the `defence_clearance` attribute in their Cognito profile before granting access

---

### Requirement 3: Account and Contact Management

**User Story:** As a sales executive, I want to manage customer accounts and their associated contacts, so that I can keep a complete record of the organizations and people I interact with.

#### Acceptance Criteria

1. WHEN a user with `sales_executive` role or higher submits a valid account payload, THE API SHALL create an Account record and return HTTP 201
2. WHEN a user submits an account update, THE API SHALL persist the changes and update the `updated_at` timestamp
3. WHEN a user requests account detail, THE API SHALL return the Account along with all associated Contacts
4. WHEN a user with `sales_executive` role or higher submits a valid contact payload for an existing Account, THE API SHALL create a Contact record linked to that Account
5. WHEN an Account is deleted, THE API SHALL cascade-delete all Contacts associated with that Account
6. THE Contact record SHALL store `full_name`, `designation`, `email`, `phone`, `role_in_decision`, and `is_primary` fields
7. WHEN a contact's `role_in_decision` is set, THE API SHALL accept only the values: `Decision Maker`, `Influencer`, `User`, or `Gatekeeper`

---

### Requirement 4: Lead Creation and Management

**User Story:** As a sales executive, I want to create and manage sales leads, so that I can track prospective opportunities from first contact through to deal closure.

#### Acceptance Criteria

1. WHEN a user with `sales_executive` role or higher submits a valid lead payload, THE API SHALL create a Lead record with status `new` and return HTTP 201
2. THE Lead record SHALL store `lead_name`, `account_id`, `lead_source`, `owner_id`, `priority`, `estimated_value`, and `expected_close_date`
3. WHEN `estimated_value` is provided in a lead payload, THE API SHALL reject values less than zero with HTTP 422
4. WHEN a lead is created, THE API SHALL insert an Audit_Log entry recording the creation action, the creator's user ID, and the initial state
5. WHEN a user updates a lead, THE API SHALL update the `updated_at` timestamp on the Lead record
6. WHEN an `admin` user requests deletion of a lead, THE API SHALL soft-delete the Lead and all associated child entities
7. WHEN a new lead is submitted, THE Duplicate_Detector SHALL compute the cosine similarity between the new lead name embedding and existing lead name embeddings
8. WHEN the cosine similarity between a new lead name and any existing lead name exceeds 0.92, THE API SHALL return HTTP 409 with a list of potential duplicate leads

---

### Requirement 5: Lead Status Lifecycle

**User Story:** As a sales manager, I want lead status transitions to follow a defined lifecycle, so that the pipeline reflects the true progression of every opportunity.

#### Acceptance Criteria

1. THE Lead_Status_Machine SHALL define the following valid statuses: `new`, `contacted`, `discovery`, `qualified`, `demo_poc`, `proposal`, `negotiation`, `won`, `lost`, `on_hold`
2. THE Lead_Status_Machine SHALL enforce the following allowed transitions:
   - `new` → `contacted`, `lost`
   - `contacted` → `discovery`, `lost`, `on_hold`
   - `discovery` → `qualified`, `lost`, `on_hold`
   - `qualified` → `demo_poc`, `proposal`, `lost`
   - `demo_poc` → `proposal`, `lost`
   - `proposal` → `negotiation`, `lost`
   - `negotiation` → `won`, `lost`, `on_hold`
   - `on_hold` → `negotiation`, `lost`
   - `won` → (terminal — no transitions allowed)
   - `lost` → (terminal — no transitions allowed)
3. WHEN a status transition request is received, THE API SHALL validate that the requested new status is in the allowed set for the current status
4. IF a requested status transition is not allowed, THEN THE API SHALL return HTTP 400 with a message listing the allowed target statuses from the current status
5. WHEN a lead status transition is completed, THE API SHALL insert an Audit_Log entry with `before_state` containing the old status and `after_state` containing the new status
6. WHEN a lead transitions to `won` or `lost`, THE API SHALL update the stage of the linked Opportunity to `closed_won` or `closed_lost` respectively
7. WHEN a lead status transition is requested, THE API SHALL only permit the requesting user to perform the change if they are the lead owner OR have `sales_manager`, `sales_head`, or `admin` role

---

### Requirement 6: Meeting Report Creation

**User Story:** As a sales executive, I want to create structured meeting reports after each customer interaction, so that meeting discussions, attendees, and follow-up actions are captured in a consistent format.

#### Acceptance Criteria

1. WHEN a user with `sales_executive` role or higher submits a valid meeting report payload, THE API SHALL atomically insert records into `meetings`, `meeting_attendees`, `meeting_reports`, `action_items`, and `audit_logs` tables within a single database transaction
2. IF the database transaction fails at any step, THEN THE API SHALL roll back all inserted rows and return HTTP 500
3. THE Meeting_Report record SHALL support two meeting types: `commercial_industrial` and `defence_aerospace`
4. WHEN the meeting type is `defence_aerospace`, THE API SHALL require `security_classification` and `department_directorate` fields and return HTTP 422 if either is absent
5. WHEN the meeting type is `commercial_industrial`, THE API SHALL accept `industry_vertical` and `plant_facility_location` as optional fields
6. WHEN a meeting report is saved, THE Meeting_Report record SHALL be associated with at least one attendee
7. WHEN a meeting report is saved, THE API SHALL asynchronously enqueue embedding generation for the concatenated `requirement_discussion` and `internal_notes` fields
8. WHEN a meeting report is saved, THE API SHALL link all Action_Items in the payload to both the meeting ID and the parent lead ID
9. WHEN a user requests PDF export of a meeting report, THE PDF_Exporter SHALL render all meeting fields, attendees, and action items into a downloadable PDF document

---

### Requirement 7: Action Item Tracking and Overdue Notifications

**User Story:** As a sales team member, I want action items to be tracked with owners and due dates, and to receive notifications when items become overdue, so that follow-ups are never missed.

#### Acceptance Criteria

1. WHEN an Action_Item is created, THE API SHALL store `action_item`, `owner_id`, `due_date`, `priority`, `lead_id`, and optionally `meeting_id`
2. WHEN an Action_Item due date is set, THE API SHALL reject due dates that fall in the past at the time of creation with HTTP 422
3. WHEN an Action_Item's `due_date` is earlier than the current date AND its status is not `completed` or `cancelled`, THE API SHALL compute `is_overdue` as `true` in the response
4. WHILE an Action_Item's status is `completed` or `cancelled`, THE API SHALL always return `is_overdue` as `false` regardless of the due date
5. WHEN the EventBridge_Scheduler triggers the daily overdue check, THE API SHALL query all Action_Items where `due_date < today` AND `status` is not `completed` or `cancelled`
6. WHEN overdue Action_Items are identified by the daily job, THE API SHALL insert a Notification record for each item's owner and send an email digest via AWS SES
7. WHEN overdue Action_Items are identified by the daily job, THE API SHALL send a push notification via AWS SNS to each item's owner
8. WHEN a user marks an Action_Item as completed, THE API SHALL permit only the item's owner or a user with `admin` role to perform the status change

---

### Requirement 8: Opportunity and Pipeline Management

**User Story:** As a sales manager, I want to track deals in a pipeline with stage, probability, and deal value, so that I can forecast revenue and prioritize opportunities.

#### Acceptance Criteria

1. WHEN a user with `sales_executive` role or higher creates an Opportunity, THE API SHALL store `opportunity_name`, `stage`, `probability`, `estimated_deal_value`, `budget_confirmed`, `decision_maker_met`, `poc_required`, `internal_approval_needed`, and `expected_order_timeline`
2. WHEN an Opportunity probability value is submitted, THE API SHALL reject any integer outside the inclusive range [0, 100] with HTTP 422
3. THE Opportunity record SHALL support the following stages: `discovery`, `qualified`, `demo_poc`, `proposal`, `negotiation`, `closed_won`, `closed_lost`
4. WHEN an Opportunity stage transitions to `closed_won` or `closed_lost`, THE API SHALL insert an Audit_Log entry recording the stage transition
5. WHEN a user with `sales_manager` role or higher requests the pipeline forecast, THE API SHALL return aggregated weighted deal values per stage, calculated as `estimated_deal_value * probability / 100`
6. WHEN a `sales_manager` requests the pipeline forecast, THE API SHALL restrict aggregation to Opportunities within that manager's team
7. WHEN a `sales_executive` requests the pipeline forecast, THE API SHALL restrict aggregation to Opportunities owned by that user

---

### Requirement 9: Document Management

**User Story:** As a sales executive, I want to attach documents to leads, meetings, and accounts, so that relevant files are accessible in context.

#### Acceptance Criteria

1. WHEN a user uploads a file, THE API SHALL generate a pre-signed S3 URL and return it to the client for direct upload
2. WHEN a file upload is requested, THE API SHALL validate the MIME type on the server side and reject disallowed types with HTTP 415
3. WHEN the uploaded file size exceeds 25 MB, THE API SHALL return HTTP 413 with a message indicating the limit
4. WHEN a document is stored, THE API SHALL record `entity_type`, `entity_id`, `file_name`, `s3_key`, `mime_type`, and `uploaded_by` in the documents table
5. THE Document record SHALL be linkable to any of the following entity types: `lead`, `meeting`, or `account`

---

### Requirement 10: Notes Management

**User Story:** As a sales user, I want to add free-text notes to leads, meetings, and accounts, so that important context is preserved alongside structured data.

#### Acceptance Criteria

1. WHEN a user submits a note with valid content, THE API SHALL store the note with `entity_type`, `entity_id`, `content`, and `created_by` fields
2. THE Note record SHALL be linkable to `lead`, `meeting`, or `account` entity types
3. WHEN a note is saved for a lead or meeting, THE API SHALL asynchronously enqueue embedding generation for the note content

---

### Requirement 11: AI Embedding Generation

**User Story:** As a platform operator, I want meeting notes and lead data to be semantically indexed, so that AI-powered search and recommendations are accurate and fast.

#### Acceptance Criteria

1. WHEN embedding generation is triggered, THE Embedding_Service SHALL call either the OpenAI `text-embedding-3-small` model or AWS Bedrock Titan model to generate a 1536-dimensional float vector
2. WHEN an embedding is stored, THE API SHALL insert a record into `notes_embeddings` with the `entity_type`, `entity_id`, `content`, `embedding`, and `model_version` fields
3. WHEN an embedding is generated for an `(entity_type, entity_id)` pair that already has a stored embedding, THE API SHALL upsert the record rather than creating a duplicate
4. WHEN the embedding API call fails, THE Embedding_Service SHALL retry up to 3 times using exponential backoff
5. IF all 3 embedding retries fail, THEN THE Embedding_Service SHALL log the error to CloudWatch at `ERROR` level and complete without raising an exception on the calling request
6. WHEN the EventBridge_Scheduler triggers the embedding retry job, THE API SHALL re-queue embedding generation for all entities that have missing embeddings
7. THE stored embedding vector SHALL always have exactly 1536 dimensions

---

### Requirement 12: Semantic Search

**User Story:** As a sales executive, I want to search meeting notes and lead data using natural language, so that I can quickly find relevant past interactions and information.

#### Acceptance Criteria

1. WHEN a user submits a semantic search query, THE API SHALL generate an embedding for the query text and execute a cosine similarity search against the `notes_embeddings` table using pgvector's `<=>` operator
2. WHEN a semantic search is executed, THE API SHALL return at most `top_k` results ordered by cosine similarity descending, where `top_k` is supplied by the caller
3. WHEN `top_k` is outside the range [1, 100], THE API SHALL reject the request with HTTP 400
4. WHEN the search query string is empty or whitespace-only, THE API SHALL reject the request with HTTP 400
5. WHEN search results are returned, THE API SHALL include `entity_type`, `entity_id`, `content_snippet` (up to 300 characters), and `similarity_score` (rounded to 4 decimal places) in each result
6. WHEN `entity_types` filters are specified, THE API SHALL restrict results to only the requested entity types
7. WHEN the Embedding_Service is unavailable during a search query, THE API SHALL degrade gracefully by returning keyword-based fallback results or an empty result set rather than an HTTP 5xx error

---

### Requirement 13: Similar Lead Discovery and Duplicate Detection

**User Story:** As a sales executive, I want to be warned about potential duplicate leads and discover similar past deals, so that effort is not duplicated and past intelligence is reused.

#### Acceptance Criteria

1. WHEN a new lead is submitted, THE Duplicate_Detector SHALL embed the lead name and query the `notes_embeddings` table for existing leads with cosine similarity above 0.92
2. WHEN potential duplicates are detected at lead creation, THE API SHALL return HTTP 409 with the list of similar leads including their names, statuses, and similarity scores
3. WHEN a user requests similar leads for an existing lead, THE API SHALL return the top 5 most semantically similar leads using cosine similarity on stored embeddings
4. WHEN duplicate detection finds no leads above the 0.92 threshold, THE API SHALL proceed with normal lead creation

---

### Requirement 14: AI Meeting Summarization and Next-Action Recommendations

**User Story:** As a sales manager, I want AI-generated summaries of meeting reports and recommended next actions for leads, so that I can quickly understand deal status without reading full reports.

#### Acceptance Criteria

1. WHEN a user with `sales_manager` role or higher requests a meeting summary, THE API SHALL send the meeting's `requirement_discussion` and `internal_notes` as context to an LLM and return the generated summary
2. WHEN a user with `sales_executive` role or higher requests a next-action recommendation for a lead, THE API SHALL send the lead's recent meeting notes and current status to an LLM and return a recommended action
3. WHEN an LLM call fails during summarization or recommendation, THE API SHALL return HTTP 503 with a message indicating the AI service is temporarily unavailable

---

### Requirement 15: Audit Logging

**User Story:** As a compliance officer, I want every data mutation to be recorded in an audit log, so that all changes are traceable and the platform meets compliance requirements.

#### Acceptance Criteria

1. WHEN any Lead, Opportunity, Meeting_Report, Action_Item, Account, or Contact record is created, updated, or deleted, THE API SHALL insert an Audit_Log record
2. THE Audit_Log record SHALL store `entity_type`, `entity_id`, `action`, `changed_by`, `before_state` (as JSONB), `after_state` (as JSONB), and `created_at`
3. WHEN a lead status changes, THE Audit_Log entry SHALL capture the old status in `before_state` and the new status and transition reason in `after_state`
4. THE Audit_Log table SHALL be insert-only; no Audit_Log record SHALL be modified or deleted after creation
5. WHEN a user with `admin` role queries the audit log, THE API SHALL return all audit entries for the requested entity

---

### Requirement 16: Competitive Intelligence Tracking

**User Story:** As a sales executive, I want to record competitive intelligence against each lead, so that the team can understand the competitive landscape and prepare differentiated responses.

#### Acceptance Criteria

1. WHEN a user submits competitor information for a lead, THE API SHALL store `competitor_name`, `strengths`, `weaknesses`, and `our_differentiators` in the competitors table linked to the lead
2. WHEN a lead is deleted, THE API SHALL cascade-delete all associated competitor records
3. WHEN a user retrieves a lead's detail, THE API SHALL include all associated competitor records in the response

---

### Requirement 17: Notification Management

**User Story:** As a platform user, I want to receive and manage in-app notifications, so that I am alerted to important events without leaving the platform.

#### Acceptance Criteria

1. WHEN a Notification is created for a user, THE API SHALL store `user_id`, `type`, `title`, `body`, `entity_type`, `entity_id`, and `is_read` fields
2. THE Notification `type` field SHALL accept only the following values: `overdue_action`, `lead_assigned`, or `meeting_reminder`
3. WHEN a user marks a notification as read, THE API SHALL update the `is_read` field to `true`
4. WHEN a user retrieves their notifications, THE API SHALL return only notifications belonging to that user

---

### Requirement 18: Dashboard and Reporting

**User Story:** As a sales manager, I want a dashboard with key metrics and pipeline visualizations, so that I can monitor team performance at a glance.

#### Acceptance Criteria

1. WHEN a user with `sales_manager` role or higher accesses the dashboard, THE API SHALL return summary metrics including total lead count by status, total weighted pipeline value, and count of overdue action items
2. WHEN a user with `sales_manager` role or higher views the pipeline forecast, THE API SHALL return per-stage aggregated weighted deal values suitable for chart rendering
3. WHEN a `sales_executive` accesses the dashboard, THE API SHALL scope all metrics to leads and action items owned by that user

---

### Requirement 19: Pagination and List Filtering

**User Story:** As a platform user, I want list views to support filtering and pagination, so that large datasets are navigable without performance degradation.

#### Acceptance Criteria

1. THE API SHALL support cursor-based pagination on all list endpoints using `after=<uuid>` and `limit` query parameters
2. WHEN the `limit` parameter is not supplied, THE API SHALL default to returning 50 records per page
3. WHEN a user filters leads by `status`, `owner_id`, or `account_id`, THE API SHALL return only records matching all specified filters
4. WHEN a user filters action items by `owner_id`, `lead_id`, or `status`, THE API SHALL return only records matching all specified filters

---

### Requirement 20: Progressive Web App (PWA) and Mobile Support

**User Story:** As a field sales executive, I want to use the platform on my mobile device including in areas with limited connectivity, so that I can capture and update information during or after customer visits.

#### Acceptance Criteria

1. THE PWA SHALL serve a valid web app manifest enabling installation on mobile home screens
2. WHEN a mobile user is offline, THE PWA SHALL queue draft meeting reports in IndexedDB and synchronize them with the API when connectivity is restored
3. WHEN an action item is overdue, THE PWA SHALL deliver a push notification to the user's mobile device via AWS SNS
4. WHEN the PWA receives a push notification payload from AWS SNS, THE PWA SHALL display the notification to the user even when the app is not in the foreground

---

### Requirement 21: Security and Data Protection

**User Story:** As a system administrator, I want the platform to enforce strong security controls, so that sensitive sales and defence data is protected from unauthorized access or disclosure.

#### Acceptance Criteria

1. THE API SHALL enforce TLS for all client-to-server and server-to-database communications
2. THE API SHALL store all database credentials, API keys, and Cognito configuration in AWS Secrets Manager and never in environment variables or source code
3. WHEN a file upload is requested, THE API SHALL validate the file's MIME type on the server side regardless of client-supplied content-type headers
4. THE API SHALL use parameterized queries via SQLAlchemy for all database interactions to prevent SQL injection
5. WHEN an S3 upload is required, THE API SHALL generate a pre-signed URL scoped to the specific object key and a limited expiry duration
6. WHEN a `defence_aerospace` meeting report is requested, THE API SHALL verify the user has the `defence_clearance` Cognito attribute before returning the report content

---

### Requirement 22: Observability and Error Handling

**User Story:** As a platform operator, I want comprehensive logging and monitoring, so that I can detect, diagnose, and resolve issues quickly.

#### Acceptance Criteria

1. THE API SHALL emit structured logs to AWS CloudWatch for every request, including `request_id`, `user_id`, `endpoint`, `status_code`, and `duration_ms`
2. WHEN an unhandled exception occurs in the API, THE API SHALL log the full stack trace to CloudWatch at `ERROR` level and return HTTP 500 with a generic error message to the client
3. WHEN the embedding retry job encounters a permanent failure after 3 retries, THE Embedding_Service SHALL emit a CloudWatch metric increment for `embedding_failure_count`
4. THE API infrastructure SHALL configure CloudWatch alarms on elevated HTTP 5xx rates and ECS CPU utilization exceeding 80%
5. WHEN a lead status transition fails validation, THE API SHALL return HTTP 400 with a structured error body listing the allowed transitions from the current status
6. WHEN a request arrives at a protected endpoint without a valid JWT, THE API SHALL return HTTP 401


---

### Requirement 23: Outlook OAuth2 Connection

**User Story:** As a sales user, I want to connect my Microsoft Outlook account to the platform, so that my emails with contacts are automatically synced into the lead timeline without manual entry.

#### Acceptance Criteria

1. WHEN a user initiates an Outlook connection, THE Outlook_Connector SHALL redirect the user to the Microsoft OAuth2 authorization endpoint requesting `Mail.ReadWrite` and `offline_access` delegated scopes
2. WHEN Microsoft redirects back with a valid authorization code, THE Outlook_Connector SHALL exchange the code for an OAuth2_Token pair (access token and refresh token) and store both tokens encrypted in AWS Secrets Manager under a key scoped to that user's ID
3. IF the OAuth2 authorization code exchange fails, THEN THE Outlook_Connector SHALL return HTTP 400 with a descriptive error message and SHALL NOT store any token data
4. WHEN an OAuth2_Token access token is within 5 minutes of expiry, THE Outlook_Connector SHALL use the stored refresh token to obtain a new access token without requiring user interaction
5. WHEN a user with `admin` role requests the list of connected users, THE API SHALL return each user's ID, email, connection status (`connected` or `disconnected`), and the timestamp of the last successful sync
6. THE API SHALL never return or log OAuth2_Token values in plaintext in any API response, log line, or CloudWatch entry
7. WHERE a user has not connected their Outlook account, THE Outlook_Connector SHALL treat all Outlook-dependent operations for that user as no-ops and return a `not_connected` status indicator

---

### Requirement 24: Email Sync to Lead Timeline

**User Story:** As a sales user, I want emails I send and receive with contacts to appear automatically on the related lead's timeline, so that I have a complete communication history without switching between tools.

#### Acceptance Criteria

1. WHEN the Outlook_Connector receives a new inbound or outbound email event for a connected user, THE Outlook_Connector SHALL retrieve the full email from the Graph_API including `subject`, `sender`, `to_recipients`, `cc_recipients`, `sent_datetime`, and `body`
2. WHEN an email is retrieved, THE Outlook_Connector SHALL extract all email addresses from the `sender`, `to_recipients`, and `cc_recipients` fields and query the Contacts table for a match on the `email` field
3. WHEN a contact match is found, THE Outlook_Connector SHALL insert an email record into the `emails` table storing `lead_id`, `outlook_message_id`, `subject`, `sender_email`, `recipient_emails`, `sent_at`, `body_full` (encrypted at rest), and `body_preview` (first 300 characters)
4. WHEN a single email matches multiple Leads via different Contacts sharing the same email address, THE Outlook_Connector SHALL insert a separate email record linked to each matching Lead
5. WHEN no contact match is found for any email address in the email, THE Outlook_Connector SHALL discard the email without storing any record
6. WHEN an email record is stored, THE API SHALL make the email visible in the Lead Detail page under an "Emails" tab, displaying `subject`, `sender_email`, `recipient_emails`, `sent_at`, `body_preview`, and an expandable "View Full Email" section showing `body_full`
7. WHEN the Outlook_Connector attempts to insert an email whose `outlook_message_id` already exists in the `emails` table for the same `lead_id`, THE Outlook_Connector SHALL skip the insertion to prevent duplicate email records

---

### Requirement 25: Real-Time Email Sync via Webhook

**User Story:** As a sales user, I want new emails to appear in the lead timeline within one minute of sending or receiving them, so that the timeline is always current.

#### Acceptance Criteria

1. WHEN a user successfully connects their Outlook account, THE Outlook_Connector SHALL register a Webhook_Subscription with the Graph_API for the user's mailbox `inbox` and `sentItems` folders with a maximum expiry of 3 days
2. WHEN the Graph_API delivers a change notification to the platform's webhook endpoint, THE API SHALL process the notification and trigger email retrieval and matching within 60 seconds of the event timestamp
3. WHEN a Webhook_Subscription has 24 hours or fewer remaining before its expiry, THE EventBridge_Scheduler SHALL renew the subscription using the Graph_API's subscription renewal endpoint so that the subscription never lapses
4. IF the Webhook_Subscription renewal fails, THEN THE Outlook_Connector SHALL retry the renewal up to 3 times using exponential backoff, and IF all retries fail THEN THE Outlook_Connector SHALL log the failure to CloudWatch at `ERROR` level and set the user's sync status to `sync_error`
5. WHEN a connected user's sync status is `sync_error`, THE API SHALL surface a "Sync Error — reconnect required" indicator in the UI sync status display for that user
6. WHEN a Webhook_Subscription is received, THE API SHALL validate the `clientState` secret in the notification payload against the stored secret before processing the event, and IF validation fails THEN THE API SHALL return HTTP 400 and discard the notification

---

### Requirement 26: Historical Email Backfill

**User Story:** As a sales user, I want my past 90 days of emails with known contacts to be imported when I first connect my Outlook account, so that the lead timeline has historical context from the moment I connect.

#### Acceptance Criteria

1. WHEN a user's Outlook account is successfully connected for the first time, THE Outlook_Connector SHALL enqueue a background backfill job to retrieve all emails from the user's mailbox sent or received within the last 90 calendar days
2. WHEN the backfill job retrieves historical emails, THE Outlook_Connector SHALL apply the same contact-matching logic defined in Requirement 24 to each retrieved email and insert matching emails into the `emails` table
3. WHEN the backfill job is running, THE API SHALL set the user's sync status to `backfill_in_progress` and return this status in the sync status indicator until the job completes
4. WHEN the backfill job completes without error, THE API SHALL update the user's sync status to `connected` and record the completion timestamp as the `last_synced_at` value
5. IF the backfill job fails or is interrupted, THEN THE Outlook_Connector SHALL log the failure to CloudWatch at `ERROR` level and set the user's sync status to `sync_error`
6. WHEN a backfill job attempts to insert an email whose `outlook_message_id` already exists in the `emails` table for the same `lead_id`, THE Outlook_Connector SHALL skip that record to ensure idempotency

---

### Requirement 27: AI-Powered Email Action Item Extraction

**User Story:** As a sales user, I want the platform to automatically identify actionable commitments in my emails and suggest them as action items, so that follow-ups are never missed.

#### Acceptance Criteria

1. WHEN a new email record is successfully inserted into the `emails` table, THE API SHALL asynchronously enqueue an AI analysis job that sends the email `body_full` to the existing LLM integration to identify actionable commitments
2. WHEN the LLM returns one or more Email_Action_Items from an email, THE API SHALL store each suggestion as an `email_action_item_suggestion` record with `lead_id`, `email_id`, `suggested_text`, `status` (defaulting to `pending`), and `created_at`
3. WHEN a user views the "Emails" tab on a Lead Detail page, THE API SHALL return all `pending` Email_Action_Item suggestions associated with emails on that lead
4. WHEN a user clicks "Add to Lead" on a suggestion, THE API SHALL create a fully linked Action_Item record with `lead_id`, `action_item` text from `suggested_text`, `owner_id` set to the requesting user's ID, and `status` set to `open`, and SHALL update the suggestion's `status` to `accepted`
5. WHEN a user dismisses an Email_Action_Item suggestion, THE API SHALL update the suggestion's `status` to `dismissed` and SHALL NOT create an Action_Item record
6. THE API SHALL never automatically create an Action_Item from an Email_Action_Item suggestion without explicit user confirmation via the "Add to Lead" action
7. WHEN the LLM call for action item extraction fails, THE API SHALL log the failure to CloudWatch at `WARN` level and SHALL NOT block or delay the email sync operation

---

### Requirement 28: Outlook Disconnect, Sync Status, and Data Retention

**User Story:** As a sales user, I want to disconnect my Outlook account at any time and see the current sync status, so that I have control over the integration and visibility into its health.

#### Acceptance Criteria

1. WHEN a user requests disconnection of their Outlook account, THE Outlook_Connector SHALL delete the user's OAuth2_Token from AWS Secrets Manager, cancel the user's active Webhook_Subscription via the Graph_API, and set the user's connection status to `disconnected`
2. WHEN a user's Outlook account is disconnected, THE API SHALL retain all previously synced email records in the `emails` table and SHALL NOT delete or modify them
3. WHEN a user's Outlook account is disconnected, THE API SHALL retain all accepted Action_Items derived from that user's email action item suggestions
4. WHEN a user views their profile settings, THE API SHALL return the user's Outlook connection status as one of: `connected`, `disconnected`, `backfill_in_progress`, or `sync_error`
5. WHEN a user's connection status is `connected`, THE API SHALL include a `last_synced_at` timestamp in the sync status response so that the UI can display "Connected — last synced X minutes ago"
6. WHEN the Webhook_Subscription cancellation call to the Graph_API fails during disconnect, THE Outlook_Connector SHALL continue with token deletion and status update, log the cancellation failure to CloudWatch at `WARN` level, and mark the connection as `disconnected`
7. WHILE a user is connected, THE API SHALL enforce that each user can only read email records synced from their own Outlook account and SHALL return HTTP 403 if a user attempts to access another user's email records
8. WHEN an `admin` user queries the Outlook connection audit log, THE API SHALL return each connected user's ID, email, connection timestamp, last sync timestamp, and connection status, and SHALL NOT include email body content in the audit response

---

### Requirement 29: Product Catalogue Management (Admin)

**User Story:** As an admin user, I want to create, update, and deactivate Products and Product_Categories in the catalogue, so that the sales team works from a single, authoritative list of company offerings.

#### Acceptance Criteria

1. WHEN a user with `admin` role submits a valid product payload, THE API SHALL create a Product record storing `name`, `code`, `description`, `category_id`, `base_price` (optional), `unit` (optional), `is_active` (defaulting to `true`), `created_at`, and `updated_at`, and return HTTP 201
2. WHEN a product creation or update request contains a `code` value that already exists in the Product_Catalogue for a different product, THE API SHALL return HTTP 409 with a message indicating the code must be unique
3. WHEN a user with `admin` role submits a valid product update, THE API SHALL persist the changes and update the `updated_at` timestamp on the Product record
4. WHEN a user with `admin` role deactivates a Product by setting `is_active` to `false`, THE API SHALL update the Product record and SHALL NOT delete or modify any existing Product_Lead or Product_Opportunity associations
5. WHEN a user with `admin` role submits a valid product category payload, THE API SHALL create a Product_Category record storing `name` and `description`, and return HTTP 201
6. WHEN any authenticated user requests the Product_Catalogue, THE API SHALL return all Product records; WHILE a user does not have the `admin` role, THE API SHALL omit edit and deactivate controls from the response metadata
7. WHEN a non-admin user submits a product create or update request, THE API SHALL return HTTP 403

---

### Requirement 30: Associate Products with Leads

**User Story:** As a sales user, I want to tag one or more Products on a Lead, so that I can record which company offerings are relevant to each sales opportunity.

#### Acceptance Criteria

1. WHEN a user with `sales_executive` role or higher submits a valid product association payload for an existing Lead, THE API SHALL create a Product_Lead record storing `lead_id`, `product_id`, and optional `notes`, and return HTTP 201
2. WHEN a product association request references a Product whose `is_active` is `false`, THE API SHALL return HTTP 422 with a message indicating the product is no longer active and cannot be added to new associations
3. WHEN a user requests the detail of a Lead, THE API SHALL include all associated Product records in the Overview section of the Lead response, regardless of the product's current `is_active` status
4. WHEN a user with `sales_executive` role or higher removes a Product_Lead association, THE API SHALL delete the Product_Lead record and return HTTP 204
5. THE API SHALL permit a Lead to have multiple Product_Lead associations and SHALL permit a Product to be associated with multiple Leads simultaneously
6. WHEN a Lead is deleted, THE API SHALL cascade-delete all Product_Lead records associated with that Lead

---

### Requirement 31: Associate Products with Opportunities (Line Items)

**User Story:** As a sales user, I want to attach Products with quantity and price to an Opportunity, so that the itemised deal value is transparently derived from the products being sold.

#### Acceptance Criteria

1. WHEN a user with `sales_executive` role or higher submits a valid line-item payload for an existing Opportunity, THE API SHALL create a Product_Opportunity record storing `opportunity_id`, `product_id`, `quantity` (integer, minimum 1), `unit_price` (decimal, minimum 0), and optional `notes`, and return HTTP 201
2. THE API SHALL compute `line_value` for every Product_Opportunity record as `quantity × unit_price` and SHALL NOT accept or store a client-supplied `line_value` value
3. WHEN a product line-item request references a Product whose `is_active` is `false`, THE API SHALL return HTTP 422 with a message indicating the product is inactive
4. WHEN a product line-item request contains a `quantity` less than 1 or a `unit_price` less than 0, THE API SHALL return HTTP 422
5. WHEN an Opportunity has one or more Product_Opportunity records, THE API SHALL compute `total_line_value` as the sum of all `line_value` values for that Opportunity and return it alongside `estimated_deal_value` in the Opportunity response
6. WHEN an Opportunity has no Product_Opportunity records, THE API SHALL return `total_line_value` as `null` in the Opportunity response and use `estimated_deal_value` as the sole deal value indicator
7. WHEN an Opportunity is deleted, THE API SHALL cascade-delete all Product_Opportunity records associated with that Opportunity

---

### Requirement 32: Pipeline Reporting by Product

**User Story:** As a sales manager, I want the pipeline forecast to break down weighted deal value by product and product category, so that I can identify which offerings drive the most pipeline and forecast revenue per product line.

#### Acceptance Criteria

1. WHEN a user with `sales_manager` role or higher requests the Pipeline_By_Product report, THE API SHALL return one row per Product that appears on at least one Opportunity, including `product_name`, `category_name`, `opportunity_count`, `total_estimated_value` (sum of `estimated_deal_value` for all linked Opportunities), and `total_weighted_value` (sum of `estimated_deal_value × probability / 100` for all linked Opportunities)
2. WHEN a `sales_executive` requests the Pipeline_By_Product report, THE API SHALL restrict the aggregation to Opportunities owned by that user
3. WHEN a `sales_manager` requests the Pipeline_By_Product report, THE API SHALL restrict the aggregation to Opportunities within that manager's team
4. THE API SHALL also return a Product_Category-level summary in the Pipeline_By_Product response, grouping `total_estimated_value` and `total_weighted_value` by `category_name`
5. WHEN computing `total_weighted_value` in the Pipeline_By_Product report, THE API SHALL count each Opportunity's weighted value once per Product it is tagged with and SHALL NOT aggregate the same Opportunity's weighted value more than once in the overall pipeline total when summing across all products
6. WHEN a user with `viewer` role requests the Pipeline_By_Product report, THE API SHALL return HTTP 403
7. WHEN no Opportunities match the requesting user's scope, THE API SHALL return an empty product rows array and zero-valued category summary

---

### Requirement 33: Product Catalogue UI

**User Story:** As a platform user, I want a searchable Catalogue page listing all active products, so that I can quickly find and select products when tagging leads and opportunities.

#### Acceptance Criteria

1. WHEN any authenticated user navigates to the Catalogue page, THE API SHALL return all Products where `is_active` is `true`, with each record including `name`, `code`, `category_name`, `base_price`, and `description`
2. WHEN a user filters the Catalogue by `category_id`, THE API SHALL return only Products belonging to that Product_Category
3. WHEN a user submits a keyword search on the Catalogue page, THE API SHALL return only Products whose `name`, `code`, or `description` contains the keyword string (case-insensitive)
4. WHEN a user with `admin` role views a product card, THE API SHALL include an `is_admin` flag set to `true` in the response so that the UI can render edit and deactivate controls alongside the product card
5. WHILE a user does not have the `admin` role, THE API SHALL set `is_admin` to `false` in product card responses so that edit and deactivate controls are not rendered
6. WHEN an `admin` user deactivates a Product from the Catalogue page, THE API SHALL update the product's `is_active` flag to `false` and exclude the product from subsequent Catalogue page responses

---

### Requirement 34: Auto-Tag Meeting Notes

**User Story:** As a sales user, I want meeting reports to be automatically tagged with relevant themes after saving, so that semantic search is more precise and competitive intelligence is automatically enriched.

#### Acceptance Criteria

1. WHEN a Meeting_Report is successfully saved, THE API SHALL asynchronously enqueue an AI tagging job that analyses the `requirement_discussion`, `client_feedback_concerns`, `competitive_landscape`, and `internal_notes` fields of that report
2. WHEN the AI tagging job completes, THE API SHALL store the returned array of Meeting_Tag values in the `meeting_tags` table linked to the `meeting_report_id`, where each stored tag value is a member of the Tag_Taxonomy
3. IF the AI tagging job returns a tag value that is not a member of the Tag_Taxonomy, THEN THE API SHALL discard that value and SHALL NOT store it
4. WHEN a Meeting_Report has associated Meeting_Tags, THE API SHALL return the tags as an array of coloured chip metadata on the Meeting Detail page response
5. WHEN a user submits a semantic search query with one or more Meeting_Tag filter values, THE API SHALL restrict results to Meeting_Reports that have at least one matching tag stored in the `meeting_tags` table
6. WHEN a user with `admin` role requests the tag frequency report, THE API SHALL return each Tag_Taxonomy value alongside the count of Meeting_Reports carrying that tag, ordered by count descending
7. WHEN AI competitive intelligence queries are executed, THE API SHALL include Meeting_Reports tagged with `competitor_mention` as additional context alongside Win_Loss_Review data
8. IF the AI tagging job fails for any reason, THEN THE API SHALL log the failure to CloudWatch at `WARN` level, and THE meeting save operation SHALL remain successful and unaffected
9. WHEN a meeting creator or a user with `admin` role submits a manual tag override for a Meeting_Report, THE API SHALL replace the current tags for that report with the submitted set and SHALL mark the tags as manually overridden
10. WHILE a Meeting_Report has manually overridden tags, THE API SHALL NOT overwrite those tags during any subsequent AI tagging re-run for that report

---

### Requirement 35: Forecasting with Historical Trends

**User Story:** As a sales manager, I want to see historical forecast accuracy and trend lines comparing predicted pipeline value against actual won deal value each month, so that I can calibrate forecasting confidence and identify improvement areas.

#### Acceptance Criteria

1. WHEN the EventBridge_Scheduler triggers the month-end snapshot job, THE API SHALL insert one Forecast_Snapshot record per pipeline stage capturing `month` (YYYY-MM), `stage`, `opportunity_count`, `total_weighted_value` (sum of `estimated_deal_value × probability / 100`), and `total_estimated_value` (sum of `estimated_deal_value`) for all active Opportunities in that stage at that point in time
2. WHEN a Forecast_Snapshot record is inserted, THE API SHALL treat it as immutable; no UPDATE or DELETE operation SHALL be performed on any Forecast_Snapshot row after insertion
3. WHEN a lead transitions to `won`, THE API SHALL insert a record into the `win_actuals` table storing `lead_id`, `opportunity_id`, `actual_value` (the `estimated_deal_value` at close), and `close_month` (YYYY-MM of the transition date)
4. WHEN a user with `sales_manager` role or higher requests the forecast trend data, THE API SHALL return monthly aggregated data for the preceding 12 calendar months, including `month`, `total_weighted_value` from the Forecast_Snapshot, `total_actual_won_value` from `win_actuals`, and `forecast_accuracy_pct` per month
5. WHEN `forecast_accuracy_pct` is computed for a month, THE API SHALL calculate it as `(sum of actual_value in win_actuals for that month / sum of total_weighted_value in Forecast_Snapshot for that month) × 100`, rounded to two decimal places
6. WHEN the denominator of the forecast accuracy calculation is zero for a given month, THE API SHALL return `forecast_accuracy_pct` as `null` for that month rather than a division-by-zero error
7. WHEN `forecast_accuracy_pct` would exceed 100 due to actual deals closing above forecast, THE API SHALL cap the returned value at 100 and SHALL NOT return a value below 0
8. WHEN a user with `sales_executive` role requests forecast trend data, THE API SHALL return HTTP 403

---

### Requirement 36: Sales Velocity Dashboard

**User Story:** As a sales manager, I want to see how quickly leads move through each pipeline stage, so that I can identify bottlenecks and coach my team to accelerate deal velocity.

#### Acceptance Criteria

1. WHEN a user with `sales_manager` role or higher requests the Sales_Velocity report, THE API SHALL compute from `audit_logs` the average number of days each lead spent in each stage, defined as the mean duration between consecutive status-change entries for that stage across all leads in scope
2. WHEN Sales_Velocity stage dwell times are computed, THE API SHALL return values that are always greater than or equal to zero days for every stage
3. WHEN a user with `sales_manager` role or higher requests deal cycle time, THE API SHALL return the average number of days between lead creation (`status = new`) and first transition to a terminal status (`won` or `lost`) across all leads in scope that have reached a terminal status
4. WHEN stage dropout rate is computed, THE API SHALL return for each stage the ratio of leads that transitioned to `lost` or `on_hold` from that stage divided by the total leads that entered that stage, expressed as a decimal in the range [0.0, 1.0]
5. WHEN the Sales_Velocity report is returned, THE API SHALL identify the bottleneck stage as the single stage with the highest average dwell time within the requesting user's scope and include it in the response as `bottleneck_stage`
6. WHILE a user has the `sales_executive` role, THE API SHALL restrict all Sales_Velocity computations to leads owned by that user only
7. WHEN the Sales_Velocity data is requested, THE API SHALL serve results from a server-side cache with a maximum age of 5 minutes; IF the cache is expired or absent, THEN THE API SHALL recompute the metrics on demand and refresh the cache

---

### Requirement 37: Win/Loss Review Workflow

**User Story:** As a sales manager, I want a structured review to be submitted every time a lead is won or lost, so that the reasons behind deal outcomes are captured and feed back into competitive intelligence.

#### Acceptance Criteria

1. WHEN a lead status transitions to `won` or `lost`, THE API SHALL include a `requires_win_loss_review: true` flag in the transition response
2. WHEN a Win_Loss_Review is submitted, THE API SHALL store `lead_id`, `outcome` (`won` or `lost`), `primary_reason`, `key_decision_factor`, `competitor_id` (optional FK to the competitors table), `lessons_learned`, `submitted_by`, and `submitted_at`
3. WHEN `primary_reason` is submitted in a Win_Loss_Review, THE API SHALL accept only the following values: `price`, `features`, `relationship`, `timeline`, `competitor`, `internal_decision`, or `no_budget`, and SHALL return HTTP 422 if any other value is provided
4. WHEN a Win_Loss_Review references a `lead_id`, THE API SHALL verify that the referenced Lead has status `won` or `lost`; IF the Lead status is any other value, THEN THE API SHALL return HTTP 422
5. WHEN a Win_Loss_Review is successfully submitted, THE API SHALL be immutable after that point; no UPDATE or DELETE operation SHALL be permitted on a submitted Win_Loss_Review record
6. WHEN a Win_Loss_Review is submitted, THE API SHALL asynchronously enqueue an embedding generation job for the concatenated `lessons_learned` and `key_decision_factor` fields and store the resulting embedding in `notes_embeddings` for use by AI competitive intelligence queries
7. WHEN 7 days have elapsed since a lead transitioned to `won` or `lost` without a Win_Loss_Review being submitted, THE EventBridge_Scheduler SHALL send a reminder notification to the lead owner via the existing notification mechanism
8. WHEN a user with `admin` or `sales_head` role requests Win_Loss_Reviews, THE API SHALL return all reviews; WHILE a user has the `sales_manager` role, THE API SHALL restrict the response to reviews for leads within that manager's team; WHILE a user has the `sales_executive` role, THE API SHALL restrict the response to reviews submitted for leads owned by that user

---

### Requirement 38: Account Health Score

**User Story:** As a sales manager, I want each account to have a computed health score, so that I can immediately see which accounts need attention and prioritise proactive outreach.

#### Acceptance Criteria

1. THE API SHALL compute an Account_Health_Score for each Account as the sum of four independent component scores, each in the range [0, 25], such that the total score is always in the range [0, 100]
2. WHEN computing the meeting recency component, THE API SHALL assign 25 points if the most recent Meeting_Report for the Account was within 30 calendar days, 15 points if within 31–90 days, 5 points if within 91–180 days, and 0 points if no meeting exists or the most recent meeting was more than 180 days ago
3. WHEN computing the action item health component, THE API SHALL assign 25 points if the Account has zero overdue Action_Items, 15 points if it has exactly one overdue Action_Item, 5 points if it has exactly two overdue Action_Items, and 0 points if it has three or more overdue Action_Items
4. WHEN computing the sentiment component, THE API SHALL send the `internal_notes` of the three most recent Meeting_Reports for the Account to the AI service and assign 25 points for positive sentiment, 15 points for neutral sentiment, 5 points for negative sentiment, and 10 points if no meeting notes are available
5. WHEN computing the pipeline score component, THE API SHALL assign 25 points if the Account has active pipeline value greater than ₹500,000, 15 points if active pipeline value is between ₹100,000 and ₹500,000 inclusive, 5 points if active pipeline value is greater than 0 and less than ₹100,000, and 0 points if there is no active pipeline
6. WHEN an Account_Health_Score is stored, THE API SHALL persist the total score, all four component scores, a `computed_at` timestamp, and an `at_risk` flag in the `account_health_scores` table
7. WHEN an Account_Health_Score total is less than 40, THE API SHALL set the `at_risk` flag to `true`; WHEN the total is 40 or greater, THE API SHALL set the `at_risk` flag to `false`
8. WHEN the EventBridge_Scheduler triggers the daily health score job, THE API SHALL recompute and upsert the Account_Health_Score for every active Account
9. WHEN a user with `sales_manager` role or higher requests account health data, THE API SHALL return the score, all four component values, the `at_risk` flag, and the `last_computed_at` timestamp for each Account in scope
10. WHEN the dashboard is loaded by a user with `sales_manager` role or higher, THE API SHALL include an At-Risk Accounts widget returning all Accounts with `at_risk` set to `true` within the requesting user's scope

---

### Requirement 39: Offline-First Meeting Reports

**User Story:** As a field sales executive, I want to complete and submit a full meeting report even when I have no network connectivity, so that I never lose data captured during or after a customer visit.

#### Acceptance Criteria

1. WHEN any field in the meeting report form is changed while the PWA is online or offline, THE PWA SHALL persist the entire current form payload (all steps: info, attendees, discussion, and actions) to the Offline_Queue in IndexedDB with a `status` of `draft`
2. WHEN the user submits the meeting report form while the device has no network connectivity, THE PWA SHALL store the complete form payload in the Offline_Queue with a `status` of `pending` and SHALL display a confirmation to the user that the report will sync when connectivity is restored
3. WHEN network connectivity is restored, THE PWA SHALL iterate all Offline_Queue items with `status` of `pending` and POST each complete payload to `POST /meetings`; WHEN a POST succeeds, THE PWA SHALL remove that item from the Offline_Queue and display a toast notification confirming the report was submitted
4. WHEN a sync attempt for an Offline_Queue item fails because the referenced Lead no longer exists on the server, THE PWA SHALL surface a conflict resolution UI offering the user the option to reassign the report to a different Lead or discard the pending report
5. WHEN the Offline_Queue contains one or more items with `status` of `pending`, THE PWA SHALL display a visual indicator showing the count of pending reports awaiting sync
6. THE Offline_Queue SHALL support multiple simultaneous pending report payloads, each identified by a unique client-generated UUID
7. WHEN a user edits a meeting report that is still in the Offline_Queue with `status` of `pending`, THE PWA SHALL update the existing Offline_Queue item with the revised payload rather than creating a duplicate entry
8. WHEN an Offline_Queue item has `status` of `synced`, THE PWA SHALL never re-submit that item to the API
9. THE PWA SHALL only remove an item from the Offline_Queue on successful API sync confirmation or explicit user discard; automatic or silent removal for any other reason SHALL NOT occur
