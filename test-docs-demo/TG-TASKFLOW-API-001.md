# Test Notes Draft
Generated: 2025-11-07 00:13:41
Status: Pending

## Query
Add Task Assignment Feature to TaskFlow API

*As a* project manager\
*I want to* assign tasks to specific team members\
*So that* I can delegate work and track who is responsible for each task

h3. Acceptance Criteria

# *Task Assignment*
#* Users can assign a task to one user from their organization
#* Only authenticated users can assign tasks
#* Cannot assign tasks to users outside organization
#* Can reassign tasks to different users
#* Can unassign tasks (set to null)
# *API Changes*
#* {{POST /api/tasks}} accepts optional {{assigned_to}} field (user_id)
#* {{PUT /api/tasks/{id}}} can update {{assigned_to}} field
#* {{GET /api/tasks}} can filter by {{assigned_to=me}} or {{assigned_to={user_id}}}
#* {{GET /api/tasks/{id}}} includes {{assigned_to}} user info in response
# *Validation Rules*
#* {{assigned_to}} must be valid user ID
#* Assigned user must exist in same organization
#* Cannot assign deleted/inactive users
#* Assignment changes logged in audit trail
# *Authorization*
#* Only task owner or assigned user can view task details
#* Only task owner or admin can change assignment
#* Regular users cannot assign tasks to others without permission
# *Notifications* (future scope)
#* User notified when task assigned to them
#* User notified when task reassigned away from them

h3. Technical Details

*Database Changes:*

* Add {{assigned_to}} column to {{tasks}} table (nullable, FK to [users.id|http://users.id])
* Add {{assigned_at}} timestamp column
* Add index on {{assigned_to}} for filtering performance

*API Endpoints Modified:*

* {{POST /api/tasks}} - accept assigned_to
* {{PUT /api/tasks/{id}}} - allow updating assigned_to
* {{GET /api/tasks}} - support assigned_to filters
* {{GET /api/tasks/{id}}} - include assignee details

*Dependencies:*

* Requires user data from Auth Service (TG-TASKFLOW-AUTH-001)
* Affects Web Dashboard task forms (TG-TASKFLOW-WEB-001)
* Uses existing authorization middleware

*Security Concerns:*

* Must prevent assigning tasks to users in other organizations (data leakage)
* Authorization checks on assignment changes
* Cannot bypass access control by assigning to yourself

## Scenarios to test for new/changed functionality
Here are the specific test scenarios for the "Task Assignment Feature" in the TaskFlow API based on the provided ticket description and relevant documentation:

- **Assign Task to User from Organization**
  - **Given** an authenticated user with a valid JWT token
  - **When** performing a `POST /api/tasks` request with a valid payload including an `assigned_to` field (user_id) within the same organization
  - **Then** the response should return `201 Created`, and the task in the database should show the assigned user ID in the `assigned_to` field.

- **Reassign Task to Different User**
  - **Given** a task exists that is currently assigned to User A
  - **When** performing a `PUT /api/tasks/{id}` request updating the `assigned_to` field to User B (within the same organization)
  - **Then** the response should return `200 OK`, and the `assigned_to` field should be updated to User B's ID in the database.

- **Unassign Task**
  - **Given** a task is assigned to User A
  - **When** performing a `PUT /api/tasks/{id}` request with the `assigned_to` field set to null
  - **Then** the response should return `200 OK`, and the task's `assigned_to` field in the database should be null.

- **Validate User Assignment Constraints**
  - **Given** an authenticated user attempting to assign a task
  - **When** sending a `POST /api/tasks` or `PUT /api/tasks/{id}` request with an `assigned_to` field containing a user ID that is deleted or inactive
  - **Then** the response should return `400 Bad Request`, and the error message should indicate that the user cannot be assigned.

- **Filter Tasks by Assigned User**
  - **Given** multiple tasks exist assigned to various users in the organization
  - **When** performing a `GET /api/tasks?assigned_to={user_id}`
  - **Then** the response should return `200 OK` with a list of tasks that are only assigned to the specified user.

- **Authorization Check on Task Access**
  - **Given** Task A is assigned to User A
  - **When** User B attempts to `GET /api/tasks/{id}` for Task A
  - **Then** the response should return `403 Forbidden`, preventing access to the task details.

- **Logging Assignment Changes**
  - **Given** a task assignment is changed
  - **When** the task is either assigned or reassigned using a `PUT /api/tasks/{id}` request
  - **Then** verify that an appropriate log entry is created in the audit trail for the changes made, including user IDs involved in the assignment change. 

These scenarios cover the new functionality of task assignment, ensuring adherence to the acceptance criteria and addressing potential risks associated with user assignment and authorization.

## Things to consider
### Key Considerations for Testing Task Assignment Feature

- **Edge Cases and Boundary Conditions**:
  - **Concurrent Updates**: Validate that when multiple users attempt to reassign the same task simultaneously, the first update succeeds while the second receives a conflict error (TEST-EDGE-003).
  - **Invalid User Assignments**: Verify assigning a task to a user ID that does not exist or belongs to a different organization results in an error.
  - **Null Assignments**: Check that when a task is unassigned (set to null), the system correctly handles this state and it does not affect the other task properties.

- **Data Validation Requirements**:
  - **User ID Validation**: Ensure that the `assigned_to` field requires a valid user ID and rejects invalid formats or nonexistent user IDs.
  - **Organization Check**: Confirm that the system prevents task assignments to users outside of the authenticated user's organization.
  - **Inactive/Deleted Users**: Test the API’s response when attempting to assign tasks to users marked as deleted or inactive.

- **Error Handling Scenarios**:
  - **Unauthorized Access**: Validate that only task owners or admins can change task assignments; any unauthorized change should return a 403 Forbidden error (TEST-ERR-004).
  - **Assignment Changes Logging**: Ensure there is an audit trail that logs all assignment changes correctly, including before and after states.
  - **Input Validation Consistency**: Verify that client-side validation aligns with API responses for assignment errors, such as invalid user IDs or organization mismatches (TEST-INT-001).

- **Performance Considerations**:
  - **API Response Time**: Measure the API’s response time for creating and updating tasks with assignments to verify that it meets acceptable thresholds during load testing scenarios.
  - **Filtering Performance**: Check that the `GET /api/tasks` endpoint efficiently filters tasks by assigned users, particularly with a large dataset.

- **Security Implications**:
  - **Data Leakage Prevention**: Ensure that API does not expose task assignment details of other users through improper access control mechanisms.
  - **JWT Authentication Integrity**: Verify that only authenticated users with valid JWT tokens can access any task assignment functionalities (AUTH).

- **Integration and Dependency Concerns**:
  - **Auth Service Dependency**: Confirm that all task assignment features integrate smoothly with the Auth Service to verify user authentication before allowing task assignments.
  - **Web Dashboard Implications**: Test that the TaskFlow Web Dashboard correctly reflects task assignments and reassignments in real-time.
  - **Database Consistency**: Ensure that all changes to the `assigned_to` field are accurately reflected in the tasks table and validated against database integrity constraints.

By focusing on these considerations, the testing will ensure a robust implementation of the Task Assignment Feature in the TaskFlow API while adhering to quality standards and requirements set forth in the documentation.

## Regression areas that could be affected
### Regression Areas for Task Assignment Feature – TaskFlow API

- **Task Creation Workflow**: 
  - Verify that the `POST /api/tasks` endpoint correctly handles task creation with the new `assigned_to` field. Ensure that valid assignments do not interfere with the task being created successfully, including all validation checks.

- **Task Update Functionality**:
  - Validate the `PUT /api/tasks/{id}` endpoint with various scenarios, including updating the `assigned_to` field. Ensure that the correct user can update task assignments and that audit logging functions properly.

- **Task Retrieval**: 
  - Confirm functionality of `GET /api/tasks` to filter tasks by `assigned_to`, ensuring that both filtering by oneself and by specific users works as expected. Check that tasks are appropriately authorized based on assignment.

- **Task Detail Access**:
  - Test `GET /api/tasks/{id}` to ensure task details show the correct `assigned_to` information, especially when the task is assigned to someone within the organization. Validate that access control restrictions are respected.

- **Task Deletion**:
  - Validate the `DELETE /api/tasks/{id}` operation to ensure tasks can still be deleted correctly and verify the refresh functionality of the task list due to deletion interactions.

- **Concurrent Updates**:
  - Test scenarios for concurrent updates on tasks involving assignment changes to ensure that optimistic locking prevents data loss and reflects correct behavior between users.

- **Authorization Checks**:
  - Conduct tests to ensure that unauthorized users cannot assign tasks to others or access tasks assigned to different users, validating against existing specifications for access control.

- **Validation and Error Handling**:
  - Verify that invalid user assignments (e.g., assigning to deleted or inactive users) trigger the correct validation errors. Also, test for edge cases like assigning tasks when the user is unauthenticated.

- **Audit Trail Verification**:
  - Ensure that any assignment changes (assign, unassign, or reassign) are logged correctly in the audit trail, reflecting all modifications made to the `assigned_to` field.

- **Web Dashboard Integration**:
  - Validate the integration of the task creation, updates, and deletions from the TaskFlow Web Dashboard to ensure forms and lists reflect changes made in the API, specifically the handling of the `assigned_to` field.

- **Notification System (Future Scope)**:
  - While not fully implemented, ensure the system is ready to integrate notification functionalities regarding task assignments, preserving the expected behavior for future developments.

These areas cover critical interaction points and workflows that could be affected by the introduction of the task assignment feature, ensuring that functionality remains intact and that new features do not introduce regressions.

## Citations
[TG-TASKFLOW-API-001#technical-scope], [TG-TASKFLOW-API-001#31-happy-path-scenarios], [TG-TASKFLOW-API-001#32-error-handling-scenarios], [TG-TASKFLOW-API-001#what-were-testing], [TG-TASKFLOW-API-001#33-edge-cases], [TG-TASKFLOW-WEB-001#31-happy-path-scenarios], [TG-TASKFLOW-WEB-001#data-requirements], [TG-TASKFLOW-API-001#34-integration-scenarios], [TG-TASKFLOW-AUTH-001#technical-scope], [TG-TASKFLOW-WEB-001#34-integration-scenarios], [TG-TASKFLOW-AUTH-001#protects-these-services], [TG-TASKFLOW-WEB-001#35-regression-tests]

---
*Generated by AI QA Assistant using gpt-4o-mini (temp: 0.1)*
*Based on 12 retrieved document chunks*
