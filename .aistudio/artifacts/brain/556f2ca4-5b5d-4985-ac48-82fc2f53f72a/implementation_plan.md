# REVISED IMPLEMENTATION PLAN: BLOX FRUITS LIVE VALUE RESEARCH & FIREBASE AUTHENTICATION

## Status: Awaiting User Approval (Zero Destructive Changes Applied)

---

## 1. No Example Prices as Real Values Discipline
- **Strict Rule**: Example numbers cited in discussions (e.g. 850M for Dragon, 570M for Kitsune, 180M for Gas, 15M for Buddha) are strictly treated as discussion examples, **never** as verified market values or seed data.
- **Catalogue Integrity**:
  - Existing catalogue records in your database remain untouched.
  - No catalogue prices will be modified or seeded based on speculative or hardcoded numbers.
  - Items with unresearched values will be displayed as `"Value unavailable"` rather than guessing or inserting arbitrary benchmarks.
  - A catalogue value is updated **only** when an actual research cycle executes with genuine public citations, valid source URLs, HTTP status verification, and timestamps.

---

## 2. Official Gemini API Pricing & Google Search Grounding Analysis

| Metric | Free Tier (Google AI Studio) | Pay-As-You-Go Beyond Free Tier |
| :--- | :--- | :--- |
| **Model** | `gemini-3.8-flash` | `gemini-3.8-flash` |
| **Search Grounding Allowance** | **5,000 free search queries / month** across all Gemini 3 series models | N/A (billed per query above 5,000) |
| **Search Query Cost After Allowance** | Free up to 5,000 queries/month | **$14.00 per 1,000 search queries** ($0.014 per query) |
| **Token Cost (Input / Output)** | Free up to standard RPM/RPD limits | $0.075 per 1M input tokens / $0.30 per 1M output tokens |
| **Catalogue Research Requirement** | ~40 tradable items in current catalogue | 40 items × 1 query = 40 queries per full cycle |
| **Expected Monthly Consumption** | Daily cycle (30 days × 40 items = 1,200 queries/mo) | Well within the **5,000 monthly free search queries** ($0.00 cost) |
| **Spending Cap Safeguard** | Billing will **NOT** be activated. Hard cap enforced on backend to never exceed 150 queries/day. |

*Billing is query-based. Under free tier limits, running a daily scheduled update of your entire catalogue consumes ~1,200 search queries per month, easily staying within the 5,000 free queries allowance at **$0.00 monthly cost**.*

---

## 3. Firestore Security Rules & Validation Architecture

### Valid Syntax (No Invalid `incoming()` References)
In standard Firestore Security Rules, the incoming payload is referenced via `request.resource.data`, and existing database documents via `resource.data`. The rules will be written using standard, syntax-validated rule expressions:

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

    // Strict server-controlled Custom Claim check
    function isAdmin() {
      return isSignedIn() && request.auth.token.admin == true;
    }

    function isOwner(userId) {
      return isSignedIn() && request.auth.uid == userId;
    }

    // 1. Users collection: private user profiles
    match /users/{userId} {
      allow get: if isOwner(userId) || isAdmin();
      allow list: if isAdmin();
      allow create: if isOwner(userId) && request.resource.data.keys().hasOnly(['uid', 'email', 'displayName', 'createdAt']);
      allow update: if isOwner(userId) && !request.resource.data.diff(resource.data).affectedKeys().hasAny(['admin', 'role']);
      allow delete: if isAdmin();
    }

    // 2. Services Catalogue (Publicly readable, Admin write-only)
    match /services/{serviceId} {
      allow read: if true;
      allow write: if isAdmin();
    }

    // 3. Fruit Mutations (e.g. Hellfire Gas nested under Gas Fruit)
    match /fruit_mutations/{mutationId} {
      allow read: if true;
      allow write: if isAdmin();
    }

    // 4. Orders:
    // Customers can ONLY create an order with status 'pending_review' and verified customerUid.
    // Customers CANNOT set prices, confirm deposits, mark payments, or edit status.
    match /orders/{orderId} {
      allow get: if isAdmin() || (isSignedIn() && resource.data.customerUid == request.auth.uid);
      allow list: if isAdmin() || (isSignedIn() && resource.data.customerUid == request.auth.uid);
      
      // Customer creation constraints:
      allow create: if isSignedIn()
        && request.resource.data.customerUid == request.auth.uid
        && request.resource.data.status == 'pending_review'
        && !request.resource.data.keys().hasAny(['staffNotes', 'paymentConfirmedBy', 'isPaid']);

      // Only authenticated Admins can update status, approve valuations, or record payments
      allow update: if isAdmin();
      allow delete: if isAdmin();
    }

    // 5. Research Results & Sources (Public read, Admin write)
    match /research_results/{resultId} {
      allow read: if true;
      allow write: if isAdmin();
    }

    match /research_sources/{sourceId} {
      allow read: if true;
      allow write: if isAdmin();
    }

    // 6. Value History Audit Log
    match /value_history/{historyId} {
      allow read: if true;
      allow write: if isAdmin();
    }

    // 7. Support Enquiries
    match /enquiries/{enquiryId} {
      allow get, list: if isAdmin();
      allow create: if true; // Public contact form
      allow update, delete: if isAdmin();
    }

    // 8. Service Completion Proofs
    match /service_evidence/{evidenceId} {
      allow read: if resource.data.staffApprovedPublic == true || isAdmin();
      allow write: if isAdmin();
    }

    // 9. Admin Settings (Schedule, Reliability thresholds)
    match /admin_settings/{settingId} {
      allow read: if true;
      allow write: if isAdmin();
    }
  }
}
```

### Server-Side Order Valuation Security
- Customers **cannot** set or manipulate service prices or escrow totals.
- When an order is submitted to `/api/orders`:
  1. The server fetches the official service record directly from the database.
  2. Total price is enforced from `service.priceHundredths`.
  3. Upfront 50% deposit and remaining 50% are calculated server-side using the strict integer half-up formula (`calculateDepositAndRemaining`).
  4. The client payload cannot override price, deposit, or status.

---

## 4. Secure Administrator Authorization via Firebase Custom Claims

### 1. Verified Admin Account
- **Administrator Email**: `brajbala1989@gmail.com` (Explicitly verified).

### 2. Server-Controlled Custom Claim Setup
- On the backend via Firebase Admin SDK (`firebase-admin/auth`):
  ```ts
  // Script / Route to assign Custom Claim
  await getAuth().setCustomUserClaims(userRecord.uid, { admin: true });
  ```
- **Security Invariant**:
  - Custom claims can **only** be minted by trusted server-side code with service credentials.
  - No user can self-assign `admin: true` in Firestore documents or frontend localStorage.
  - Firestore Security Rules evaluate `request.auth.token.admin == true`.
  - Backend API middleware `requireStaffAuth` verifies the Firebase ID Token with `verifyIdToken(token)` and enforces `decodedToken.admin === true`.
  - If a user logs in with an email other than `brajbala1989@gmail.com`, or does not have `admin: true`, administrative UI tabs (`Live Value Manager`, `Change Catalogue`, `Support Inbox`) and backend endpoints return 401/403 and are strictly hidden.

---

## 5. Cloud Firestore as Single Production Source of Truth

### 1. Clear Database Role Separation
- **Active Production Database**: Cloud Firestore (`ai-studio-bloxfruitsservic-556f2ca4-5b5d-4985-ac48-82fc2f53f72a`).
- **Backup / Archive**: Neon PostgreSQL remains untouched and unchanged as a standalone backup database.
- **No Silent Dual-Writes / No Conflicting State**:
  - Once migration is verified and approved, all reads and writes route directly to Firestore.
  - There is no silent fallback that could create split-brain records (e.g. order in Neon but not Firestore).
  - PostgreSQL tables remain dormant unless you explicitly request a rollback or approve data removal.

---

## 6. Secure Automatic Scheduling & Webhook Protection

### 1. Webhook Authentication (`/api/cron/research-update`)
- The endpoint is protected with a shared secret header or query token:
  `Authorization: Bearer <CRON_SECRET>` or `x-cron-secret: <CRON_SECRET>`.
- Calls without the matching secret are immediately rejected with `401 Unauthorized`.

### 2. Rate Limits, Cooldowns & Duplicate Protection
- **Cooldown Window**: Minimum 4 hours between full research cycles. If a trigger arrives before the cooldown expires, the server responds with `{ skipped: true, reason: 'Cooldown active' }`.
- **Concurrency Lock**: An active in-progress research lock prevents parallel or duplicate executions if multiple pings occur simultaneously.
- **Request Pacing**: 400ms delay between consecutive Gemini calls to respect free tier rate limits.

---

## 7. Improved Value Verification & Attribution Requirements

1. **Genuinely Independent Sources**:
   - The engine validates domain names (`sourceUrl`).
   - Multiple pages from the same root domain or known scrapers/mirrors are deduplicated and counted as only **one** source.
   - At least 2 distinct root domains are required for automatic updates.
2. **Conflict Detection**:
   - If sources report values diverging by more than 15%, the item is flagged as `needs_review` / `conflict` and **never** auto-updated.
3. **Audit Trail**:
   - Every verified change logs: `oldValueM`, `newValueM`, `sourceUrls`, `retrievedAt`, `changeReason`, and `approvedBy`.
4. **Attribution Display**:
   - UI displays clickable direct source links with domain labels (e.g., *Blox Fruits Values*, *FruityBlox Community Trading*) adhering to Google Search Grounding attribution requirements.

---

## 8. Explicit Preservation of Existing Features

The following working systems are explicitly protected and untouched:
- **Existing Service Catalogue**: All current raid, trial, boss, and quest services.
- **50/50 Escrow Calculator**: Integer hundredths math, odd hundredth split rules, deposit coverage checks.
- **Customer Order Tracking**: Unguessable `publicToken` URLs and `orderCode` formatting.
- **Staff Workflow**: Deposit confirmation, verification checkboxes, in-progress, completion steps.
- **Discord Proofs Wall**: Customer consent filters and masked Roblox usernames.
- **Support Inbox**: In-app enquiry creation, staff notes, and search.
- **Order History**: All previous customer orders and payment records.

---

## 9. Verification & Testing Plan

Actual test commands to be executed upon your approval:

1. **Unit & Integration Suite (`npm test`)**:
   - `tests/calculator.test.ts` (Escrow math & rounding)
   - `tests/public_orders.test.ts` (Order flow & token lookups)
   - `tests/admin_actions.test.ts` (Staff actions & item verification)
   - `tests/support_and_proofs.test.ts` (Support tickets & Discord proofs)
   - `tests/admin_catalog_and_auth.test.ts` (Admin auth & CRUD)
   - `tests/value_research.test.ts` (Grounded research, reliability rules, cron secret)
   - `tests/market_trends.test.ts` (Market trends widget & response schema)
2. **TypeScript & Static Analysis (`npm run lint`)**:
   - `tsc --noEmit` to verify type safety across client, server, and Firestore rules.
3. **Production Build (`npm run build`)**:
   - `vite build` to verify clean production asset bundling.
4. **Firestore Security Rules Audit**:
   - Automated check verifying permission rejection on unauthorized order price edits or role escalations.
5. **Admin Claim Authorization Test**:
   - Verify that non-admin Google accounts cannot access administrative endpoints or UI components.
