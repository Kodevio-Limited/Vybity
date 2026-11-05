# Vybity — API Specification (validated v1)

Consolidated and screen-validated version of `Vybity - REST API Specification v1.md`. Field-level
validation notes are in `ASSUMPTIONS_AND_QUESTIONS.md`; workflow ordering is in
`SCREEN_TO_API_MAPPING.md`. Items marked ⚠ are assumptions pending confirmation.

---

## 1. Conventions

| Topic | Rule |
| --- | --- |
| Base URL | `https://api.vybity.com/v1` (placeholder) |
| Format | JSON, UTF-8. Media uploads go direct to storage via presigned URL |
| Auth | `Authorization: Bearer <access_token>`. Access 15 min, refresh 30 days, rotates on use |
| Success envelope | `{ "data": ..., "meta"?: ... }` |
| Error envelope | `{ "error": { "code", "message", "fields"? } }` |
| Pagination | `?limit=20&cursor=<opaque>` → `meta.next_cursor` (null on last page) |
| Timestamps | ISO 8601 UTC |
| Distance/duration | meters; minutes unless field ends `_seconds` |
| Idempotency | POST creates accept `Idempotency-Key` |
| OTP | 6 digits, 5 min TTL, 5 attempts, resend +30 s, ≤5 sends/hour |
| Privacy | Public contributions expose only `handle` + `avatar` |

**Status codes:** 200/201/202/204 success · 400/401/403/404/409/410/422/423/429 errors.

---

## 2. Shared types

### 2.1 Enums
| Enum | Values |
| --- | --- |
| `reliability_level` | `high`, `medium`⚠, `low` |
| `crowd_level` | `light`, `moderate`, `busy`, `packed` |
| `noise_level` | `quiet`, `moderate`, `loud` |
| `wait_bucket` | `none`, `5_min`, `10_min`, `20_plus_min` |
| `parking_level` | `easy`, `moderate`, `difficult`, `full`, `not_applicable` (+ response-only `limited`⚠) |
| `atmosphere` | `cozy`, `lively`, `romantic`, `casual`, `trendy`, `loud_music` |
| `environment` | `outdoors`, `indoors` |
| `not_right_now_reason` | `too_crowded`, `too_loud`, `too_expensive`, `too_far`, `not_the_right_atmosphere`, `changed_my_mind`, `other` |
| `travel_mode` | `walking`, `driving`, `transit` |
| `account_type` | `consumer`, `provider` |
| `location_permission` | `granted`, `while_using`, `denied` |
| `otp_purpose` | `email_verification`, `password_reset` |
| `claim_status` | `none`, `draft`, `pending_review`, `approved`, `rejected`⚠ |
| `permission_module` | `overview`, `live_updates`, `promotions_calendar`, `offers_coupons`, `analytics`, `location_profile`, `reports` |
| `publish_mode` | `now`, `schedule` |
| `live_update_status` | `scheduled`, `published`, `expired` |
| `offer_status` | `scheduled`, `active`, `expired`⚠ |
| `calendar_item_type` | `update`, `special`, `event`, `coupon`, `reminder` |
| `report_status` | `pending_review`, `resolved` |
| `dispute_status` | `open`⚠, `in_progress`, `resolved`⚠ |
| `team_status` | `active`, `invited`⚠, `disabled`⚠ |
| `weekday` | `monday` … `sunday` |

`wait_minutes` is always numeric in responses (0 = none), because Compare shows values (25 min) that are
not submission buckets.

### 2.2 Objects
**Author:** `{ handle, avatar:{id,url} }`.
**RightNow:** `{ noise|null, wait_minutes|null, crowd|null, parking|null, status:"available"|"limited_current_information" }` (three null when limited).
**PlaceCard:** `id, name, distance_m, is_open, reliability:{level}, updated_at, right_now|null, tags[], live_biz_update|null, event|null, offer|null, why_this_fits|null, is_saved`.

---

## 3. Authentication (app)

| Method | Path | Auth | Purpose |
| --- | --- | --- | --- |
| POST | `/auth/register` | Public | Register (`email,password,password_confirmation,accepted_terms,device?`) → 201 `{user_id,email,email_verified:false,otp}` |
| POST | `/auth/login` | Public | Sign In → 200 `{access_token,refresh_token,expires_in,user}` |
| POST | `/auth/social/google` · `/auth/social/apple` | Public | `{id_token,authorization_code?,full_name?,device?}` → 200 +`is_new_user` |
| POST | `/auth/forgot-password` | Public | `{email}` → 200 `{message,otp}` (always 200) |
| POST | `/auth/otp/verify` | Public | `{email,otp,purpose}` → tokens (email) or `{reset_token,...}` (reset) |
| POST | `/auth/otp/resend` | Public | `{email,purpose}` → `{otp}` |
| POST | `/auth/reset-password` | Public(token) | `{reset_token,password,password_confirmation}` → `{password_reset:true}` |
| POST | `/auth/refresh` | Public | `{refresh_token}` → new tokens |
| POST | `/auth/logout` | User | `{refresh_token,all_devices?}` → 204 |

**Validation:** password ≥8 with a letter+number ⚠; `password_confirmation == password`;
`accepted_terms == true`.
**Errors:** `409 EMAIL_ALREADY_REGISTERED`, `422 VALIDATION_FAILED/PASSWORD_MISMATCH`,
`401 INVALID_CREDENTIALS`, `403 EMAIL_NOT_VERIFIED` (+`otp`), `423 ACCOUNT_LOCKED` (+`retry_after_seconds`),
`401 SOCIAL_TOKEN_INVALID`, `409 EMAIL_LINKED_TO_PASSWORD_ACCOUNT`⚠, `400 OTP_INVALID`(+`attempts_remaining`),
`410 OTP_EXPIRED`, `429 OTP_ATTEMPTS_EXCEEDED/RESEND_COOLDOWN/SEND_LIMIT`,
`410 RESET_TOKEN_EXPIRED`, `401 REFRESH_TOKEN_INVALID`.

### 3.1 Identity
| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/me/identity` | `{handle,avatar,created_at}` |
| POST | `/me/identity/regenerate` ⚠ | new handle+avatar (≤5/day) |
| PUT | `/me/account-type` | `{account_type}` → `{account_type,next}` |

---

## 4. Content & onboarding
| Method | Path | Auth | Purpose |
| --- | --- | --- | --- |
| GET | `/content/onboarding` | Public | `{slides[]}` |
| PUT | `/me/onboarding` | User | `{status:"completed"|"skipped"}` → `{onboarding_completed:true}` |
| GET | `/content/legal/{slug}` | Public | `{slug,title,version,updated_at,body_markdown}` |

---

## 5. Location
| Method | Path | Purpose |
| --- | --- | --- |
| GET/PUT | `/me/location-permission` | `{status}` |
| GET | `/locations/search?q=&lat=&lng=` | `{items:[{id,label,address_line,city,lat,lng}]}` |
| GET/PUT | `/me/location` | `{source:"gps"|"manual",lat?,lng?,location_id?}` → `{location:{city,label,lat,lng,source}}` |

---

## 6. Home & Chat
| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/home?lat=&lng=` | `{location:{city},time_of_day,prompt_suggestions:[{id,text}],active_visit|null}` |
| POST | `/chat/sessions` | `{text,suggestion_id?,location?}` → 201 `{session_id,messages[]}` |
| POST | `/chat/sessions/{id}/messages` | `{text|quick_reply_id}` → `{message,results|null}` |
| GET | `/chat/sessions/{id}/messages?cursor=` | `{messages[],results}` |
| POST | `/chat/sessions/{id}/places/{pid}/not-right-now` | `{reason,other_text?}` → `{removed_place_id,results}` |

`message` = `{id,role,text,quick_replies:[{id,label}],created_at}`; `results` =
`{summary_text,places:[PlaceCard]}`.

---

## 7. Places
| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/places/{id}?lat=&lng=` | full Details object (name, distance, reliability, opening_hours, address, last_updated_at, right_now, live_biz_update, event, offer, community_updates[first 3], is_saved, is_following) |
| GET | `/places/{id}/community-updates?cursor=&limit=` | `{items:[{id,author,text,posted_at,media[],signals?}],meta}` |
| GET | `/places/{id}/reliability` | `{place_name,level,community_updates_count,live_biz_updates_count,most_recent_update_at,explanation}` |
| GET | `/places/{id}/why-not-here?session_id=` | `{crowd,wait_minutes,parking,reliability:{level}}` |

---

## 8. Compare
`GET /places/compare?ids=a,b[,c]&lat=&lng=` → `{places:[{id,name,environment,crowd,noise,wait_minutes,distance_m,parking,amenities[],updated_at,reliability}]}`. 2–3 ids ⚠.

---

## 9. Directions & navigation
| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/places/{id}/directions?origin_lat=&origin_lng=&mode=` | `{place,right_now,reliability,travel:{mode,distance_m,duration_minutes,arrive_at},route:{polyline,bounds}}` |
| POST | `/navigation/sessions` | `{place_id,origin,mode}` → 201 `{session_id,route,eta_minutes,remaining_distance_m,arrive_at,next_step,place,right_now,reliability}` |
| PATCH | `/navigation/sessions/{id}` | `{location,heading?}` → `{eta_minutes,remaining_distance_m,next_step,arrived,visit?}` |
| DELETE | `/navigation/sessions/{id}` | 204 |

---

## 10. Visits
| Method | Path | Purpose |
| --- | --- | --- |
| POST | `/visits` | `{place_id,method,location,navigation_session_id?}` → 201 `{visit}` |
| GET | `/visits/active` | `{visit|null}` |
| PATCH | `/visits/{id}/end` ⚠ | → `{visit:{...,ended_at}}` |

`visit` = `{id,place:{id,name},arrived_at,verified,contribution_prompt_at}`. Errors `422 TOO_FAR_FROM_PLACE`, `409 VISIT_ALREADY_ACTIVE`.

---

## 11. Contribute
| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/contribute/places?lat=&lng=&limit=` | `{items:[PlaceCard]}` |
| POST | `/media/uploads` | `{kind,content_type,size_bytes}` → `{media_id,upload_url,upload_headers,expires_in_seconds}` |
| POST | `/media/{id}/complete` | → `{status,thumbnail_url}` |
| POST | `/places/{id}/community-updates` | `{visit_id?,crowd?,noise?,wait?,parking?,atmosphere[]?,note?,media_ids[]?}` → 201 `{update,reward}` |
| PUT/DELETE | `/places/{id}/follow` | `{notifications}` → `{following:true}` / 204 |

At least one signal required. Errors `422 NO_SIGNAL_PROVIDED/MEDIA_NOT_READY`, `429 UPDATE_COOLDOWN`.
`reward` = `{xp_awarded,xp_total,level,credibility:{level},new_badges[]}`.

---

## 12. Saved
| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/me/saved-places?lat=&lng=&cursor=&limit=` | `{items:[PlaceCard],meta}` |
| PUT/DELETE | `/me/saved-places/{id}` | `{is_saved:true}` / 204 |

---

## 13. Profile & account
| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/me/profile` | `{handle,avatar,xp,level,credibility,next_level_xp?}` |
| GET | `/me/badges` ⚠ | `{items:[{id,name,description,icon_url,earned,earned_at,progress}]}` |
| GET | `/me/rewards` ⚠ | `{items:[{id,title,description,xp_required,unlocked}]}` |
| GET | `/me/favourites` ⚠ | alias of saved |
| GET | `/me/notifications?cursor=` | `{items[],meta.unread_count}` |
| POST | `/me/notifications/read` | `{ids[]|all}` |
| GET/PATCH | `/me/notification-settings` ⚠ | `{push_enabled,place_updates,community_updates,rewards}` |
| POST | `/me/devices` | `{platform,push_token}` |
| GET/PATCH | `/me` | profile; email change → 202 `{pending_email,otp}` |
| POST | `/me/email/verify` | `{otp}` → 200 `{email}` |
| POST | `/me/password` | `{current_password,new_password,new_password_confirmation}` → 204 |
| GET/PATCH | `/me/settings` ⚠ | `{distance_unit,language}` |
| POST/DELETE | `/me/account-deletion` | `{password,reason?}` → 202 `{scheduled_for}` / cancel |

Errors: `400 CURRENT_PASSWORD_INCORRECT/OTP_INVALID`, `409 EMAIL_ALREADY_REGISTERED`,
`422 PASSWORD_MISMATCH/PASSWORD_REUSED`⚠.

---

## 14. Error code table (app)

| HTTP | Codes |
| --- | --- |
| 400 | `OTP_INVALID`, `CURRENT_PASSWORD_INCORRECT` |
| 401 | `INVALID_CREDENTIALS`, `TOKEN_EXPIRED`, `REFRESH_TOKEN_INVALID`, `SOCIAL_TOKEN_INVALID` |
| 403 | `EMAIL_NOT_VERIFIED` |
| 404 | `NOT_FOUND` |
| 409 | `EMAIL_ALREADY_REGISTERED`, `VISIT_ALREADY_ACTIVE` |
| 410 | `OTP_EXPIRED`, `RESET_TOKEN_EXPIRED` |
| 422 | `VALIDATION_FAILED`, `PASSWORD_MISMATCH`, `PASSWORD_REUSED`, `NO_SIGNAL_PROVIDED`, `TOO_FAR_FROM_PLACE`, `MEDIA_NOT_READY` |
| 423 | `ACCOUNT_LOCKED` |
| 429 | `OTP_RESEND_COOLDOWN`, `OTP_SEND_LIMIT`, `OTP_ATTEMPTS_EXCEEDED`, `UPDATE_COOLDOWN`, `RATE_LIMITED` |

---

# Part 2 — Business Portal API

All under `/business`. Conventions from §1; portal-specific rules below.

| Topic | Rule |
| --- | --- |
| Accounts | No register; portal user is a `provider` account created in the app ⚠ |
| Roles | `owner` (claimant), `staff` (invited) |
| Active location | one per account from the token; `X-Location-Id` reserved for multi-location ⚠ |
| Claim gating | until `approved`, only auth + claim + `/business/me`; else `403 CLAIM_NOT_APPROVED` |
| Module permission | staff only assigned modules; else `403 FORBIDDEN_MODULE`; Team/Settings owner-only ⚠ |
| Dates | ISO 8601 with offset, in location timezone |
| Notify followers | publishing sends push; optional `notify_followers` (default true) ⚠ |

## 15. Portal auth
`POST /business/auth/login` → 200 `{access_token,refresh_token,expires_in,user:{id,name,email,role,permissions[]},business:{location_id,name,label,claim_status}}`.
Errors `401 INVALID_CREDENTIALS`, `403 ACCOUNT_NOT_PROVIDER`, `423 ACCOUNT_LOCKED`.
Recovery mirrors app §3: `/business/auth/forgot-password`, `/otp/verify`, `/otp/resend`,
`/reset-password`, `/refresh`, `/logout`.

## 16. Claim
| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/business/claims/current` | `{claim|null}` |
| GET | `/business/locations/search?q=` | `{items:[{id,name,address_line,claimable}]}` |
| POST | `/business/claims` | `{location_id}` → 201 `{claim:status:"draft"}` |
| GET | `/business/claims/{id}/requirements` | `{fields[],documents[]}` |
| POST | `/business/uploads` + `/{id}/complete` | `purpose:claim_document|gallery` |
| PUT | `/business/claims/{id}/verification` | `{fields,documents}` |
| GET | `/business/claims/{id}` | claim summary |
| POST | `/business/claims/{id}/submit` | 202 `{status:"pending_review"}` |

Errors `409 LOCATION_ALREADY_CLAIMED/CLAIM_ALREADY_OPEN`, `422 VERIFICATION_FIELDS_INVALID/DOCUMENT_REQUIRED/CLAIM_INCOMPLETE`.

## 17. Dashboard modules
| Method | Path | Permission | Purpose |
| --- | --- | --- | --- |
| GET | `/business/overview` | overview | right_now, followers, recent_engagement, upcoming_scheduled_posts, need_attention |
| GET | `/business/community-updates?cursor=` | overview | paged engagement |
| GET | `/business/live-updates?status=&cursor=` | live_updates | list |
| POST | `/business/live-updates` | live_updates | `{text,publish_mode,scheduled_for?,expires_at?,notify_followers?}` |
| DELETE | `/business/live-updates/{id}` ⚠ | live_updates | 204 |
| GET | `/business/calendar?from=&to=` | promotions_calendar | `{items[]}` |
| POST | `/business/calendar/items` | promotions_calendar | `{item_type,title,publish_mode,scheduled_at?,notify_followers?}` |
| DELETE | `/business/calendar/items/{id}` ⚠ | promotions_calendar | 204 |
| GET | `/business/offers?status=&cursor=` | offers_coupons | list |
| POST | `/business/offers` | offers_coupons | `{title,terms,publish_mode,valid_from,valid_to?,code,quantity_limit?,notify_followers?}` |
| PATCH/DELETE | `/business/offers/{id}` ⚠ | offers_coupons | edit/end |
| GET | `/business/analytics?range=30d` | analytics | six blocks + aggregate |
| GET/PUT | `/business/location-profile` | location_profile | profile |
| POST/DELETE | `/business/location-profile/gallery` | location_profile | media |
| GET | `/business/team` | owner | members |
| POST | `/business/team/invitations` | owner | `{email,name?,permissions[]?}` |
| PUT | `/business/team/{id}/permissions` | owner | `{permissions[]}` |
| DELETE | `/business/team/{id}` ⚠ | owner | 204 |
| GET | `/business/reports?status=&cursor=` | reports | list |
| GET | `/business/reports/{id}` | reports | detail |
| GET | `/business/disputes?cursor=` | reports | list |
| POST | `/business/disputes` | reports | `{subject,related_content?,report_id?,details}` |

## 18. Portal settings
| Method | Path | Purpose |
| --- | --- | --- |
| GET/PATCH | `/business/me` | `{name,restaurant_name,phone,email}` / `role` |
| POST | `/business/me/password` | 204 |

## 19. Portal errors
| HTTP | Codes |
| --- | --- |
| 403 | `CLAIM_NOT_APPROVED`, `FORBIDDEN_MODULE`, `ACCOUNT_NOT_PROVIDER` |
| 409 | `LOCATION_ALREADY_CLAIMED`, `CLAIM_ALREADY_OPEN`, `CODE_ALREADY_USED`, `ALREADY_A_MEMBER` |
| 422 | `VERIFICATION_FIELDS_INVALID`, `DOCUMENT_REQUIRED`, `CLAIM_INCOMPLETE`, `EXPIRY_BEFORE_PUBLISH`, `VALIDITY_RANGE_INVALID`, `ADDRESS_CHANGE_REQUIRES_REVIEW` |

---

## 20. Endpoint index (counts)

- **App:** 9 auth + 3 identity + 3 content + 3 location + 5 chat/home + 4 places + 1 compare +
  4 navigation + 3 visits + 5 contribute/media + 2 saved + ~14 profile/account = **56 endpoints**.
- **Portal:** 7 auth + 8 claim + 21 modules + 2 settings = **38 endpoints**.
- **Total: ~94 endpoints** (including ⚠ placeholders). Missing by design: app-side **report** and
  **redeem coupon**, and the whole **admin** surface (see `ASSUMPTIONS_AND_QUESTIONS.md` D15/D16).
