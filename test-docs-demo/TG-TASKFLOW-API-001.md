# Task API CRUD Operations Test Guideline
ID: TG-TASKFLOW-API-001
Version: 1.2
Feature: RESTful Task Management API
Type: API
Last Updated: 2025-11-07
Owner: QA Team

## 1. Feature Context
### What We're Testing
RESTful API endpoints for task CRUD operations in TaskFlow project management system. Covers creating, reading, updating, and deleting tasks with proper validation, error handling, and data integrity.

### Technical Scope
- **Components**: Task API Service, Task Repository, Validation Layer
- **APIs**:
  - `POST /api/tasks` - Create task
  - `GET /api/tasks/{id}` - Get task by ID
  - `GET /api/tasks` - List tasks with filters
  - `PUT /api/tasks/{id}` - Update task
  - `DELETE /api/tasks/{id}` - Delete task
- **Database**: tasks table (PostgreSQL)
- **External Services**: None (self-contained API)
- **Consumes This API**: TaskFlow Web Dashboard (TG-TASKFLOW-WEB-001)
- **Authentication**: Requires JWT tokens from TG-TASKFLOW-AUTH-001

## 2. Risk Analysis
### Critical Risks
- **RISK-001**: SQL injection via unvalidated input → **Test Focus**: Verify input sanitization on all fields
- **RISK-002**: Unauthorized access to other users' tasks → **Test Focus**: Test authentication and authorization boundaries
- **RISK-003**: Data loss on concurrent updates → **Test Focus**: Verify optimistic locking or conflict detection
- **RISK-004**: Invalid data persisted → **Test Focus**: Test validation rules comprehensively
- **RISK-005**: Data leakage due to assignment to users outside the organization → **Test Focus**: Validate that tasks are only assigned to users within the same organization

### Known Issues & Bugs
- **BUG-TF-101**: Task title truncation at 255 chars not validated → **Verify**: Returns 400 error for titles >255 chars
- **BUG-TF-203**: Deleted tasks still appear in search results → **Verify**: Soft-deleted tasks excluded from queries

## 3. Test Scenarios

### 3.1 Happy Path Scenarios
#### TEST-HP-001: Create Valid Task
**Component**: Task API Service
**Endpoint**: POST /api/tasks
**Feature Tags**: task-creation, validation, api-crud
**Test Type**: Happy Path
**Priority**: P1
**Prerequisites**: Authenticated user with valid JWT token
**Affects**: Web dashboard task form, task list views

**Given**: Authenticated user with valid API token
**When**: POST /api/tasks with valid payload:
```json
{
  "title": "Implement login feature",
  "description": "Add OAuth2 authentication",
  "status": "TODO",
  "priority": "HIGH",
  "due_date": "2025-12-01"
}
```
**Then**:
- Returns 201 Created
- Response includes task ID and all fields
- Task persisted in database
**Verify**:
- Returned task.id is UUID format
- task.created_at is current timestamp
- task.created_by matches authenticated user

---

#### TEST-HP-002: Retrieve Task by ID
**Component**: Task API Service
**Endpoint**: GET /api/tasks/{id}
**Feature Tags**: task-retrieval, api-crud
**Test Type**: Happy Path
**Priority**: P1
**Prerequisites**: Task exists in database
**Affects**: Web dashboard task detail page

**Given**: Task exists with ID "123e4567-e89b-12d3-a456-426614174000"
**When**: GET /api/tasks/123e4567-e89b-12d3-a456-426614174000
**Then**:
- Returns 200 OK
- Response body contains complete task data
**Verify**:
- All fields present: id, title, description, status, priority, due_date, created_at, updated_at
- No sensitive user data exposed

---

#### TEST-HP-003: List Tasks with Pagination
**Component**: Task API Service
**Endpoint**: GET /api/tasks
**Feature Tags**: task-list, pagination, api-crud
**Test Type**: Happy Path
**Priority**: P1
**Prerequisites**: Multiple tasks exist in database
**Affects**: Web dashboard main task list

**Given**: 25 tasks exist in system
**When**: GET /api/tasks?page=1&limit=10
**Then**:
- Returns 200 OK
- Response contains exactly 10 tasks
- Includes pagination metadata
**Verify**:
- `total_count: 25`
- `page: 1`
- `has_next_page: true`
- Tasks ordered by created_at DESC

---

#### TEST-HP-004: Filter Tasks by Status
**Component**: Task API Service
**Endpoint**: GET /api/tasks
**Feature Tags**: task-filtering, task-list, api-crud
**Test Type**: Happy Path
**Priority**: P2
**Prerequisites**: Tasks with various statuses exist
**Affects**: Web dashboard status filter functionality

**Given**: Tasks with status TODO, IN_PROGRESS, DONE exist
**When**: GET /api/tasks?status=TODO
**Then**:
- Returns 200 OK
- Only tasks with status=TODO returned
**Verify**:
- All returned tasks have status "TODO"
- Count matches expected TODO tasks

---

#### TEST-HP-005: Update Task Status
**Component**: Task API Service
**Endpoint**: PUT /api/tasks/{id}
**Feature Tags**: task-update, status-change, api-crud
**Test Type**: Happy Path
**Priority**: P1
**Prerequisites**: Task exists, authenticated user has permission
**Affects**: Web dashboard task status badges, audit logs

**Given**: Task with ID "abc123" has status "TODO"
**When**: PUT /api/tasks/abc123 with payload:
```json
{
  "status": "IN_PROGRESS"
}
```
**Then**:
- Returns 200 OK
- Task status updated in database
- updated_at timestamp refreshed
**Verify**:
- Only status field changed
- Other fields unchanged
- Audit log entry created

---

#### TEST-HP-006: Delete Task
**Component**: Task API Service
**Endpoint**: DELETE /api/tasks/{id}
**Feature Tags**: task-deletion, soft-delete, api-crud
**Test Type**: Happy Path
**Priority**: P1
**Prerequisites**: Task exists, authenticated user has permission
**Affects**: Web dashboard task list, search results
**Covers Risk**: RISK-004

**Given**: Task with ID "xyz789" exists
**When**: DELETE /api/tasks/xyz789
**Then**:
- Returns 204 No Content
- Task marked as deleted (soft delete)
**Verify**:
- Task not visible in GET /api/tasks
- Task still exists in database with deleted_at timestamp
- Can be recovered if needed

---

#### TEST-HP-007: Assign Task to User (Happy Path)
**Component**: Task API Service
**Endpoint**: POST /api/tasks
**Feature Tags**: task-assignment, api-crud
**Test Type**: Happy Path
**Priority**: P1
**Prerequisites**: Authenticated user with valid JWT token
**Affects**: Web dashboard task form, task list views

**Given**: Authenticated user with valid API token
**When**: POST /api/tasks with valid payload including assigned_to:
```json
{
  "title": "Implement new feature",
  "description": "Details of the feature",
  "status": "TODO",
  "assigned_to": "user_id_here"
}
```
**Then**:
- Returns 201 Created
- Response includes task ID and all fields
- Task persisted in database with assigned user ID
**Verify**:
- Assigned user ID matches the one in the payload

---

#### TEST-HP-008: Reassign Task (Happy Path)
**Component**: Task API Service
**Endpoint**: PUT /api/tasks/{id}
**Feature Tags**: task-reassignment, api-crud
**Test Type**: Happy Path
**Priority**: P1
**Prerequisites**: Task exists with ID `task_id`, assigned to user A
**Affects**: Web dashboard task reassignment

**Given**: Task exists with ID `task_id`
**When**: Sending a PUT request to update the task with a new `assigned_to` field:
```json
{
  "assigned_to": "new_user_id_here"
}
```
**Then**:
- Returns 200 OK
- Verify the task's `assigned_to` field is updated in the database.

---

#### TEST-HP-009: Unassign Task
**Component**: Task API Service
**Endpoint**: PUT /api/tasks/{id}
**Feature Tags**: task-unassignment, api-crud
**Test Type**: Happy Path
**Priority**: P1
**Prerequisites**: Task exists with ID `task_id`
**Affects**: Web dashboard task unassignment

**Given**: Task exists with ID `task_id`
**When**: Sending a PUT request to set the `assigned_to` field to `null`:
```json
{
  "assigned_to": null
}
```
**Then**:
- Returns 200 OK
- Verify the `assigned_to` field in the database is null.

---

#### TEST-HP-010: Filter Tasks by Assigned User
**Component**: Task API Service
**Endpoint**: GET /api/tasks
**Feature Tags**: task-filtering, api-crud
**Test Type**: Happy Path
**Priority**: P1
**Prerequisites**: Multiple tasks exist in the system
**Affects**: Web dashboard task list

**Given**: Multiple tasks exist in the system
**When**: Sending a GET request with the filter `assigned_to={user_id}`:
```plaintext
GET /api/tasks?assigned_to=user_id_here
```
**Then**:
- Returns 200 OK
- Verify that only tasks assigned to the specified user are included in the response.

---

#### TEST-HP-011: Authorization Check on Task Assignment
**Component**: Task API Service
**Endpoint**: PUT /api/tasks/{id}
**Feature Tags**: authorization, api-crud
**Test Type**: Security Test
**Priority**: P1
**Prerequisites**: User who is neither the task owner nor an admin
**Affects**: Task management authorization

**Given**: User who is neither the task owner nor an admin
**When**: Sending a PUT request to update the task's assignment
**Then**:
- Returns 403 Forbidden
- Verify that no changes are made to the task's `assigned_to` field.

---

#### TEST-HP-012: Concurrency Handling for Task Assignment Updates
**Component**: Task API Service
**Endpoint**: PUT /api/tasks/{id}
**Feature Tags**: concurrency, task-assignment
**Test Type**: Concurrency Test
**Priority**: P1
**Prerequisites**: Two users attempt to reassign the same task
**Affects**: Task assignment integrity

**Given**: Two users (User A and User B) are attempting to reassign the same task simultaneously
**When**: Both send simultaneous PUT requests to update the `assigned_to` field
**Then**:
- Verify that one update succeeds (200 OK) while the other returns a 409 Conflict.

---

### 3.2 Error Handling Scenarios
#### TEST-ERR-001: Create Task with Missing Required Field
**Component**: Task API Service, Validation Layer
**Endpoint**: POST /api/tasks
**Feature Tags**: validation, error-handling, input-validation
**Test Type**: Negative Test
**Priority**: P1
**Prerequisites**: Authenticated user
**Affects**: Web dashboard form validation feedback
**Covers Risk**: RISK-004

**Given**: Authenticated user
**When**: POST /api/tasks with payload missing title:
```json
{
  "description": "Some description",
  "status": "TODO"
}
```
**Then**:
- Returns 400 Bad Request
- Error message: "title is required"
**Verify**:
- No task created in database
- Error response includes field name
- No stack trace in response

---

#### TEST-ERR-002: Retrieve Non-Existent Task
**Component**: Task API Service
**Endpoint**: GET /api/tasks/{id}
**Feature Tags**: error-handling, not-found
**Test Type**: Negative Test
**Priority**: P2
**Prerequisites**: None
**Affects**: Web dashboard 404 error page

**Given**: No task exists with ID "nonexistent-id"
**When**: GET /api/tasks/nonexistent-id
**Then**:
- Returns 404 Not Found
- Error message: "Task not found"
**Verify**:
- No database errors logged
- Response time <100ms

---

#### TEST-ERR-003: Update Task with Invalid Data
**Component**: Task API Service, Validation Layer
**Endpoint**: PUT /api/tasks/{id}
**Feature Tags**: validation, error-handling, status-validation
**Test Type**: Negative Test
**Priority**: P1
**Prerequisites**: Task exists
**Affects**: Web dashboard task edit form
**Covers Risk**: RISK-004

**Given**: Task exists with ID "task123"
**When**: PUT /api/tasks/task123 with invalid status:
```json
{
  "status": "INVALID_STATUS"
}
```
**Then**:
- Returns 400 Bad Request
- Error message: "status must be one of: TODO, IN_PROGRESS, DONE"
**Verify**:
- Task unchanged in database
- Valid status values listed in error

---

#### TEST-ERR-004: Unauthorized Task Access
**Component**: Task API Service, Auth Middleware
**Endpoint**: GET /api/tasks/{id}
**Feature Tags**: authorization, security, access-control
**Test Type**: Security Test
**Priority**: P0
**Prerequisites**: Multi-user environment
**Affects**: Data privacy, user isolation
**Covers Risk**: RISK-002

**Given**: Task belongs to User A, current user is User B
**When**: GET /api/tasks/{userA_task_id} as User B
**Then**:
- Returns 403 Forbidden
- Error message: "Access denied"
**Verify**:
- No task data returned
- No user information leaked in error

---

#### TEST-ERR-005: SQL Injection Attempt
**Component**: Task API Service
**Endpoint**: GET /api/tasks
**Feature Tags**: security, sql-injection, input-sanitization
**Test Type**: Security Test
**Priority**: P0
**Prerequisites**: None
**Affects**: System security
**Covers Risk**: RISK-001

**Given**: Malicious user attempts SQL injection
**When**: GET /api/tasks?title=' OR '1'='1
**Then**:
- Returns 400 Bad Request OR returns empty results
- No SQL error exposed
**Verify**:
- Query parameter properly escaped
- No database access with malicious query
- Security event logged

---

### 3.3 Edge Cases
#### TEST-EDGE-001: Very Long Task Title
**Component**: Task API Service, Validation Layer
**Endpoint**: POST /api/tasks
**Feature Tags**: validation, edge-case, character-limit
**Test Type**: Boundary Test
**Priority**: P2
**Prerequisites**: None
**Affects**: Web dashboard character count
**Covers Bug**: BUG-TF-101

**Given**: Title with 500 characters
**When**: POST /api/tasks with 500-char title
**Then**:
- Returns 400 Bad Request
- Error: "title must be 255 characters or less"
**Verify**:
- Validation before database insertion
- Character count accurate (not byte count)

---

#### TEST-EDGE-002: Task with Future Due Date
**Component**: Task API Service
**Endpoint**: POST /api/tasks
**Feature Tags**: date-validation, edge-case
**Test Type**: Boundary Test
**Priority**: P3
**Prerequisites**: None
**Affects**: Task scheduling features

**Given**: Current date is 2025-11-05
**When**: Create task with due_date "2030-01-01"
**Then**:
- Returns 201 Created
- Task created successfully
**Verify**:
- No date validation errors
- Date stored correctly in UTC

---

#### TEST-EDGE-003: Concurrent Task Updates
**Component**: Task API Service, Database Layer
**Endpoint**: PUT /api/tasks/{id}
**Feature Tags**: concurrency, race-condition, optimistic-locking
**Test Type**: Concurrency Test
**Priority**: P1
**Prerequisites**: Same task accessed by multiple users
**Affects**: Data integrity
**Covers Risk**: RISK-003

**Given**: Task with ID "concurrent-test"
**When**: Two users update same task simultaneously
**Then**:
- First update succeeds (200 OK)
- Second update returns 409 Conflict
**Verify**:
- Optimistic locking prevents data loss
- Error message explains conflict
- Updated_at timestamp can resolve conflict

---

#### TEST-EDGE-004: Empty Task List
**Component**: Task API Service
**Endpoint**: GET /api/tasks
**Feature Tags**: empty-state, edge-case
**Test Type**: Edge Case
**Priority**: P2
**Prerequisites**: No tasks for current user
**Affects**: Web dashboard empty state display

**Given**: No tasks exist for current user
**When**: GET /api/tasks
**Then**:
- Returns 200 OK
- Empty array: `{"data": [], "total_count": 0}`
**Verify**:
- No null or undefined response
- Proper pagination metadata

---

### 3.4 Integration Scenarios
#### TEST-INT-001: Create Task and Verify in Database
**Component**: Task API Service, Database Layer
**Endpoint**: POST /api/tasks
**Feature Tags**: integration, database-verification
**Test Type**: Integration Test
**Priority**: P1
**Prerequisites**: Database access for verification
**Affects**: Data persistence layer

**Given**: Clean database state
**When**: POST /api/tasks with valid data
**Then**:
- API returns 201 Created
- Direct database query shows task exists
**Verify**:
- All fields match request payload
- Database constraints enforced (NOT NULL, etc.)
- Timestamps auto-generated correctly

---

#### TEST-INT-002: Task Created Event Published
**Component**: Task API Service, Event Bus
**Endpoint**: POST /api/tasks
**Feature Tags**: integration, event-publishing, messaging
**Test Type**: Integration Test
**Priority**: P2
**Prerequisites**: Event bus configured
**Affects**: Real-time notifications, webhooks

**Given**: Event listener configured
**When**: POST /api/tasks successfully creates task
**Then**:
- "task.created" event published to event bus
- Event payload includes task ID and user ID
**Verify**:
- Event published within 100ms
- Event format matches schema
- No PII in event payload

---

### 3.5 Regression Tests
#### TEST-REG-001: Deleted Tasks Not in Search
**Component**: Task API Service
**Endpoint**: GET /api/tasks
**Feature Tags**: regression, soft-delete, search
**Test Type**: Regression Test
**Priority**: P1
**Prerequisites**: Soft delete functionality enabled
**Affects**: Search results, task lists
**Previous Bug**: BUG-TF-203

**Context**: Previously deleted tasks appeared in search results
**Given**: Task was deleted (soft delete)
**When**: GET /api/tasks with search filters
**Then**: Deleted task NOT in results

---

## 4. Dependencies & Integration Points
### External Dependencies
- **PostgreSQL**: v14+, tasks table with proper indexes
- **Authentication Service**: JWT token validation required (TG-TASKFLOW-AUTH-001)
- **Event Bus**: RabbitMQ for task events (optional)

### Consumed By
- **Web Dashboard**: TG-TASKFLOW-WEB-001 (all CRUD operations)
- **Mobile App**: (future)

### API Contracts
- **Request Headers**:
  - `Authorization: Bearer <token>` (required)
  - `Content-Type: application/json` (for POST/PUT)
- **Response Format**: Consistent JSON structure
  ```json
  {
    "data": {},
    "error": null,
    "metadata": {}
  }
  ```

### Data Requirements
- **Test Users**: At least 2 users with different permissions
- **Test Tasks**: Variety of statuses, priorities, due dates
- **Database**: Seeded with realistic test data

## 5. Key Verification Points
### Always Verify
- [ ] Authentication required for all endpoints
- [ ] Input validation on all user-supplied data
- [ ] SQL injection prevention
- [ ] Proper HTTP status codes
- [ ] Error messages don't expose sensitive data
- [ ] Audit logging for all mutations (CREATE, UPDATE, DELETE)

### Common Failure Points
- Pagination off-by-one errors
- Timezone handling for due_dates
- Soft delete not respected in queries
- Concurrent update race conditions
- Authorization bypass via parameter tampering

---
*Generated by AI QA Assistant using gpt-4o-mini (temp: 0.1)*
*Based on 12 retrieved document chunks*