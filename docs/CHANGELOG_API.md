# Vybity — API Changelog (draft → canonical)

Every endpoint **removed**, **added**, or **changed** from `Vybity - REST API Specification v1.md` /
`API_SPECIFICATION.md` to `FINAL_API_SPECIFICATION.md`. Reason column states the Rule-1 justification.
`correction.md` was empty (0 bytes) at the time of writing, so no owner corrections are applied.

## A. Removed

| Endpoint (was) | Reason |
| --- | --- |
| `GET /me/favourites` | Identical to `GET /me/saved-places`; no screen distinguishes them. |
| `GET /me/location-permission` | OS permission is device-local; nothing server-side reads it. |
| `PUT /me/location-permission` | Same; the app already knows the OS permission. |
| `GET /me/settings` | No designed screen; distance unit/language are client locale. |
| `PATCH /me/settings` | Same. Deferred to appendix. |
| `GET /me/badges` | Badges frame is hidden/empty; no screen. Deferred to appendix. |
| `GET /me/rewards` | No frame. Deferred to appendix. |
| `GET /me/notifications` | Profile row exists but no frame. Deferred to appendix. |
| `POST /me/notifications/read` | Same. |
| `GET /me/notification-settings` | No frame. Deferred to appendix. |
| `PATCH /me/notification-settings` | Same. |
| `POST /me/identity/regenerate` | The "refresh" glyph is inside the *Continue as Provider* button, not a regenerate control; stable identity is intentional and regeneration is an abuse vector. |
| `POST /navigation/sessions` | Navigation runs on-device; the server only needs one arrival event (`POST /visits`). |
| `PATCH /navigation/sessions/{id}` | Same; live position is device-local. Arrival is reported via `POST /visits`. |
| `DELETE /navigation/sessions/{id}` | No server session exists to end. |
| `PATCH /visits/{id}/end` | No screen; visits expire server-side and on the next arrival. |
| `GET /content/onboarding` *kept* | (Not removed — retained for CMS-managed copy.) |

## B. Added

| Endpoint (new) | Reason |
| --- | --- |
| `POST /auth/social/{provider}` | Consolidates `/auth/social/google` + `/auth/social/apple` (same responsibility). |
| `POST /business/auth/register` | Portal self sign-up so owners no longer need the phone app first (§0 D13). |
| `GET /business/invitations/{token}` | Invitee reviews an invitation before accepting (staff invite flow, D13). |
| `POST /business/invitations/{token}/accept` | Invitee sets a password and activates membership (D13). |
| `POST /business/live-updates/{id}/end` | Replaces `DELETE` with an audited lifecycle transition (D20). |
| `POST /business/offers/{id}/end` | Replaces `DELETE offers/{id}` with an audited lifecycle transition (D20). |
| `POST /business/calendar/items/{id}/cancel` | Replaces `DELETE` with an audited lifecycle transition (D20). |
| `POST /places/{id}/reports` **PROPOSED-NO-FIGMA** | App-side producer for the portal's Reports/Need Attention (D15); no Figma screen. |
| `POST /offers/{id}/redeem` **PROPOSED-NO-FIGMA** | App-side producer for the portal's Coupon Redemptions analytics (D15); no Figma screen. |
| `/admin-api/auth/*` | Separate admin auth realm (login/refresh/logout) (D14, D22). |
| `/admin-api/dashboard` | Pending-work summary for operators. |
| `/admin-api/users`, `/users/{id}`, `PATCH /users/{id}/status` | User management / suspension (Rule 4). |
| `/admin-api/claims`, `POST /claims/{id}/approve`, `POST /claims/{id}/reject` | Claim review queue — the portal is otherwise unusable (Rule 4). |
| `GET/PUT /admin-api/claim-requirements` | Claim form is admin-configurable (Verify Your Business renders dynamically). |
| `/admin-api/reports`, `POST /reports/{id}/decision` | Report moderation (Rule 4). |
| `/admin-api/disputes`, `PATCH /disputes/{id}` | Dispute resolution (Rule 4). |
| `/admin-api/community-updates`, `POST /…/{id}/decision` | Community update moderation (Rule 4). |
| `/admin-api/media`, `POST /media/{id}/decision` | Photo/gallery moderation, approves `pending_review` media (Rule 4). |
| `/admin-api/places`, `POST /…`, `PATCH /…/{id}`, `POST /places/import` | Place import/edit/dedup — the place supply has no other owner (§6.4). |
| `/admin-api/gamification/badges`, `PUT /…/{id}`, `PATCH /settings/xp-rules` | Badge + XP rules are admin-managed, not hard-coded (§6.6). |
| `/admin-api/content/legal*`, `/admin-api/content/onboarding` | Legal + onboarding content management (Rule 4). |
| `/admin-api/audit-logs` | Traceability for every admin mutation (Rule 5). |

## C. Changed

| Endpoint | Change | Reason |
| --- | --- | --- |
| `POST /places/{id}/community-updates` | `visit_id` optional → **required** | Presence gating is a core rule; no anonymous-to-place contributions (§6.2). |
| `POST /places/{id}/community-updates` | Adds `409 ALREADY_CONTRIBUTED`, `422 VISIT_NOT_ACTIVE`, `429 RATE_LIMITED` | Enforce one update per visit, window, and daily cap (§6.2). |
| `POST /visits` | `Idempotency-Key` required; `method` auto/manual both call one arrival service; returns `contribution_due_at` | Prevent duplicate visits; one canonical arrival (§2.6, §6.3). |
| `GET /visits/active` | Adds derived `contribution_due`; is now the reconciliation source for the prompt | Server-scheduled prompt, not a phone timer (D11). |
| `GET /places/{id}/community-updates` | `signals` fixed; media ownership checked | Consistency with Details. |
| `GET /places/compare` | 2–3 ids enforced; duplicates rejected; order preserved | Design vs button conflict resolved (D19). |
| `POST /business/live-updates` | Status set derives `ended`; publishing idempotent | Lifecycle + no duplicate follower pushes. |
| `PATCH /business/offers/{id}` | Editable only while `scheduled` | Ended/active offers must not be silently mutated (D20). |
| Follower fields (`overview.followers`, `analytics.follower_growth`, `aggregate.followers`) | Single source of truth = `COUNT(Follow)` | Fixes 12,450 vs 4,389 contradiction (D16, §6.5). |
| `reliability_level` | `high|medium|low` → `high|medium|low|insufficient_data` | Thin data must be explicit, never fabricated (D17). |
| `live_update_status` / `offer_status` | Add `ended`; `calendar_item_status` added | Lifecycle transitions (D20). |
| `wait` (submit) vs `wait_minutes` (read) | Formally separated | Compare/Details show values that are not buckets (§5). |
| `POST /me/account-deletion` | Response includes content-handling policy | Deletion semantics defined (§2.11, Q8). |
| Business portal account creation | "created in the phone app" → "portal sign-up **or** app" | Removes dependency (D13). |
| `DELETE /business/team/{id}` | Kept, but semantics = disable (audited) | Revoke access without losing audit history. |

## D. Kept (candidates that survived Rule 1)

| Endpoint | Why kept |
| --- | --- |
| `GET/PUT /me/location` | Home header + "nearby" ranking depend on it. |
| `POST /me/devices` | Follower/arrival push is impossible without a token. |
| `GET /content/onboarding` + `PUT /me/onboarding` | Copy is CMS-managed and completion drives routing across devices. |
| `PUT /me/account-type` | Identity ready screen has two buttons; server must route. |
| `POST /auth/logout` (portal) | No control is drawn, but session revocation is required. |
| `GET /business/community-updates` | Overview "View All" needs pagination. |

## E. Deliberately not created

View switches, month/week toggles, modal open/close, unsent review previews, card selection/Cancel,
local-only search filters, and any public route for the internal AI/rules services (§9).

## F. Owner corrections (`correction.md`, applied 2026-10-14)

| Correction | Endpoint change |
| --- | --- |
| Remove §4 Content & onboarding | **Removed** consumer `GET /content/onboarding`, `PUT /me/onboarding`, `GET /content/legal/{slug}`; **removed** admin `GET/POST/PATCH /admin-api/content/legal[/{slug}]` and `GET/PUT /admin-api/content/onboarding`. Splash/Onboarding/Legal are frontend-owned. |
| Location as lat/lng **and** standard string/GPS formats | **Changed** `PUT /me/location` and `GET /locations/search` to also accept `location` (coordinate string / DMS / geohash / Plus Code) and `q` (free text). WGS84 canonical (§2.13, D27). |
| Remove `POST /chat/…/not-right-now` | **Removed.** Not Right Now is client-side (D24); the sheet filters the local result list. |
| "Not that many APIs" in §7 Places | **Removed** `GET /places/{id}/reliability` and `GET /places/{id}/why-not-here`; both folded into `GET /places/{id}` (D25). Places is now `GET /places/{id}` + `GET /places/{id}/community-updates`. |
| Saved by id; `place_id → location_id → {name,lat,lng}` | **Changed** model: `Place` (POI) 1:N `Location` (branch); Saved and place reads key on `place_id` and echo `location{location_id,name,lat,lng}` (D26). |
| "How are route/ETA computed?" | **Clarified** (no endpoint change): routing/ETA come from a maps provider server-side (`GET /places/{id}/directions`); navigation sessions stay removed (D9, D28). |
| "Are all Contribute APIs needed?" | **Confirmed kept**: every Contribute endpoint maps 1:1 to a designed screen (D14 §6.10). |
| Trim Profile & account | **Removed** `GET /me/profile` (merged into `GET /me`); badges/rewards/notifications/settings remain out of MVP (appendix). |
| New admin dashboard to approve each business | **Confirmed**: `/admin-api` is the approval surface; claim approve/reject already present (D14/D22). |

> `OPENAPI.yaml` regenerated to match (now 103 paths). `correction.md` is no longer empty, so the former
> Q1 ("corrections missing") is closed.
