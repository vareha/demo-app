# Web Dashboard Task Management Test Guideline
ID: TG-TASKFLOW-WEB-001
Version: 1.2
Feature: TaskFlow Web Dashboard UI
Type: E2E
Last Updated: 2025-11-11
Owner: QA Team

## 1. Feature Context
### What We're Testing
Web-based dashboard for managing tasks in TaskFlow project management system. Covers task creation, viewing, updating, deletion, and assigning tasks to users through browser UI with real-time updates and responsive design.

### Technical Scope
- **Components**: React Dashboard, Task List Component, Task Form Component, WebSocket Client
- **Pages**:
  - `/dashboard` - Main task list view
  - `/task/new` - Create task form
  - `/task/{id}` - Task detail view
  - `/task/{id}/edit` - Edit task form
- **Backend APIs**: Consumes TG-TASKFLOW-API-001 endpoints
- **External Services**: WebSocket server for real-time updates
- **Authentication**: Requires authentication from TG-TASKFLOW-AUTH-001

## 2. Risk Analysis
### Critical Risks
- **RISK-001**: XSS via unescaped task content → **Test Focus**: Verify HTML escaping in task titles/descriptions
- **RISK-002**: State desync between UI and server → **Test Focus**: Test optimistic updates and rollback
- **RISK-003**: Form validation bypassed → **Test Focus**: Client-side validation matches server rules
- **RISK-004**: Memory leak from unclosed WebSocket → **Test Focus**: Test connection cleanup on navigation
- **RISK-005**: Data leakage by assigning tasks to users outside organization → **Test Focus**: Validate authorization on task assignments

### Known Issues & Bugs
- **BUG-UI-301**: Task list doesn't refresh after delete → **Verify**: List updates immediately after deletion
- **BUG-UI-402**: Long task titles overflow card boundaries → **Verify**: Text truncates with ellipsis

## 3. Test Scenarios

### 3.1 Happy Path Scenarios
#### TEST-HP-001: View Task Dashboard
**Component**: React Dashboard, Task List Component
**Page**: /dashboard
**Feature Tags**: dashboard, task-list, ui-navigation
**Test Type**: Happy Path
**Priority**: P1
**Prerequisites**: Authenticated user with existing tasks
**Consumes API**: GET /api/tasks (TG-TASKFLOW-API-001)

**Given**: User logged in and has 5 tasks
**When**: Navigate to /dashboard
**Then**:
- All 5 tasks displayed in list
- Tasks sorted by creation date (newest first)
- Each task shows title, status, priority
**Verify**:
- Loading spinner shows while fetching
- Empty state not shown
- Task cards clickable

---

#### TEST-HP-002: Create New Task via Form
**Component**: Task Form Component
**Page**: /task/new
**Feature Tags**: task-creation, form-submission, ui-crud
**Test Type**: Happy Path
**Priority**: P1
**Prerequisites**: Authenticated user
**Consumes API**: POST /api/tasks (TG-TASKFLOW-API-001)

**Given**: User on /task/new page
**When**: Fill form with:
- Title: "Design database schema"
- Description: "Create ERD for new features"
- Status: "TODO"
- Priority: "HIGH"
- Click "Create Task" button
**Then**:
- Task created successfully
- Redirected to /dashboard
- New task visible in list
- Success notification shown
**Verify**:
- Form fields cleared after submit
- API called with correct payload
- Task appears at top of list

---

#### TEST-HP-003: View Task Details
**Component**: Task Detail Component
**Page**: /task/{id}
**Feature Tags**: task-detail, ui-navigation
**Test Type**: Happy Path
**Priority**: P1
**Prerequisites**: Task exists in database
**Consumes API**: GET /api/tasks/{id} (TG-TASKFLOW-API-001)

**Given**: Task with ID "task123" exists
**When**: Click on task card in dashboard
**Then**:
- Navigated to /task/task123
- All task details displayed
- Action buttons visible (Edit, Delete)
**Verify**:
- Title, description, status, priority, due date all shown
- Created/updated timestamps formatted correctly
- Back button returns to dashboard

---

#### TEST-HP-004: Edit Task Details
**Component**: Task Form Component
**Page**: /task/{id}/edit
**Feature Tags**: task-update, form-submission, optimistic-ui
**Test Type**: Happy Path
**Priority**: P1
**Prerequisites**: Task exists, user has edit permission
**Consumes API**: PUT /api/tasks/{id} (TG-TASKFLOW-API-001)

**Given**: On task detail page for "task123"
**When**: Click "Edit" → Update status to "IN_PROGRESS" → Click "Save"
**Then**:
- Task updated successfully
- Returned to detail view
- Status badge shows "IN_PROGRESS"
**Verify**:
- Optimistic UI update (immediate feedback)
- API call completes successfully
- Other fields unchanged

---

#### TEST-HP-005: Delete Task
**Component**: Task Detail Component, Confirmation Modal
**Page**: /task/{id}
**Feature Tags**: task-deletion, ui-crud, modal-interaction
**Test Type**: Happy Path
**Priority**: P1
**Prerequisites**: Task exists, user has delete permission
**Consumes API**: DELETE /api/tasks/{id} (TG-TASKFLOW-API-001)
**Covers Risk**: RISK-002

**Given**: On task detail page for "task456"
**When**: Click "Delete" → Confirm in modal
**Then**:
- Task deleted
- Redirected to dashboard
- Task removed from list
**Verify**:
- Confirmation modal shows warning
- Cancel button aborts deletion
- Success message displayed

---

#### TEST-HP-006: Real-Time Task Update
**Component**: WebSocket Client, Task List Component
**Page**: /dashboard
**Feature Tags**: real-time, websocket, live-updates
**Test Type**: Happy Path
**Priority**: P2
**Prerequisites**: WebSocket connection active, multiple users
**Affects**: Real-time collaboration experience

**Given**: Dashboard open, WebSocket connected
**When**: Another user updates task status
**Then**:
- Task status updates in real-time without refresh
- Update animation plays
**Verify**:
- WebSocket message received
- UI updates within 500ms
- No page flicker

---

#### TEST-HP-007: Assign Task to User Within Organization
**Component**: Task Form Component
**Page**: /task/{id}/edit
**Feature Tags**: task-assignment, ui-crud
**Test Type**: Happy Path
**Priority**: P1
**Prerequisites**: Task exists, user has edit permission
**Consumes API**: PUT /api/tasks/{id} (TG-TASKFLOW-API-001)

**Given**: On task detail page for "task123"
**When**: Assign task to a valid user within the organization using `PUT /api/tasks/{id}` with `{"assigned_to": "valid_user_id"}`
**Then**:
- Task assigned successfully
- Assigned user details reflected in the task detail view
**Verify**:
- `assigned_to` field updated in the database
- Audit trail logs the assignment

---

#### TEST-HP-008: Reassign Task to Different User
**Component**: Task Form Component
**Page**: /task/{id}/edit
**Feature Tags**: task-assignment, ui-crud
**Test Type**: Happy Path
**Priority**: P1
**Prerequisites**: Task exists, user has edit permission
**Consumes API**: PUT /api/tasks/{id} (TG-TASKFLOW-API-001)

**Given**: Task assigned to user "userA"
**When**: Reassign the task to another valid user within the same organization using `PUT /api/tasks/{id}` with `{"assigned_to": "valid_user_id_2"}`
**Then**:
- Task reassigned successfully
- Audit trail logs the reassignment

---

#### TEST-HP-009: Unassign Task
**Component**: Task Form Component
**Page**: /task/{id}/edit
**Feature Tags**: task-assignment, ui-crud
**Test Type**: Happy Path
**Priority**: P1
**Prerequisites**: Task assigned to user
**Consumes API**: PUT /api/tasks/{id} (TG-TASKFLOW-API-001)

**Given**: Task is assigned to a user
**When**: Unassign the task by setting the `assigned_to` field to null using `PUT /api/tasks/{id}` with `{"assigned_to": null}`
**Then**:
- Task unassigned successfully
- `assigned_to` field set to null in the database

---

### 3.2 Error Handling Scenarios
#### TEST-ERR-001: Create Task with Empty Title
**Component**: Task Form Component, Client Validation
**Page**: /task/new
**Feature Tags**: validation, error-handling, form-validation
**Test Type**: Negative Test
**Priority**: P1
**Prerequisites**: None
**Covers Risk**: RISK-003

**Given**: On task creation form
**When**: Leave title empty → Click "Create Task"
**Then**:
- Form submission blocked
- Error message under title field: "Title is required"
- Submit button disabled until valid
**Verify**:
- No API call made
- Form remains on page
- Other fields retain values

---

#### TEST-ERR-002: Attempt to Assign Task to User Outside Organization
**Component**: Task Form Component
**Page**: /task/{id}/edit
**Feature Tags**: task-assignment, error-handling
**Test Type**: Negative Test
**Priority**: P1
**Prerequisites**: Task exists
**Consumes API**: PUT /api/tasks/{id} (TG-TASKFLOW-API-001)

**Given**: Task exists
**When**: Attempt to assign task to a user from a different organization using `PUT /api/tasks/{id}` with `{"assigned_to": "invalid_user_id"}`
**Then**:
- Verify the response indicates failure (403 Forbidden) with an appropriate error message.

---

#### TEST-ERR-003: Validate Task Assignment with Invalid User ID
**Component**: Task Form Component
**Page**: /task/{id}/edit
**Feature Tags**: task-assignment, validation
**Test Type**: Negative Test
**Priority**: P1
**Prerequisites**: Task exists
**Consumes API**: PUT /api/tasks/{id} (TG-TASKFLOW-API-001)

**Given**: Task exists
**When**: Attempt to assign the task using an invalid user ID using `PUT /api/tasks/{id}` with `{"assigned_to": "invalid_user_id"}`
**Then**:
- Verify the response indicates failure (400 Bad Request) with an error message specifying the validation rule.

---

### 3.3 Edge Cases
#### TEST-EDGE-001: Very Long Task Description
**Component**: Task Form Component
**Page**: /task/new
**Feature Tags**: edge-case, text-input, ui-layout
**Test Type**: Boundary Test
**Priority**: P3
**Prerequisites**: None

**Given**: On task creation form
**When**: Enter 5000-character description
**Then**:
- Form accepts input
- Textarea expands appropriately
- Full description saved
**Verify**:
- No UI layout breaking
- Scroll appears in textarea
- Character count shown (if applicable)

---

#### TEST-EDGE-002: Rapid Task Status Changes
**Component**: Task Detail Component, Optimistic UI
**Page**: /task/{id}/edit
**Feature Tags**: edge-case, concurrency, optimistic-ui
**Test Type**: Stress Test
**Priority**: P2
**Prerequisites**: Task exists
**Consumes API**: PUT /api/tasks/{id} (TG-TASKFLOW-API-001)
**Covers Risk**: RISK-002

**Given**: On task detail page
**When**: Change status 5 times rapidly (toggle)
**Then**:
- All changes queued and processed
- Final state correct
- No race conditions
**Verify**:
- Optimistic UI handles rapid changes
- Server state matches final UI state

---

### 3.4 Integration Scenarios
#### TEST-INT-001: Form Validation Matches API
**Component**: Task Form Component, Validation Layer
**Page**: /task/new
**Feature Tags**: integration, validation-sync, api-contract
**Test Type**: Integration Test
**Priority**: P1
**Prerequisites**: None
**Consumes API**: POST /api/tasks (TG-TASKFLOW-API-001)

**Given**: On task creation form
**When**: Submit various invalid payloads
**Then**:
- Client validation catches same errors as server
- Error messages consistent with API
**Verify**:
- Title required (both client & API)
- Status enum values match
- Date format validation aligned

---

### 3.5 Regression Tests
#### TEST-REG-001: Task List Refreshes After Delete
**Component**: Task List Component
**Page**: /dashboard
**Feature Tags**: regression, ui-refresh, state-management
**Test Type**: Regression Test
**Priority**: P1
**Prerequisites**: Task exists in list
**Consumes API**: DELETE /api/tasks/{id} (TG-TASKFLOW-API-001)
**Previous Bug**: BUG-UI-301

**Context**: Previously list didn't update after deletion
**Given**: Task visible in list
**When**: Delete task successfully
**Then**: Task immediately removed from list UI

---

## 4. Dependencies & Integration Points
### External Dependencies
- **Task API**: TG-TASKFLOW-API-001 (all CRUD endpoints)
- **Auth Service**: TG-TASKFLOW-AUTH-001 (JWT token management)
- **WebSocket Server**: wss://taskflow.example.com/ws for real-time updates

### Browser Support
- **Desktop**: Chrome 90+, Firefox 88+, Safari 14+, Edge 90+
- **Mobile**: iOS Safari 14+, Chrome Mobile 90+
- **Screen Sizes**: 320px (mobile) to 2560px (4K desktop)

### Data Requirements
- **Test Users**: User with existing tasks for list views
- **Empty State**: User with zero tasks for empty state testing
- **Test Tasks**: Various statuses, priorities, due dates for filtering

## 5. Key Verification Points
### Always Verify
- [ ] All user input properly escaped (XSS prevention)
- [ ] Form validation matches API validation
- [ ] Loading states shown during async operations
- [ ] Error messages user-friendly and actionable
- [ ] Responsive design works on mobile/tablet/desktop
- [ ] Keyboard navigation functional (accessibility)

### Common Failure Points
- Form validation bypassed by manipulating DOM
- WebSocket reconnection not handling missed messages
- Optimistic updates not rolling back on API failure
- Memory leaks from event listeners not cleaned up
- Stale data shown after navigation
- Double-submit on button mashing

---

**Version History**:
- v1.1 - Initial version
- v1.2 - Added tests for task assignment feature and updated risk analysis related to task assignments.