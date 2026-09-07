# CREXPERT.AI — Software Requirements Specification

| Field | Value |
|---|---|
| **Document ID** | SRS-CREXPERT-001 |
| **Version** | 1.1 |
| **System** | CREXPERT.AI — AI-powered Commercial Real Estate Marketplace |
| **Environment** | Production — `https://crexpert.ai` |
| **Purpose** | Define what the system shall do, for product alignment and QA test design |

---

## 1. Introduction

### 1.1 Purpose
This document specifies functional, data, security, and non-functional requirements for CREXPERT.AI. Each requirement has a unique ID (`REQ-<MODULE>-<nnn>`) for traceability to test cases.

### 1.2 Conventions

| Label | Meaning |
|---|---|
| **Priority** | P1 = critical · P2 = core · P3 = supporting |
| **Shall / Must** | Mandatory behaviour |
| **Should** | Expected but lower priority |

### 1.3 Definitions

| Term | Meaning |
|---|---|
| **Listing** | CRE property record. Lifecycle: draft → published → archived / sold |
| **Inquiry / Lead** | Investor contact request on a listing |
| **Conversation** | 1:1 message thread between investor and agent |
| **Saved Search** | Persisted search filter with optional email alerts |
| **Promotion** | Approved featured listing or marketplace banner placement |
| **Rexi** | In-app AI assistant |
| **Active view** | UI mode: Investor or Agent (independent of account role) |

---

## 2. Scope

### 2.1 In Scope
Public marketplace, listing detail, investor workspace (My Hub), agent workspace, account settings, messaging, AI search, Rexi assistant, and admin console.

### 2.2 Out of Scope
Third-party map/census accuracy, Firebase platform SLAs, email infrastructure internals.

### 2.3 Platform Context
- **Frontend:** Next.js (App Router), React
- **Auth:** Firebase Authentication + server session (`POST/DELETE /api/auth/session`)
- **Data:** Cloud Firestore (realtime) + Firebase HTTPS Callables (~100 functions)
- **Storage:** Firebase Storage for listing media and attachments
- **Important:** Route protection is client-side; authorisation must be enforced on callables and Firestore rules, not only in the UI.

---

## 3. Users & Permissions

### 3.1 Actors

| Actor | Description |
|---|---|
| **Anonymous** | Browse marketplace and listing details only |
| **Investor** | Save listings/searches, inquire, message agents, use My Hub |
| **Agent** | All investor capabilities + listing CRUD, leads, import/export, promotions |
| **Admin** | Platform console: users, listings, moderation, reports, promotions, analytics |
| **System** | Scheduled jobs: vitals, digests, peer medians, AI enrichment |

### 3.2 Permission Matrix

| Capability | Anon | Investor | Agent | Admin |
|---|:--:|:--:|:--:|:--:|
| Browse published listings | ✅ | ✅ | ✅ | ✅ |
| View own draft/archived listing | ⛔ | ⛔ | ✅ | ✅ |
| Save listing / saved search | ⛔ | ✅ | ✅ | ✅ |
| Submit inquiry / message agent | ⛔ | ✅ | ✅ | ✅ |
| Create / publish / delete listing | ⛔ | ⛔ | ✅ own | ✅ |
| View leads | ⛔ | ⛔ | ✅ own | ✅ |
| Request promotion | ⛔ | ⛔ | ✅ own | ✅ |
| Report listing | ⛔ | ✅ | ✅ | ✅ |
| Access `/admin/**` | ⛔ | ⛔ | ⛔ | ✅ |
| Suspend user / moderate listing | ⛔ | ⛔ | ⛔ | ✅ |

---

## 4. Application Structure

### 4.1 Key Routes

| Area | Routes |
|---|---|
| **Public** | `/marketplace`, `/listings/[slug]`, `/agents/[slug]`, `/signin`, `/signup`, `/legal/*` |
| **Investor** | `/my-hub`, `/saved`, `/saved-searches`, `/messages`, `/messages/[id]` |
| **Agent** | `/agent`, `/agent/leads`, `/agent/listings/new`, `/agent/listings/[id]` |
| **Account** | `/account`, `/account/profile`, `/account/notifications`, `/account/investor-preferences`, `/account/agent-profile`, `/account/security`, `/account/privacy` |
| **Admin** | `/admin`, `/admin/users`, `/admin/listings`, `/admin/reports`, `/admin/analytics`, `/admin/settings` |

### 4.2 Access Rules

| ID | Requirement | Pri |
|---|---|:--:|
| REQ-ROLE-001 | Unauthenticated users accessing protected routes shall be redirected to sign-in and returned after login | P1 |
| REQ-ROLE-002 | Non-admin users accessing `/admin/**` shall be redirected to `/marketplace` | P1 |
| REQ-ROLE-003 | Header shall provide Investor / Agent view switcher; selection persists across sessions | P1 |
| REQ-ROLE-004 | Investor view shows Marketplace + My Hub; Agent view shows Listings + Leads | P2 |
| REQ-ROLE-005 | Users without `agentEnabled` shall not access agent features until onboarding completes | P1 |
| REQ-ROLE-006 | All permissions in §3.2 shall be enforced server-side on callables and Firestore, independent of UI | P1 |

---

## 5. Functional Requirements

### 5.1 Authentication & Session

| ID | Requirement | Pri |
|---|---|:--:|
| REQ-AUTH-001 | System shall provide sign-up at `/signup` (name, email, password, phone, location) and sign-in at `/signin` | P1 |
| REQ-AUTH-002 | Password shall require ≥8 chars, uppercase, lowercase, digit, and special character | P1 |
| REQ-AUTH-003 | Sign-up shall validate email format, name (2–100 chars), phone (7–40 chars), and location (2–200 chars) | P1 |
| REQ-AUTH-004 | Sign-in shall support forgot-password via email reset link | P1 |
| REQ-AUTH-005 | On login, system shall mint server session via `POST /api/auth/session`; sign-out shall clear it | P1 |
| REQ-AUTH-006 | Expired session shall prompt re-authentication | P1 |
| REQ-AUTH-007 | Security page shall offer sign-out (current device) and sign-out everywhere (all devices) | P1 |
| REQ-AUTH-008 | Email verification shall be supported; profile shall show verified/unverified state | P1 |
| REQ-AUTH-009 | Agent sign-up shall route to agent onboarding before agent features unlock | P1 |
| REQ-AUTH-010 | Signed-in users visiting `/signin` or `/signup` shall be redirected away | P2 |

### 5.2 Marketplace

| ID | Requirement | Pri |
|---|---|:--:|
| REQ-MKT-001 | Marketplace at `/marketplace` shall support For Lease and For Sale tabs (For Lease default) | P1 |
| REQ-MKT-002 | Users shall filter by property type, price, size, location, tenancy, utilities, spotlight badges, zoning, and listed-within days | P1 |
| REQ-MKT-003 | Keyword search shall match address, title, and agent name | P2 |
| REQ-MKT-004 | Results shall paginate (default 20, max 100) and sort by newest, price, acreage, relevance, completeness, or saves | P1 |
| REQ-MKT-005 | Filter, sort, and page state shall be reflected in the URL for deep linking | P1 |
| REQ-MKT-006 | Only published, non-rejected listings shall appear for non-owners | P1 |
| REQ-MKT-007 | Result cards shall show type, price (or "Price upon request"), title, size, address, and image count | P2 |
| REQ-MKT-008 | Save/bookmark on a card shall prompt sign-in when unauthenticated | P1 |
| REQ-MKT-009 | Empty results shall show guidance to broaden filters | P2 |
| REQ-MKT-010 | Active filters shall display as removable chips with Clear all | P2 |
| REQ-MKT-011 | AI search box shall accept natural language (max 300 chars) and map to structured filters | P1 |
| REQ-MKT-012 | Marketplace banners shall respect audience, schedule, priority, and active flag | P2 |

### 5.3 Listing Detail Page

| ID | Requirement | Pri |
|---|---|:--:|
| REQ-LIST-001 | Listing URL shall resolve by slug-id or bare 20-char ID | P2 |
| REQ-LIST-002 | Unknown listing shall show "Listing not found" (prefer HTTP 404) | P1 |
| REQ-LIST-003 | Page shall show title, type, address, price, gallery, listed/updated dates, and days on market | P1 |
| REQ-LIST-004 | Inquire CTA shall open inquiry modal; Message/Call/Email shall respect agent public-contact settings | P1 |
| REQ-LIST-005 | Populated sections only: About, Highlights, Details, Amenities, Features, Spaces | P2 |
| REQ-LIST-006 | Location module shall show map with Directions, Aerial, Map, and Commute modes | P2 |
| REQ-LIST-007 | Demographics module shall show US Census ACS data for 1/3/5-mile rings | P2 |
| REQ-LIST-008 | Similar listings, transportation, and nearby amenities shall display when data exists | P2 |
| REQ-LIST-009 | Documents shall download via signed URLs only | P1 |
| REQ-LIST-010 | Report listing action shall be available to authenticated users | P2 |

### 5.4 Agent Listing Management

| ID | Requirement | Pri |
|---|---|:--:|
| REQ-LMG-001 | Agent workspace at `/agent` shall list own listings in tabs: All, Drafts, Published, Archived, Sold, Promotions | P1 |
| REQ-LMG-002 | Workspace shall support New Listing, Import, Export, title search, and sort (Created / Needs Attention) | P1 |
| REQ-LMG-003 | Listing wizard shall have sections: Basic, Media, Highlights, Property Details, Spaces (office/retail), Sale Details, Lease Details — shown based on property/transaction type | P1 |
| REQ-LMG-004 | Wizard shall auto-save drafts and show publish-readiness checklist with field count | P1 |
| REQ-LMG-005 | Publish shall require: title, description, transaction type, property type, street address, city, state, valid US ZIP | P1 |
| REQ-LMG-006 | Save as Draft shall persist without publish validations | P1 |
| REQ-LMG-007 | Lifecycle actions: publish, unpublish, archive, unarchive, mark sold/unsold, delete — each only for own listings from valid states | P1 |
| REQ-LMG-008 | Published listing edits shall remain published and update marketplace | P1 |
| REQ-LMG-009 | Rejected moderation status shall block agent re-publish until admin clears | P1 |
| REQ-LMG-010 | Address entry shall use US Places autocomplete; price unit options shall match transaction type | P1 |
| REQ-LMG-011 | Completeness score (0–100) and vitals (fresh/aging/stale, top/mid/low) shall reflect listing fill level | P2 |
| REQ-LMG-012 | AI Review shall be advisory only and never block publishing | P2 |

### 5.5 Media & Documents

| ID | Requirement | Pri |
|---|---|:--:|
| REQ-MEDIA-001 | Images: JPEG/PNG/WebP, max 10 MB · Videos: MP4/MOV/WebM, max 200 MB · Documents: PDF only, max 25 MB | P1 |
| REQ-MEDIA-002 | External video links: max 5, must be http(s), max 2048 chars each | P2 |
| REQ-MEDIA-003 | Disallowed file types shall be rejected with clear error messages | P1 |
| REQ-MEDIA-004 | Upload URLs shall be scoped to the owning agent's listing only | P1 |
| REQ-MEDIA-005 | Profile photo and company logo: JPEG/PNG/WebP | P1 |

### 5.6 CSV Import & Export

| ID | Requirement | Pri |
|---|---|:--:|
| REQ-IMPX-001 | Import shall accept max 200 rows; default to dry-run before commit | P1 |
| REQ-IMPX-002 | Required columns: title, transactionType, propertyType; all imports create drafts | P1 |
| REQ-IMPX-003 | Export shall support scope filters (all/draft/published/archived/sold) for own listings only | P1 |
| REQ-IMPX-004 | Multi-value columns use `\|` separator; price accepts `$` and commas | P2 |

### 5.7 Inquiries & Leads

| ID | Requirement | Pri |
|---|---|:--:|
| REQ-LEAD-001 | Inquiry modal shall require message (max 600 chars); unauthenticated users prompted to sign in | P1 |
| REQ-LEAD-002 | Agent leads page shall filter by All / New / Contacted / Closed | P1 |
| REQ-LEAD-003 | Agent shall update lead status, follow-up date, contact log, and outcome per business rules | P1 |
| REQ-LEAD-004 | Closing a lead requires an outcome; outcome only set when status is closed | P1 |
| REQ-LEAD-005 | Agent shall only access leads on own listings | P1 |

### 5.8 Messaging

| ID | Requirement | Pri |
|---|---|:--:|
| REQ-MSG-001 | Messages list and thread views at `/messages` and `/messages/[id]` | P1 |
| REQ-MSG-002 | Message shall have body (max 2000 chars) and/or up to 5 attachments (max 25 MB each) | P1 |
| REQ-MSG-003 | Only conversation participants may read/write; messages update in realtime | P1 |
| REQ-MSG-004 | Message action on listing/agent card shall open or create scoped conversation | P2 |

### 5.9 Saved Properties & Searches

| ID | Requirement | Pri |
|---|---|:--:|
| REQ-SAVE-001 | User shall save/unsave listings; saved page shows save date and price-drop indicator | P1 |
| REQ-SAVE-002 | Saved listings shall be private and idempotent (no duplicates) | P1 |
| REQ-SRCH-001 | Saved search requires structured filter, NLP text, or investor preferences | P1 |
| REQ-SRCH-002 | Alerts support instant/daily/weekly digest; pausing stops alerts | P1 |
| REQ-SRCH-003 | Re-running a saved search shall reproduce marketplace results | P1 |

### 5.10 My Hub (Investor Dashboard)

| ID | Requirement | Pri |
|---|---|:--:|
| REQ-HUB-001 | My Hub shall show: Your Focus, Saved Properties, Saved Searches, Recent Agents, Recent Inquiries, Recent Searches, Recently Viewed, Market Trends, Top Markets, Trending Searches, Recommended | P1 |
| REQ-HUB-002 | Each module shall have a defined empty state for new users | P2 |
| REQ-HUB-003 | Recommendations shall use investor preferences and saved activity | P2 |

### 5.11 AI Assistant (Rexi) & AI Search

| ID | Requirement | Pri |
|---|---|:--:|
| REQ-AI-001 | Rexi launcher shall be available on marketplace and listing pages | P2 |
| REQ-AI-002 | Assistant shall support sessions (create, rename, delete), text input, and listing context when opened from a listing | P1 |
| REQ-AI-003 | Assistant shall answer only from supplied listing context; shall not invent properties or give financial/legal advice | P1 |
| REQ-AI-004 | Assistant shall report exact match counts from context and resist prompt injection | P1 |
| REQ-AI-005 | AI field enrichment and document extraction shall be advisory; never overwrite without user acceptance | P1 |
| REQ-AI-006 | NLP search shall detect alert intent and offer Save as Alert | P2 |

### 5.12 Account & Settings

| ID | Requirement | Pri |
|---|---|:--:|
| REQ-ACCT-001 | Account nav: Overview, Profile, Notifications, Preferences, Agent profile, Security, Data & privacy | P1 |
| REQ-ACCT-002 | Profile edits shall enforce field validation (name, phone, location, email) | P1 |
| REQ-ACCT-003 | Notifications page shall control email toggles and digest cadence (daily/weekly/off) | P1 |
| REQ-ACCT-004 | Investor preferences: asset classes, deal size range, geography, accreditation | P1 |
| REQ-ACCT-005 | Agent profile: company, license, bio, specialties, public contact visibility (email off by default) | P1 |
| REQ-ACCT-006 | Agent onboarding shall require license details and terms acceptance | P1 |
| REQ-ACCT-007 | Data export shall deliver user data by email; account deletion shall require confirmation and sign out user | P1 |

### 5.13 Promotions

| ID | Requirement | Pri |
|---|---|:--:|
| REQ-PROMO-001 | Agent shall request featured and/or banner promotion for own listings | P1 |
| REQ-PROMO-002 | Agent may withdraw pending requests; admin approves/rejects with required dates and notes | P1 |
| REQ-PROMO-003 | Agent shall not self-approve promotions | P1 |

### 5.14 Notifications

| ID | Requirement | Pri |
|---|---|:--:|
| REQ-NOTIF-001 | Header bell shall show unread indicator; user sees only own notifications | P1 |
| REQ-NOTIF-002 | Email notifications shall respect account notification preferences | P1 |

### 5.15 Listing Reports & Moderation

| ID | Requirement | Pri |
|---|---|:--:|
| REQ-MOD-001 | Authenticated users shall report listings (spam, fraud, inaccurate, inappropriate, other) with optional note | P1 |
| REQ-MOD-002 | Flagged listings de-prioritised; rejected listings removed from public surfaces | P1 |
| REQ-MOD-003 | Reporter identity shall not be exposed to listing agent | P1 |

### 5.16 Admin Console

| ID | Requirement | Pri |
|---|---|:--:|
| REQ-ADMIN-001 | Admin-only access to dashboard KPIs, growth charts, promotion queue, reported listings | P1 |
| REQ-ADMIN-002 | User management: list, filter, search, suspend, verify agent | P1 |
| REQ-ADMIN-003 | Listing management: filter by status/moderation, moderate, unpublish, feature | P1 |
| REQ-ADMIN-004 | Report queue: flag, archive, or reject listing with reviewer notes | P1 |
| REQ-ADMIN-005 | Analytics overview with for-sale/for-lease map; banner management in settings | P1 |
| REQ-ADMIN-006 | All admin mutations shall write immutable audit log entries | P1 |

---

## 6. Data Validation Rules

Key constraints for acceptance and boundary testing. Server shall enforce the same rules as the client.

### 6.1 User Profile

| Field | Constraint |
|---|---|
| displayName | 2–100 chars |
| phone | 7–40 chars; digits, spaces, dashes, parentheses, dots, leading + |
| location | 2–200 chars |
| email | Valid email format |

### 6.2 Listing — Publish Requirements

| Field | Constraint |
|---|---|
| title | 1–200 chars |
| description | Required to publish (max 5000) |
| transactionType | sale · lease · sale_lease |
| propertyType | One of 16 types; subtype must match type |
| address, city, state | Required to publish |
| zip | US format: 12345 or 12345-6789 |
| price | ≥ 0; unit must match transaction type |
| officeSpaces | Max 50 suites |
| amenities / features | Max 40 items each |

### 6.3 Inquiry & Lead

| Field | Constraint |
|---|---|
| message | 1–600 chars |
| leadStatus | new · contacted · closed |
| leadOutcome | won · lost · no_response (required when closed) |
| followUpAt | Must not be in the past |

### 6.4 Messaging & AI

| Field | Constraint |
|---|---|
| message body | 1–2000 chars (or attachment required) |
| attachments | Max 5, max 25 MB each |
| Rexi message | 1–1000 chars |
| saved search name | 1–80 chars |

### 6.5 Key Enumerations

| Domain | Values |
|---|---|
| Listing status | draft · published · archived · sold |
| Moderation | approved · flagged · rejected · unreviewed |
| Transaction type | sale · lease · sale_lease |
| Lead status | new · contacted · closed |
| Digest frequency | instant · daily · weekly · off |
| Report reason | spam · fraud · inaccurate · inappropriate · other |

### 6.6 Pagination Limits

| Surface | Default | Max |
|---|---|---|
| Marketplace page size | 20 | 100 |
| Saved listings / searches | 50 | 100 |
| Leads | 100 | 200 |
| Notifications | 20 | 50 |
| CSV import rows | — | 200 |

---

## 7. Non-Functional Requirements

| ID | Requirement | Pri |
|---|---|:--:|
| REQ-NFR-001 | Marketplace search shall return results within 3 s (p95) | P1 |
| REQ-NFR-002 | Listing detail primary content shall render before third-party modules (maps, demographics) | P1 |
| REQ-NFR-003 | Every async surface shall show loading, empty, and error states — never a blank region | P1 |
| REQ-NFR-004 | Failed operations shall show readable errors without leaving UI stuck | P1 |
| REQ-NFR-005 | App shall work on latest two versions of major browsers; usable from 320 px to 2560 px width | P1 |
| REQ-NFR-006 | All controls keyboard-accessible; modals trap focus and close on Escape | P1 |
| REQ-NFR-007 | Form errors associated with inputs for assistive technology | P1 |
| REQ-NFR-008 | Currency (USD), sqft, and dates formatted consistently across the app | P2 |
| REQ-NFR-009 | Derived values (cap rate, price/sqft, days on market) update when inputs change | P1 |
| REQ-NFR-010 | Counts shown in marketplace, hub, and admin shall agree for the same underlying data | P1 |

---

## 8. Security Requirements

| ID | Requirement | Pri |
|---|---|:--:|
| REQ-SEC-001 | Firestore rules shall enforce §3.2 permission matrix for all direct client reads/writes | P1 |
| REQ-SEC-002 | Callables shall derive ownership and role from auth token — never trust client-supplied IDs or roles | P1 |
| REQ-SEC-003 | All validation rules in §6 enforced server-side; bypassing UI shall produce same rejections | P1 |
| REQ-SEC-004 | Firebase App Check required on all callables | P1 |
| REQ-SEC-005 | Storage rules shall restrict writes to owning agent's listing paths | P1 |
| REQ-SEC-006 | Document download URLs time-limited; no enumeration across listings | P1 |
| REQ-SEC-007 | User-generated content rendered safely — no stored XSS; rich text sanitised | P1 |
| REQ-SEC-008 | CSV export shall neutralise formula injection | P1 |
| REQ-SEC-009 | Rate limiting on inquiries, messaging, reports, AI, import, and auth attempts | P1 |
| REQ-SEC-010 | Session cookie HttpOnly, Secure, SameSite; invalidated on sign-out and password change | P1 |
| REQ-SEC-011 | PII exposed publicly only when owner enables showEmailPublicly / showPhonePublicly | P1 |
| REQ-SEC-012 | Admin callables verify admin role server-side; UI redirect is not access control | P1 |
| REQ-SEC-013 | AI assistant shall not disclose other users' data or perform privileged actions | P1 |

---

## 9. UI Standards

| ID | Requirement | Pri |
|---|---|:--:|
| REQ-UI-001 | Global header: brand, nav (view-dependent), messages, notifications, avatar menu | P1 |
| REQ-UI-002 | Authenticated users shall not see sign-in/sign-up CTAs in header | P1 |
| REQ-UI-003 | Destructive actions (delete listing, delete account, clear all) require confirmation | P1 |
| REQ-UI-004 | Successful mutations show toast or inline confirmation | P2 |
| REQ-UI-005 | Page titles follow `<Context> · CREXPERT.AI` pattern | P2 |
| REQ-UI-006 | Footer links to Privacy and Terms | P3 |

---

## 10. Open Items

Items below need product-owner confirmation before being used as pass/fail test oracles.

### 10.1 Known Observations

| # | Issue |
|---|---|
| ANOMALY-01 | Unknown listing/agent URLs return HTTP 200 instead of 404 |
| ANOMALY-02 | Security page promises password/MFA management but only shows sign-out options |
| ANOMALY-03 | My Hub Market Trends shows `—` for all metrics (empty state or broken job?) |
| ANOMALY-04 | Null-price listings sort order undefined when sorting by price |
| ANOMALY-05 | Highlights editor counter (2000) may not match stored limit (12000) |
| ANOMALY-06 | Rexi listing counts may not match marketplace totals |
| ANOMALY-08 | Admin analytics may contain placeholder data — verify with admin account |
| ANOMALY-10 | Authenticated account pages may show sign-in/sign-up links in SSR header |

### 10.2 Verification Gaps

- Admin console requirements not validated without admin account access
- Firestore security rules not reviewed rule-by-rule
- Email delivery (verification, alerts, digests) not tested end-to-end
- Sold-listing public visibility policy undefined
- Data retention after listing/account deletion undefined

---

## Appendix A — Requirement Index

| Module | IDs | Count |
|---|---|---|
| Authentication | REQ-AUTH-001 – 010 | 10 |
| Roles & access | REQ-ROLE-001 – 006 | 6 |
| Marketplace | REQ-MKT-001 – 012 | 12 |
| Listing detail | REQ-LIST-001 – 010 | 10 |
| Listing management | REQ-LMG-001 – 012 | 12 |
| Media | REQ-MEDIA-001 – 005 | 5 |
| Import/Export | REQ-IMPX-001 – 004 | 4 |
| Leads | REQ-LEAD-001 – 005 | 5 |
| Messaging | REQ-MSG-001 – 004 | 4 |
| Saved properties | REQ-SAVE-001 – 002 | 2 |
| Saved searches | REQ-SRCH-001 – 003 | 3 |
| My Hub | REQ-HUB-001 – 003 | 3 |
| AI / Rexi | REQ-AI-001 – 006 | 6 |
| Account | REQ-ACCT-001 – 007 | 7 |
| Promotions | REQ-PROMO-001 – 003 | 3 |
| Notifications | REQ-NOTIF-001 – 002 | 2 |
| Moderation | REQ-MOD-001 – 003 | 3 |
| Admin | REQ-ADMIN-001 – 006 | 6 |
| Non-functional | REQ-NFR-001 – 010 | 10 |
| Security | REQ-SEC-001 – 013 | 13 |
| UI | REQ-UI-001 – 006 | 6 |
| **Total** | | **123** |

---

*End of document — SRS-CREXPERT-001 v1.1*
