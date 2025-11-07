# Authentication & Session Management Test Guideline
ID: TG-TASKFLOW-AUTH-001
Version: 1.2
Feature: User Authentication & Session Management
Type: API + E2E
Last Updated: 2025-11-07
Owner: QA Team

## 1. Feature Context
### What We're Testing
Authentication flow including user login, JWT token management, session lifecycle, token refresh, and logout functionality. Critical security component protecting both API and web dashboard access. This document is updated to include the new Task Assignment feature within the TaskFlow API.

### Technical Scope
- **Components**: Auth Service, Token Manager, Session Store, Auth Middleware, Task Management Service
- **APIs**:
  - `POST /api/auth/login` - User login with credentials
  - `POST /api/auth/refresh` - Refresh expired JWT token
  - `POST /api/auth/logout` - End user session
  - `GET /api/auth/me` - Get current user profile
  - `POST /api/tasks` - Create a new task (includes optional `assigned_to` field)
  - `PUT /api/tasks/{id}` - Update existing task (can modify `assigned_to` field)
  - `GET /api/tasks` - Retrieve tasks (can filter by `assigned_to`)
  - `GET /api/tasks/{id}` - Retrieve a specific task (includes `assigned_to` information)
- **Database**: users table, sessions table, tasks table (PostgreSQL)
- **External Services**: Auth Service (TG-TASKFLOW-AUTH-001)
- **Protects**: All Task API endpoints (TG-TASKFLOW-API-001), Web Dashboard (TG-TASKFLOW-WEB-001)

## 2. Risk Analysis
### Critical Risks
- **RISK-001**: Credential leakage in logs or responses → **Test Focus**: Verify passwords never logged or returned
- **RISK-002**: Token replay attacks → **Test Focus**: Ensure tokens expire and cannot be reused after logout
- **RISK-003**: Session fixation attacks → **Test Focus**: Regenerate session ID after login
- **RISK-004**: Weak password acceptance → **Test Focus**: Test password complexity requirements
- **RISK-005**: Brute force attacks → **Test Focus**: Verify rate limiting on login endpoint
- **RISK-006**: Data leakage through task assignment → **Test Focus**: Ensure users can only assign tasks to members of their own organization

### Known Issues & Bugs
- **BUG-AUTH-501**: Token refresh race condition → **Verify**: Concurrent refresh requests handled correctly
- **BUG-AUTH-602**: Session not invalidated on logout → **Verify**: Token unusable after logout

## 3. Test Scenarios

### 3.1 Happy Path Scenarios
#### TEST-HP-001: Successful Login
**Component**: Auth Service, Session Store
**Endpoint**: POST /api/auth/login
**Feature Tags**: authentication, login, jwt-token, session-management
**Test Type**: Happy Path
**Priority**: P0
**Prerequisites**: User exists in database
**Affects**: All protected endpoints, web dashboard access

**Given**: User exists with email "test@example.com" and password "SecurePass123!"
**When**: POST /api/auth/login with:
```json
{
  "email": "test@example.com",
  "password": "SecurePass123!"
}
```
**Then**:
- Returns 200 OK
- Response includes JWT access token and refresh token
- Session created in database
**Verify**:
- Access token is valid JWT format
- Token expiry set to 15 minutes
- Refresh token expiry set to 7 days
- User ID encoded in token payload

---

#### TEST-HP-002: Access Protected Resource with Token
**Component**: Auth Middleware, Token Manager
**Endpoint**: GET /api/auth/me
**Feature Tags**: authentication, token-validation, protected-resource
**Test Type**: Happy Path
**Priority**: P0
**Prerequisites**: Valid JWT token
**Affects**: All API endpoints requiring authentication

**Given**: Valid JWT token
**When**: GET /api/auth/me with header `Authorization: Bearer <token>`
**Then**:
- Returns 200 OK
- Response includes user profile (id, email, name)
**Verify**:
- No password field in response
- Token validated correctly
- User data accurate

---

#### TEST-HP-003: Refresh Expired Token
**Component**: Auth Service, Token Manager
**Endpoint**: POST /api/auth/refresh
**Feature Tags**: token-refresh, session-management, jwt-rotation
**Test Type**: Happy Path
**Priority**: P1
**Prerequisites**: Access token expired, valid refresh token
**Affects**: Long-running sessions, mobile app persistence
**Covers Risk**: RISK-002

**Given**: Access token expired, valid refresh token available
**When**: POST /api/auth/refresh with:
```json
{
  "refresh_token": "<valid_refresh_token>"
}
```
**Then**:
- Returns 200 OK
- New access token issued
- Refresh token rotated (new one issued)
**Verify**:
- New access token valid for 15 minutes
- Old refresh token invalidated
- Session updated with new tokens

---

#### TEST-HP-004: Logout and Invalidate Session
**Component**: Auth Service, Session Store
**Endpoint**: POST /api/auth/logout
**Feature Tags**: logout, session-invalidation, security
**Test Type**: Happy Path
**Priority**: P1
**Prerequisites**: User logged in with active session
**Affects**: Token security, session cleanup
**Covers Risk**: RISK-002

**Given**: User logged in with valid token
**When**: POST /api/auth/logout with token
**Then**:
- Returns 204 No Content
- Token invalidated
- Session deleted from database
**Verify**:
- Token cannot be used for subsequent requests
- Refresh token also invalidated
- Session record marked as logged_out

---

#### TEST-HP-005: Assign Task to User (Happy Path)
**Component**: Task Management Service
**Endpoint**: POST /api/tasks
**Feature Tags**: task-assignment, happy-path, task-management
**Test Type**: Happy Path
**Priority**: P0
**Prerequisites**: Authenticated user exists with valid JWT token
**Affects**: Task assignment functionality

**Given**: Authenticated user with a valid JWT token
**When**: Sending a POST request with a valid task payload including the optional `assigned_to` field:
```json
{
  "title": "Implement new feature",
  "description": "Details of the feature",
  "status": "TODO",
  "assigned_to": "user_id_here"
}
```
**Then**:
- Returns a 201 Created response
- Verify the task is persisted with the assigned user ID.

---

#### TEST-HP-006: Reassign Task (Happy Path)
**Component**: Task Management Service
**Endpoint**: PUT /api/tasks/{id}
**Feature Tags**: task-reassignment, happy-path, task-management
**Test Type**: Happy Path
**Priority**: P0
**Prerequisites**: Task exists with ID `task_id`, assigned to user A
**Affects**: Task reassignment functionality

**Given**: Task exists with ID `task_id` and is assigned to user A
**When**: Sending a PUT request to update the task with a new `assigned_to` field:
```json
{
  "assigned_to": "new_user_id_here"
}
```
**Then**:
- Returns a 200 OK response
- Verify the task's `assigned_to` field is updated in the database.

---

#### TEST-HP-007: Unassign Task
**Component**: Task Management Service
**Endpoint**: PUT /api/tasks/{id}
**Feature Tags**: task-unassignment, happy-path, task-management
**Test Type**: Happy Path
**Priority**: P0
**Prerequisites**: Task exists with ID `task_id`
**Affects**: Task unassignment functionality

**Given**: Task exists with ID `task_id`
**When**: Sending a PUT request to set the `assigned_to` field to `null`:
```json
{
  "assigned_to": null
}
```
**Then**:
- Returns a 200 OK response
- Verify the `assigned_to` field in the database is null.

---

### 3.2 Error Handling Scenarios
#### TEST-ERR-001: Login with Invalid Password
**Component**: Auth Service, Validation Layer
**Endpoint**: POST /api/auth/login
**Feature Tags**: error-handling, security, brute-force-protection
**Test Type**: Negative Test
**Priority**: P1
**Prerequisites**: User exists, wrong password provided
**Affects**: Security posture, user enumeration prevention
**Covers Risk**: RISK-005

**Given**: User exists but wrong password provided
**When**: POST /api/auth/login with incorrect password
**Then**:
- Returns 401 Unauthorized
- Error message: "Invalid email or password"
**Verify**:
- Generic error (doesn't specify which field wrong)
- Failed attempt logged
- Rate limit counter incremented
- No timing attack vulnerability

---

#### TEST-ERR-002: Login with Non-Existent User
**Component**: Auth Service
**Endpoint**: POST /api/auth/login
**Feature Tags**: error-handling, security, user-enumeration
**Test Type**: Security Test
**Priority**: P0
**Prerequisites**: Email not in system
**Affects**: User enumeration prevention
**Covers Risk**: RISK-001

**Given**: Email doesn't exist in system
**When**: POST /api/auth/login with non-existent email
**Then**:
- Returns 401 Unauthorized
- Error message: "Invalid email or password"
**Verify**:
- Same error as wrong password (no user enumeration)
- Response time similar to valid user attempt

---

#### TEST-ERR-003: Access Protected Resource Without Token
**Component**: Auth Middleware
**Endpoint**: GET /api/auth/me
**Feature Tags**: authorization, error-handling, token-required
**Test Type**: Negative Test
**Priority**: P0
**Prerequisites**: No auth token provided
**Affects**: API security enforcement

**Given**: No authorization header provided
**When**: GET /api/auth/me without token
**Then**:
- Returns 401 Unauthorized
- Error message: "Authentication required"
**Verify**:
- Clear error message
- No resource data leaked

---

#### TEST-ERR-004: Access with Expired Token
**Component**: Auth Middleware, Token Manager
**Endpoint**: GET /api/auth/me
**Feature Tags**: token-expiry, error-handling, session-timeout
**Test Type**: Negative Test
**Priority**: P1
**Prerequisites**: Token expired (>15 minutes old)
**Affects**: Session timeout behavior

**Given**: JWT access token expired (>15 minutes old)
**When**: GET /api/auth/me with expired token
**Then**:
- Returns 401 Unauthorized
- Error message: "Token expired"
**Verify**:
- Suggests using refresh token
- Token expiry correctly validated

---

#### TEST-ERR-005: Brute Force Login Attempt
**Component**: Auth Service, Rate Limiter
**Endpoint**: POST /api/auth/login
**Feature Tags**: security, rate-limiting, brute-force-protection
**Test Type**: Security Test
**Priority**: P0
**Prerequisites**: Rate limiting configured (5 attempts / 15 min)
**Affects**: Account security, system stability
**Covers Risk**: RISK-005

**Given**: Rate limit set to 5 failed attempts per 15 minutes
**When**: POST /api/auth/login with wrong password 6 times
**Then**:
- First 5 attempts: 401 Unauthorized
- 6th attempt: 429 Too Many Requests
- Error: "Too many login attempts. Try again in X minutes"
**Verify**:
- Rate limit per IP address
- Lockout duration correct
- Successful login resets counter

---

#### TEST-ERR-006: Assign Task to User Outside Organization (Negative Test)
**Component**: Task Management Service
**Endpoint**: POST /api/tasks
**Feature Tags**: task-assignment, error-handling, task-management
**Test Type**: Negative Test
**Priority**: P1
**Prerequisites**: Authenticated user exists
**Affects**: Task assignment functionality

**Given**: Authenticated user
**When**: Submitting a task creation request with `assigned_to` set to a user from a different organization:
```json
{
  "title": "Task for external user",
  "description": "This should fail",
  "status": "TODO",
  "assigned_to": "external_user_id"
}
```
**Then**:
- Returns a 403 Forbidden response
- Verify that no task was created in the database.

---

### 3.3 Edge Cases
#### TEST-EDGE-001: Weak Password Rejected
**Component**: Auth Service, Password Validator
**Endpoint**: POST /api/auth/register
**Feature Tags**: validation, password-complexity, security
**Test Type**: Boundary Test
**Priority**: P1
**Prerequisites**: None
**Affects**: Account creation, password security
**Covers Risk**: RISK-004

**Given**: User registration with weak password "12345"
**When**: POST /api/auth/register (or update password)
**Then**:
- Returns 400 Bad Request
- Error: "Password must be at least 8 characters with 1 uppercase, 1 lowercase, 1 number, 1 special character"
**Verify**:
- Password complexity enforced
- Clear requirements communicated

---

### 3.4 Integration Scenarios
#### TEST-INT-001: End-to-End Login Flow (Web + API)
**Component**: Auth Service, Web Login Page, Token Storage
**Page**: /login
**Feature Tags**: integration, e2e, login-flow, web-authentication
**Test Type**: Integration Test
**Priority**: P1
**Prerequisites**: Web app and API both running
**Consumes**: TG-TASKFLOW-WEB-001 login page

**Given**: User on web login page (/login)
**When**: Enter credentials → Click "Login" button
**Then**:
- API login request succeeds
- Tokens stored in browser (httpOnly cookie or localStorage)
- Redirected to /dashboard
- Subsequent API calls include auth token
**Verify**:
- Token securely stored
- Auth header automatically added
- Dashboard loads user-specific data

---

### 3.5 Regression Tests
#### TEST-REG-001: Token Usable After Logout
**Component**: Auth Service, Token Blacklist
**Endpoint**: POST /api/auth/logout, GET /api/auth/me
**Feature Tags**: regression, logout, token-invalidation
**Test Type**: Regression Test
**Priority**: P0
**Prerequisites**: User logged out
**Affects**: Token security
**Previous Bug**: BUG-AUTH-602

**Context**: Previously tokens worked after logout
**Given**: User logged out successfully
**When**: Attempt to use old access token
**Then**: Returns 401 Unauthorized

---

## 4. Dependencies & Integration Points
### External Dependencies
- **PostgreSQL**: Users, sessions, and tasks tables required
- **Redis** (optional): For rate limiting and token blacklist
- **Email Service** (future): For password reset

### Protects These Services
- **Task API**: TG-TASKFLOW-API-001 (all endpoints require authentication)
- **Web Dashboard**: TG-TASKFLOW-WEB-001 (login/logout, token management)

### JWT Configuration
- **Algorithm**: HS256 (or RS256 for asymmetric)
- **Access Token TTL**: 15 minutes
- **Refresh Token TTL**: 7 days
- **Issuer**: "taskflow-auth"
- **Secret**: Must be strong, environment-specific

### Security Headers
- `Authorization: Bearer <token>` - Required for protected endpoints
- Response includes: `X-RateLimit-Remaining`, `X-RateLimit-Reset`

### Data Requirements
- **Test Users**:
  - Active user: test@example.com / SecurePass123!
  - Locked user: locked@example.com (for rate limit testing)
  - Admin user: admin@example.com (for role-based testing)

## 5. Key Verification Points
### Always Verify
- [ ] Passwords never logged, returned, or exposed
- [ ] JWT signatures validated on every request
- [ ] Tokens expire at correct intervals
- [ ] Rate limiting prevents brute force
- [ ] Sessions invalidated on logout
- [ ] Token refresh rotates refresh tokens
- [ ] HTTPS enforced in production

### Common Failure Points
- Password hashing algorithm weak (use bcrypt/argon2)
- JWT secret hard-coded or weak
- Token expiry not checked correctly
- Race conditions in token refresh
- Session fixation vulnerabilities
- CORS misconfiguration exposing tokens
- Tokens stored in localStorage (XSS risk)
- Rate limiting bypassed via IP spoofing

---

This updated document reflects the addition of the Task Assignment feature with the inclusion of new test scenarios while preserving all existing tests and their structure. The version has been incremented to 1.2, and the last updated date has been modified accordingly.