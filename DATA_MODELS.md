# Vybity — Data Models

Database models and relationships derived from the screens and the API spec. Written stack-agnostically
(fields, types, constraints); a relational rendering is shown, with notes for a document store
(MongoDB) where relevant. `id` is a string (UUID/ULID) unless noted. All timestamps are ISO 8601 UTC.

Enums are collected in `API_SPECIFICATION.md` §2.1 and repeated here for the fields that use them.

---

## 1. Entity map

```
User (account) ──1:1── Identity(handle, avatar)
   │  │  │
   │  │  └─1:N─ Device
   │  ├─1:N─ Notification / NotificationSetting
   │  ├─1:N─ Session(refresh tokens)
   │  ├─1:N─ Contribution (community update)
   │  ├─1:N─ Visit ──1:1─ NavigationSession
   │  ├─1:N─ SavedPlace
   │  ├─1:N─ Follow(place)
   │  ├─1:N─ ChatSession ──1:N─ ChatMessage
   │  ├─1:N─ Media
   │  ├─1:N─ Reward / Badge(granted)
   │  └─1:N─ AccountDeletion
   │
Location(place) ──1:N─ LocationHours
   │        ├─1:N─ Contribution
   │        ├─1:N─ LiveBizUpdate
   │        ├─1:N─ Offer ──1:N─ OfferRedemption
   │        ├─1:N─ CalendarItem
   │        ├─1:N─ GalleryMedia
   │        ├─1:N─ Follow
   │        ├─1:N─ Report
   │        └─1:N─ Claim ──1:N─ ClaimDocument
   │
Claim ──N:1── User (claimant)   │   Claim ──1:N── ClaimRequirement(config)
BusinessMember (Team) ──N:1── User, N:1── Location
Dispute ──N:1── BusinessMember/Location, N:1── Report?
```

---

## 2. Identity & auth

### User
| Field | Type | Notes |
| --- | --- | --- |
| `id` | string (PK) | |
| `email` | string, unique, indexed | lower-cased |
| `password_hash` | string, nullable | null for social-only accounts |
| `email_verified` | bool | |
| `account_type` | enum `consumer`/`provider` | default `consumer` |
| `onboarding_completed` | bool | |
| `location_permission` | enum | default `denied` |
| `is_locked` | bool | 423 `ACCOUNT_LOCKED` |
| `failed_login_count` | int | lockout logic |
| `created_at`, `updated_at` | timestamp | |

### Identity (1:1 User)
| Field | Type | Notes |
| --- | --- | --- |
| `user_id` | FK User | |
| `handle` | string, unique | `@city-guide22`; system generated |
| `avatar_id` | string | 3D avatar asset |
| `avatar_url` | string | |
| `created_at` | timestamp | |
| `regenerate_count_today` | int | ≤5/day ⚠ |

### Session / refresh token
| Field | Type | Notes |
| --- | --- | --- |
| `id`, `user_id`, `refresh_token_hash` | | rotates on use |
| `platform` | enum `ios`/`android`/`web` | |
| `expires_at` | timestamp | 30 days |
| `revoked_at` | timestamp, nullable | |
| `created_at` | timestamp | |

### Device
`id`, `user_id`, `platform`, `push_token`, `push_enabled`, `updated_at`.

### OTP
| Field | Type | Notes |
| --- | --- | --- |
| `email`, `code_hash`, `purpose` | enum otp_purpose | |
| `expires_at` | timestamp | 300 s |
| `attempts` | int | ≤5 |
| `resend_available_at` | timestamp | +30 s |
| `created_at` | timestamp | send-limit 5/hour |

### PasswordResetToken
`id`, `user_id`, `token_hash`, `expires_at` (600 s), `used_at`.

---

## 3. Location & place

### Location (a place)
| Field | Type | Notes |
| --- | --- | --- |
| `id` | string (PK) | |
| `name` | string | "The Garden Kitchen" |
| `address_line1` | string | "214 Willow Ave" |
| `city` | string | "Kansas City" / placeholder "Dhaka" |
| `lat`, `lng` | number, indexed (geo) | |
| `timezone` | string | for hours + scheduling |
| `is_open` | bool (derived) | from hours + now (cache) |
| `amenities` | string[] | "Patio", "WiFi", "Bar" (Compare) |
| `environment` | enum `outdoors`/`indoors` | Compare row |
| `description` | string | portal Location Profile |
| `phone`, `email`, `website`, `instagram` | string | contact/links |
| `claim_status` | enum claim_status | mirrors active claim |
| `created_at`, `updated_at` | timestamp | |

### LocationHours
`location_id`, `weekday` (enum), `open_time`, `close_time`, `closed` bool. Unique (`location_id`,`weekday`).

### RightNow (derived/denormalised per location)
`location_id`, `noise` (noise_level|null), `wait_minutes` (int|null), `crowd` (crowd_level|null),
`parking` (parking_level|null), `status` (`available`|`limited_current_information`),
`reliability_level`, `updated_at`. Recomputed from contributions + live updates.

### GeocodeResult (search index, not user data)
`id`, `label`, `address_line`, `city`, `lat`, `lng` — served by `GET /locations/search`.

---

## 4. Chat & discovery

### ChatSession
`id`, `user_id`, `origin_lat`, `origin_lng`, `status` (`active`/`closed`), `created_at`.

### ChatMessage
`id`, `session_id`, `role` (`user`/`assistant`), `text`, `created_at`.
Related: `ChatQuickReply` (`id`,`message_id`,`label`), `ChatResult` (`id`,`message_id`,`place_id`,`why_this_fits`,`rank`), `NotRightNowFeedback` (`id`,`session_id`,`place_id`,`reason`,`other_text`,`created_at`).

### SavedPlace
`user_id`, `place_id`, `created_at`. Unique (`user_id`,`place_id`).
*(`/me/favourites` aliases this table ⚠.)*

### Follow
`user_id`, `place_id`, `notifications` bool, `created_at`. Unique (`user_id`,`place_id`).

---

## 5. Visits & navigation

### NavigationSession
`id`, `user_id`, `place_id`, `mode` (travel_mode), `origin_lat`, `origin_lng`, `route_polyline`,
`eta_minutes`, `remaining_distance_m`, `status` (`active`/`ended`), `created_at`, `ended_at`.

### Visit
| Field | Type | Notes |
| --- | --- | --- |
| `id`, `user_id`, `place_id` | | |
| `method` | enum `auto`/`manual` | |
| `navigation_session_id` | FK nullable | |
| `arrived_at` | timestamp | |
| `verified` | bool | location check |
| `contribution_prompt_at` | timestamp | arrived_at + 10 min |
| `ended_at` | timestamp nullable | |
Unique partial: one active visit per user (`ended_at is null`).

---

## 6. Contributions & media

### Contribution (Community Update)
| Field | Type | Notes |
| --- | --- | --- |
| `id`, `user_id`, `place_id` | | |
| `visit_id` | FK nullable | links to verified visit |
| `crowd` | crowd_level nullable | |
| `noise` | noise_level nullable | |
| `wait` | wait_bucket nullable | submission bucket |
| `wait_minutes` | int nullable | resolved value for display |
| `parking` | parking_level nullable | |
| `atmosphere` | atmosphere[] | 1–3 ⚠ |
| `note` | string(≤500) nullable | |
| `status` | `submitted` | moderation later |
| `created_at` | timestamp | |

### ContributionMedia (join)
`contribution_id`, `media_id`, `order` (≤4 ⚠).

### Media
| Field | Type | Notes |
| --- | --- | --- |
| `id`, `user_id` | | |
| `kind` | `photo`/`video` | |
| `content_type`, `size_bytes` | | |
| `storage_key`, `url`, `thumbnail_url` | | |
| `status` | `processing`/`ready`/`failed` | |
| `purpose` | `community`/`claim_document`/`gallery` | |
| `created_at` | timestamp | |

---

## 7. Rewards

### UserReward (progression)
`user_id`, `xp_total`, `level`, `credibility_level` (reliability_level), `updated_at`.

### Badge
`id`, `name`, `description`, `icon_url`, `criteria` (rule), `xp_required?`.

### UserBadge
`user_id`, `badge_id`, `earned`, `earned_at`, `progress`.

### Reward (catalog)
`id`, `title`, `description`, `xp_required`, `unlocked` (derived per user).

---

## 8. Notifications

### Notification
`id`, `user_id`, `type` (`place_update`/`community_update`/`reward`/`system`), `title`, `body`,
`place_id?`, `read` bool, `created_at`. Index (`user_id`,`created_at`).

### NotificationSetting
`user_id`, `push_enabled`, `place_updates`, `community_updates`, `rewards`.

### UserSetting
`user_id`, `distance_unit` (`km`/`mi`), `language`.

---

## 9. Business portal

### Claim
| Field | Type | Notes |
| --- | --- | --- |
| `id`, `user_id` (claimant), `location_id` | | |
| `status` | enum claim_status | `none`/`draft`/`pending_review`/`approved`/`rejected` ⚠ |
| `submitted_at`, `decided_at`, `decided_by` | timestamp/nullable | admin action (no screen) |
| `message` | string | review banner copy |
| `created_at` | timestamp | |

### ClaimRequirement (admin-configured template)
`key`, `label`, `placeholder`, `type`, `required`, `validation` — for fields and documents.
### ClaimValue
`claim_id`, `key`, `value`.
### ClaimDocument
`claim_id`, `key`, `media_id`, `file_name`, `size_bytes`.

### BusinessMember (Team)
`id`, `location_id`, `user_id`, `name`, `email`, `role` (`owner`/`staff`),
`status` (`active`/`invited`/`disabled` ⚠), `permissions` (permission_module[]), `created_at`.
Unique (`location_id`,`email`).

### LiveBizUpdate
`id`, `location_id`, `text`(≤280 ⚠), `category` (`environment_status` ⚠),
`status` (`scheduled`/`published`/`expired`), `publish_mode`, `scheduled_for`, `published_at`,
`expires_at`, `created_by` (BusinessMember/User), `notify_followers`, `created_at`.

### CalendarItem
`id`, `location_id`, `item_type` (`update`/`special`/`event`/`coupon`/`reminder`), `title`,
`publish_mode`, `scheduled_at`, `status`, `offer_id?`, `created_by`, `created_at`.

### Offer
| Field | Type | Notes |
| --- | --- | --- |
| `id`, `location_id`, `title`, `terms` | | |
| `status` | `scheduled`/`active`/`expired` ⚠ | |
| `publish_mode`, `valid_from`, `valid_to` | | |
| `code` | string | unique per location |
| `quantity_limit` | int nullable | list shows "50 available" |
| `quantity_available` | int (derived) | limit − redemptions |
| `notify_followers`, `created_by`, `created_at` | | |

### OfferRedemption
`id`, `offer_id`, `user_id`, `redeemed_at`, `location_id`. (No app redeem screen yet ⚠.)

### GalleryMedia
`id`, `location_id`, `media_id`, `status` (`approved`/`pending_review`), `order`, `created_at`.

### Report
| Field | Type | Notes |
| --- | --- | --- |
| `id`, `location_id` | | |
| `affected_content_type` | `live_update`/`offer`/`gallery_media`/`contribution` | |
| `affected_content_id` | string | |
| `label` | string | "Live Biz Update" |
| `reason` | string | "Reported as misleading" |
| `status` | `pending_review`/`resolved` | |
| `reported_at` | timestamp | |
| `reporter_user_id?` | FK nullable | no app report screen yet ⚠ |
| `disputable` | bool (derived) | |

### Dispute
`id`, `location_id`, `submitted_by` (BusinessMember), `subject`, `related_content`, `report_id?`,
`details`, `status` (`open`/`in_progress`/`resolved` ⚠), `submitted_at`, `updated_at`.

### AnalyticsSnapshot (aggregation source)
Per location/day: `date`, `followers`, `followers_delta`, `notifications_delivered`,
`notifications_opened`, `post_views`, `coupon_views`, `coupon_redemptions`, `community_updates`.
Series endpoints read from here.

### AccountDeletion
`user_id`, `requested_at`, `scheduled_for` (30 days ⚠), `cancelled_at?`, `reason?`.

### LegalContent
`slug` (`privacy-policy`/`terms`), `title`, `version`, `updated_at`, `body_markdown`.

### OnboardingSlide
`id`, `order`, `title`, `body`, `image_url`.

---

## 10. Key relationships & constraints (summary)

| Relationship | Cardinality | Rule |
| --- | --- | --- |
| User–Identity | 1:1 | created at registration |
| User–Visit | 1:N | only one **active** visit at a time |
| Visit–Contribution | 1:0..1 | a contribution may reference a visit |
| User–SavedPlace | 1:N | unique (user, place) |
| User–Follow | 1:N | unique (user, place) |
| Location–Contribution | 1:N | feeds RightNow |
| Location–LiveBizUpdate/Offer/CalendarItem | 1:N | authored by business |
| Location–Claim | 1:N | one open claim at a time |
| Location–BusinessMember | 1:N | owner + staff |
| Offer–OfferRedemption | 1:N | quantity_available derived |
| Report–Dispute | 1:0..1 | a dispute may target a report |

**Indexes:** geo on `Location(lat,lng)`; `Contribution(place_id,created_at desc)`;
`Notification(user_id,created_at desc)`; unique `LocationClaim(location_id) where status in
(draft,pending_review)`; unique `Offer(location_id,code)`.

---

## 11. Stack note

The written brief proposes **Node/Express + MongoDB**; the analysis task template mentions
**Django/DRF**. The schema above is neutral. If MongoDB: embed `LocationHours`, `RightNow`, `gallery`,
and `notification settings` as sub-documents; keep `Contribution`, `Offer`, `Report` as collections with
`location_id`. If relational: add join tables shown in §6 and §9. **Confirm the target stack** (open
question D22).
