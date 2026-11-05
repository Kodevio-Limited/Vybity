# Vybity — Final API Specification & Architecture (canonical)

**Status:** single source of truth for the Vybity REST API. Supersedes `API_SPECIFICATION.md`,
`SCREEN_TO_API_MAPPING.md`, and `Vybity - REST API Specification v1.md` (kept for history).
Machine-readable twin: `OPENAPI.yaml`. Endpoint-level deltas: `CHANGELOG_API.md`.

**Basis:** Figma file `SbnpZCb45gYYPhmshOWC8D`. Verified this session: the file has **one page
`App Ui` (`0:1`)**; the "Business Portal" is a node/section (`671:2529`) *inside* that page holding 27
business frames (Overview `925:328`, Claim status `871:2023`, Live Biz Updates `691:3361`, Promotions
Calendar `700:3748`, Offers, Analytics, Location Profile, Team, Reports/Disputes, Settings, modals).
There is **no admin panel in Figma** (confirmed). Written brief: Detailed Scope of Work v1.0 (Aug 4, 2026).

> **Owner corrections applied (2026-10-14).** `correction.md` is now populated and every correction is
> applied in this revision, indexed in §0.1. Summary: onboarding + legal content endpoints removed
> (frontend-owned), `POST /chat/…/not-right-now` removed, Places trimmed 4→2 endpoints,
> `place_id → location{location_id, location_name, lat, lng}` identifier model, location accepted as
> coordinates **or** standard string/GPS formats, routing/ETA delegated to a maps provider, and
> Profile trimmed to required endpoints.

> **BLOCKER — target-stack override recorded.** Scope of Work v1.0 says **MongoDB**; the task mandates
> **PostgreSQL + PostGIS**. Decision D1 below adopts Postgres/PostGIS and records the reason. If the
> owner insists on MongoDB, the geo query layer and the index plan in §4 must be re-derived.

---

## 0. Decision + reason log (everything decidable, decided)

| # | Decision | Reason |
| --- | --- | --- |
| D1 | **PostgreSQL + PostGIS** (not MongoDB) | Geospatial "near me" ranking, strong relational integrity for visits/claims/follows, DRF + `django.contrib.gis` maturity, transactional idempotency. Recorded as a flagged deviation from the SoW. |
| D2 | Version all public routes at `/api/v1` | Task requirement; enables non-breaking evolution. |
| D3 | Consolidate social login → `POST /auth/social/{provider}` | Google + Apple share one responsibility; one handler, one failure surface. |
| D4 | **Remove `/me/favourites`** | Byte-for-byte alias of `/me/saved-places`; no screen distinguishes them. |
| D5 | **Remove `/me/location-permission` (GET+PUT)** | OS location permission is device-local; the app reads it directly. Server only needs the *chosen* location (`PUT /me/location`). |
| D6 | **Remove `/me/settings`** | distance unit/language are client locale concerns; no designed screen. |
| D7 | **Remove `/me/badges`, `/me/rewards`, `/me/notifications*`, `/me/notification-settings`** from MVP | Profile rows exist but no frames exist. Contracts parked in §14 appendix. |
| D8 | **Remove `/me/identity/regenerate`** | The "refresh" glyph is inside the *Continue as Provider* button, not a regenerate control; identity is system-generated and stable; regeneration is an abuse/impersonation vector. |
| D9 | **Remove all `/navigation/sessions` (POST/PATCH/DELETE)** | Turn-by-turn + ETA run on-device from `GET /places/{id}/directions`. Server needs only a single arrival event → `POST /visits`. No server session to create/patch/delete. |
| D10 | **Remove `PATCH /visits/{id}/end`** | No screen. Visits expire server-side (`visit_max_duration`) and on a new arrival. |
| D11 | **Arrival prompt is server-scheduled push** | A Celery ETA job fires at `contribution_prompt_at`; the client reconciles via `GET /visits/active`. No phone timer is authoritative. |
| D12 | **`visit_id` is REQUIRED for a community update** | Presence gating is a core product rule; contributions cannot be anonymous-to-a-place. |
| D13 | **Portal has its own sign-up** (`POST /business/auth/register`) + staff email-link invite accept | Removes the dependency "portal user must first install the phone app". |
| D14 | **Admin API added** (`/admin-api`) with separate auth/roles | Claim approval, report moderation, dispute resolution, place import, content, XP/badge rules have no portal equivalent. |
| D15 | **Content reporting + coupon redemption are designed as proposed endpoints** (no Figma screen) | Portal Analytics/Reports depend on them; they are marked `PROPOSED-NO-FIGMA`. |
| D16 | **Follower count SoT = `COUNT(follow records)`** | Single derivation for Overview, Analytics series, and `new_this_week`. Resolves the 12,450 vs 4,389 contradiction. |
| D17 | **Reliability levels = `high/medium/low/insufficient_data`** | `medium` kept (design only shows High/Low); `insufficient_data` is the thin-data state (never fabricate). |
| D18 | **Atmosphere is multi-select, 1–3** | UI shows 6 chips with no exclusivity affordance (assumption, §13). |
| D19 | **Compare accepts 2–3 places** | Design shows 2 columns; button says "Compare 3 places". |
| D20 | Lifecycle verbs replace destructive DELETE on publishables | `…/end`, `…/cancel` are state transitions, audited, and preserve history/analytics. |
| D21 | **Idempotency-Key** required on arrival + all create/publish POSTs | Prevents duplicate visits, contributions, and duplicate follower pushes. |
| D22 | Admin = separate auth realm + `admin_role`; every mutation audited | Rule 4 requirement; blast-radius isolation from business tokens. |
| D23 | **Remove `/content/onboarding`, `PUT /me/onboarding`, `/content/legal/{slug}`** | Owner: splash/onboarding fully frontend-controlled; legal text bundled client-side. Matching admin content endpoints removed. |
| D24 | **Remove `POST /chat/sessions/{id}/places/{pid}/not-right-now`** | Owner directive. The Not Right Now sheet is client-side only; a rejection refines the local result list. |
| D25 | **Trim Places from 4 endpoints to 2** | Owner: "too many APIs". `reliability` and `why-not-here` data folded into `GET /places/{id}`; only `GET /places/{id}` and `GET /places/{id}/community-updates` remain. |
| D26 | **Identifier model = `place_id → location{location_id, location_name, lat, lng}`** | Owner directive. `Place` (POI) 1:N `Location` (branch); MVP is 1 place : 1 location; Saved and all place reads key on `place_id` and echo the `location` object. |
| D27 | **Location accepted as coordinates *or* standard strings** | Owner directive. WGS84 decimal degrees canonical; also accept `"lat,lng"`, DMS, geohash, Plus Code, or a free-text place string (geocoded). |
| D28 | **Routing/ETA via a maps provider (server-side)** | Owner wants accuracy. Directions/ETA/polyline from Google Directions/Mapbox/OSRM server-side; turn-by-turn stays on-device. |
| D29 | **Profile & account trimmed to required endpoints** | Owner: "keep just required". `GET /me/profile` merged into `GET /me`; badges/rewards/notifications/settings stay out of MVP (appendix). |

### 0.1 Owner corrections (`correction.md`) → where applied

| Correction (owner) | Applied in |
| --- | --- |
| Remove §4 Content & onboarding (slides, status, legal) | D23; §1, §8.1, §8.3, §8.4, §10, §11, §14 |
| Location as latitude/longitude **and** standard string/GPS formats | D27; §2.13, §6.8, §8.1, §8.4, §10 A2 |
| Remove `POST /chat/…/not-right-now` | D24; §8.1, §8.4, §9.3, §10 A4, §11 |
| "Why `{id}` and `lat/lng`?" / "what is cursor?" | Explainers in §8.1 and §2.7 |
| New admin dashboard to approve each business | §8.3 (D14/D22); emphasised in §4 |
| "Not that many APIs in §7 Places" | D25; §8.1 (4→2), §10 A4/A5, §11 |
| "How are route/ETA computed?" | D28; §2.14 |
| "Are all Contribute APIs needed?" | Confirmed 1:1 with UI in §8.1 (Contribute, Add Photo/Video, Review, Follow) and §6.10 |
| Saved by id with place→location hierarchy | D26; §3.2, §3.5, §6.8, §11 |
| Trim Profile & account to required | D29; §8.1, §14, §16 |
| Splash/onboarding frontend-owned | D23; §1, §11, §13 |

---

## 1. Product, surfaces, and content ownership

Vybity answers *"what is this place like right now, before I go?"* via an in-app conversation, live
community observations, and verified business posts. Launch city: **Kansas City**. Anonymous identity.
The UI never uses the words "AI" or "chatbot" (copy rule CR-1, §9.6).

| Surface | Users | Tech | Auth realm |
| --- | --- | --- | --- |
| Consumer mobile app | Visitors | Flutter | consumer token |
| Business portal | Owner / staff | Next.js + Tailwind | provider token |
| Admin panel | Vybity staff | Next.js + Tailwind | admin token |
| Backend | all | Django REST + PostGIS + Celery/Redis | — |

**Content ownership (who may write what on a place)**

| Content | Writer | Source |
| --- | --- | --- |
| Right Now verdicts | computed from contributions + live updates | §6.1 |
| Community Updates | visitors with a verified visit | §8.1 (consumer) |
| Live Biz Update / Event / Offer / Location Profile | business (portal) | §8.2 (portal) |
| Reports / moderation state | app (report) + admin (decision) | §6.7, §8.3 |
| Analytics | derived, read-only | §6.5 |

> **Frontend-owned UI:** Splash, Onboarding, and the Legal text are bundled in the clients (owner
> correction); no API serves them. `GET /content/onboarding`, `PUT /me/onboarding` and
> `GET /content/legal/{slug}` are removed (D23).

---

## 2. Architecture

### 2.1 Component diagram

```
                    ┌──────────────────────┐   ┌──────────────────────┐
 Flutter app ─────► │                      │   │  Next.js Business    │ ──► /business/*
 (consumer tok)     │   Django REST API    │◄──┤  Portal (provider)   │
                    │   (DRF, /api/v1)     │   └──────────────────────┘
                    │                      │   ┌──────────────────────┐
                    │  - auth (JWT)        │◄──┤  Next.js Admin       │ ──► /admin-api/*
                    │  - rules services    │   └──────────────────────┘
                    │  - idempotency       │
                    └───┬───────────┬──────┘
                        │           │ (service calls, no public route)
        ┌───────────────┘           └────────────────┐
        ▼                                            ▼
┌───────────────┐   ┌────────────────────┐   ┌────────────────────┐
│ PostgreSQL16  │   │ Celery workers      │   │ Redis               │
│ + PostGIS     │   │ (broker + scheduler)│   │ cache, rate limit,  │
│ (source of    │   │ expiry, ETA push,   │   │ broker              │
│  truth)       │   │ media, analytics    │   └────────────────────┘
└───────────────┘   └─────────┬──────────┘
                              │
   ┌──────────┬───────────┬───┴────────┬──────────────┬────────────┐
   ▼          ▼           ▼            ▼              ▼            ▼
LLM provider  Maps/Routes  Object      FCM (push)   Email/SMS    Geocoder/
(intent+      (directions, storage     (delivery)   (OTP)        Places
 explain)     ETA)         (S3 media)                              (import)
```

### 2.2 Required external services & keys

| Service | Purpose | Key(s) | Notes |
| --- | --- | --- | --- |
| LLM provider | intent extraction + explanation text | `LLM_API_KEY`, `LLM_MODEL` | structured JSON output; never source of places |
| Maps/Routing | directions, ETA, polyline, distance | `MAPS_API_KEY` | server-normalised; no straight-line estimates |
| Places data | seed/import places | `PLACES_API_KEY` | Google Places or OSM; see §6.4 |
| Geocoder | address search (`/locations/search`) | reuses MAPS/PLACES key | — |
| Push | follower/arrival notifications | `FCM_SERVER_KEY`, project id | delivery receipts only |
| Email/SMS | OTP + invites | `SMTP_*` / `SMS_API_KEY` | OTP sender |
| Object storage | media (photos/videos/docs) | `S3_*` (bucket, region, access, secret) | presigned PUT; private bucket |
| Redis | Celery broker, cache, rate limits | `REDIS_URL` | — |
| Apple/Google OAuth | social sign-in | `GOOGLE_CLIENT_ID`, `APPLE_*` | token verification |

JWT signing: `JWT_SECRET`, rotate quarterly. All secrets from env/secret store; never logged.

### 2.3 Infrastructure & sizing assumptions

| Component | Baseline | Scaling note |
| --- | --- | --- |
| Django API | 2–3 uvicorn/gunicorn pods, 2 vCPU / 4 GB | horizontal behind LB; stateless |
| PostgreSQL | 4 vCPU / 16 GB, 100 GB SSD, PostGIS, PITR | read replica for analytics reads |
| Redis | 1 node, 2 GB | separate logical DBs for broker/cache/limits |
| Celery workers | 2 default + 1 beat | ETA tasks (arrival prompts) need a beat + timer |
| Object storage | S3-compatible, versioned, lifecycle → cold after 90 d | private; served via signed CDN URLs |
| Estimated load (launch) | ~50k MAU, 30 rps peak API, ~2k contributions/day | DB plan validated at 10× |

### 2.4 Auth & token strategy

- **Access token:** JWT (RS256), 15 min TTL, claims `sub`, `realm` (`consumer|provider|admin`),
  `scopes`, `jti`. Validated statelessly.
- **Refresh token:** opaque, 30 days, **rotated on every use**, stored hashed (`Session` table).
  Reuse of a rotated token revokes the whole session family (theft detection).
- **Realms:** consumer, provider, admin are separate issuers/audiences. A provider token cannot call
  `/admin-api`; an admin token cannot act on consumer resources.
- **Logout:** revokes the presented refresh token; `all_devices:true` revokes all sessions.
- **Password reset / email change:** revokes sessions on success. Email is not authoritative until OTP
  verified.
- **Social:** server verifies the provider `id_token`; no client-trusted identity.

### 2.5 Rate limiting (Redis token bucket; keyed per IP + per user + per route)

| Class | Limit |
| --- | --- |
| Auth (register/login/otp/forgot) | 10/min per IP; OTP: 5 sends/hour, resend ≥30 s, 5 verify attempts |
| Reads (search/places/chat GET) | 120/min per user |
| Chat turns | 20/min, 300/day per user |
| Writes (save/follow/contribute/report) | 30/min per user |
| Media uploads | 20/hour per user |
| Business publish actions | 60/hour per location |
| Admin | 300/min per admin |

Exceeding → `429 RATE_LIMITED` with `retry_after_seconds`.

### 2.6 Idempotency

`Idempotency-Key` header is **required** on: `POST /visits`, `POST /media/uploads`,
`POST /places/{id}/community-updates`, all `POST /business/*` publish/create routes, all
`POST /admin-api/*` decisions. Server stores `(user, route, key) → response` for 24 h; a replay returns
the original response, no side effects. Missing key on a required route → `422 IDEMPOTENCY_KEY_REQUIRED`.

### 2.7 Pagination, filtering, envelopes

- `?limit=20&cursor=<opaque>`; `meta.next_cursor` null on last page. Default 20, max 100
  (`422 VALIDATION_FAILED` above max).
- Success: `{ "data": ..., "meta"?: {...} }`.
  Error: `{ "error": { "code", "message", "fields"?, "request_id" } }` (the HTTP status carries the class).
- All timestamps ISO 8601 UTC. Distances metres. Durations minutes unless suffixed `_seconds`.

### 2.8 Error format & code catalogue

| HTTP | Code(s) |
| --- | --- |
| 400 | `OTP_INVALID`, `CURRENT_PASSWORD_INCORRECT` |
| 401 | `INVALID_CREDENTIALS`, `TOKEN_EXPIRED`, `REFRESH_TOKEN_INVALID`, `SOCIAL_TOKEN_INVALID` |
| 403 | `EMAIL_NOT_VERIFIED`, `CLAIM_NOT_APPROVED`, `FORBIDDEN_MODULE`, `ACCOUNT_NOT_PROVIDER`, `ADMIN_FORBIDDEN` |
| 404 | `NOT_FOUND` |
| 409 | `EMAIL_ALREADY_REGISTERED`, `EMAIL_LINKED_TO_PASSWORD_ACCOUNT`, `VISIT_ALREADY_ACTIVE`, `LOCATION_ALREADY_CLAIMED`, `CLAIM_ALREADY_OPEN`, `CODE_ALREADY_USED`, `ALREADY_A_MEMBER`, `INVITATION_ALREADY_ACCEPTED` |
| 410 | `OTP_EXPIRED`, `RESET_TOKEN_EXPIRED`, `INVITATION_EXPIRED` |
| 422 | `VALIDATION_FAILED`, `PASSWORD_MISMATCH`, `PASSWORD_REUSED`, `NO_SIGNAL_PROVIDED`, `TOO_FAR_FROM_PLACE`, `VISIT_NOT_ACTIVE`, `MEDIA_NOT_READY`, `MEDIA_LIMIT_EXCEEDED`, `UPDATE_COOLDOWN`, `IDEMPOTENCY_KEY_REQUIRED`, `EXPIRY_BEFORE_PUBLISH`, `VALIDITY_RANGE_INVALID`, `VERIFICATION_FIELDS_INVALID`, `DOCUMENT_REQUIRED`, `CLAIM_INCOMPLETE`, `OFFER_NOT_REDEEMABLE`, `REPORT_TARGET_INVALID` |
| 423 | `ACCOUNT_LOCKED` |
| 429 | `OTP_RESEND_COOLDOWN`, `OTP_SEND_LIMIT`, `OTP_ATTEMPTS_EXCEEDED`, `RATE_LIMITED` |

### 2.9 Media pipeline

1. Client → `POST /media/uploads` (`kind`, `content_type`, `size_bytes`) → server validates type/size
   and returns `media_id` + presigned `upload_url`.
2. Client PUTs bytes directly to storage (never through Django).
3. Client → `POST /media/{id}/complete`; server verifies object exists, enqueues `media.process`
   (thumbnail/transcode/virus scan). Status `processing → ready|failed`.
4. Only `ready` media may be attached to a contribution/gallery. Ownership checked.
5. Limits: photo ≤10 MB; video ≤50 MB / 30 s; ≤4 attachments per contribution.
6. Serving: private bucket + short-lived signed URLs; EXIF GPS stripped on ingest (privacy).

### 2.10 Notification pipeline

- Followers of a location + users with an active visit are the audiences.
- Publish (live update / offer / calendar event) → Celery `fanout` job → one `Notification` row per
  recipient + FCM send; `delivered` receipt and `opened` (client ack) tracked separately.
- **Arrival prompt:** on visit creation, enqueue `prompt.arrival` with ETA = `contribution_prompt_at`.
  It re-checks the visit is still active before sending. Client also reconciles via `GET /visits/active`.
- Dedupe: one push per (user, entity, event) via idempotency key; honors `notify_followers`.
- Account with no device token → skipped, still recorded.

### 2.11 Privacy & data retention

- Public identity = `handle` + `avatar` only; never name/email/phone; Contributors are pseudonymous.
- No public photos of people; business gallery + community media only. EXIF stripped.
- Retention: OTP 10 min after use; rate-limit keys 24 h; `Notification` 180 d; media until content
  deleted + 30 d; analytics snapshots aggregated (no PII); audit logs 2 years.
- Account deletion: 30-day grace (`scheduled_for`); on purge, authored contributions are detached to
  a system identity (aggregate data preserved), media deleted, sessions revoked.
- Location is stored at the granularity the user chose (city / GPS point), never published.

### 2.12 Audit logging

Every admin mutation and every business publish/end/cancel writes an immutable `AuditLog`
(`actor_id`, `realm`, `action`, `entity_type`, `entity_id`, `before`, `after`, `ip`, `created_at`).
Admin queue reads are logged too. No secrets/OTPs in audit payloads.

### 2.13 Location input formats & standards (correction §5)

Canonical representation is **WGS84 (EPSG:4326) decimal degrees**, stored as PostGIS
`geography(Point,4326)`. Clients may send location in any of these forms; the server normalises to
`lat`/`lng`:

| Form | Example | Field |
| --- | --- | --- |
| Coordinates | `39.0997, -94.5786` | `lat`, `lng` (numbers) |
| Device GPS | OS-provided fix | `source:"gps"`, `lat`, `lng` |
| Coordinate string | `"39.0997,-94.5786"` | `location` (string) |
| DMS | `"39°05'59\"N 94°34'43\"W"` | `location` (string) |
| Geohash / Plus Code | `"9yu5…"` / `"86HF+QX Kansas City"` | `location` (string) |
| Free text / address | `"Kansas City"` | `q` (geocoded) |

Rules: `lat`∈[−90,90], `lng`∈[−180,180]; an unparseable string → `422 VALIDATION_FAILED`; a string
that geocodes to >1 match returns candidates (for `/locations/search`) or the best match (for
`PUT /me/location`). Never trust a client coordinate without range validation.

### 2.14 Routing & ETA provider (correction §9)

Route, polyline, distance, ETA and `arrive_at` come from a **maps/routing provider** (Google Directions
/ Mapbox / self-hosted OSRM) called **server-side** in `GET /places/{id}/directions`. The server never
returns a straight-line distance as an ETA and the client never fabricates a route. Turn-by-turn
rendering and the arrival geofence stay on-device; arrival is reported to `POST /visits` (§9). Provider
errors/timeouts/rate limits degrade to a safe message + retry (never a guessed route).

---

## 3. Data model (PostgreSQL + PostGIS)

`id` = UUIDv7 unless noted. All `created_at/updated_at` UTC. Geo columns are `geography(Point,4326)`.
Enum values are enforced by lookup tables/`CheckConstraint`, never free text.

### 3.1 Accounts & auth
| Entity | Key fields | Relations / indexes |
| --- | --- | --- |
| `User` | id, email (unique, citext), password_hash (null for social), account_type, email_verified, is_locked, failed_login_count, created_at | 1:1 Identity, 1:N Session/Device/Visit/Follow/SavedPlace/ChatSession/Media/Contribution |
| `Identity` | user_id (PK/FK), handle (unique), avatar_id, avatar_url | 1:1 User |
| `Session` | id, user_id, refresh_hash, realm, platform, expires_at, revoked_at, rotated_from | idx (user_id, revoked_at) |
| `Device` | id, user_id, platform, push_token, push_enabled, updated_at | uniq (push_token) |
| `Otp` | id, email, code_hash, purpose, expires_at, attempts, send_count, resend_available_at | idx (email, purpose, created_at) |
| `PasswordResetToken` | id, user_id, token_hash, expires_at, used_at | — |
| `AdminUser` | id, email, password_hash, admin_role, is_active | separate realm |

### 3.2 Places
| Entity | Key fields | Notes |
| --- | --- | --- |
| `Place` | id, name, category, description, phone, email, website, instagram, source (`imported|business|manual`), external_ref, status (`active|hidden`) | The POI/business. **uniq (external_ref, source)**; idx (category) |
| `Location` | id, **place_id (FK)**, location_name, address_line1, city, region, country, **geo** `geography(Point,4326)`, timezone, environment, amenities[] | A branch under a Place (D26). **GIST(geo)**; idx (place_id), (city). MVP: 1 Location per Place |
| `LocationHours` | location_id, weekday, open_time, close_time, closed | uniq (location_id, weekday) |
| `RightNowSnapshot` | location_id (PK), place_id, crowd, noise, wait_minutes, parking, reliability_level, status, computed_at | derived cache, per branch |
| `PlaceAlias` | place_id, alias | dedup merge history |
| `ImportBatch` | id, source, file/ref, counts, created_by | admin import audit |

### 3.3 Discovery
| Entity | Key fields | Notes |
| --- | --- | --- |
| `ChatSession` | id, user_id, origin_geo, status, created_at | — |
| `ChatMessage` | id, session_id, role, text, structured_intent JSONB, created_at | idx (session_id, created_at) |
| `ChatQuickReply` | id, message_id, label, value | — |
| `ChatResult` | id, message_id, place_id, location_id, rank, why_this_fits | — |

> **No Not Right Now entity:** the sheet is client-side in MVP (D24); a rejection refines the local
> result list and is not persisted server-side.

### 3.4 Visits & contributions
| Entity | Key fields | Notes |
| --- | --- | --- |
| `Visit` | id, user_id, place_id, location_id, method (`auto|manual`), arrived_at, arrived_geo, distance_m, verified, contribution_prompt_at, contribution_due_at, status (`active|expired`), expired_at | **uniq partial (user_id) WHERE status='active'**; idx (location_id, arrived_at) |
| `Contribution` | id, user_id, place_id, location_id, visit_id (FK NOT NULL), crowd, noise, wait_bucket, wait_minutes, parking, atmosphere[], note, status (`published|hidden|removed`), created_at | idx (location_id, created_at desc); uniq (visit_id); uniq (user_id, visit_id) |
| `ContributionMedia` | contribution_id, media_id, position | uniq (contribution_id, position) |
| `Media` | id, owner_id, kind, purpose (`community|claim_document|gallery`), content_type, size_bytes, storage_key, url, thumbnail_url, status (`processing|ready|failed`), created_at | idx (owner_id, status) |

### 3.5 Social & rewards
| Entity | Key fields | Notes |
| --- | --- | --- |
| `Follow` | user_id, place_id, notifications, created_at | **uniq (user_id, place_id)**; idx (place_id) — **follower SoT** |
| `SavedPlace` | user_id, place_id, location_id?, created_at | **uniq (user_id, place_id)** (D26/D29); response echoes `location{id,name,lat,lng}` so the client can compute distance |
| `UserProgression` | user_id (PK), xp_total, level, credibility_level, updated_at | — |
| `Badge` | id, code (unique), name, description, icon_url, criteria JSONB, active | admin-managed |
| `UserBadge` | user_id, badge_id, earned_at, progress | uniq (user_id, badge_id) |
| `XpRule` | id, action, xp, daily_cap, active | admin-managed table |

### 3.6 Business
| Entity | Key fields | Notes |
| --- | --- | --- |
| `BusinessAccount` | id, owner_user_id, name, status (`active|suspended`) | provider realm |
| `BusinessMember` | id, business_account_id, **place_id**, user_id?, email, name, role (`owner|staff`), status (`invited|active|disabled`), permissions[] | uniq (place_id, email); idx (user_id) |
| `Invitation` | id, business_account_id, email, role, permissions[], token_hash, expires_at, accepted_at | — |
| `Claim` | id, user_id, **place_id**, status, submitted_at, decided_at, decided_by, decision_reason, allow_resubmit | **uniq partial (place_id) WHERE status in (draft,pending_review)** |
| `ClaimRequirement` | key, kind (`field|document`), label, placeholder, type, required, validation JSONB, active | admin-configurable |
| `ClaimValue` / `ClaimDocument` | claim_id, key, value / media_id, file_name, size_bytes | — |
| `LiveBizUpdate` | id, **place_id**, text(≤280), category, status (`scheduled|published|expired|ended`), publish_mode, scheduled_for, published_at, expires_at, notify_followers, created_by | idx (place_id, status, scheduled_for) |
| `CalendarItem` | id, **place_id**, item_type, title, publish_mode, scheduled_at, status (`scheduled|active|cancelled|completed`), offer_id?, created_by | idx (place_id, scheduled_at) |
| `Offer` | id, **place_id**, title, terms, status (`scheduled|active|expired|ended`), publish_mode, valid_from, valid_to, code, quantity_limit, notify_followers, created_by | **uniq (place_id, code)** |
| `OfferRedemption` | id, offer_id, user_id, place_id, redeemed_at | uniq (offer_id, user_id) when single-use |
| `GalleryMedia` | id, **place_id**, media_id, status (`pending_review|approved|rejected`), position | — |
| `Report` | id, **place_id**, affected_content_type (`live_update|offer|gallery_media|contribution`), affected_content_id, label, reason, note, reporter_user_id?, status (`pending_review|upheld|dismissed`), action (`none|remove|restore`), decided_by, reported_at | idx (status, reported_at) |
| `Dispute` | id, **place_id**, submitted_by_member_id, subject, related_content, report_id?, details, status (`open|in_progress|resolved`), decision, submitted_at | idx (status) |
| `AnalyticsDaily` | place_id, date, followers, followers_delta, notif_delivered, notif_opened, post_views, coupon_views, coupon_redemptions, community_updates | uniq (place_id, date) |

> Business-authored content keys on **`place_id`** (the business); live conditions (Right Now, visits,
> contributions) key on **`location_id`** (the branch the user was at). With one branch per place in
> MVP they resolve 1:1.
| `AuditLog` | id, actor_id, realm, action, entity_type, entity_id, before JSONB, after JSONB, ip, created_at | idx (entity_type, entity_id), (actor_id, created_at) |

### 3.7 Geo & performance indexes
- `Location` GIST on `geo`; B-tree on `place_id`, `city`.
- `Place` B-tree on `category`, `status`.
- `Contribution(location_id, created_at DESC)` — Right Now window scans.
- `Follow(place_id)` — follower counts without table scans.
- `Visit` partial unique on active; `Contribution(visit_id)` unique; `Contribution(user_id, place_id, created_at)` for cooldown checks.
- `ChatMessage(session_id, created_at)`; `Notification(user_id, created_at DESC)`.
- Monthly partitioning on `Contribution`, `AuditLog`, `AnalyticsDaily` at scale.

---

## 4. Roles & permission matrix

| Capability | Visitor | Provider owner | Provider staff | Admin |
| --- | --- | --- | --- | --- |
| Chat / discover / save / follow | ✅ | ✅ | ✅ | — |
| Arrive + contribute | ✅ | ✅ | ✅ | — |
| Report content (proposed) | ✅ | ✅ | ✅ | — |
| Redeem offer (proposed) | ✅ | ✅ | ✅ | — |
| Claim a location | — | ✅ | — | ✅ (decide) |
| Live updates / calendar / offers / gallery | — | ✅ | ✅ per `permission_module` | — |
| Analytics / reports (read) | — | ✅ | ✅ per module | ✅ |
| Team & settings | — | ✅ | — | — |
| Moderate reports / approve claims / disputes | — | — | — | ✅ |
| Place import · legal/onboarding content · XP & badge rules | — | — | — | ✅ |
| Suspend users | — | — | — | ✅ |

Enforcement is **server-side** on every route. `claim_status != approved` → `403 CLAIM_NOT_APPROVED`
(claim + auth routes exempt). Missing module → `403 FORBIDDEN_MODULE`. **Only an admin can move a claim
to `approved`** (owner correction): the business portal is fully gated until the admin approves that
business panel.

---

## 5. Enums & shared objects

| Enum | Values |
| --- | --- |
| `reliability_level` | `high`, `medium`, `low`, `insufficient_data` |
| `crowd_level` | `light`, `moderate`, `busy`, `packed` |
| `noise_level` | `quiet`, `moderate`, `loud` |
| `wait_bucket` (submit only) | `none`, `5_min`, `10_min`, `20_plus_min` |
| `parking_level` | `easy`, `moderate`, `difficult`, `full`, `not_applicable` (+ computed `limited`) |
| `atmosphere` | `cozy`, `lively`, `romantic`, `casual`, `trendy`, `loud_music` |
| `environment` | `outdoors`, `indoors` |
| `not_right_now_reason` | `too_crowded`, `too_loud`, `too_expensive`, `too_far`, `not_the_right_atmosphere`, `changed_my_mind`, `other` |
| `travel_mode` | `walking`, `driving`, `transit` |
| `account_type` | `consumer`, `provider` |
| `otp_purpose` | `email_verification`, `password_reset` |
| `claim_status` | `none`, `draft`, `pending_review`, `approved`, `rejected` |
| `permission_module` | `overview`, `live_updates`, `promotions_calendar`, `offers_coupons`, `analytics`, `location_profile`, `reports` |
| `publish_mode` | `now`, `schedule` |
| `live_update_status` | `scheduled`, `published`, `expired`, `ended` |
| `offer_status` | `scheduled`, `active`, `expired`, `ended` |
| `calendar_item_type` | `update`, `special`, `event`, `coupon`, `reminder` |
| `calendar_item_status` | `scheduled`, `active`, `cancelled`, `completed` |
| `report_status` | `pending_review`, `upheld`, `dismissed` |
| `report_action` | `none`, `remove`, `restore` |
| `dispute_status` | `open`, `in_progress`, `resolved` |
| `team_status` | `invited`, `active`, `disabled` |
| `visit_status` | `active`, `expired` |
| `contribution_status` | `published`, `hidden`, `removed` |
| `weekday` | `monday` … `sunday` |

**Shared objects**
- `Author` = `{ handle, avatar:{ id, url } }` (never name/email).
- `RightNow` = `{ crowd|null, noise|null, wait_minutes|null, parking|null, status:"available"|"limited_current_information", updated_at }`.
- `PlaceCard` = `{ id, name, distance_m, is_open, reliability:{level}, updated_at, right_now|null, tags[], live_biz_update|null, event|null, offer|null, why_this_fits|null, is_saved }`.
- `Page` = `{ items[], meta:{ next_cursor } }`.
- `Reliability` = `{ level, community_updates_count, live_biz_updates_count, most_recent_update_at, explanation }`.

## 6. Core rules & algorithms (Rule 2)

### 6.1 Right Now algorithm (one service: `right_now.compute(location)`)

Used by PlaceCard, Details, Compare, the why-not-here sheet (via Details), directions, and recommendations. One implementation.

| Parameter | Value (configurable) | Rule |
| --- | --- | --- |
| Observation window | **last 90 min** for community signals; live biz updates valid while `published` and not expired (max 6 h) | Older community updates never affect the verdict. |
| Minimum contributors | **≥3 distinct users** in-window → `available`; **1–2** → computed but `reliability=low`; **0** → `limited_current_information` | Distinct-user count, not row count (anti-spam). |
| Credibility weight | `w = credibility[user] × recency`, recency `= 0.5^(age_min/45)` | A high-credibility fresh update outweighs several stale/low ones. |
| Aggregation | per signal independently: **weighted mode** for categorical (`crowd`,`noise`,`parking`), **weighted median** for `wait_minutes` | Never average unrelated signals (no crowd-vs-parking mixing). |
| Live update bias | A recent (`≤2 h`) published live update overrides community for the signal it speaks to (e.g., explicit wait time) | Business is authoritative about its own state. |
| `wait_minutes` | submission `wait_bucket` maps `none→0, 5_min→5, 10_min→10, 20_plus_min→25`; live updates may set an explicit value | Read responses are numeric. |
| Staleness | `updated_at = max(newest contribution, newest live update)`. Newest age > **6 h** → `limited_current_information` | Stale data is never shown as current. |
| `parking=limited` | computed display label when weighted result is `difficult`/`full`, or when parking evidence is thin but present | Submission chips never include `limited`. |
| Reliability | `high` if ≥5 weighted-eq contributors and top wins >60% and newest ≤20 min; `medium` if ≥3 and newest ≤45 min; `low` if 1–2; else `insufficient_data` | Confidence, not just recency. |
| Thin data | Return `status=limited_current_information`, four signal fields `null`; the card still shows Reliability + Live Biz Update + Event; never invent | Empty state is explicit. |

### 6.2 Presence-gated contribution

| Rule | Value |
| --- | --- |
| `visit_id` | **REQUIRED** on `POST /places/{id}/community-updates` (D12) |
| Visit validity | Visit must exist, belong to the caller, be `status=active`, and not be past `contribution_due_at` (= `arrived_at + 3 h`) → else `422 VISIT_NOT_ACTIVE` |
| Arrival verification | At `POST /visits`, caller's `location` must be within **150 m** (configurable) of `Location.geo` → else `422 TOO_FAR_FROM_PLACE`; store `distance_m` + `verified` |
| Arrival radius policy | Refuse if device accuracy (if provided) > 200 m; a `verified` visit is the only visit that can contribute |
| Time window after arrival | Contributions accepted from `arrived_at` up to `arrived_at + 3 h` |
| One update per visit | Unique `(visit_id)` — a second submit on the same visit → `409 ALREADY_CONTRIBUTED` (or reuse the update route) |
| Place cooldown | Same user + same place within **6 h** → `429 UPDATE_COOLDOWN` |
| Daily cap | 10 contributions/day/user → `429 RATE_LIMITED` |
| Media | ≤4 items, all `ready` and owned → else `422 MEDIA_NOT_READY` / `MEDIA_LIMIT_EXCEEDED` |
| Note | ≤500 chars |
| Signals | At least one of crowd/noise/wait/parking/atmosphere/note → else `422 NO_SIGNAL_PROVIDED` |
| XP | Awarded exactly once per successful submit (idempotent). |

### 6.3 Arrival prompt (server-scheduled, D11)

1. `POST /visits` creates the visit and enqueues `prompt.arrival` (Celery ETA, `contribution_prompt_at =
   arrived_at + 10 min`).
2. At ETA the worker re-checks the visit is `active`; if so, sends a push "You're at {place}…" and
   records a `Notification`.
3. The client schedules a **local** notification as a convenience only and reconciles on launch/resume
   via `GET /visits/active` (which returns `contribution_due`). The phone timer is never authoritative.
4. If the user deleted the app or has no token, the visit still expires on schedule.

### 6.4 Places source, dedup, editing, claimability

| Step | Rule |
| --- | --- |
| Seed/import | Admin `POST /admin-api/places/import` ingests a Places provider dump (name, address, geo, hours, external_ref). Batches are recorded in `ImportBatch`. |
| Deduplication | Match on `external_ref`; else name-normalised + **<75 m** geo distance. Collisions merge into the surviving location and record `PlaceAlias`; merges are audited and reversible. |
| Editing | Admin can edit any field or hide a place (`status=hidden`). Business edits are limited to the Location Profile (hours/contact/links/description/media) — never name/geo (re-verification required). |
| Claimability | `claimable = location.claim_status in (none) and status=active`. A location with an open/approved claim is not claimable. |
| One claim | Partial unique index allows only one `draft|pending_review` claim per location. |

### 6.5 Follower count & analytics derivation (D16)

- **Single source of truth:** `COUNT(Follow) WHERE location_id = X` (denormalised to a counter updated
  transactionally on follow/unfollow).
- `overview.followers.total` = that count. `followers.new_this_week` = follows with
  `created_at >= now-7d`.
- `analytics.follower_growth.change` = `followers(period_end) − followers(period_start)`;
  `.change_percent` = change / start. The daily series reads `AnalyticsDaily.followers`.
- `analytics.aggregate.followers` **must equal** `overview.followers.total` for the same period.
  The draft's 12,450 vs 4,389 is a mock-data bug, not two metrics (see §12).
- Rollup job snapshots counts nightly at 00:05 location-local.

### 6.6 XP, level, credibility, badges

**XP rules (`XpRule` table, admin-editable):**

| Action | XP | Cap |
| --- | --- | --- |
| Submit community update | +50 | — |
| Include a photo | +15 | per update |
| Verified visit (auto or manual) | +10 | — |
| First update of the day | +10 | daily |
| Update flagged then upheld by admin | **−50** (clawback) | — |

**Level:** threshold table `level_n_xp = 100 × n(n+1)/2` (L1=0, L2=100, L3=300, L4=600, L5=1000,
L6=1500, L7=2100 …). `level` = highest n with `xp_total ≥ threshold`. `next_level_xp` returned in
`GET /me`.

**Credibility:** recency-weighted agreement of a user's signals with the eventual Right Now verdict over
their last 20 verified contributions. `≥0.7 → high`, `0.4–0.7 → medium`, `<0.4 → low`; new users start
`medium`. Feeds the Right Now weight (6.1) and is surfaced on Profile.

**Badges (`Badge.criteria` JSONB, admin-managed):** e.g. `first_update` (1 contribution),
`photo_contributor` (10 photo contributions), `neighborhood_expert` (25 contributions at ≥5 places),
`trusted_voice` (credibility `high` sustained 30 d). A nightly job evaluates and grants `UserBadge`;
grants are idempotent.

### 6.7 Coupon redemption & content reporting (PROPOSED — no Figma screen, D15)

Designed because the portal's Analytics (redeem) and Reports (report) depend on the producing side.

- **Reporting:** `POST /places/{id}/reports` (app) creates a `Report` on a live update, offer, gallery
  media, or contribution. Fields: `affected_content_type`, `affected_content_id`, `reason` (enum +
  `other`), `note?`. One open report per (reporter, content). Then the admin `decision` endpoint
  applies `action` (`none|remove|restore`). Portal sees reports in Reports/Need Attention.
- **Redemption:** `POST /offers/{id}/redeem` records an `OfferRedemption` (unique per
  `(offer_id, user_id)`), decrements `quantity_available`, rejects non-`active`/expired/out-of-stock
  → `422 OFFER_NOT_REDEEMABLE`. Portal Analytics reads views vs redemptions.
- Both are marked **PROPOSED-NO-FIGMA** in §8 so the owner can accept or drop them.

### 6.8 Identifier model (place ↔ location, D26)

- `Place` = the POI/business (`place_id`). All app reads and Saved are keyed by `place_id`.
- `Location` = a physical branch under a Place: `{ location_id, location_name, lat, lng }`.
- API responses expose both: `{ "id": "<place_id>", "name": "...", "location": { "location_id":
  "...", "name": "...", "lat": 39.0997, "lng": -94.5786 } }`. `distance_m` is computed from the
  caller's `lat`/`lng` to the location's coordinates.
- MVP is **1 place : 1 location** (Kansas City); the model is 1:N so branches can be added later
  without an API break. Saved references `place_id` and echoes its `location`.

### 6.9 "Not right now" & "Why not here" — client-side (D24/D25)

Both sheets are rendered on the device. "Not right now" filters the current result list locally;
"why not here" reads the `GET /places/{id}` payload. Neither adds an endpoint or a server entity.

### 6.10 Contribute API surface — confirmed against the UI (owner Q §11)

Every Contribute endpoint maps to a designed screen, so all are kept: `GET /contribute/places`
(Contribute `641:1072`), `POST /media/uploads` + `/complete` (Add Photo or Video `847:149`),
`POST /places/{id}/community-updates` (Community Update → Review → Updated), `PUT/DELETE
/places/{id}/follow` (After click done `771:782`). Nothing extra.

---

## 7. State machines

Transition triggers: **U**=user/app, **B**=business, **A**=admin, **S**=system/worker.

**Claim (D-claim)** — admin decision is the only path to `approved`; rejection may allow one resubmit.
```
none ──U: POST /business/claims──► draft ──U: submit──► pending_review
pending_review ──A: approve──► approved            (portal modules unlock)
pending_review ──A: reject(allow_resubmit)──► rejected ──U: resubmit──► pending_review
pending_review ──A: reject(no resubmit)──► rejected (terminal)
```

**Visit** — one active per user; expires automatically; `POST /visits` is the only creation trigger.
```
(none) ──U: POST /visits (auto|manual, within radius)──► active
active ──S: now > contribution_due_at (arrived_at+3h)──► expired
active ──S: new arrival starts (superseded)──► expired
active ──U: (no manual end endpoint; D10)
```

**Community update** — moderation is post-hoc; auto-publish unless flagged.
```
(none) ──U: POST /places/{id}/community-updates (visit_id required)──► published
published ──A: remove──► removed    published ──A: hide──► hidden
published ──A(uphold report)──► removed
hidden ──A: restore──► published    removed ──A: restore──► published
```

**Live Biz Update** — server decides status from time; publishing is idempotent.
```
(none) ──B: POST (publish_mode=now)──► published
(none) ──B: POST (publish_mode=schedule)──► scheduled ──S: scheduled_for──► published
published ──S: now > expires_at──► expired
published|scheduled ──B: POST /end──► ended (terminal; audited)
```

**Offer** — lifecycle-limited edit (only `scheduled` offers are editable).
```
(none) ──B: POST (now)──► active      (none) ──B: POST (schedule)──► scheduled
scheduled ──B: PATCH (allowed only here)──► scheduled
scheduled ──S: valid_from──► active
active ──S: valid_to / quantity exhausted──► expired
scheduled|active ──B: POST /offers/{id}/end──► ended (terminal)
```

**Calendar item** — scheduled items never read as active early.
```
(none) ──B: POST──► scheduled ──S: scheduled_at──► active ──S: end──► completed
scheduled ──B: POST /cancel──► cancelled (terminal)
```

**Report** — business cannot decide its own reports; only admin.
```
(none) ──U: POST /places/{id}/reports──► pending_review
pending_review ──A: decision(uphold, action=remove|restore|none)──► upheld
pending_review ──A: decision(dismiss)──► dismissed    (both terminal)
```

**Dispute** — owned by the business, decided by admin.
```
(none) ──B: POST /business/disputes──► open
open ──A: pick-up──► in_progress ──A: resolve──► resolved (terminal)
```

**Business member / invitation**
```
(none) ──owner: POST /business/team/invitations──► invited
invited ──S: expires_at──► (expired, INVITATION_EXPIRED) ──owner: reinvite──► invited
invited ──invitee: POST /business/invitations/{token}/accept──► active
active ──owner: DELETE /business/team/{id}──► disabled
```

**Navigation:** removed (D9). Turn-by-turn state is device-local.

## 8. Endpoint catalog (Rule 1 — every endpoint justified)

Legend — **Consumer:** app / portal / admin / system. **Screen** refers to Figma frames verified in
metadata. Endpoints marked `PROPOSED-NO-FIGMA` are deliberate additions with no design (D15).
Endpoints with no screen, no admin need, and no rule were **removed** (see §15 / `CHANGELOG_API.md`).

### 8.1 Consumer app (`/api/v1`)

| Method & path | Why it exists | Consumer | Screen / rule | Fails if removed |
| --- | --- | --- | --- | --- |
| `POST /auth/register` | Create an email account and send the verification code | app | Register `76:32` | No sign-up |
| `POST /auth/login` | Authenticate and return routing flags | app | Sign In `74:518` | No sign-in |
| `POST /auth/social/{provider}` | Sign in/up with Google/Apple | app | Sign In/Register buttons | No social auth |
| `POST /auth/forgot-password` | Start reset (no enumeration) | app | Forgot Pass `76:173` | No recovery |
| `POST /auth/otp/verify` | Verify email or exchange reset code for a reset token | app | OTP `76:409` | Email never verifiable; no reset |
| `POST /auth/otp/resend` | Reissue an OTP | app | OTP "Send again" | User stuck on expired code |
| `POST /auth/reset-password` | Set a new password with the reset token | app | Create New Password `76:475` | No recovery |
| `POST /auth/refresh` | Rotate the session | app | §2.4 | 15-min sessions only |
| `POST /auth/logout` | Revoke refresh token(s) | app | Profile > Log Out `849:1094` | Sessions outlive intent |
| `GET /me/identity` | Show the system handle + avatar | app | Identity ready `208:336` | No public identity shown |
| `PUT /me/account-type` | Record consumer vs provider intent and where to go next | app | Identity ready buttons | Can't route provider to portal |
| `GET /locations/search` | Geocode a typed address or place string | app | Add address manually `847:865` | Manual location impossible |
| `GET /me/location` | Return the current city/location for the header | app | Home header | Header has no city |
| `PUT /me/location` | Persist GPS or a chosen address/coordinate string (D27) | app | Allow Location `847:961` | Home can't know where "nearby" is |
| `GET /home` | Home payload (city, time-of-day, chips, active visit) | app | Home `225:1012`, Home after visit `536:394` | Home can't render |
| `POST /chat/sessions` | Start a discovery conversation | app | Chat `225:1209` | No discovery |
| `POST /chat/sessions/{id}/messages` | Send a turn / quick reply; get ranked places | app | Chat, Result `139:1912` | Discovery stalls |
| `GET /chat/sessions/{id}/messages` | Reopen a conversation | app | Chat history | History lost on restart |
| `GET /places/{id}` | Full Details payload — includes `location{location_id,name,lat,lng}`, Right Now, Reliability detail (counts + explanation) and the Why-not-here fields, so no separate reliability/why-not-here endpoints are needed | app | Details `847:474`; Reliability `380:661`; Why not here `293:323` | No place page |
| `GET /places/{id}/community-updates` | Paginate the community feed | app | Details "see all" | Only 3 updates ever visible |
| `GET /places/compare` | Side-by-side compare 2–3 places | app | Compare `251:1871` | Compare blank |
| `GET /places/{id}/directions` | Route + ETA + arrive-at | app | Get Direction `470:199` | No directions |
| `GET /contribute/places` | Places that most need a fresh update | app | Contribute `641:1072` | Contribute blank |
| `PUT /places/{id}/follow` | Follow + receive updates | app | After click done `771:782` | No followers → notify pipeline dead |
| `DELETE /places/{id}/follow` | Unfollow | app | follow state | Can't unsubscribe |
| `POST /places/{id}/community-updates` | Submit a presence-gated observation | app | Community Update/Review/Updated | The loop's input is gone |
| `POST /places/{id}/reports` **PROPOSED-NO-FIGMA** | Report content (portal reports depend on it) | app | §6.7 | Portal Reports has no producer |
| `POST /offers/{id}/redeem` **PROPOSED-NO-FIGMA** | Redeem a coupon (portal analytics depends on it) | app | §6.7 | Coupon redemptions can never occur |
| `POST /media/uploads` | Presigned upload start | app | Add Photo/Video `847:149` | No media |
| `POST /media/{id}/complete` | Confirm upload; trigger processing | app | Add Photo/Video | Media stuck `processing` |
| `POST /visits` | Report arrival (auto or manual) → presence gate | app | Manual Arrival `522:375` | Contributions impossible |
| `GET /visits/active` | Active visit + server-derived prompt due time | app | Home after visit, Show it after 10 min `518:74` | Prompt/visit state lost |
| `GET /me/saved-places` | Saved list | app | Saved `631:97` | Saved blank |
| `PUT /me/saved-places/{id}` | Save | app | save heart | Can't save |
| `DELETE /me/saved-places/{id}` | Unsave | app | save heart | Can't unsave |
| `GET /me` | Profile + progression + personal information (one call) | app | Profile `849:1094`, Personal Information `666:2257` | Profile and info blank |
| `PATCH /me` | Change email (OTP-gated) | app | Personal Information | Can't change email |
| `POST /me/email/verify` | Confirm the new email | app | email OTP | Email change never completes |
| `POST /me/password` | Change password | app | Change Password `666:2385` | Can't rotate password |
| `POST /me/devices` | Register push token | app | notifications | Followers can't be pushed |
| `POST /me/account-deletion` | Schedule deletion (30 d) | app | Account Deletion | Can't delete account |
| `DELETE /me/account-deletion` | Cancel deletion in grace | app | Account Deletion | No undo |

> **Owner Q answers.** (1) In `/places/{id}?lat=&lng=`, `{id}` is the **Place** identifier (PK); `lat`
> / `lng` are the **caller's** current coordinates, used to compute `distance_m` for that place — they
> are not the place's own location. (2) `cursor` is an opaque pagination token returned as
> `meta.next_cursor` (null on the last page); send it back to fetch the next page — it is not an id.
> (3) **Removed (owner correction, D23/D24):** `GET /content/onboarding`, `PUT /me/onboarding`,
> `GET /content/legal/{slug}` (frontend-owned) and `POST /chat/…/not-right-now` (client-side).

### 8.2 Business portal (`/business`)

| Method & path | Why it exists | Consumer | Screen / rule | Fails if removed |
| --- | --- | --- | --- | --- |
| `POST /business/auth/register` | Portal self sign-up (D13) | portal | Log in (new) | Owner must install the app first |
| `POST /business/auth/login` | Provider sign-in + claim routing | portal | Log in `820:10473` | No portal access |
| `POST /business/auth/refresh` | Rotate provider session | portal | §2.4 | Short sessions only |
| `POST /business/auth/logout` | Revoke provider session | portal | — (no control drawn but required) | Sessions outlive intent |
| `POST /business/auth/forgot-password` | Recovery start | portal | Forgot Password `820:10505` | No recovery |
| `POST /business/auth/otp/verify` | Verify + issue reset token | portal | Verify Email `820:10524` | Recovery blocked |
| `POST /business/auth/otp/resend` | Resend code | portal | Resend Code | Blocked on expiry |
| `POST /business/auth/reset-password` | Set new password | portal | Create New Password `820:10553` | Recovery blocked |
| `GET /business/invitations/{token}` | Show invite details before accepting | portal | Staff invite email (new) | Invitee can't review |
| `POST /business/invitations/{token}/accept` | Accept invite, set password, activate member | portal | Staff invite email (new) | Staff can never join |
| `GET /business/claims/current` | Show claim status / resume | portal | Claim status `871:2023` | Owner can't see decision |
| `GET /business/locations/search` | Find the business to claim | portal | Find your business `868:1645` | Can't claim |
| `POST /business/claims` | Open a claim draft | portal | Is this your business? `868:1765` | Can't claim |
| `GET /business/claims/{id}/requirements` | Admin-configured verification form | portal | Verify Your Business `869:1840` | Dynamic form can't render |
| `GET /business/claims/{id}` | Review claim summary | portal | Review your claim `871:1920` | Can't review |
| `PUT /business/claims/{id}/verification` | Save entered fields + docs | portal | Continue | Claim incomplete |
| `POST /business/claims/{id}/submit` | Submit for review | portal | Submit `871:1978` | Claim never enters queue |
| `POST /business/uploads` | Presigned upload for claim docs / gallery | portal | Upload File | No docs/gallery |
| `POST /business/uploads/{id}/complete` | Confirm upload | portal | Upload File | Upload stuck |
| `GET /business/overview` | Dashboard summary | portal | Overview `925:328` | Overview blank |
| `GET /business/community-updates` | Paged Recent Engagement | portal | View All | Only 3 items |
| `GET /business/live-updates` | List updates + status | portal | Live Biz Updates `691:3361` | List blank |
| `POST /business/live-updates` | Author a live update | portal | Create Update modal `698:3585` | No live updates |
| `POST /business/live-updates/{id}/end` | End early (lifecycle, audited) | portal | list action (D20) | Cannot stop a live update |
| `GET /business/calendar` | Month/week/schedule data | portal | Promotions Calendar `700:3748` | Calendar blank |
| `POST /business/calendar/items` | Schedule an item | portal | Schedule Item modal `702:4841` | Can't schedule |
| `POST /business/calendar/items/{id}/cancel` | Cancel a scheduled item | portal | (D20) | Can't cancel |
| `GET /business/offers` | Offer list | portal | Offers `710:5700` | Offers blank |
| `POST /business/offers` | Create offer | portal | Create Offer modal `710:5618` | No offers |
| `PATCH /business/offers/{id}` | Edit a not-yet-active offer | portal | (D20) | Typos become permanent |
| `POST /business/offers/{id}/end` | End an offer | portal | (D20) | Can't stop promotion |
| `GET /business/analytics` | Charts + aggregate cards | portal | Analytics `702:5049` | Analytics blank |
| `GET /business/location-profile` | Public profile data | portal | Location Profile `791:8570` | Profile blank |
| `PUT /business/location-profile` | Edit hours/contact/links/description | portal | Edit Profile `799:8923` | Can't edit |
| `POST /business/location-profile/gallery` | Attach media (pending review) | portal | Add Media | No gallery |
| `DELETE /business/location-profile/gallery/{media_id}` | Remove gallery media | portal | gallery action | Can't remove |
| `GET /business/team` | Team list | portal | Team `801:9377` | Team blank |
| `POST /business/team/invitations` | Invite staff | portal | Invite Staff | Can't add staff |
| `PUT /business/team/{id}/permissions` | Assign module permissions | portal | Assign Permissions modal `807:9668` | Staff over/under-permissioned |
| `DELETE /business/team/{id}` | Remove staff | portal | Actions column | Can't revoke access |
| `GET /business/reports` | Reports affecting the business | portal | Reports (`808:9892`) | Reports blank |
| `GET /business/reports/{id}` | Report detail + content | portal | Need Attention | Can't inspect a report |
| `GET /business/disputes` | Dispute list | portal | Disputes | Disputes blank |
| `POST /business/disputes` | Submit a dispute/support request | portal | Submit dispute modal `817:10251` | Can't dispute |
| `GET /business/me` | Profile settings | portal | Settings–My Profile `927:610` | Settings blank |
| `PATCH /business/me` | Update name/phone/email | portal | Save Changes | Can't edit settings |
| `POST /business/me/password` | Update password | portal | Settings–Security `951:642` | Can't rotate password |

### 8.3 Admin panel (`/admin-api`) — no Figma (D14)

> **Owner confirmation:** this is the second dashboard. It is the only surface that can approve a
> business (claim) panel and control moderation, content, XP/badge rules and place data. When a claim
> is approved the corresponding business portal unlocks (§4).

| Method & path | Why it exists | Consumer | Screen / rule | Fails if removed |
| --- | --- | --- | --- | --- |
| `POST /admin-api/auth/login` | Separate admin realm (D22) | admin | — | No admin access |
| `POST /admin-api/auth/refresh` | Rotate admin session | admin | §2.4 | Short sessions |
| `POST /admin-api/auth/logout` | Revoke admin session | admin | — | Sessions outlive intent |
| `GET /admin-api/dashboard` | Pending-work summary | admin | Rule 4 | No queue overview |
| `GET /admin-api/users` | Search consumer/provider accounts | admin | Rule 4 | Cannot support users |
| `GET /admin-api/users/{id}` | Account detail + activity | admin | Rule 4 | Investigations blocked |
| `PATCH /admin-api/users/{id}/status` | Suspend/reactivate | admin | Rule 4 | Abuse uncontainable |
| `GET /admin-api/claims` | Claim review queue | admin | Rule 4 | Claims never decided → portal locked |
| `POST /admin-api/claims/{id}/approve` | Approve claim | admin | Claim SM §7 | Portal modules never unlock |
| `POST /admin-api/claims/{id}/reject` | Reject with reason (+resubmit) | admin | Claim SM §7 | No rejection path |
| `GET /admin-api/claim-requirements` | Read verification template | admin | Verify Your Business form | Form unconfigurable |
| `PUT /admin-api/claim-requirements` | Change what the claim form asks | admin | Rule 4 | Form hard-coded |
| `GET /admin-api/reports` | Moderation queue | admin | Rule 4 | Reports never actioned |
| `POST /admin-api/reports/{id}/decision` | Dismiss/uphold + remove/restore | admin | Report SM §7 | Reported content stays |
| `GET /admin-api/disputes` | Dispute queue | admin | Rule 4 | Disputes unattended |
| `PATCH /admin-api/disputes/{id}` | Update/resolve a dispute | admin | Dispute SM §7 | Disputes never resolve |
| `GET /admin-api/community-updates` | Community moderation queue | admin | Rule 4 | Harmful updates stay public |
| `POST /admin-api/community-updates/{id}/decision` | Hide/remove/restore an update | admin | Contribution SM §7 | No moderation |
| `GET /admin-api/media` | Photo moderation queue | admin | Rule 4 | Bad gallery/community photos stay |
| `POST /admin-api/media/{id}/decision` | Approve/reject media | admin | Gallery `pending_review` | Gallery never approved |
| `GET /admin-api/places` | Place inventory | admin | Rule 4 | Cannot curate places |
| `POST /admin-api/places` | Create a place manually | admin | §6.4 | Gaps never fixable |
| `PATCH /admin-api/places/{id}` | Edit/hide a place | admin | §6.4 | Wrong data permanent |
| `POST /admin-api/places/import` | Bulk import/geo-dedupe | admin | §6.4 | No supply at launch |
| `GET /admin-api/gamification/badges` | Read badge catalogue | admin | Rule 4 | Badges unmanageable |
| `PUT /admin-api/gamification/badges/{id}` | Create/update badge + criteria | admin | §6.6 | Badge rules frozen in code |
| `PATCH /admin-api/settings/xp-rules` | Tune XP values | admin | §6.6 | Economy un-tunable |
| `GET /admin-api/audit-logs` | Query the audit trail | admin | Rule 5 | No accountability |

**Deliberately NOT created:** view toggles, month/week switches, modal open/close, unsent review
previews, selection/Cancel, local-only search filters, or any endpoint whose only caller is client UI
state. Also not created: a public HTTP route for any internal AI/rules service (§9).

### 8.4 Request-field validation catalogue (every request field: type / required / validation / error)

Shared: `items` arrays default empty; all ids are UUID strings; unknown fields → `422 VALIDATION_FAILED`.

| Endpoint | Field | Type | Req | Validation | Error code |
| --- | --- | --- | --- | --- | --- |
| `POST /auth/register` | email | string | ✔ | RFC-5322, ≤254, unique | `EMAIL_ALREADY_REGISTERED` |
| | password | string | ✔ | ≥8, ≥1 letter, ≥1 number | `VALIDATION_FAILED` |
| | password_confirmation | string | ✔ | equals password | `PASSWORD_MISMATCH` |
| | accepted_terms | bool | ✔ | must be true | `VALIDATION_FAILED` |
| | device.platform/push_token | string | ✘ | `ios|android`; token ≤255 | `VALIDATION_FAILED` |
| `POST /auth/login` | email,password | string | ✔ | non-empty | `INVALID_CREDENTIALS`/`ACCOUNT_LOCKED` |
| `POST /auth/social/{provider}` | id_token | string | ✔ | verified with provider | `SOCIAL_TOKEN_INVALID` |
| | authorization_code | string | apple only | non-empty | `SOCIAL_TOKEN_INVALID` |
| `POST /auth/otp/verify` | email | string | ✔ | — | `OTP_INVALID` |
| | otp | string | ✔ | exactly 6 digits | `OTP_INVALID`/`OTP_EXPIRED` |
| | purpose | enum | ✔ | `email_verification|password_reset` | `VALIDATION_FAILED` |
| `POST /auth/reset-password` | reset_token | string | ✔ | valid, unexpired, unused | `RESET_TOKEN_EXPIRED` |
| | password,password_confirmation | string | ✔ | as register | `PASSWORD_MISMATCH` |
| `PUT /me/account-type` | account_type | enum | ✔ | `consumer|provider` | `VALIDATION_FAILED` |
| `GET /locations/search` | q | string | ✔ | 2–120 chars | `VALIDATION_FAILED` |
| | lat,lng | number | ✘ | −90..90 / −180..180 | `VALIDATION_FAILED` |
| `PUT /me/location` | source | enum | ✔ | `gps|manual` | `VALIDATION_FAILED` |
| | lat,lng | number | if gps | ranges | `VALIDATION_FAILED` |
| | location | string | if manual | coordinate string / DMS / geohash / Plus Code | `VALIDATION_FAILED` |
| | location_id | uuid | if manual | a picked `/locations/search` result | `NOT_FOUND` |
| | q | string | if manual | free-text place to geocode | `VALIDATION_FAILED` |
| `POST /chat/sessions` | text | string | ✔ | 1–500 | `VALIDATION_FAILED` |
| | suggestion_id | uuid | ✘ | exists | `NOT_FOUND` |
| | location.lat/lng | number | ✘ | ranges | `VALIDATION_FAILED` |
| `POST /chat/sessions/{id}/messages` | text | string | one-of | 1–500 | `VALIDATION_FAILED` |
| | quick_reply_id | uuid | one-of | exists in session | `NOT_FOUND` |
| `GET /places/compare` | ids | csv uuid | ✔ | 2–3, distinct, exist | `VALIDATION_FAILED` |
| `PUT /places/{id}/follow` | notifications | bool | ✔ | — | `VALIDATION_FAILED` |
| `POST /places/{id}/community-updates` | visit_id | uuid | ✔ | owned, active, in-window | `VISIT_NOT_ACTIVE` |
| | crowd/noise/parking | enum | ✘ | enum | `VALIDATION_FAILED` |
| | wait | enum | ✘ | `wait_bucket` | `VALIDATION_FAILED` |
| | atmosphere | enum[] | ✘ | 1–3 distinct | `VALIDATION_FAILED` |
| | note | string | ✘ | ≤500 | `VALIDATION_FAILED` |
| | media_ids | uuid[] | ✘ | ≤4, owned, `ready` | `MEDIA_NOT_READY`/`MEDIA_LIMIT_EXCEEDED` |
| | (all signals) | — | ≥1 present | crowd∨noise∨wait∨parking∨atmosphere∨note | `NO_SIGNAL_PROVIDED` |
| `POST /visits` | place_id | uuid | ✔ | exists | `NOT_FOUND` |
| | method | enum | ✔ | `auto|manual` | `VALIDATION_FAILED` |
| | location.lat/lng | number | ✔ | ranges; within radius | `TOO_FAR_FROM_PLACE` |
| `POST /media/uploads` | kind | enum | ✔ | `photo|video` | `VALIDATION_FAILED` |
| | content_type | string | ✔ | image/* or video/* allowlist | `VALIDATION_FAILED` |
| | size_bytes | int | ✔ | photo ≤10 MB, video ≤50 MB | `VALIDATION_FAILED` |
| `POST /me/password` | current_password | string | ✔ | matches | `CURRENT_PASSWORD_INCORRECT` |
| | new_password,new_password_confirmation | string | ✔ | as register, not reused | `PASSWORD_MISMATCH`/`PASSWORD_REUSED` |
| `PATCH /me` | email | string | ✔ | valid, unique | `EMAIL_ALREADY_REGISTERED` |
| `POST /me/account-deletion` | password | string | ✔ | matches | `CURRENT_PASSWORD_INCORRECT` |
| `POST /business/auth/register` | email,password,name,business_name | string | ✔ | email distinct realm; password rules | `EMAIL_ALREADY_REGISTERED` |
| `POST /business/claims` | location_id | uuid | ✔ | exists, claimable | `LOCATION_ALREADY_CLAIMED`/`CLAIM_ALREADY_OPEN` |
| `PUT /business/claims/{id}/verification` | fields | object | ✔ | matches requirements | `VERIFICATION_FIELDS_INVALID` |
| | documents | object | per req | required keys present, uploads `ready` | `DOCUMENT_REQUIRED` |
| `POST /business/live-updates` | text | string | ✔ | 1–280 | `VALIDATION_FAILED` |
| | publish_mode | enum | ✔ | `now|schedule` | `VALIDATION_FAILED` |
| | scheduled_for | datetime | if schedule | future | `VALIDATION_FAILED` |
| | expires_at | datetime | ✘ | > publish time | `EXPIRY_BEFORE_PUBLISH` |
| `POST /business/offers` | title,terms | string | ✔ | ≤120 / ≤500 | `VALIDATION_FAILED` |
| | valid_from | datetime | ✔ | — | `VALIDATION_FAILED` |
| | valid_to | datetime | ✘ | > valid_from | `VALIDITY_RANGE_INVALID` |
| | code | string | ✔ | uniq per location | `CODE_ALREADY_USED` |
| | quantity_limit | int | ✘ | ≥1 | `VALIDATION_FAILED` |
| `POST /business/disputes` | subject,details | string | ✔ | ≤120 / ≤2000 | `VALIDATION_FAILED` |
| `POST /business/team/invitations` | email | string | ✔ | not a member | `ALREADY_A_MEMBER` |
| | permissions | enum[] | ✘ | subset of modules | `VALIDATION_FAILED` |
| `POST /admin-api/claims/{id}/reject` | reason | string | ✔ | ≤500 | `VALIDATION_FAILED` |
| `POST /admin-api/reports/{id}/decision` | decision | enum | ✔ | `dismiss|uphold` | `VALIDATION_FAILED` |
| | action | enum | if uphold | `none|remove|restore` | `VALIDATION_FAILED` |
| (all create/publish POSTs) | Idempotency-Key | header | ✔ | UUID/opaque | `IDEMPOTENCY_KEY_REQUIRED` |

## 9. AI chat architecture (Rule 3 — the LLM never invents places)

### 9.1 Pipeline (5 stages; only stage 5 touches language)

```
user text ──► [1] Intent Extractor (LLM, structured JSON, no free prose)
                 └► {intent, category, filters{...}, clarifications_needed[]}
            ──► [2] Clarifier (deterministic rules)       user answers via quick_reply_id
            ──► [3] Retrieval (PostGIS + live data)       eligible places ONLY
            ──► [4] Ranker (deterministic scoring)        top N PlaceCards + why_this_fits
            ──► [5] Explainer (LLM)                       ONE summary sentence from returned facts
                 └► response {message, quick_replies, results{summary_text, places[]}}
```

The DB is the **only** source of places. Stage 5 is given the place list as input and may only
paraphrase it; its output is validated against the returned ids (any unknown id → reject, fall back to
the template sentence). If retrieval is empty, the LLM is not asked to "find something".

### 9.2 Stage contracts

**Stage 1 — Intent Extractor** input: `{latest_user_text, last_3_turns, user_location{city,lat,lng},
now, allowed_enums}`. Output JSON (schema-validated):

```json
{ "intent": "find_place|refine|smalltalk|unsupported",
  "category": "restaurant|park|cafe|bar|gym|other|null",
  "filters": { "noise": "quiet|null", "crowd": "light|moderate|busy|packed|null",
               "wait_max_min": null, "parking": "easy|moderate|null",
               "atmosphere": ["cozy"], "open_now": true, "max_distance_m": null,
               "price": "low|medium|high|null", "party_size": null },
  "free_text_summary": "quiet dinner nearby",
  "clarifications_needed": ["max_distance_m"],
  "confidence": 0.0 }
```
Enums are closed; unknown values → null (never a new enum). No place names are accepted as output.

**Stage 2 — Clarifier** (no LLM). If a required slot is missing (`clarifications_needed` non-empty and
the turn limit allows), return a single question with 1–3 `quick_replies` (e.g. *Walking Distance*).
Ask at most **one** question per turn and at most **2** clarifying turns per session; after that,
proceed with defaults (walking distance 2 km, open now).

**Stage 3 — Retrieval.** SQL/PostGIS only: strict radius from `origin_geo`, `open_now` from
`LocationHours`, category from place taxonomy, plus Right Now signals from the Right Now service (§6.1).
Hard filters applied in SQL; soft preferences handled in ranking. Returns candidate rows (cap 200).

**Stage 4 — Ranker** (deterministic). Score = weighted sum of: preference match, `Distance` decay,
freshness (`updated_at`), reliability, crowd/wait fit. Ties broken by distance. Emits `why_this_fits`
as a **template** filled with real field values (e.g. *"Quiet now, 1.2 km, opens till 10 PM"*). No prose
invention.

**Stage 5 — Explainer.** Input: the ranked list (id, name, signals) + `free_text_summary`. Output: a
single `summary_text`. Validator: every entity mentioned must be in the returned set. On failure →
`"I found a few places that fit what you're looking for."`.

### 9.3 "Not right now" and "Why not here?" (client-side, D24/D25)

- **Not right now** is **client-side in MVP**: the sheet filters the current result list locally and
  reapplies the same ranking penalties (e.g. `too_loud` demotes loud places) using the already-displayed
  place data. No endpoint, no LLM call. A server-side refinement endpoint can be added later if
  cross-session learning is required.
- **Why not here** is rendered **client-side from the `GET /places/{id}` payload** (which already
  carries `crowd`, `wait_minutes`, `parking` and `reliability`). The comparison against the active search
  is done in the app; deterministic, no LLM.

### 9.4 Fallback when there are no results

Return `results = null` (or `places: []`) plus one honest `message` from a fixed set:
"Nothing nearby matches that right now." Offer `quick_replies` to relax the top filter (distance →
category → time). Never fabricate a place or an open status.

### 9.5 Latency, streaming, caching, cost, logging

| Concern | Rule |
| --- | --- |
| Latency | Target p95 < 2.5 s. Stage 5 may stream tokens over SSE (`text/event-stream`) on
`POST …/messages` when `Accept: text/event-stream`; places are sent as a final `results` event. Non-stream clients get one JSON body. |
| Timeouts | LLM 6 s hard cap; retrieval 300 ms; on LLM timeout, return the template summary + places. |
| Caching | Intent result cached per `(normalized_text, location_bucket, daypart)` 15 min; retrieval results cached per `(filters, geohash6)` 60 s; Right Now cached 60 s per location. |
| Cost limits | Per-user 300 chat turns/day; per-session ≤6 LLM calls; only stage 5 receives long context; stage 1 prompt is small and enum-bound. Daily org budget alert. |
| Logging | Log `request_id`, intent JSON, retrieved ids, chosen ids, latency, token spend, model. Never log raw user text with PII beyond session, never log secrets/OTPs. |
| Prompt injection | Stage 1 output is schema-validated and enum-closed; retrieved content (business text, notes, place names) is passed to stage 5 as **data**, never as instructions, wrapped in a delimited block; stage 5 output is validated against returned ids; no tool access, no URLs; any instruction-like content in user text or place data is ignored. |
| Model downtime | Circuit breaker → deterministic template replies + retrieval-only results. |
| Copy rule | No user-facing string may contain "AI", "chatbot", "agent", "model", "prompt", "LLM". The assistant speaks as "Vybity". Enforced by a lint test over response strings. |

### 9.6 No "AI" label — copy rules (CR-1)

- The assistant is addressed as *Vybity*; replies are first-person, plain, and short.
- Error/empty states are human ("Nothing matches that right now"), never technical.
- A CI grep test fails the build if banned tokens appear in any user-facing resource bundle.

---

## 10. End-to-end workflows (one scenario per workflow, with JSON)

Envelope: success `{ "data": …, "meta"? }`; error `{ "error": { code, message, fields?, request_id } }`.

### A1. Create account → identity
`POST /auth/register`
```json
{ "email": "sam@example.com", "password": "hunter2pass", "password_confirmation": "hunter2pass",
  "accepted_terms": true, "device": { "platform": "ios", "push_token": "fcm-abc" } }
```
```json
{ "data": { "user_id": "u_01H...", "email": "sam@example.com", "email_verified": false,
  "otp": { "expires_in_seconds": 300, "resend_available_in_seconds": 30 } } }
```
→ `POST /auth/otp/verify` `{ "email":"sam@example.com","otp":"482913","purpose":"email_verification" }`
→ `200 { "data": { "access_token":"…","refresh_token":"…","expires_in":900,
"user": { "id":"u_01H...","handle":"@city-guide22","account_type":"consumer","email_verified":true,
"onboarding_completed":false }, "is_new_user": true } }`
→ `GET /me/identity` → `{ "data": { "handle":"@city-guide22","avatar":{"id":"av_3","url":"https://…"},
"created_at":"2026-10-14T18:02:11Z" } }` → `PUT /me/account-type { "account_type":"consumer" }`.

### A2. Set location
`PUT /me/location { "source":"gps","lat":39.0997,"lng":-94.5786 }`
→ `200 { "data": { "location": { "city":"Kansas City","label":"Kansas City","lat":39.0997,
"lng":-94.5786,"source":"gps" } } }`.
Standard string/GPS formats are also accepted (D27, §2.13):
`PUT /me/location { "source":"manual", "location":"39°05'59\"N 94°34'43\"W" }` or
`{ "source":"manual", "q":"Kansas City" }` → the same `{ location }` object.

### A3. Sign in + recovery
`POST /auth/login { "email":"sam@example.com","password":"hunter2pass" }`
→ `200 { "data": { "access_token":"…","refresh_token":"…","expires_in":900,
"user": { "id":"u_01H...","handle":"@city-guide22","account_type":"consumer","email_verified":true,
"onboarding_completed":true } } }`. Recovery: `POST /auth/forgot-password` (always 200) →
`POST /auth/otp/verify {purpose:"password_reset"}` → `{ "reset_token":"rt_…",
"reset_token_expires_in_seconds":600 }` → `POST /auth/reset-password` → `{ "password_reset": true }`.

### A4. Discover (chat → results)
`POST /chat/sessions { "text":"somewhere quiet for dinner" }`
→ `201 { "data": { "session_id":"cs_1","messages":[
{ "id":"m1","role":"user","text":"somewhere quiet for dinner","created_at":"…" },
{ "id":"m2","role":"assistant","text":"Got it. How far are you willing to travel?",
  "quick_replies":[{"id":"qr_walk","label":"Walking Distance"}],"created_at":"…" } ],
"results": null } }`
→ `POST /chat/sessions/cs_1/messages { "quick_reply_id":"qr_walk" }`
→ `200 { "data": { "message": { "id":"m3","role":"assistant",
"text":"I found a few places that fit what you're looking for.","quick_replies":[],"created_at":"…" },
"results": { "summary_text":"I found a few places that fit what you're looking for.",
"places":[ { "id":"p_1","name":"The Garden Kitchen","distance_m":1500,"is_open":true,
"reliability":{"level":"high"},"updated_at":"2026-10-14T17:59:00Z",
"right_now":{"crowd":"moderate","noise":"quiet","wait_minutes":10,"parking":"difficult",
"status":"available","updated_at":"2026-10-14T17:59:00Z"},
"tags":[],"live_biz_update":{"text":"Outdoor seating is currently available","posted_at":"…"},
"event":{"title":"Live Music","starts_at":"2026-10-14T20:00:00Z"},
"offer":{"title":"20% off dinner","description":"…"},"why_this_fits":"Quiet now, 1.5 km",
"is_saved":false } ] } } }`
→ **NOT RIGHT NOW / WHY NOT HERE are client-side** (D24/D25): the app filters the local result list for
"not right now" and renders "why not here" from the `GET /places/{id}` payload. No request.

### A5. Evaluate, compare, save
`GET /places/p_1?lat=39.09&lng=-94.57` → Details object (as A4 card + `location{id,name,lat,lng}`,
`opening_hours`, `address`, `right_now`, `reliability{level,community_updates_count,
live_biz_updates_count,most_recent_update_at,explanation}`, `community_updates[3]`, `is_saved`,
`is_following`). This one call also backs the Reliability and Why-not-here sheets (D25).
`GET /places/compare?ids=p_1,p_2&lat=39.09&lng=-94.57`
→ `200 { "data": { "places":[ { "id":"p_1","name":"The Garden Kitchen","environment":"outdoors",
"crowd":"moderate","noise":"quiet","wait_minutes":10,"distance_m":1500,"parking":"limited",
"amenities":["Patio","WiFi"],"updated_at":"…","reliability":{"level":"high"} }, { … p_2 … } ] } }`.
`PUT /me/saved-places/p_1` → `{ "data": { "is_saved": true } }`.

### A6. Navigate and arrive (navigation runs on-device — D9)
`GET /places/p_1/directions?origin_lat=39.09&origin_lng=-94.57&mode=walking`
→ `200 { "data": { "place":{"id":"p_1","name":"The Garden Kitchen","address":{"city":"Kansas City"}},
"right_now": {…}, "reliability":{"level":"high"},
"travel":{"mode":"walking","distance_m":1500,"duration_minutes":18,"arrive_at":"2026-10-14T19:35:00Z"},
"route":{"polyline":"…","bounds":{…}} } }`.
On-device geofence fires → `POST /visits` with `Idempotency-Key: vis-abc`
```json
{ "place_id":"p_1", "method":"auto", "location": { "lat":39.0998, "lng":-94.5787 } }
```
```json
{ "data": { "visit": { "id":"v_1","place": { "id":"p_1","name":"The Garden Kitchen" },
  "arrived_at":"2026-10-14T19:12:00Z","verified":true,
  "contribution_prompt_at":"2026-10-14T19:22:00Z","contribution_due_at":"2026-10-14T22:12:00Z",
  "status":"active" } } }
```
Server enqueues the 19:22 push. `GET /visits/active` → same visit (with `contribution_due:false` until
19:22).

### A7. Contribute (presence-gated) → follow
`POST /media/uploads { "kind":"photo","content_type":"image/jpeg","size_bytes":812345 }`
→ `201 { "data": { "media_id":"md_1","upload_url":"https://s3/…","upload_headers":{…},
"expires_in_seconds":600 } }` → PUT bytes → `POST /media/md_1/complete` → `{ "data": {
"status":"ready","thumbnail_url":"https://…" } }`.
`POST /places/p_1/community-updates` with `Idempotency-Key: con-1`
```json
{ "visit_id":"v_1","crowd":"moderate","noise":"quiet","wait":"10_min","parking":"difficult",
  "atmosphere":["cozy"],"note":"Got seated immediately.","media_ids":["md_1"] }
```
```json
{ "data": { "update": { "id":"c_1","status":"published","created_at":"2026-10-14T19:30:00Z" },
  "reward": { "xp_awarded": 65, "xp_total": 1305, "level": 7, "credibility": { "level":"high" },
  "new_badges": ["photo_contributor"] } } }
```
Errors: missing `visit_id`→`422 VISIT_NOT_ACTIVE`; no signals→`422 NO_SIGNAL_PROVIDED`; 2nd submit on
`v_1`→`409 ALREADY_CONTRIBUTED`; within 6 h same place→`429 UPDATE_COOLDOWN`.
`PUT /places/p_1/follow { "notifications": true }` → `{ "data": { "following": true } }`.

### A8. Profile & account
`GET /me` → `{ "data": { "id":"u_01H...","email":"sam@example.com","email_verified":true,
"handle":"@city-guide22","avatar":{…},"account_type":"consumer","xp":1240,"level":7,
"credibility":{"level":"high"},"next_level_xp":2100,"created_at":"…" } }` (D29 — one call for
Profile + Personal Information).
`PATCH /me { "email":"new@example.com" }` → `202 { "data": { "pending_email":"new@example.com",
"otp": { "expires_in_seconds":300,"resend_available_in_seconds":30 } } }` →
`POST /me/email/verify { "otp":"119204" }` → `{ "data": { "email":"new@example.com" } }`.
`POST /me/account-deletion { "password":"hunter2pass" }` → `202 { "data": {
"scheduled_for":"2026-11-13T18:30:00Z" } }`.

### P1. Portal sign in + recovery
`POST /business/auth/login { "email":"owner@garden.com","password":"…" }`
→ `200 { "data": { "access_token":"…","refresh_token":"…","expires_in":900,
"user": { "id":"bu_1","name":"Sarah Jessie","email":"owner@garden.com","role":"owner",
"permissions":["overview","live_updates","promotions_calendar","offers_coupons","analytics",
"location_profile","reports"] },
"business": { "location_id":"p_1","name":"Garden Kitchen","label":"Business Account",
"claim_status":"approved" } } }`. Recovery mirrors A3 under `/business/auth/*`.

### P2. Claim your business
`GET /business/claims/current` → `{ "data": { "claim": null } }`.
`GET /business/locations/search?q=garden` → `{ "data": { "items":[ { "id":"p_1",
"name":"The Garden Kitchen","address_line":"214 Willow Ave","claimable":true } ] } }`.
`POST /business/claims { "location_id":"p_1" }` → `201 { "data": { "claim": { "id":"cl_1",
"status":"draft","location": { "id":"p_1","name":"The Garden Kitchen" } } } }`.
`GET /business/claims/cl_1/requirements` → `{ "data": { "fields":[ { "key":"reg_no","label":"Registration No.","placeholder":"…","type":"text","required":true,"validation":{"max":40} } ],"documents":[ { "key":"permit","label":"Business Permit","accepted_formats":["pdf","jpg"],"max_size_mb":10,"required":true } ] } }`.
Upload via `/business/uploads` (purpose `claim_document`) →
`PUT /business/claims/cl_1/verification { "fields":{"reg_no":"KC-88213"},"documents":{"permit":"up_9"} }`
→ `POST /business/claims/cl_1/submit` → `202 { "data": { "status":"pending_review",
"submitted_at":"…","message":"Your business claim has been submitted…" } }`.
Admin decides (§8.3) → next `GET /business/claims/current` shows `approved`.

### P3. Daily dashboard
`GET /business/overview` → `{ "data": { "business": { "name":"Garden Kitchen",
"label":"Business Account" }, "right_now": { "crowd":"moderate","noise":"quiet","wait_minutes":10,
"parking":"not_applicable","status":"available" }, "followers": { "total":12450,
"new_this_week":5 }, "recent_engagement":[ { "id":"c_1","author": { "handle":"@city-guide22",
"avatar": {…} },"text":"It's pretty quiet inside right now.","posted_at":"…" } ],
"upcoming_scheduled_posts":[ { "id":"lu_9","title":"Happy Hour Specials Starting Soon!",
"scheduled_for":"2026-10-14T16:00:00Z","status":"scheduled" } ],
"need_attention":[ { "id":"r_1","type":"reported_content","title":"Reported Content",
"message":"A user reported a recent photo…","report_id":"r_1" } ] } }`.

### P4. Post a live update
`POST /business/live-updates` with `Idempotency-Key: lu-1`
```json
{ "text":"Outdoor seating is currently available and heaters are on.","publish_mode":"now",
  "expires_at":"2026-10-15T02:00:00Z","notify_followers":true }
```
```json
{ "data": { "id":"lu_10","text":"Outdoor seating…","category":"environment_status",
"status":"published","published_at":"2026-10-14T19:14:00Z","scheduled_for":null,
"expires_at":"2026-10-15T02:00:00Z","created_by": { "id":"bu_1","name":"Sarah Jessie" } } }
```
Followers receive one push (idempotent). `POST /business/live-updates/lu_10/end` → `{ "status":"ended" }`.

### P5. Promotions calendar
`GET /business/calendar?from=2026-10-01&to=2026-10-31` → `{ "data": { "items":[ { "id":"ci_1",
"type":"event","title":"Live Music Friday","starts_at":"2026-10-17T20:00:00Z","status":"scheduled",
"offer_id":null } ] } }`. `POST /business/calendar/items { "item_type":"event",
"title":"Live Music Friday","publish_mode":"schedule","scheduled_at":"2026-10-17T20:00:00Z",
"notify_followers":true }` → `201 { "data": { "id":"ci_1", … } }`.

### P6. Create an offer
`POST /business/offers { "title":"20% off Dinner","terms":"20% off selected dinner items.",
"publish_mode":"schedule","valid_from":"2026-10-14T17:00:00Z","valid_to":"2026-10-21T02:00:00Z",
"code":"SWEET","quantity_limit":50,"notify_followers":true }` → `201`; `409 CODE_ALREADY_USED` on reuse.
(App-side redemption: `POST /offers/o_1/redeem` → `{ "data": { "redeemed":true,
"quantity_available":49 } }`.)

### P7. Analytics
`GET /business/analytics?range=30d` → `{ "data": { "follower_growth": { "change":589,
"change_percent":15.5,"series":[ { "date":"2026-09-15","value":11861 } ] }, "notifications": {
"delivered":12130,"opened":3047,"open_rate":25.1,"series":[…] }, "post_engagement": {
"avg_views_per_post":679,"total_views":20361,"series":[…] }, "coupons": { "views":5837,
"redemptions":1497,"series":[…] }, "community_updates": { "total":617,"avg_per_day":21,
"series":[…] }, "aggregate": { "followers":12450,"followers_change_percent":15.5,
"notifications_delivered":12130,"notification_open_rate":25.1,"post_views":20361,
"avg_views_per_post":679,"coupon_views":5837,"coupon_redemptions":1497 } } }` (note
`aggregate.followers == overview.followers.total`).

### P8. Location profile
`GET /business/location-profile` → hours, contact, links, description, `gallery[]` with
`status:approved|pending_review`. `PUT /business/location-profile { "hours":[…],"contact":{…},
"links":{…},"description":"…" }` → updated; address change → `422 ADDRESS_CHANGE_REQUIRES_REVIEW`.
`POST /business/location-profile/gallery { "upload_id":"up_3" }` → media `pending_review`.

### P9. Team
`POST /business/team/invitations { "email":"maya@gmail.com", "permissions":["overview",
"community_updates"] }` → member `status:"invited"` + email link.
Invitee: `GET /business/invitations/{token}` → details; `POST /business/invitations/{token}/accept
{ "name":"Maya Chowdhury","password":"…","password_confirmation":"…" }` → member `active`.
`PUT /business/team/bm_2/permissions { "permissions":["overview","live_updates"] }`.

### P10. Reports & disputes
`GET /business/reports?status=pending_review` → `{ "data": { "items":[ { "id":"r_1",
"affected_content": { "type":"live_update","label":"Live Biz Update","excerpt":"Weekend brunch menu…" },
"reason":"Reported as misleading","status":"pending_review","reported_at":"…" } ] } }`.
`POST /business/disputes { "subject":"Dispute on branch menu report",
"related_content":"Live Biz Update from Sept 2","report_id":"r_1","details":"…" }` → `201` dispute
`status:"open"`.

### P11. Portal settings
`PATCH /business/me { "name":"Sarah Jessie","phone":"+14155550123" }` → updated; email change → `202`
+ OTP as A8. `POST /business/me/password` → `204`.

### AD1. Admin decision (claim)
`POST /admin-api/claims/cl_1/approve { "note":"Verified." }` → `200 { "data": { "id":"cl_1",
"status":"approved","decided_at":"…","decided_by":"ad_1" } }`; writes an `AuditLog`; owner notified.
Reject: `POST /admin-api/claims/cl_1/reject { "reason":"Permit unreadable","allow_resubmit":true }`.
Report decision: `POST /admin-api/reports/r_1/decision { "decision":"uphold","action":"remove",
"note":"Misleading" }` → content removed; reporter + business notified.

## 11. Traceability matrix (Rule 6 — UI element ↔ endpoint field)

**A. UI element → endpoint field.** Every displayed value originates from a field. Verified frames.

| UI element (frame) | Endpoint | Field |
| --- | --- | --- |
| Register email/password/retype/terms | `POST /auth/register` | `email,password,password_confirmation,accepted_terms` |
| OTP 6 boxes (counter) | `POST /auth/otp/verify` | `otp`,`resend_available_in_seconds` |
| Identity ready `@city-guide22` + 3D avatar | `GET /me/identity` | `handle,avatar.url` |
| Home header "Kansas City" | `GET /home` | `location.city` |
| "Good Afternoon" | `GET /home` | `time_of_day` |
| TRY ASKING chips | `GET /home` | `prompt_suggestions[].text` |
| YOU'RE VISITING card | `GET /home` / `GET /visits/active` | `active_visit.place.name` |
| Result: name / distance / Reliability / Updated | result payload | `places[].name, distance_m, reliability.level, updated_at` |
| Result: Right Now line | result payload | `places[].right_now.{crowd,noise,wait_minutes}` |
| Result: Needs Update / Limited Info tags | result payload | `places[].tags[]`, `right_now.status` |
| Result: Live Biz Update / Event / Offer | result payload | `live_biz_update.text`, `event.title,starts_at`, `offer.title` |
| Result: Why this fits | result payload | `places[].why_this_fits` |
| Compare rows (Environment/Crowd/Noise/Wait/Distance/Parking/Amenities/Freshness/Reliability) | `GET /places/compare` | `environment,crowd,noise,wait_minutes,distance_m,parking,amenities[],updated_at,reliability.level` |
| Reliability sheet (4 / 0 / 24 min / explanation) | `GET /places/{id}` | `reliability.community_updates_count,live_biz_updates_count,most_recent_update_at,explanation` |
| Why not here (Crowd/Wait/Parking/Reliability) | `GET /places/{id}` (client-side sheet) | `right_now.{crowd,wait_minutes,parking},reliability.level` |
| Details: Opening Hours / Address / Last Updated | `GET /places/{id}` | `opening_hours.display,address.line1,last_updated_at` |
| Details community feed `@city-guide22, 3 min, text` | `GET /places/{id}/community-updates` | `author.handle,posted_at,text` |
| Get Direction "Arrive around 7:35 PM" | `GET /places/{id}/directions` | `travel.arrive_at` |
| Show it after 10 min prompt | `GET /visits/active` + push | `contribution_prompt_at` |
| Community Update chips (Crowd/Noise/Wait/Parking/Atmosphere) | `POST /places/{id}/community-updates` | `crowd,noise,wait,parking,atmosphere[]` |
| Updated "+50 XP" | `POST …/community-updates` | `reward.xp_awarded` |
| Follow prompt | `PUT /places/{id}/follow` | `following` |
| Saved cards | `GET /me/saved-places` | PlaceCard fields |
| Profile XP / LVL 7 / Credibility High | `GET /me` | `xp,level,credibility.level` |
| Overview Right Now / Total Followers / +5 | `GET /business/overview` | `right_now.*, followers.total, followers.new_this_week` |
| Overview Recent Engagement | `GET /business/overview` | `recent_engagement[].author.handle,text,posted_at` |
| Overview Upcoming Scheduled Posts | `GET /business/overview` | `upcoming_scheduled_posts[].title,scheduled_for,status` |
| Overview Need Attention | `GET /business/overview` | `need_attention[].{title,message,report_id}` |
| Live Biz Updates rows (Environment Status, Published/Expired, time) | `GET /business/live-updates` | `category,status,published_at` |
| Calendar month grid | `GET /business/calendar` | `items[].{type,title,starts_at,status}` |
| Offers row (title/terms/validity/code/50 available) | `GET /business/offers` | `title,terms,valid_from,valid_to,code,quantity_available` |
| Analytics 6 blocks + aggregate | `GET /business/analytics` | `follower_growth,notifications,post_engagement,coupons,community_updates,aggregate` |
| Location Profile hours/contact/links/gallery | `GET /business/location-profile` | `hours[],contact,links,description,gallery[]` |
| Team table (name/email/status) | `GET /business/team` | `members[].{name,email,status,role}` |
| Reports table | `GET /business/reports` | `affected_content.label,reason,status,reported_at` |
| Disputes list | `GET /business/disputes` | `subject,related_content,status,submitted_at` |
| Settings fields | `GET /business/me` | `name,restaurant_name,email,phone,role` |

**B. Endpoint field → a UI element or a rule.** Every response field is either shown or feeds a rule:
- `reliability.level` shown (Details/Compare) **and** feeds ranking (§9) and Right Now (§6.1).
- `tags[].needs_update` shown **and** drives `GET /contribute/places` ordering (§6.1 staleness).
- `distance_m` shown **and** used as a hard filter (§9.2 stage 3).
- `contribution_prompt_at` drives the scheduled push (§6.3), not directly displayed.
- `credibility.level` shown on Profile **and** weights Right Now (§6.1).

**C. Orphans (flagged, both directions).**

*UI element with no endpoint (orphan), and disposition:*
| Element | Frame | Disposition |
| --- | --- | --- |
| Profile rows Badges / Rewards / Notifications / Settings | `849:1094` | No frames → appendix §14; rows deferred or hidden (Q4). |
| Home "Search/manual close" chrome | `225:1012` | Client-side only; no field needed. |
| Compare "Cancel" / selection mode | `251:1502` | Client-side only (explicitly excluded). |
| NOT RIGHT NOW sheet | `287:183` | Client-side only (D24); filters the local result list. |
| Onboarding slides + "Skip" | `78:561/859/899` | Frontend-owned (D23); no endpoint. |
| Privacy Policy / Terms body | `666:2455`, `666:2478` | Frontend-bundled text (D23); no endpoint. |
| Reports table column "STAFF MEMBER" | `808:9892` | Copy error, not a field (see §12). |
| `OFFFER` typo on Saved card | `631:97` | Copy error; the field is `offer.title`. |

*Endpoint field with no UI (system-only), and why it stays:*
| Field | Consumer | Reason |
| --- | --- | --- |
| `contribution_due_at` | app (reconcile) | Enforces the contribution window; no visible label. |
| `is_new_user` | app routing | Decides Identity ready vs Home. |
| `meta.next_cursor` | app | Pagination control. |
| `request_id` | all | Support/observability. |
| `verified`, `distance_m` on visit | server | Presence-gate integrity (§6.2). |
| `reliability.explanation` | app | Shown in the Reliability sheet. |

## 12. Contradictions found and resolved

| # | Contradiction | Resolution |
| --- | --- | --- |
| C1 | Overview Total Followers **12,450** vs Analytics aggregate **4,389** (+589) | One SoT: `COUNT(Follow)`. `aggregate.followers == overview.followers.total`. The 4,389 is mock data (§6.5). |
| C2 | `wait_bucket` enum vs Compare showing **25 min** | Submissions use buckets; reads expose numeric `wait_minutes` (§5). |
| C3 | Parking `limited` in Why-not-here/Compare but not a submission chip | `limited` is a computed display label; never submitted (§6.1). |
| C4 | Reliability shows only High/Low; draft enum had `medium` | Keep `medium` + add `insufficient_data`; design-only-High/Low is a data gap (Q3). |
| C5 | Button "Compare 3 places" vs design showing 2 columns | Accept 2–3 (§0 D19). |
| C6 | "Splash 1–3" in draft but only Splash 2/3 exist | No API impact; corrected in §1. |
| C7 | Reports table column "STAFF MEMBER" | Copy error → label should be **CONTENT TYPE** (matches `affected_content.type`) (Q9). |
| C8 | Portal had no Register screen (draft §18) yet needs accounts | Portal self sign-up + invite accept added (D13). |
| C9 | Draft §17 Q1 said "portal missing" | Stale — portal exists (`671:2529`, 27 frames); removed as a question. |
| C10 | Scope says MongoDB; task mandates PostGIS | Decision D1; recorded as flagged deviation. |
| C11 | `live-update status` had no `ended` but we replace DELETE with `/end` | Added `ended` to `live_update_status`/`offer_status`; added `calendar_item_status`. |
| C12 | Draft "one location per account from token" vs possible multi-location | MVP = one active location; `X-Location-Id` reserved (Q9). |
| C13 | Draft had a Content/onboarding API section; owner says splash/onboarding are frontend-owned | Removed `GET /content/onboarding`, `PUT /me/onboarding`, `GET /content/legal/{slug}` and matching admin content endpoints (D23). |
| C14 | Draft had `/places/{id}/reliability` + `/why-not-here` as separate calls; owner: "not that many APIs" | Folded into `GET /places/{id}`; sheets rendered client-side (D25). |
| C15 | Draft keyed everything on one `location`; owner wants `place_id → location_id → {name,lat,lng}` | Split `Place` (POI) and `Location` (branch); Saved keyed on `place_id`, response echoes `location` (D26). |

## 13. Assumptions (not seen in Figma)

| # | Assumption | Where |
| --- | --- | --- |
| AS1 | `medium` reliability level exists | §5, §6.1 |
| AS2 | Atmosphere is multi-select 1–3 | §6.2, D18 |
| AS3 | Compare allows 2–3 places | D19 |
| AS4 | Arrival radius 150 m, accuracy gate 200 m | §6.2 |
| AS5 | Contribution window = arrival + 3 h; prompt at +10 min | §6.2, §6.3 |
| AS6 | Cooldown 6 h/place, 10/day cap | §6.2 |
| AS7 | Photo ≤10 MB; video ≤50 MB/30 s; ≤4 attachments; note ≤500 | §2.9, §6.2 |
| AS8 | XP table, level thresholds, badge criteria | §6.6 |
| AS9 | Deletion grace 30 days; content detached on purge | §2.11 |
| AS10 | OTP 6 digits / 5 min / 5 attempts / resend 30 s / 5 per hour | §2.5 |
| AS11 | One branch per place in MVP; `X-Location-Id` reserved for future branches | §12 C12/C15, D26 |
| AS12 | Portal staff invite via email link + password set | D13, §10 P9 |
| AS13 | Admin panel UI is unspecified (no Figma) | §8.3 |
| AS14 | Coupon redemption + content report have no app screen | §6.7 (PROPOSED) |
| AS15 | Base URL `https://api.vybity.com/v1`; page size 20/max 100 | §2.7 |
| AS16 | Splash, Onboarding and Legal text are bundled in the clients (owner) | D23, §1 |
| AS17 | Not Right Now is client-side; no server refinement in MVP | D24, §9.3 |
| AS18 | Location strings (DMS/geohash/Plus Code/free text) geocoded with a best-match policy | D27, §2.13 |

## 14. Appendix — conditional / post-MVP endpoints (build only if confirmed)

Parked because no screen exists (D7). Thin contracts; not in the MVP path list.

- `GET /me/badges` → `Page` of `{id,name,description,icon_url,earned,earned_at,progress}`.
- `GET /me/rewards` → `Page` of `{id,title,description,xp_required,unlocked}`.
- `GET /me/notifications?cursor=` → `Page` + `meta.unread_count`; `POST /me/notifications/read {ids[]|all}`.
- `GET/PATCH /me/notification-settings` → `{push_enabled,place_updates,community_updates,rewards}`.
- `GET/PATCH /me/settings` → `{distance_unit:km|mi, language}`.

## 15. Changelog (summary — full detail in `CHANGELOG_API.md`)

- **Removed:** `/me/favourites`, `/me/location-permission` (GET+PUT), `/me/settings`,
  `/me/badges`, `/me/rewards`, `/me/notifications*`, `/me/notification-settings` (→ appendix),
  `/me/identity/regenerate`, all `/navigation/sessions` (POST/PATCH/DELETE), `PATCH /visits/{id}/end`.
- **Added:** `POST /auth/social/{provider}` (merge), `POST /business/auth/register`,
  `GET/POST /business/invitations/{token}*`, `POST /business/live-updates/{id}/end`,
  `POST /business/offers/{id}/end`, `POST /business/calendar/items/{id}/cancel`,
  `POST /places/{id}/reports` (proposed), `POST /offers/{id}/redeem` (proposed),
  the entire `/admin-api/*` surface.
- **Changed:** `visit_id` optional→required on contributions; reliability enum +=
  `medium,insufficient_data`; statuses += `ended/cancelled/completed`; follower SoT fixed;
  `wait` (submit) split from `wait_minutes` (read).
- **Owner corrections (this revision, §0.1):** removed `/content/onboarding`, `PUT /me/onboarding`,
  `/content/legal/{slug}` (frontend-owned) and their admin counterparts; removed
  `POST /chat/…/not-right-now` (client-side); removed `/places/{id}/reliability` and
  `/places/{id}/why-not-here` (folded into `GET /places/{id}`); removed `GET /me/profile` (merged into
  `GET /me`); added the `Place`→`Location` model, location string-format inputs, and maps-provider
  routing/ETA.

## 16. Open questions (business unknowns only — everything else is decided)

1. **MongoDB vs PostGIS** — confirm D1 (Postgres/PostGIS) stands; otherwise the geo/index plan changes.
2. **Reliability** — is `medium` real, and is `insufficient_data` the correct thin-data label?
3. **Badges / Rewards / Notifications / Settings** — confirm MVP-excluded (appendix); the Profile rows would then be hidden.
4. **Content report + coupon redemption from the app** — keep the two PROPOSED endpoints or drop them?
5. **Compare limit** — 2 or 3 places (design shows 2)?
6. **Atmosphere** — multi-select 1–3, or single?
7. **Account deletion** — 30-day grace correct, and should authored updates be detached or deleted?
8. **Staff permissions** — can Team/Settings be delegated, or owner-only (assumed owner-only)?
9. **Branches** — confirm one branch per place at launch (the model allows many, D26).
10. **Legal text** — confirmed bundled in the client, or should a CMS endpoint be re-added later?




