# Authentication & Session Management Test Guideline
ID: TG-TASKFLOW-AUTH-001  
Version: 1.2  
Feature: User Authentication & Session Management  
Type: API + E2E  
Last Updated: 2025-11-07  
Owner: QA Team  

## 1. Feature Context
### What We're Testing
Authentication flow including user login, JWT token management, session lifecycle, token refresh, logout functionality, and task assignment feature for task management. Critical security component protecting both API and web dashboard access.

### Technical Scope
- **Components**: Auth Service, Token Manager, Session Store, Auth Middleware
- **APIs**: 
  - `POST /api/auth/login` - User login with credentials
  - `POST /api/auth/refresh` - Refresh expired JWT token
  - `POST /api/auth/logout` - End user session
  - `GET /api/auth/me` - Get current user profile
  - `POST /api/tasks` - Create task with optional `assigned_to` field
  - `PUT /api/tasks/{id}` - Update task to assign to a user
  - `GET /api/tasks` - Retrieve tasks, filtering by `assigned_to`
  - `GET /api/tasks/{id}` - Get task details including assignee information
- **Database**: users table, sessions table, tasks table (PostgreSQL)
- **External Services**: None (self-contained auth system)
- **Protects**: All Task API endpoints (TG-TASKFLOW-API-001), Web Dashboard (TG-TASKFLOW-WEB-001)

## 2. Risk Analysis
### Critical Risks
- **RISK-001**: Credential leakage in logs or responses → **Test Focus**: Verify passwords never logged or returned
- **RISK-002**: Token replay attacks → **Test Focus**: Ensure tokens expire and cannot be reused after logout
- **RISK-003**: Session fixation attacks → **Test Focus**: Regenerate session ID after login
- **RISK-004**: Weak password acceptance → **Test Focus**: Test password complexity requirements
- **RISK-005**: Brute force attacks → **Test Focus**: Verify rate limiting on login endpoint
- **RISK-006**: Unauthorized task assignment → **Test Focus**: Ensure only authorized users can assign tasks

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

#### TEST-HP-005: Assign Task to User
**Component**: Task API  
**Endpoint**: POST /api/tasks  
**Feature Tags**: task-assignment, api  
**Test Type**: Happy Path  
**Priority**: P0  
**Prerequisites**: User authenticated, user exists within the same organization  
**Affects**: Task management functionality  

**Given**: User is authenticated and part of the organization  
**When**: POST /api/tasks with:  
```json
{
  "title": "New Task",
  "description": "Task description",
  "assigned_to": "<valid_user_id>"
}
```  
**Then**: 
- Returns 201 Created
- Task is created with `assigned_to` field populated  
**Verify**: 
- Task is stored in database with correct assignment
- `assigned_to` is valid and corresponds to an active user in the same organization  

---

#### TEST-HP-006: Update Task Assignment
**Component**: Task API  
**Endpoint**: PUT /api/tasks/{id}  
**Feature Tags**: task-assignment, api  
**Test Type**: Happy Path  
**Priority**: P0  
**Prerequisites**: Task exists, user is the owner or admin  
**Affects**: Task management functionality  

**Given**: Task with id `{id}` exists and user is authorized  
**When**: PUT /api/tasks/{id} with:  
```json
{
  "assigned_to": "<another_valid_user_id>"
}
```  
**Then**: 
- Returns 200 OK
- Task assignment updated  
**Verify**: 
- Task in database reflects new `assigned_to` user  

---

#### TEST-HP-007: Unassign Task
**Component**: Task API  
**Endpoint**: PUT /api/tasks/{id}  
**Feature Tags**: task-assignment, api  
**Test Type**: Happy Path  
**Priority**: P0  
**Prerequisites**: Task exists, user is the owner or admin  
**Affects**: Task management functionality  

**Given**: Task with id `{id}` exists and user is authorized  
**When**: PUT /api/tasks/{id} with:  
```json
{
  "assigned_to": null
}
```  
**Then**: 
- Returns 200 OK
- Task assignment is cleared  
**Verify**: 
- Task in database has `assigned_to` field set to null  

---

#### TEST-HP-008: Retrieve Task with Assignee Info
**Component**: Task API  
**Endpoint**: GET /api/tasks/{id}  
**Feature Tags**: task-assignment, api  
**Test Type**: Happy Path  
**Priority**: P0  
**Prerequisites**: Task exists  
**Affects**: Task management functionality  

**Given**: Task with id `{id}` exists  
**When**: GET /api/tasks/{id}  
**Then**: 
- Returns 200 OK
- Response includes task details and `assigned_to` info  
**Verify**: 
- Assignee details correctly populated in response  

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

#### TEST-ERR-006: Assign Task to Invalid User
**Component**: Task API  
**Endpoint**: POST /api/tasks  
**Feature Tags**: task-assignment, validation  
**Test Type**: Negative Test  
**Priority**: P1  
**Prerequisites**: User authenticated  
**Affects**: Task creation functionality  

**Given**: User is authenticated  
**When**: POST /api/tasks with:  
```json
{
  "title": "New Task",
  "description": "Task description",
  "assigned_to": "<invalid_user_id>"
}
```  
**Then**: 
- Returns 400 Bad Request
- Error message: "Assigned user must exist in the same organization"  
**Verify**: 
- Validation checks are enforced correctly  

---

#### TEST-ERR-007: Assign Task to User Outside Organization
**Component**: Task API  
**Endpoint**: POST /api/tasks  
**Feature Tags**: task-assignment, validation  
**Test Type**: Negative Test  
**Priority**: P1  
**Prerequisites**: User authenticated  
**Affects**: Task creation functionality  

**Given**: User is authenticated  
**When**: POST /api/tasks with:  
```json
{
  "title": "New Task",
  "description": "Task description",
  "assigned_to": "<user_id_from_different_org>"
}
```  
**Then**: 
- Returns 403 Forbidden
- Error message: "Cannot assign tasks to users outside your organization"  
**Verify**: 
- Proper authorization checks are in place  

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

#### TEST-EDGE-002: Concurrent Token Refresh
**Component**: Token Manager, Session Store  
**Endpoint**: POST /api/auth/refresh  
**Feature Tags**: concurrency, race-condition, token-refresh  
**Test Type**: Concurrency Test  
**Priority**: P1  
**Prerequisites**: Same user, multiple devices  
**Affects**: Multi-device session management  
**Covers Bug**: BUG-AUTH-501  

**Given**: Two devices refresh token simultaneously  
**When**: Both send refresh requests within 100ms  
**Then**: 
- First request succeeds (new tokens issued)
- Second request fails gracefully (401 or 409)
- User not logged out unexpectedly  
**Verify**: 
- Race condition handled
- User can re-authenticate if needed  

---

#### TEST-EDGE-003: Long Session Duration
**Component**: Token Manager  
**Endpoint**: POST /api/auth/refresh  
**Feature Tags**: session-management, token-expiry, edge-case  
**Test Type**: Boundary Test  
**Priority**: P2  
**Prerequisites**: Session idle for extended period  
**Affects**: Long-running sessions  

**Given**: User logged in, session idle for 6 days 23 hours  
**When**: Use refresh token just before 7-day expiry  
**Then**: 
- Refresh succeeds
- New 7-day refresh token issued
- Session extended  
**Verify**: 
- Expiry calculated correctly
- No sudden session termination  

---

#### TEST-EDGE-004: Token with Tampered Signature
**Component**: Token Manager, JWT Validator  
**Endpoint**: GET /api/auth/me  
**Feature Tags**: security, token-tampering, jwt-validation  
**Test Type**: Security Test  
**Priority**: P0  
**Prerequisites**: JWT with modified payload  
**Affects**: Token security, data integrity  

**Given**: Valid JWT token with modified payload  
**When**: GET /api/auth/me with tampered token  
**Then**: 
- Returns 401 Unauthorized
- Error: "Invalid token signature"  
**Verify**: 
- Signature validation works
- Tampering detected
- Security event logged  

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

#### TEST-INT-002: Token Expiry During Active Session
**Component**: Token Manager, Web Dashboard, Auto-Refresh Logic  
**Page**: /dashboard  
**Feature Tags**: integration, token-refresh, session-continuity  
**Test Type**: Integration Test  
**Priority**: P1  
**Prerequisites**: Active user session, token about to expire  
**Consumes**: TG-TASKFLOW-WEB-001 dashboard  

**Given**: User actively using dashboard, token expires  
**When**: Next API call made with expired token  
**Then**: 
- Client detects 401 error
- Automatically refreshes token
- Original request retried with new token
- User uninterrupted  
**Verify**: 
- Refresh happens transparently
- No data loss
- Single refresh attempt (not infinite loop)  

---

#### TEST-INT-003: Logout Across Multiple Tabs
**Component**: Auth Service, Web Dashboard, Cross-Tab Communication  
**Page**: Multiple browser tabs  
**Feature Tags**: integration, logout, multi-tab, session-sync  
**Test Type**: Integration Test  
**Priority**: P2  
**Prerequisites**: User logged in with multiple tabs  
**Consumes**: TG-TASKFLOW-WEB-001  

**Given**: User logged in with 3 browser tabs open  
**When**: Logout clicked in one tab  
**Then**: 
- All tabs detect logout
- All tabs redirect to login page
- Token invalidated globally  
**Verify**: 
- Broadcast channel or storage event used
- No tab retains access  

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

In this updated document, new test scenarios for task assignment functionality have been added, existing tests preserved, and metadata updated accordingly. The version has been incremented and the last updated date has been modified as well.