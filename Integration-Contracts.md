# Integration Contracts

**Parent:** [[AI-Coding-Best-Practices]]
**Related:** [[Canonical Documentation]] · [[Externalized State]] · [[Review and Audit]] · [[Agent Self-Verification]]

---

## What Is an Integration Contract?

An Integration Contract is a **mandatory section in every feature plan** that explicitly specifies how new code connects to existing code. It documents:

- Which files will be modified and why
- Which files will be created and where they fit
- What API contracts (routes, endpoints, interfaces) are introduced or changed
- What data model changes are required
- How new components integrate with existing navigation and state
- What code depends on this feature, and what this feature depends on

The Integration Contract is not aspirational. It's not "we'll connect it if we have time." It's the executable specification for integration — the section that gets verified as part of the deep audit in Step 10 of the [[AI-Coding-Best-Practices#The Development Loop|Development Loop]].

---

## The Orphaned Code Problem

Orphaned or disconnected code is identified as the **#1 failure mode in AI-assisted development** across multiple independent research initiatives.

### What It Looks Like

The AI writes code that works perfectly in isolation but isn't connected to anything:

- A database table is created but never queried or populated
- An API route is implemented but never registered in the router
- A React component is written but never imported or rendered anywhere
- A utility function exists but is never called
- A state management slice is created but never integrated with the store
- A webhook endpoint exists but is never triggered from anywhere

The code passes unit tests. It compiles. It looks correct. But in the running application, it's dead — a ghost in the codebase.

### Why It Happens

This failure mode is peculiar to AI-assisted development because:

1. **Locality of reasoning.** The AI tends to reason about tasks in isolation — "implement the booking table" or "add a confirmation email route." Each task works. But the AI doesn't automatically verify that the pieces connect to *each other*.

2. **Missing context integration.** Even when the AI reads the codebase carefully, it may not fully understand all the connection points — where routers are registered, how components are composed, how state flows through the application. The explicit connections often live in configuration files, entry points, or patterns that aren't immediately obvious.

3. **No automatic verification.** Unlike compilation errors or test failures, orphaned code produces no signal. A function that's never called still compiles. A table that's never used still migrates. The orphaned code doesn't break anything — it just sits there.

4. **The human review bottleneck.** In human code review, spotting orphaned code requires reading the entire feature implementation plus understanding what should have been connected. This is cognitively expensive. Under deadline pressure (which AI-assisted projects often face), it's easy to miss. Research on AI code quality finds that reviewers often miss integration failures even when explicitly looking for them.

### The Cost

- **Silent bugs that surface late.** The orphaned code lives in production until a feature that depends on it fails mysteriously.
- **Rework and refactoring.** When the orphaned code is discovered, it often requires plumbing entire layers of the application to connect it.
- **Trust erosion.** Teams that experience repeated orphaned code failures start over-validating AI output, reducing the time savings the AI provides.

---

## The Integration Contract Template

Every feature plan must include an Integration Contract section. Use this template:

### Files Modified

```
- [app/api/routes.ts]
  - Adds POST /bookings endpoint
  - Registers BookingsController

- [app/store/index.ts]
  - Imports and registers bookings reducer
  - Hooks into store initialization

- [app/components/Navigation.tsx]
  - Adds link to /bookings page
  - Updates navigation state to include bookings
```

### Files Created

```
- [app/api/controllers/BookingsController.ts]
  - New: Handles all booking-related HTTP logic
  - Depends on: BookingModel, BookingValidator, BookingService
  - Used by: POST /bookings route in routes.ts

- [app/models/BookingModel.ts]
  - New: Defines Booking table schema and methods
  - Depends on: Database connection from app/db/index.ts
  - Used by: BookingsController, bookings state slice

- [app/store/slices/bookings.ts]
  - New: Redux slice for booking state management
  - Depends on: BookingModel for type definitions
  - Used by: BookingForm component, BookingList component
```

### API Contracts

```
POST /api/bookings
  Request:
    {
      "userId": string,
      "serviceId": string,
      "date": ISO8601 string,
      "notes": string (optional)
    }
  Response:
    {
      "id": string,
      "status": "pending" | "confirmed" | "cancelled",
      "createdAt": ISO8601 string
    }
  Error responses: 400 (invalid input), 401 (unauthorized), 409 (conflict)

GET /api/bookings/:id
  Response: { ...booking object }
  Error responses: 404 (not found), 401 (unauthorized)
```

### Data Model Changes

```
- New table: Bookings
  - Columns: id (PK), user_id (FK to Users), service_id (FK to Services),
            date (timestamp), status (enum), created_at, updated_at
  - Indexes: (user_id, created_at), (service_id, date)
  - Migration: Add triggers to update Users.last_booking_date

- Modified table: Services
  - Add column: booking_count (integer, default 0)
  - Add index: (booking_count) for sorting by popularity
  - Backward compatible: existing queries unaffected
```

### Component Interfaces

```
- [BookingForm] (new)
  - Props: { onSubmit: (booking) => void, serviceId: string }
  - Renders in: Page /bookings/new (new page)
  - Dispatches: Redux actions booking/createBooking
  - Uses: BookingValidator (import from app/utils)

- [BookingList] (new)
  - Props: { userId: string, status?: string }
  - Renders in: Page /my-bookings (existing, modified)
  - Reads from: Redux state.bookings.list
  - Depends on: BookingCard component (existing, no changes)

- [Navigation] (modified)
  - New link: "My Bookings" → /my-bookings
  - Icon: Uses existing icon library (material-ui)
  - Auth guard: Only visible when user.isAuthenticated
```

### Dependencies

```
Upstream (what this feature depends on):
  - Users authentication system (app/auth)
    * Uses: user context, login validation

  - Services catalog (app/models/ServiceModel)
    * Uses: service list, service details

  - Database connection (app/db)
    * Uses: knex instance for migrations and queries

Downstream (what will depend on this feature):
  - Notifications system (planned)
    * Will trigger: booking confirmation emails
    * Will read: Bookings table (status changes)

  - Admin dashboard (planned)
    * Will read: Bookings data for analytics

  - Payment system (planned)
    * Will depend on: Booking.status = "confirmed" trigger
```

---

## Integration Contract Verification Checklist

During the deep audit (Step 10), verify every point in the Integration Contract:

### Files Modified
- [ ] Does each referenced file exist?
- [ ] Does each modification appear in the diff?
- [ ] Are imports added to files that use new exports?
- [ ] Are configuration updates applied (routes registered, state slice added to store)?
- [ ] Do modifications maintain backward compatibility where needed?

### Files Created
- [ ] Is each file created in the correct directory with correct naming?
- [ ] Does the file get imported and used somewhere (not orphaned)?
- [ ] Do type exports match what dependents expect?
- [ ] Are dependencies available (no circular imports)?
- [ ] Does the file follow project conventions (linting, structure)?

### API Contracts
- [ ] Are all new routes registered in the router?
- [ ] Do request/response shapes match the contract in all code paths?
- [ ] Are all error cases listed in the contract actually handled?
- [ ] Are all endpoints tested (unit test for handler + integration test for route)?
- [ ] Do authentication/authorization checks match the spec?

### Data Model Changes
- [ ] Do migration files exist and are they in the correct order?
- [ ] Are foreign key constraints created and tested?
- [ ] Are indexes created for all columns listed in the contract?
- [ ] Are existing queries still valid (backward compatibility)?
- [ ] Do ORM models (if applicable) reflect the new schema?

### Component Interfaces
- [ ] Is each new component actually rendered somewhere?
- [ ] Do props match the contract in the render locations?
- [ ] Are state dispatches received and handled (Redux/Zustand/Context)?
- [ ] Are all component imports in place?
- [ ] Do conditional renders (auth guards, feature flags) work as intended?

### Dependencies
- [ ] Does the code actually import from upstream dependencies?
- [ ] Are all upstream dependencies available (versions, installation)?
- [ ] Have downstream systems been notified of changes they depend on?
- [ ] If new tables are added, do downstream readers have appropriate migrations?
- [ ] Are there any circular dependencies (A depends on B, B depends on A)?

---

## Common Failure Patterns

### 1. Dead Routes

**Pattern:** API route is implemented and tested but never registered in the router.

```javascript
// ORPHANED: Route exists but isn't wired
// app/api/controllers/BookingsController.ts
export async function createBooking(req, res) {
  // Implementation is correct
}

// ❌ File: app/api/routes.ts
// Route was never added:
// router.post('/bookings', createBooking)
```

**How to catch it:** Integration Contract must list exactly which files register the route. Verification checks that registration code exists in routes.ts.

### 2. Orphaned Database Tables

**Pattern:** Table is created (migration runs) but never queried.

```sql
-- ORPHANED: Table exists but nothing reads/writes it
CREATE TABLE bookings (
  id UUID PRIMARY KEY,
  user_id UUID,
  service_id UUID,
  date TIMESTAMP
);

-- No INSERT, UPDATE, or SELECT from bookings anywhere in the codebase
```

**How to catch it:** Integration Contract specifies the table. Verification searches the codebase for queries matching the table name — if count = 0 after excluding the migration, it's orphaned.

### 3. Unused Components

**Pattern:** React component is written but never imported anywhere.

```typescript
// ORPHANED: Component is well-written but unused
// app/components/BookingForm.tsx
export function BookingForm({ serviceId, onSubmit }) {
  // Implementation is solid
}

// ❌ BookingForm is imported nowhere
// ❌ Page /bookings/new doesn't exist or uses a different component
```

**How to catch it:** Integration Contract says "BookingForm renders in Page /bookings/new". Verification checks that /bookings/new exists and imports BookingForm.

### 4. Missing Navigation Links

**Pattern:** Page exists but isn't accessible from navigation.

```typescript
// Page exists at /my-bookings
// Component tree:
//   Layout
//     Navigation
//       - Home link
//       - Services link
//       - ❌ Missing: My Bookings link
//     Router → /my-bookings works if URL is typed directly

// But users can't discover it
```

**How to catch it:** Integration Contract specifies "Navigation updated with link to /my-bookings". Verification renders Navigation component and checks for the link.

### 5. Disconnected State Management

**Pattern:** Redux action/reducer exists but is never dispatched.

```typescript
// ORPHANED: Slice is well-structured but nothing uses it
// app/store/slices/bookings.ts
const bookingsSlice = createSlice({
  name: 'bookings',
  // Full implementation
});

// ❌ store/index.ts doesn't include bookingsSlice
// OR
// ❌ No component dispatches booking actions
```

**How to catch it:** Integration Contract specifies "store/index.ts registers bookings reducer". Verification checks that registration code exists. Also checks that at least one action is dispatched from a component.

### 6. Missing Error Handling Paths

**Pattern:** API contract lists error response codes but handlers don't implement them.

```typescript
// Contract says:
// POST /bookings
//   Error responses: 400 (invalid input), 401 (unauthorized), 409 (conflict)

// ❌ Implementation:
export async function createBooking(req, res) {
  const booking = await Booking.create(req.body);
  res.json(booking); // No validation, no 400. No auth check, no 401.
                     // No conflict detection, no 409.
}
```

**How to catch it:** For each error code in the API contract, verify that there's a corresponding code path and test case.

---

## Integration Contracts Across Architectures

### Monolith (Single Codebase, Single Deployment)

All files, tables, and routes exist in one repo. Integration Contract focuses on file interconnections, import chains, and route registration:

```
Files Modified: [app/routes.ts], [app/models/User.ts], [app/store/index.ts]
API Contracts: POST /api/users/register, GET /api/users/me
Data Model: Users table, Sessions table
Component Interfaces: LoginForm, ProfilePage
Dependencies: AuthContext → LoginForm → ProfilePage
```

**Verification is straightforward:** All references are to files in the same repo. Grep and import analysis suffice.

### Microservices (Multiple Services)

Each service is a separate repo with separate deployment. Integration Contract must specify cross-service dependencies:

```
This is the Booking Service.

Upstream dependencies:
  - User Service: POST /auth/verify-token, GET /users/:id
  - Service Catalog Service: GET /services/:id

Downstream dependencies:
  - Notification Service: Will consume Bookings.created events from message queue
  - Billing Service: Will read bookings from database replication stream

Files Modified: [services/booking/routes.ts], [services/booking/models/Booking.ts]
API Contracts: POST /bookings, GET /bookings/:id (BookingService)
Data Model: Bookings table, with foreign key: (service_id → ServiceCatalogService)
Dependencies:
  - HTTP dependency: Calls UserService.verify-token on every request
  - Event dependency: Publishes to booking.created topic (Kafka/RabbitMQ)
  - Data dependency: Reads from ServiceCatalog read replica
```

**Verification is harder:** Must check cross-service contracts. Tools:
- Verify HTTP calls match actual service schemas (hit /swagger, request a live endpoint)
- Verify event publishing matches consumer schemas (check message queue topic)
- Verify data replication is configured (check replication lag)

### Frontend/Backend Split (Separate Repos)

Frontend and backend are deployed separately but tightly coupled. Integration Contract lives in both:

**Backend Integration Contract:**
```
New endpoints: POST /api/bookings, GET /api/bookings/:id
Response schema: { id, userId, serviceId, date, status, createdAt }
Auth: Bearer token required
```

**Frontend Integration Contract:**
```
New pages: /bookings, /bookings/new
API calls: POST /api/bookings, GET /api/bookings/:id
State: Redux slice for bookings
Components: BookingForm, BookingList
Navigation: Link to /bookings added to Navigation
```

**Verification:** Check that frontend API calls match backend response schema exactly. Schema mismatches are the most common failure mode in frontend/backend splits.

---

## Integration Contract Examples

### Example 1: Booking System Feature

**Parent feature plan:** [Booking System Implementation]

**Integration Contract:**

**Files Modified:**
- `app/api/routes.ts` — Register POST /bookings and GET /bookings/:id
- `app/store/index.ts` — Import and register bookings reducer
- `app/components/Navigation.tsx` — Add link to /bookings

**Files Created:**
- `app/api/controllers/BookingsController.ts` — Handles booking endpoints
- `app/models/BookingModel.ts` — Booking schema and queries
- `app/store/slices/bookings.ts` — Redux state for bookings
- `app/pages/Bookings.tsx` — Main bookings page
- `app/pages/BookingDetail.tsx` — Single booking detail view
- `app/components/BookingForm.tsx` — Form component for creating bookings
- `app/components/BookingList.tsx` — List component for displaying bookings

**API Contracts:**
- `POST /api/bookings` — Create booking (auth required)
- `GET /api/bookings/:id` — Fetch booking details (auth required)
- `GET /api/bookings` — List user's bookings with filters (auth required)

**Data Model Changes:**
- New table: `bookings` (id, user_id, service_id, date, status, created_at, updated_at)
- Foreign key: bookings.user_id → users.id
- Foreign key: bookings.service_id → services.id
- Index: (user_id, created_at) for fast user booking retrieval

**Component Interfaces:**
- `BookingForm` — Props: { serviceId, onSubmit }, used by /bookings/new
- `BookingList` — Props: { userId, status }, used by /bookings
- `Navigation` — Updated: added "My Bookings" link

**Dependencies:**
- Upstream: Users auth system, Services catalog, Database
- Downstream: Notification system (will email confirmations), Admin dashboard (will query bookings)

**Verification Result:** ✓ All files created and connected. All routes registered. All components rendered. All tables migrated. All redux actions dispatched. Integration complete.

### Example 2: Authentication Feature

**Parent feature plan:** [User Authentication Implementation]

**Integration Contract:**

**Files Modified:**
- `app/store/index.ts` — Register auth reducer
- `app/App.tsx` — Wrap app with AuthProvider
- `app/api/routes.ts` — Register POST /auth/login, POST /auth/logout

**Files Created:**
- `app/context/AuthContext.tsx` — Auth state (user, token, loading)
- `app/store/slices/auth.ts` — Redux slice (redundant with context, but handles persistence)
- `app/api/controllers/AuthController.ts` — Login/logout logic
- `app/models/SessionModel.ts` — Session schema and validation
- `app/pages/Login.tsx` — Login page
- `app/pages/Register.tsx` — Registration page
- `app/components/ProtectedRoute.tsx` — Route guard component

**API Contracts:**
- `POST /auth/login` — Login user, returns JWT token
- `POST /auth/logout` — Invalidate session
- `POST /auth/register` — Create new user account
- `GET /auth/me` — Get current user (protected)

**Data Model Changes:**
- Modify Users table: add password_hash, email_verified, created_at columns
- New table: Sessions (id, user_id, token_hash, expires_at, created_at)
- Index: Sessions.user_id for session lookup
- Trigger: Delete expired sessions daily

**Component Interfaces:**
- `ProtectedRoute` — Wraps routes that require auth
- `LoginForm` — Props: { onLoginSuccess }, used by /login
- `Navigation` — Updated: show "Login" when not authed, "Logout" when authed

**Dependencies:**
- Upstream: None (auth is foundational)
- Downstream: All protected pages, all API routes requiring auth

**Verification Result:**
- ✓ AuthContext provides user state to entire app
- ✓ Redux auth slice syncs with localStorage for persistence
- ✓ ProtectedRoute guards /bookings, /my-profile, /admin
- ✓ All API routes check auth token before processing
- ✓ Sessions table created and indexed
- ✓ Navigation shows correct buttons based on auth state
- ❌ **Issue found:** /register page created but no link in Navigation. Added "Sign Up" link to fix.
- ❌ **Issue found:** Password reset flow specified but not implemented. Moved to future phase.

---

## Anti-Patterns and How to Avoid Them

### Anti-Pattern 1: Vague Contracts

**Bad:**
```
Files Modified: routes, models, components (which ones?)
Dependencies: stuff in the database (what stuff?)
```

**Good:**
```
Files Modified:
- app/api/routes.ts (adds POST /bookings)
- app/models/BookingModel.ts (adds queries)
- app/store/slices/bookings.ts (NEW, not modified)

Dependencies:
- Upstream: UserModel (for user_id validation), ServicesModel (for service_id FK)
- Downstream: NotificationService (will read bookings.status changes)
```

**Why it matters:** Vague contracts can't be verified. Verification finds nothing, and integration bugs slip through.

### Anti-Pattern 2: Contracts That Never Get Verified

**Bad:** Integration Contract is written as part of the plan but not checked during audit.

**Good:** Step 10 (Deep Audit) includes a dedicated sub-task:
```
[ ] Deep Audit → Integration Contract Verification
  - [ ] Verify all Files Modified exist and are changed as specified
  - [ ] Verify all Files Created are connected and not orphaned
  - [ ] Verify all API routes are registered
  - [ ] Verify all data model changes are migrated
  - [ ] Verify all components are rendered
  - [ ] Verify all dependencies (upstream and downstream) are satisfied
```

**Why it matters:** Without verification, the contract is decoration. Making verification mandatory and auditable is what prevents orphaned code.

### Anti-Pattern 3: Contracts Without Clear Upstream/Downstream

**Bad:**
```
Dependencies: Depends on stuff, other stuff might use this
```

**Good:**
```
Upstream (what this depends on):
  - app/models/UserModel.ts — Uses for user_id validation in createBooking
  - app/auth/middleware.ts — Uses for bearer token verification on protected routes

Downstream (what depends on this):
  - app/pages/AdminDashboard.tsx — Will query Bookings table for analytics
  - NotificationService (separate repo) — Will subscribe to bookings.created events
```

**Why it matters:** If you don't explicitly list what depends on this feature, you won't notify those downstream systems when the feature changes. Breaking changes leak into dependent code.

### Anti-Pattern 4: Contracts That Don't Account for State

**Bad:**
```
Component Interfaces: BookingForm component
```

**Good:**
```
Component Interfaces:
- BookingForm (new)
  - Props: { onSubmit: (booking) => void, serviceId: string }
  - Dispatches Redux action: bookings/createBooking
  - Reads from Redux state: auth.user (for user_id)
  - Rendered in: app/pages/BookingCreate.tsx (new page, imported and rendered in Router)
```

**Why it matters:** State management is invisible if not documented. If BookingForm dispatches redux/createBooking but that action is never registered, or if it reads auth.user but auth isn't initialized, the component breaks silently.

### Anti-Pattern 5: Post-Hoc Contracts

**Bad:** Integration Contract is written *after* implementation to match what was built.

**Good:** Integration Contract is written *before* implementation and used as verification spec.

**Why it matters:** Post-hoc contracts are narratives, not specifications. They describe what happened, not what should happen. They can't catch orphaned code — they only rationalize it.

---

## Getting Started: Integration Contracts in Your Workflow

If you're not using Integration Contracts yet:

1. **Add to feature plan template.** Make Integration Contract a required section (along with success criteria and scope bounds).

2. **Write it before implementation.** During planning phase (Steps 2–4), draft the contract. It forces you to think through connections before code is written.

3. **Verify it during audit (Step 10).** Make Integration Contract verification a mandatory part of the deep audit checklist. Don't move to Step 11 (Remediation) until IC is verified.

4. **Keep a living register.** Maintain a doc or spreadsheet listing all active Integration Contracts:
   - Feature name
   - Status (planned, in-progress, audited, shipped)
   - Verification date
   - Known issues or downstream impacts

5. **Build verification tooling.** Write scripts that automate parts of verification:
   - Check that API routes are registered: grep routes.ts for endpoint names
   - Check that components are rendered: grep for component imports in pages
   - Check that tables are queried: grep for table names in migrations + source code
   - Check that redux slices are registered: parse store/index.ts

6. **Review with the team.** Include Integration Contract verification in code review. Ask: "Are all the connections in the IC actually in the code?"

---

## Integration Contracts and Canonical Documentation

Integration Contracts and Canonical Documentation serve complementary purposes and should reference each other:

**Integration Contract** (written before/during implementation):
- Specifies: "Here's what this feature will connect to"
- Temporal scope: What's being built right now
- Audience: AI agents, implementation team, reviewers
- Update: Once per feature, before code review

**Canonical Documentation** (updated after implementation):
- Documents: "Here's what this feature actually is and how to use it"
- Temporal scope: The permanent state of that feature in the codebase
- Audience: Future teams building features that depend on this
- Update: After each feature ships (Step 14 of Development Loop)

**Example:**
- Integration Contract says: "BookingForm component is created and rendered in /bookings/new page. Redux bookings slice is registered in store. POST /api/bookings endpoint created."
- Canonical Document (written after shipment) says: "Booking system lets users request services for a specific date. Here's how to create bookings, query them, and handle status changes. Here are the data models. Here's what depends on bookings (notifications, admin dashboard)."

When building the next feature that depends on bookings, the research phase (Step 1) reads the Canonical Documentation. The Canonical Doc references "see Integration Contract for implementation details" if the reviewer needs to understand the internal wiring.

---

## Sources

### Failure Mode Research

- Anthropic, "2026 Agentic Coding Trends Report"
- Microsoft, "Taxonomy of failure modes in AI agents" (2025)
- IEEE Spectrum, "Silent Failures in AI-Generated Code" (2025)
- VentureBeat, "The Hidden Cost of AI Code: Integration Failures and Technical Debt" (2025)

### Related Best Practices

- Boris Cherny, Claude Code workflow — integration verification patterns (2026)
- Addy Osmani, "Agentic Engineering" — architectural validation in AI workflows (2026)
- Thoughtworks, "Spec-driven development" — specification precision and verification (2025)
