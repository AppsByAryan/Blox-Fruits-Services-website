# Blox Fruit Services: Live Value Research & Firebase Custom Claims Authentication

A production-grade architectural blueprint to upgrade Blox Fruit Services with server-controlled Firebase Custom Claims admin authentication, Google Search Grounding for autonomous live fruit and gamepass market valuations, hardened Firestore Security Rules with backend price enforcement, single-source-of-truth Firestore migration while preserving Neon as a cold backup, and multi-tier testing verification.

---

## User Review & Critical Decisions

> [!IMPORTANT]
> **Admin Account Verification Gate**: Before any custom claims are minted or permissions modified, please confirm which Google Account should receive the server-controlled `admin: true` custom claim:
> - **Detected Session Email**: `shelendra.ubuntu@gmail.com` *(Recommended default for current session)*
> - **Alternative Previously Mentioned**: `brajbala1989@gmail.com`
> - *Action Required*: Confirm the primary administrator email address upon approving this plan. No administrative claim will be issued without your explicit selection.

> [!NOTE]
> **Zero Destructive Changes & Zero Billing Activation**:
> - Billing will **not** be enabled. Grounding operates strictly within Google AI Studio's 5,000 monthly free search queries allowance.
> - Neon PostgreSQL remains 100% untouched as an offline backup database.
> - No example prices (e.g., speculative 850M Dragon or 570M Kitsune) will ever be seeded or used as real catalogue values.

---

## 1. Overview & Core Concept

### What It Delivers
Blox Fruit Services is a specialized platform offering in-game raid carries, race V4 awakenings, boss defeats, and quest progression, paid strictly via in-game fruits and gamepasses with a strict 50% upfront / 50% post-completion escrow model. This upgrade integrates two critical capabilities:

1. **Firebase Authentication with Server-Minted Custom Claims**: Replaces manual admin password authentication with Google Sign-In backed by cryptographically signed Firebase ID tokens containing `{ admin: true }`. The frontend strictly isolates and hides administrative controls for non-admin accounts, and the backend validates every sensitive endpoint against Firebase Admin SDK.
2. **Grounding-Powered Live Market Research Engine**: Researches live trading values for in-game fruits and gamepasses using Gemini 3.8 Flash with Google Search Grounding. Replaces manual guesswork with verifiable, multi-source evidence, deduplicating domains, detecting price divergence, logging audit trails with source URLs and timestamps, and formatting attribution according to Google standards.

### Target Personas & Security Posture
- **Roblox Players (Customers)**: Browse services, calculate required fruits using 50/50 escrow splits, place orders with unguessable tracking tokens, view transparent trade valuations with direct source citations, and track orders without creating accounts. Customers cannot alter catalogue prices, adjust order totals, modify payment statuses, or escalate permissions.
- **Verified Staff Administrators**: Authenticate via Google Sign-In with validated `admin: true` claims. Gain access to the Live Value Manager, Catalogue Editor, Order Escrow Processing (50% deposit confirmation, completion verification), and Support Ticket Inbox.

---

## 2. User Experience & Visual Design

### Key User Flows

```
[Customer Journey]
Browse Catalogue ──> Select Service ──> Input In-Game Fruits ──> Live 50/50 Escrow Breakdown ──> Submit Order ──> Track via Private Token
                                                                                                                        │
                                                                                                                        ▼
[Staff Admin Journey]                                                                                   Discord Proofs Wall
Google Sign-In ──> Firebase ID Token Check ──> Custom Claims Verification (admin: true)
                         │
                         ├── If Claim Missing: Standard Customer View (Admin UI Strictly Hidden)
                         └── If Claim Valid: Full Admin Console Access
                                   │
                                   ├── Live Value Research Tab (Domain-deduplicated citations & review)
                                   ├── Catalogue & Escrow Price Manager (Backend validated)
                                   ├── Order Management (Verify 50% deposit, assign staff, complete)
                                   └── Support Tickets & Evidence Inbox
```

### Aesthetic Direction & Visual Identity
- **Atmosphere**: High-contrast, dark-mode gaming console aesthetic with deep slate/obsidian foundations (`bg-slate-950`), subtle border definitions (`border-slate-800`), and warm amber accents (`text-amber-400`, `bg-amber-500/20`).
- **Typography**: Clean, readable sans-serif (`Plus Jakarta Sans`) for navigation and content headers, paired with monospace (`JetBrains Mono`) for escrow figures, order tracking tokens, and trading values.
- **Attribution Display**: In accordance with Google Search Grounding attribution guidelines, grounded research cards prominently display clickable source links with verified domain labels (e.g., `bloxfruitsvalues.com`, `fruityblox.com`) alongside retrieval timestamps.
- **Administrative UI Isolation**: All admin buttons, tabs, edit toggles, and staff-only notes are strictly excluded from the DOM when a user lacks verified administrative claims—no disabled buttons or empty containers that leak administrative capability.

---

## 3. Key Product Decisions & Trade-Offs

### 1. No Example Prices as Real Values
- **Principle**: Speculative discussion numbers (e.g., 850M for Dragon, 570M for Kitsune, 180M for Gas, 15M for Buddha) must **never** be inserted as real catalogue values, seeded into database schemas, or used to overwrite existing records.
- **Implementation**: The existing catalogue prices remain intact. Any newly tracked or unresearched item displays a clear `"Value pending research"` badge. A price update is applied if and only if verified through a live research cycle backed by at least two independent domains, HTTP status verification, and audit timestamps.

### 2. Google Search Grounding Economics & Spending Caps
- **Official Model & Pricing**: `gemini-3.8-flash` in Google AI Studio.
- **Search Grounding Free Tier**: **5,000 free search queries per month** across Gemini 2.0 and 3 series models per project.
- **Overage Rate (If Ever Activated)**: $14.00 per 1,000 search queries ($0.014 per query) beyond 5,000/month. Token rates are $0.075 / 1M input tokens and $0.30 / 1M output tokens.
- **Catalogue Sizing**: Current service and fruit catalogue comprises ~35-40 distinct items. Running a complete research sweep uses ~40 search grounding queries.
- **Monthly Usage Projection**:
  - Weekly update (4 runs/mo): 160 queries/month (3.2% of free allowance).
  - Daily update (30 runs/mo): 1,200 queries/month (24% of free allowance).
  - Cost under free tier: **$0.00 / month**.
- **Hard Safeguard Controls**:
  - Billing is **not** activated.
  - Rate limiting caps research queries at 100 queries/day.
  - Concurrency mutex prevents overlapping executions.
  - Cooldown enforce a minimum 4-hour window between full catalogue cycles.

### 3. Valid Firestore Security Rules & Server-Authoritative Pricing
- **Elimination of Invalid Syntax**: Replace non-existent `incoming()` helper calls with valid Firestore Security Rules primitives: `request.resource.data` (incoming write) and `resource.data` (existing document).
- **Anti-Tampering Guard**: Customers cannot set or alter `priceHundredths`, `depositRequiredHundredths`, `paymentStatus`, `status`, or `staffNotes`.
- **Backend Price Calculation**: When an order is submitted to `/api/orders`:
  1. The server fetches the canonical service from Firestore by ID.
  2. The server computes the exact 50% deposit and 50% balance using the verified integer half-up formula (`Math.ceil(total / 2)` and `Math.floor(total / 2)`).
  3. The server sets initial status to `'pending_deposit'` and signs the record. Client-supplied price overrides are rejected.

### 4. Server-Controlled Firebase Custom Claims
- **Claim Structure**: `request.auth.token.admin == true`.
- **Provisioning**: Custom claims are minted exclusively on the server using `firebase-admin/auth` (`setCustomUserClaims(uid, { admin: true })`).
- **No Self-Assignment**: Firestore Security Rules strictly block any write to role fields or administrative document paths from client SDKs.
- **Dual-Layer Enforcement**:
  - *Client*: `api.adminCheckSession` verifies `user.getIdTokenResult().claims.admin === true`. Unauthorized users cannot view admin views or initiate admin actions.
  - *Server*: All `/api/admin/*` endpoints extract the Bearer ID token, verify it via `adminAuth.verifyIdToken(token)`, and confirm `decodedToken.admin === true` before processing.

### 5. Single Production Source of Truth (Firestore) & Untouched Neon Backup
- **Production Truth**: Cloud Firestore database `ai-studio-bloxfruitsservic-0b8414ac-1b89-445c-ba0b-47f4771a2aad` is the authoritative production database.
- **Neon Standalone Backup**: Neon PostgreSQL remains 100% frozen and untouched as a verified cold backup. No active writes route to Neon, and no dual-write fallback is permitted, preventing split-brain or divergent order records.
- **Decommissioning Gate**: Neon will not be modified or decommissioned without explicit subsequent user approval.

### 6. Secured Cron Endpoint & Execution Safeguards
- **Protected Route**: `/api/cron/research-update` requires `Authorization: Bearer <CRON_SECRET>` or `x-cron-secret: <CRON_SECRET>`.
- **Execution Locks**: Memory/Firestore mutex flag (`research_in_progress`) prevents duplicate concurrent executions.
- **Minimum Cooldown**: 4 hours between full passes. Pings within the cooldown return HTTP 200 with `{ status: 'skipped', reason: 'cooldown_active' }`.

### 7. Multi-Source Independent Value Verification & Conflict Resolution
- **Apex Domain Deduplication**: Sources are grouped by root domain (e.g. `bloxfruitsvalues.com` vs `fruityblox.com`). Subdomains and scrapers echoing the same content do not qualify as independent sources.
- **Consensus Requirement**: Auto-updating requires at least 2 distinct independent apex domains with agreement within ±15%.
- **Divergence Handling**: When valuations conflict by >15%, the system records the raw findings, marks the item as `needs_review`, keeps the previous approved value active, and alerts staff.
- **Audit History**: Every valuation records `itemKey`, `oldValue`, `newValue`, `sources` (URLs + snippets), `timestamp`, and `approvalStatus`.

### 8. Strict Preservation of Core Workflows
- **Service Catalogue**: Raid carries, race awakenings, boss assistance, and custom quest bundles remain identical.
- **50/50 Escrow Math**: Deposit and balance calculations using integer hundredths are preserved.
- **Order Tracking**: Order IDs (`BX-XXXXXX`) and 256-bit unguessable public tracking tokens (`/order?token=...`) continue unchanged.
- **Discord Community Proofs**: Screenshot submission and verified proof wall with customer anonymity toggles are preserved.
- **Customer Support**: Ticket and enquiry handling remain fully operational.

---

## 4. Technical Architecture & Data Strategy

### System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                 CLIENT LAYER (Browser)                                  │
├───────────────────────────────────────────┬─────────────────────────────────────────────┤
│               Public Views                │             Admin Console Views             │
│  - LandingPage & Service Catalog          │  - LiveValueResearchTab (Market Trends)     │
│  - 50/50 Escrow Calculator (OrderForm)    │  - AdminCatalogManager (CRUD & Pricing)     │
│  - OrderStatusPage (via Public Token)     │  - AdminOrdersList & OrderDetails           │
│  - SupportPage & Discord Proofs Wall      │  - SupportInboxTab                          │
│                                           │  * Rendered ONLY when claims.admin === true │
└─────────────────────┬─────────────────────┴──────────────────────┬──────────────────────┘
                      │                                            │
           Public API & Token Queries                     Bearer Firebase ID Token
                      │                                            │
                      ▼                                            ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                               BACKEND LAYER (Express Server)                            │
├───────────────────────────────────────────┬─────────────────────────────────────────────┤
│             Public Middleware             │              Admin Middleware               │
│  - /api/services (Public Catalog)         │  - requireAdminAuth (verifyIdToken)         │
│  - /api/orders (Server price calculation) │  - /api/admin/services (Create/Edit)        │
│  - /api/orders/:token (Public lookup)     │  - /api/admin/orders/:id (Confirm deposit)  │
│  - /api/market-trends (Cached valuations) │  - /api/admin/research (Manual trigger)     │
├───────────────────────────────────────────┴─────────────────────────────────────────────┤
│                                Background Automation                                    │
│  - /api/cron/research-update (Protected by CRON_SECRET, 4h cooldown, mutex lock)        │
│  - Gemini 3.8 Flash + Google Search Grounding Engine (Rate-limited, max 100/day)        │
└─────────────────────┬────────────────────────────────────────────┬──────────────────────┘
                      │                                            │
           Direct Firestore SDK                        Firebase Admin SDK
                      │                                            │
                      ▼                                            ▼
┌──────────────────────────────────────────────────┐  ┌───────────────────────────────────┐
│        PRODUCTION DATABASE: Cloud Firestore      │  │        COLD BACKUP: Neon DB       │
│  - /services/{serviceId}                         │  │  - PostgreSQL tables frozen       │
│  - /orders/{orderId} (Status & 50/50 escrow)     │  │  - Zero production writes         │
│  - /research_results/{resultId} (Citations/Audit)│  │  - Untouched until user approval  │
│  - /enquiries/{enquiryId} (Support tickets)      │  └───────────────────────────────────┘
│  - /service_evidence/{evidenceId} (Proofs)       │
└──────────────────────────────────────────────────┘
```

### Firestore Security Rules Architecture

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Default deny catch-all
    match /{document=**} {
      allow read, write: if false;
    }

    // Helper functions
    function isSignedIn() {
      return request.auth != null;
    }

    function isAdmin() {
      return isSignedIn() && request.auth.token.admin == true;
    }

    function isCustomerOwner(userId) {
      return isSignedIn() && request.auth.uid == userId;
    }

    // 1. Services Catalogue: Public read, Admin write
    match /services/{serviceId} {
      allow read: if true;
      allow write: if isAdmin();
    }

    // 2. Orders:
    // Read: Admins, or authenticated customer matching order customerUid
    // Create: Verified customers creating pending orders with initial status only
    // Update/Delete: Admins ONLY (prevent customer tampering with prices or payment status)
    match /orders/{orderId} {
      allow get: if isAdmin() || (isSignedIn() && resource.data.customerUid == request.auth.uid);
      allow list: if isAdmin();
      
      allow create: if isSignedIn()
        && request.resource.data.customerUid == request.auth.uid
        && request.resource.data.status == 'pending_deposit'
        && !request.resource.data.keys().hasAny(['depositConfirmedAt', 'depositConfirmedBy', 'completedAt', 'staffNotes']);

      allow update: if isAdmin();
      allow delete: if isAdmin();
    }

    // 3. Research Results & Verifications
    match /research_results/{resultId} {
      allow read: if true;
      allow write: if isAdmin();
    }

    // 4. Value History Audit Log
    match /value_history/{historyId} {
      allow read: if true;
      allow write: if isAdmin();
    }

    // 5. Support Enquiries
    match /enquiries/{enquiryId} {
      allow get, list: if isAdmin();
      allow create: if true; // Public contact form
      allow update, delete: if isAdmin();
    }

    // 6. Community Proofs Wall
    match /service_evidence/{evidenceId} {
      allow read: if resource.data.staffApprovedPublic == true || isAdmin();
      allow write: if isAdmin();
    }
  }
}
```

### Component State & Claims Verification Mapping

| Component | Visible To | Trigger / Action | Authorization Enforcement |
| :--- | :--- | :--- | :--- |
| `AdminLoginPage` | Public / Staff | Click "Sign in with Google" | `signInWithPopup(auth, provider)` retrieves ID token; backend validates custom claim. |
| `AdminDashboardPage` | Admin only | Navigate to `/admin` | Decodes token; if `claims.admin !== true`, redirects to home and unmounts admin views. |
| `LiveValueResearchTab` | Admin only | Manual research trigger | `POST /api/admin/research/trigger` with Bearer token. Rate-limited on backend. |
| `CatalogueManager` | Admin only | Update price or service info | `PUT /api/admin/services/:id` verified with Admin SDK. |
| `OrderEscrowReview` | Admin only | Confirm 50% deposit / mark complete | `PATCH /api/admin/orders/:id/status` validated by Admin SDK; emits tracking update. |
| `OrderStatusPage` | Public | Enter private tracking token | `GET /api/orders/track?token=...` returns sanitized order state without leaking staff notes. |

---

## 5. Verification & Testing Protocol

Execution of tests will be conducted across five strict boundaries upon approval:

1. **Unit & Integration Suite (`npm test`)**:
   - `tests/calculator.test.ts`: 50/50 escrow math, integer hundredths, odd splits.
   - `tests/public_orders.test.ts`: Order creation, unguessable token validation, public status lookups.
   - `tests/admin_actions.test.ts`: Deposit confirmation, carry progression, proof publishing.
   - `tests/value_research.test.ts`: Google Search Grounding parser, domain deduplication, conflict divergence (>15%), cron secret auth.
   - `tests/admin_catalog_and_auth.test.ts`: Firebase ID token validation, custom claim verification, rejection of non-admin tokens.
2. **ESLint & Rules Syntax Validation (`npm run lint`)**:
   - Runs TypeScript checks and `@firebase/eslint-plugin-security-rules` against `firestore.rules` to ensure zero syntax or scope errors.
3. **Production Build Compilation (`npm run build`)**:
   - Executes Vite bundle compilation and typecheck to verify production readiness.
4. **Firestore Security Rules Unit Tests**:
   - Executes tests against Firestore Rules using `@firebase/rules-unit-testing` to verify:
     - Rejection of customer price alteration on order creation.
     - Rejection of status modification by non-admin users.
     - Enforcement of `admin: true` claim on sensitive collections.
5. **Admin Claim Authorization Tests**:
   - Verifies that a valid Google user lacking `{ admin: true }` receives 403 Forbidden on all administrative endpoints and cannot render administrative components.
   - Verifies that the verified administrator account receives full authorized access.
