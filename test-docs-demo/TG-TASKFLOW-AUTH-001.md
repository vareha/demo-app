# Test Notes Draft
Generated: 2025-11-07 09:35:50
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
- **Assign Task to User within Organization**
  - **Given**: Authenticated user with valid JWT token.
  - **When**: Perform `POST /api/tasks` with payload including valid `assigned_to` user ID from the same organization.
  - **Then**: 
    - Returns 201 Created.
    - Assigned user ID included in the task response.
    - Check task is correctly persisted in the database with the `assigned_to` field populated.

- **Reassign Task to Different User**
  - **Given**: Task exists with an assigned user.
  - **When**: Perform `PUT /api/tasks/{id}` with a different valid `assigned_to` user ID from the same organization.
  - **Then**: 
    - Returns 200 OK.
    - The task's `assigned_to` field is updated in the response and database.
    - Confirm that assignment change is logged in the audit trail.

- **Unassign Task**
  - **Given**: Task is assigned to a user.
  - **When**: Perform `PUT /api/tasks/{id}` and set `assigned_to` to null.
  - **Then**: 
    - Returns 200 OK.
    - The `assigned_to` field in the task details is null.
    - Ensure audit trail logs the unassignment.

- **Authorization Check for Assignment**
  - **Given**: User B (not the owner) attempts to assign a task owned by User A.
  - **When**: Perform `PUT /api/tasks/{id}` to change the `assigned_to` user.
  - **Then**: 
    - Returns 403 Forbidden.
    - Confirm no task data returned, ensuring proper enforcement of authorization rules.

- **Filter Tasks by Assigned User**
  - **Given**: Multiple tasks with different assigned users exist.
  - **When**: Perform `GET /api/tasks?assigned_to={user_id}` for a specific user.
  - **Then**: 
    - Returns 200 OK.
    - Only tasks assigned to the specified user are returned in the response.

- **Validate User Assignment** 
  - **Given**: Authenticated user.
  - **When**: Perform `POST /api/tasks` with an invalid `assigned_to` user ID (either non-existent or a user from another organization).
  - **Then**: 
    - Returns 400 Bad Request.
    - Ensure the error message appropriately describes the validation failure (e.g., "assigned_to must be a valid user from the same organization").

- **Task Details Include Assigned User Info**
  - **Given**: Task exists with an assigned user.
  - **When**: Perform `GET /api/tasks/{id}`.
  - **Then**: 
    - Returns 200 OK.
    - Response body includes the `assigned_to` user info as expected.

## Things to consider
### Testing Considerations for Task Assignment Feature

- **Edge Cases and Boundary Conditions:**
  - Test assigning a task to a user with the maximum allowed user ID length.
  - Verify behavior when assigning tasks with a `null` value for `assigned_to`.
  - Check system response when attempting to assign a task to users with invalid or malformed user IDs.
  - Assess performance with a high number of tasks being assigned/reassigned in quick succession (stress test).

- **Data Validation Requirements:**
  - Ensure that the `assigned_to` field accepts only valid user IDs (existing and active users within the organization).
  - Validate that attempting to assign a task to a user outside the authenticated user's organization returns an appropriate error message.
  - Verify that task assignment is prevented for deleted or inactive users, and an error is returned.
  - Validate the presence of an audit trail entry when tasks are assigned, reassigned, or unassigned, checking for correct logging of user actions.

- **Error Handling Scenarios:**
  - Verify system behavior when an unauthenticated user attempts to assign a task (should return 401 Unauthorized).
  - Check for proper error messages when trying to reassign a task without permission (access control enforcement).
  - Test the response when trying to assign tasks concurrently in a multi-user environment to ensure there are no race conditions or data integrity issues (conflict scenarios).

- **Performance Considerations:**
  - Measure the API response times for task assignment operations to ensure they meet specified performance metrics.
  - Evaluate the performance implications of filtering tasks by the `assigned_to` field with a large dataset to ensure that the added index on `assigned_to` improves query performance.

- **Security Implications:**
  - Test for data leakage by attempting to assign tasks to users without appropriate permissions and ensure access control is enforced.
  - Verify that sensitive user information is not exposed through task details accessible via the GET endpoints.
  - Conduct SQL injection tests on the `assigned_to` field during task assignment API calls to ensure input sanitization is effective.

- **Integration and Dependency Concerns:**
  - Ensure integration with the Auth Service is functioning correctly; test that authenticated user IDs correctly populate the assignment options.
  - Confirm that changes in task assignment reflect appropriately in the Web Dashboard in real time, including verification of the UI update mechanisms.
  - Test the impact of the API changes on related components in the Web Dashboard, ensuring no regressions occur due to the addition of the task assignment functionality.

By addressing these considerations, testing can comprehensively cover functional, non-functional, and security aspects of the new Task Assignment feature implemented in the TaskFlow API.

## Regression areas that could be affected
### Regression Areas for Task Assignment Feature in TaskFlow API

- **Task Creation Flow:**  
  - **POST /api/tasks**: Verify task creation with and without the `assigned_to` field. Ensure that the creation process completes without errors and that the task can be viewed with the correct assignment in subsequent GET requests.

- **Task Update Flow:**  
  - **PUT /api/tasks/{id}**: Ensure task updates still function correctly. Test updates to both the `assigned_to` field and any other fields (like `status`, `priority`, etc.), confirming that only the intended fields change and proper auditing occurs.

- **Task Retrieval and Filtering:**  
  - **GET /api/tasks**: Validate the `assigned_to` filtering feature works without issues. Test retrieval of tasks assigned to specific users or for the current user (`assigned_to=me`) remains accurate.
  - **GET /api/tasks/{id}**: Check that task details correctly include the `assigned_to` field information, and ensure user permissions are respected.

- **Concurrent Updates:**  
  - **Race Conditions**: Test concurrency scenarios where two users may attempt to update the same task's assignment. Validate the response for handling conflicts properly (should return 409 Conflict).

- **Authorization Paths:**  
  - Ensure that users without permission cannot assign tasks to others. Validate the behavior of the system when a user tries to assign tasks improperly (e.g., unauthorized users trying to use the `PUT` endpoint to change assignments).

- **Error Handling Scenarios:**
  - Test error responses when attempting to assign tasks with invalid `user_id`, to users outside the organization, or to inactive/deleted users (related to the validation rules specified).

- **Notification System (Future Scope):**  
  - While not implemented, verify placeholders or indications that notifications should happen upon assignment and reassignment events, ensuring this integration is prepared for future development.

- **Integration with Web Dashboard:**
  - **TG-TASKFLOW-WEB-001** relates to the visual representation of tasks, so confirm that task assignment visibility (in the web UI) correctly reflects backend changes, especially in task lists and detail views.

- **Deleting Tasks:**  
  - Verify that task deletion (`DELETE /api/tasks/{id}`) functionality still works. Confirm that task removal from lists updates immediately after deletion in the web UI to reflect no erroneous data remaining.

- **Empty State Handling:**  
  - Ensure that the system handles situations where no tasks exist due to unassignment correctly and that task assignments do not disrupt the expected empty state UI behavior.

- **Performance and Security Checks:**  
  - Validate that new changes do not introduce performance degradation or security flaws related to task assignment, particularly around data leakage or unauthorized data access.

This list encompasses areas that share functionality or intersect with the changes from adding the task assignment feature, ensuring comprehensive regression testing coverage.

## Citations
[TG-TASKFLOW-API-001#technical-scope], [TG-TASKFLOW-API-001#31-happy-path-scenarios], [TG-TASKFLOW-API-001#32-error-handling-scenarios], [TG-TASKFLOW-API-001#what-were-testing], [TG-TASKFLOW-API-001#33-edge-cases], [TG-TASKFLOW-WEB-001#31-happy-path-scenarios], [TG-TASKFLOW-WEB-001#data-requirements], [TG-TASKFLOW-API-001#34-integration-scenarios], [TG-TASKFLOW-AUTH-001#technical-scope], [TG-TASKFLOW-WEB-001#34-integration-scenarios], [TG-TASKFLOW-AUTH-001#protects-these-services], [TG-TASKFLOW-WEB-001#35-regression-tests]

---
*Generated by AI QA Assistant using gpt-4o-mini (temp: 0.1)*
*Based on 12 retrieved document chunks*
