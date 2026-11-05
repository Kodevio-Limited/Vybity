# Vybity - REST API Specification v1

Derived screen by screen from the Figma file Vybity App (page App Ui, 51 frames). Every request field maps to an input in the UI, and every response field maps to something the UI displays. Items marked ⚠ are assumptions where the design is ambiguous; they are collected in Section 17 for confirmation.

## 0. Project overview (read this first)

Vybity tells a person what a physical place is like right now, before they go, using conditions reported by people who were actually there. It launches in Kansas City. The user never talks to other users: they talk to the app, and the app answers from live environment data. its basically an ai applicatio. user will basicaaly talk to ai ai will suggest user phisical placess based on user conversation. if user say for find park ai will find park. if user search for resturatnt(through ai) 

### 0.1 Core principles

- **Know Before You Go:** the product answers "what is it like there right now?".
- **Environment first:** data describes the place (crowd, noise, wait, parking, atmosphere), never other people.
- **Anonymous by design:** system-generated username and avatar, no personal photos, no social feed, no followers-of-people.
- **Invisible technology:** the chat is never labelled "AI" or "chatbot" in the UI.
- **User decides:** the app suggests and explains; the user chooses where to go.
- **Presence-gated contribution:** a user can only contribute an update after physically visiting the place.

### 0.2 Three surfaces and the proposed stack

| Surface | Users | Designed in Figma? | Proposed tech |
| --- | --- | --- | --- |
| Consumer mobile app (page "App Ui") | Visitors | Yes, about 50 screens | Flutter |
| Business portal (page "Business Portal") | Business owners and staff | Yes, about 28 screens | Next.js + tailwine css |
| Admin / moderation platform | Vybity staff | No screens. Described in the scope document only | Next.js + tailwine css |
| Backend and database | All of the above | n/a | Django |

Sections 1 to 17 specify the mobile app API, Sections 18 to 30 the business portal API, and Sections 31 to 34 the workflows and how both connect. The admin API is not specified yet (see 0.5).

### 0.3 The one real workflow

Everything in the product is one loop. The mobile app and the business portal are the two sides of it.

1. **Ask:** the user opens Home and chats with the app ("somewhere quiet for dinner"). The app may ask one follow-up, such as distance.
2. **Get places:** the app returns place cards (restaurants, parks, anywhere) each with a Right Now line such as "Quiet", "10 min wait", "Moderate crowd".
3. **Open a place:** Details shows Right Now, Live Biz Update, Events, Offers, and Community Updates (comments, some with photos).
4. **Go:** the user starts navigation and the app detects arrival.
5. **Contribute:** after arriving, the user is asked what it is like (crowd, noise, wait, parking, atmosphere, a note, an optional photo) and submits a Community Update. They earn XP and are offered to follow the place.
6. **Loop closes:** that update feeds the Right Now verdicts and the Community Updates list that the next user sees, and feeds the business's Overview and Analytics in the portal.

On the business side: the owner signs in, verifies, claims their specific location, and then manages Live Biz Updates, the Promotions Calendar, Offers, the Location Profile, Team, and Reports. Analytics reads back what the app is doing.

### 0.4 Who controls what on a place's Details screen

| Content | Written by | Source in this spec |
| --- | --- | --- |
| Right Now verdicts (noise, wait, crowd, parking) | Computed from community contributions | Sections 7, 11 |
| Community Updates (notes and photos) | Visitors who were physically present | Section 11 |
| Live Biz Update | Business, through the portal | Sections 22, 33 |
| Events | Business, through the Promotions Calendar | Section 23 |
| Offers and coupons | Business, through the portal | Section 24 |
| Location Profile (hours, contact, photos) | Business, through the portal | Section 26 |
| Analytics | Derived from app activity, read-only | Section 25 |

### 0.5 Assumptions and gaps for the reviewer

- **Admin vs portal:** the product owner described Live Biz Updates, Events, and Offers as coming from the "admin panel". The Figma designs create them from the business portal under the owner's own login. This spec follows the Figma. If a separate internal admin posts on a business's behalf, that endpoint set is missing and must be added.
- **Admin API not specified:** claim approval, report moderation, and dispute resolution have no Figma screens. They are needed for the portal to work end to end.
- **Not in the designs:** the app screens for reporting content and redeeming a coupon do not exist, though the portal shows reports and coupon redemptions.
- **Marks:** items marked ⚠ are assumptions made by the author, not read from a design. Open questions are collected in Section 17 (app) and Section 34 (portal).
- **Placeholders:** the base URL, default page sizes, and limits are placeholders to confirm with the backend team.

#comment form samiul:: create apis & related screen and stuff for admin cz there need to have an separte admin dashbord. 

## 1. Conventions

| Topic | Rule |
| --- | --- |
| Base URL | `https://api.vybity.com/v1` (placeholder) |
| Format | JSON, UTF-8. Photos and videos upload directly to storage through a presigned URL (Section 11.2) |
| Auth | `Authorization: Bearer <access_token>`. Access token lives 15 min, refresh token 30 days and rotates on use. Endpoints marked Public need no token |
| Success envelope | `{ data, meta? }` |
| Error envelope | `{ error: { code, message, fields? } }` where `fields` maps field name to message |
| Pagination | `?limit=20&cursor=<opaque>`, response `meta.next_cursor` (null on last page) |
| Timestamps | ISO 8601 UTC. The client renders `3 mins ago`, `Updated 3 mins ago`, `Arrive around 7:35 PM` |
| Distance / duration | Distance in meters (client renders `1.5 km`). Durations in minutes unless the field ends in `_seconds` |
| Idempotency | POST endpoints that create records accept an `Idempotency-Key` header |
| OTP limits | 6 digits, valid 5 min, 5 verify attempts per code, resend cooldown 30 s, max 5 sends per hour |
| Privacy | Public contributions only ever expose `handle` and `avatar`, never real name or email (Onboarding 6: Explore freely. Stay anonymous.) |

## 2. Shared types

### 2.1 Enums

| Enum | Values | UI source |
| --- | --- | --- |
| `reliability_level` | `high`, `medium`, `low` | Reliability: High / Low (medium ⚠) |
| `crowd_level` | `light`, `moderate`, `busy`, `packed` | Community Update > Crowd |
| `noise_level` | `quiet`, `moderate`, `loud` | Community Update > Noise |
| `wait_bucket` | `none`, `5_min`, `10_min`, `20_plus_min` | Community Update > Wait (submissions only) |
| `parking_level` | `easy`, `moderate`, `difficult`, `full`, `not_applicable`. Responses may also return `limited` | Community Update > Parking. Why not here and Compare show Limited ⚠ |
| `atmosphere` | `cozy`, `lively`, `romantic`, `casual`, `trendy`, `loud_music` | Community Update > Atmosphere |
| `environment` | `outdoors`, `indoors` | Compare > Environment |
| `not_right_now_reason` | `too_crowded`, `too_loud`, `too_expensive`, `too_far`, `not_the_right_atmosphere`, `changed_my_mind`, `other` | Not Right Now sheet |
| `travel_mode` | `walking`, `driving`, `transit` | Chat chip Walking Distance (others ⚠) |
| `account_type` | `consumer`, `provider` | Identity ready buttons |
| `location_permission` | `granted`, `while_using`, `denied` | Allow Location ⚠ |
| `otp_purpose` | `email_verification`, `password_reset` | OTP screen |

In responses, wait is always numeric `wait_minutes` (0 means none) because Compare shows values like 25 min that are not a submission bucket.

### 2.2 Objects

**Author** (public identity): `handle` (e.g. `@city-guide22`), `avatar.id`, `avatar.url`.

**RightNow**: `noise` (noise\_level), `wait_minutes` (int), `crowd` (crowd\_level), `status` (`available` or `limited_current_information`). When status is `limited_current_information` the other three are null and the card shows Limited Current Information.

**PlaceCard** (used by Results, Saved, Contribute, Home):

| Field | Type | UI element |
| --- | --- | --- |
| `id` | string |  |
| `name` | string | The Garden Kitchen |
| `distance_m` | int | 1.5 km |
| `is_open` | bool | Open chip |
| `reliability.level` | reliability\_level | Reliability: High |
| `updated_at` | timestamp | Updated 3 mins ago |
| `right_now` | RightNow or null | Right Now: Quiet, 10 min wait, Moderate Crowd |
| `tags` | string\[\] | `needs_update` shows the Needs Update tag |
| `live_biz_update` | `{ text, posted_at }` or null | Live Biz Update |
| `event` | `{ title, starts_at }` or null | Event: Live Music, Today at 8 PM |
| `offer` | `{ title, description }` or null | Offer ⚠ (Saved card label reads OFFFER) |
| `why_this_fits` | string or null | Why this fits (personalised to the search) |
| `is_saved` | bool | Saved state |

## 3. Authentication

### 3.1 `POST /auth/register` - Public - Register screen

| Field | Type | Req | Rules |
| --- | --- | --- | --- |
| `email` | string | yes | valid email |
| `password` | string | yes | min 8, at least one letter and one number ⚠ |
| `password_confirmation` | string | yes | must equal password (Retype Password) |
| `accepted_terms` | bool | yes | must be true (Check box ⚠) |
| `device.platform`, `device.push_token` | string | no | ios or android, push token |

**201 response:** `user_id`, `email`, `email_verified` (false), `otp.expires_in_seconds` (300), `otp.resend_available_in_seconds` (30). Client goes to OTP screen.

**Errors:** 409 `EMAIL_ALREADY_REGISTERED`, 422 `VALIDATION_FAILED` (with `fields`), 422 `PASSWORD_MISMATCH`.

### 3.2 `POST /auth/login` - Public - Sign In screen

Request: `email`, `password`, `device` (optional, same as 3.1).

**200 response:** `access_token`, `refresh_token`, `expires_in` (seconds), `user` = { `id`, `handle`, `avatar`, `account_type`, `email_verified`, `onboarding_completed`, `location_permission` }. The user flags let the client route to Onboarding, Allow Location, or Home.

**Errors:** 401 `INVALID_CREDENTIALS`, 403 `EMAIL_NOT_VERIFIED` (body includes an `otp` object and the client routes to OTP), 423 `ACCOUNT_LOCKED` (includes `retry_after_seconds`).

### 3.3 `POST /auth/social/google` and `POST /auth/social/apple` - Public - Google / Apple buttons (Sign In and Register)

Request: `id_token` (required). Apple only: `authorization_code` (required), `full_name` (optional, Apple sends it once and it is never shown publicly). `device` optional.

**200 response:** same as 3.2 plus `is_new_user` (bool). When true the client routes to Identity ready (3.10).

**Errors:** 401 `SOCIAL_TOKEN_INVALID`, 409 `EMAIL_LINKED_TO_PASSWORD_ACCOUNT` ⚠.

### 3.4 `POST /auth/forgot-password` - Public - Forgot Pass screen

Request: `email` (required).

**200 response:** `message` (generic text), `otp.expires_in_seconds`, `otp.resend_available_in_seconds`. Always returns 200, even for unknown emails, to prevent account enumeration. Client goes to OTP with purpose `password_reset`.

### 3.5 `POST /auth/otp/verify` - Public - OTP screen (Verify OTP, 6 boxes)

| Field | Type | Req | Rules |
| --- | --- | --- | --- |
| `email` | string | yes |  |
| `otp` | string | yes | exactly 6 digits |
| `purpose` | otp\_purpose | yes |  |

**200 response, purpose `email_verification`:** `access_token`, `refresh_token`, `expires_in`, `user`, `is_new_user` (true). Client goes to Identity ready.

**200 response, purpose `password_reset`:** `reset_token` (string), `reset_token_expires_in_seconds` (600). Client goes to Create New Password.

**Errors:** 400 `OTP_INVALID` (includes `attempts_remaining`), 410 `OTP_EXPIRED`, 429 `OTP_ATTEMPTS_EXCEEDED`.

### 3.6 `POST /auth/otp/resend` - Public - Send again link

Request: `email`, `purpose`. **200 response:** `otp.expires_in_seconds`, `otp.resend_available_in_seconds`. **Errors:** 429 `OTP_RESEND_COOLDOWN` (includes `retry_after_seconds`), 429 `OTP_SEND_LIMIT`.

### 3.7 `POST /auth/reset-password` - Public (reset token) - Create New Password screen

Request: `reset_token`, `password`, `password_confirmation` (all required, same rules as 3.1). **200 response:** `password_reset` (true). All existing sessions are revoked. Client returns to Sign In ⚠ (no auto-login).

**Errors:** 410 `RESET_TOKEN_EXPIRED`, 422 `PASSWORD_MISMATCH`, 422 `VALIDATION_FAILED`.

### 3.8 `POST /auth/refresh` - Public

Request: `refresh_token`. **200 response:** new `access_token`, `refresh_token`, `expires_in`. **Errors:** 401 `REFRESH_TOKEN_INVALID`.

### 3.9 `POST /auth/logout` - User - Profile > Log Out

Request: `refresh_token`, `all_devices` (bool, default false). **204** no body.

### 3.10 Identity ready screen

**`GET /me/identity`** - User. Response: `handle` (@city-guide22), `avatar.id`, `avatar.url` (3D avatar), `created_at`. The system generates this identity at account creation.

**`POST /me/identity/regenerate`** - User ⚠ (the refresh icon on the screen). No body. Response: new `handle` and `avatar`. Limit 5 per day.

**`PUT /me/account-type`** - User. Request: `account_type`. Continue sends `consumer`, Continue as Provider sends `provider`. Response: `account_type`, `next` (`app` or `business_onboarding` ⚠).

## 4. Splash and Onboarding

Splash 1-3 are static with no API calls.

**`GET /content/onboarding`** - Public. Response: `slides[]` each with `id`, `order`, `title`, `body`, `image_url`. Current content: Know Before You Go; Information from the people and places that know; Explore freely. Stay anonymous. The client may bundle these and treat this endpoint as an override.

**`PUT /me/onboarding`** - User. Request: `status` (`completed` or `skipped`, from the Skip button). Response: `onboarding_completed` (true).

## 5. Location

**`PUT /me/location-permission`** - User - Allow Location screen and Profile > Location Permission. Request: `status` (location\_permission). Response: `status`. **`GET /me/location-permission`** returns the same object.

**`GET /locations/search?q=<text>&lat=&lng=`** - User - Add address manually (Address Details field). Response: `items[]` with `id`, `label`, `address_line`, `city`, `lat`, `lng`.

**`PUT /me/location`** - User - sets the current location shown in the header (Kansas City).

| Field | Type | Req | Notes |
| --- | --- | --- | --- |
| `source` | `gps` or `manual` | yes |  |
| `lat`, `lng` | number | if gps |  |
| `location_id` | string | if manual | id from `/locations/search` |

Response: `location` = { `city`, `label`, `lat`, `lng`, `source` }. **`GET /me/location`** returns the same.

## 6. Home and Chat

### 6.1 `GET /home?lat=&lng=` - User - Home screen

Response:

| Field | UI element |
| --- | --- |
| `location.city` | Kansas City header |
| `time_of_day` (`morning`, `afternoon`, `evening`, `night`) | Good Afternoon |
| `prompt_suggestions[]` = { `id`, `text` } | TRY ASKING chips (Somewhere quiet for dinner, What is good right now, Something fun tonight, Somewhere relaxed nearby) |
| `active_visit` = null or { `visit_id`, `place` = { `id`, `name`, `city` } } | YOU ARE VISITING card with View Details and Update this Place (Home after visit) |

### 6.2 `POST /chat/sessions` - User - first message from the input Tell me what you are looking for

| Field | Type | Req | Notes |
| --- | --- | --- | --- |
| `text` | string | yes | max 500 ⚠ |
| `suggestion_id` | string | no | set if a TRY ASKING chip was tapped |
| `location` | `{ lat, lng }` | no | defaults to saved current location |

**201 response:** `session_id`, `messages[]` (the user message and the assistant reply, shape in 6.3).

### 6.3 `POST /chat/sessions/{session_id}/messages` - User - Chat screen

Request (one of the two is required): `text`, or `quick_reply_id` (a chip such as Walking Distance).

**200 response:**

| Field | Notes |
| --- | --- |
| `message.id`, `message.role` (`assistant`), `message.text` | e.g. Got it. How far are you willing to travel? |
| `message.quick_replies[]` = { `id`, `label` } | e.g. Walking Distance. Empty when results are returned |
| `message.created_at` |  |
| `results` | null while the assistant is still clarifying. Otherwise { `summary_text`, `places[]` (PlaceCard) } with summary\_text = I found a few places that fit what you are looking for. |

The user message echo has the same shape with `role` = `user`.

**`GET /chat/sessions/{session_id}/messages?cursor=`** - User. Response: `messages[]` oldest first, plus `results` of the latest turn.

### 6.4 `POST /chat/sessions/{session_id}/places/{place_id}/not-right-now` - User - Not Right Now sheet

Request: `reason` (not\_right\_now\_reason, required), `other_text` (required when reason is `other`, max 200). The text on the sheet says this refines results for this search only.

**200 response:** `removed_place_id`, `results` (same shape as 6.3, re-ranked without that place).

## 7. Places

### 7.1 `GET /places/{place_id}?lat=&lng=` - User - Details screen

| Field | Type | UI element |
| --- | --- | --- |
| `id`, `name` | string | The Garden Kitchen |
| `distance_m` | int | 1.5 km |
| `reliability.level` | reliability\_level | Reliability: High |
| `opening_hours` | `{ display, open_time, close_time, timezone, is_open }` | Opening Hours: 8:00 AM - 10:00 PM |
| `address` | `{ line1, city, lat, lng }` | Address: Kansas City |
| `last_updated_at` | timestamp | Last Updated: 8 mins ago |
| `right_now` | RightNow | Right Now: Quiet, 10 min wait, Moderate Crowd |
| `live_biz_update` | `{ text, posted_at }` or null | Live Biz Update: Outdoor seating is currently available, 3 mins ago |
| `event` | `{ title, starts_at }` or null | Event: Live Music, Today - 8:00 PM |
| `offer` | `{ title, description }` or null | Offer: 20% off dinner, Enjoy 20% off selected dinner items tonight. |
| `community_updates[]` | first 3 items, each { `id`, `author` (Author), `text`, `posted_at`, `media[]` } | Community Update list: @city-guide22, 3 min, quote text |
| `is_saved`, `is_following` | bool | save and follow state |

### 7.2 `GET /places/{place_id}/community-updates?cursor=&limit=` - User

Full paginated list behind the Community Update section of Details. Items have the same shape as `community_updates[]` above, plus optional `signals` = { `crowd`, `noise`, `wait_minutes`, `parking`, `atmosphere[]` }.

### 7.3 `GET /places/{place_id}/reliability` - User - Reliability sheet

Response: `place_name`, `level`, `community_updates_count` (4), `live_biz_updates_count` (0), `most_recent_update_at` (Most recent update: 24 min ago), `explanation` (How this was assembled: Vybity combines available recent community observations and verified business updates to create the current Right Now summary).

### 7.4 `GET /places/{place_id}/why-not-here?session_id=` - User - WHY NOT HERE? sheet

Response: `crowd` (crowd\_level), `wait_minutes`, `parking` (parking\_level, may be `limited`), `reliability.level`. `session_id` is optional and used to tailor the comparison to the current search.

## 8. Compare

**`GET /places/compare?ids=<id1>,<id2>[,<id3>]&lat=&lng=`** - User - Compare screen (selection mode with Cancel and Compare 3 places is client-side). Two or three ids are required.

Response: `places[]` in the requested order, each with:

| Field | UI row |
| --- | --- |
| `id`, `name` | column header (Garden Kitchen, Downtown Bistro) |
| `environment` | Environment: Outdoors / Indoors |
| `crowd` | Crowd: Moderate / Busy |
| `noise` | Noise: Quiet / Loud |
| `wait_minutes` | Wait: 10 min / 25 min |
| `distance_m` | Distance: 1.2 km / 2.4 km |
| `parking` | Parking: Easy / Limited |
| `amenities[]` | Amenities: Patio, WiFi / Bar, WiFi |
| `updated_at` | Freshness: 8 min ago / 4 min ago |
| `reliability.level` | Reliability: High / High |

The section title CURRENT ENVIRONMENT is static text.

## 9. Directions and Navigation

### 9.1 `GET /places/{place_id}/directions?origin_lat=&origin_lng=&mode=` - User - Get Direction screen

Response: `place` = { `id`, `name`, `address.city` }, `right_now`, `reliability.level`, `travel` = { `mode`, `distance_m`, `duration_minutes`, `arrive_at` (Arrive around 7:35 PM) }, `route` = { `polyline`, `bounds` }.

### 9.2 `POST /navigation/sessions` - User - Start navigation

Request: `place_id`, `origin` = { `lat`, `lng` }, `mode` (default `walking`).

**201 response:** `session_id`, `route.polyline`, `eta_minutes` (6 min), `remaining_distance_m` (1.5 km remaining), `arrive_at`, `next_step` = { `instruction` (Turn Left), `maneuver` (`turn_left`), `distance_m` (IN 200M) }, plus the same `place`, `right_now`, `reliability` block as 9.1.

### 9.3 `PATCH /navigation/sessions/{session_id}` - User - live position updates

Request: `location` = { `lat`, `lng` }, `heading` (optional). Response: `eta_minutes`, `remaining_distance_m`, `next_step`, `arrived` (bool, true within 100 m of the place ⚠). When `arrived` is true the response also includes `visit` (see 10.1).

### 9.4 `DELETE /navigation/sessions/{session_id}` - User

Ends navigation. **204** no body.

## 10. Visits and Arrival

### 10.1 `POST /visits` - User - Manual Arrival (confirm Am I at The Garden Kitchen) and automatic arrival

| Field | Type | Req | Notes |
| --- | --- | --- | --- |
| `place_id` | string | yes |  |
| `method` | `auto` or `manual` | yes | manual comes from the confirm prompt |
| `location` | `{ lat, lng }` | yes | used to verify the visit |
| `navigation_session_id` | string | no |  |

**201 response:** `visit` = { `id`, `place` = { `id`, `name` }, `arrived_at`, `verified` (bool), `contribution_prompt_at` (arrived\_at + 10 min) }. The client schedules the prompt You are at the Garden Kitchen. What is it like there right now? for `contribution_prompt_at` (screen Show it after 10 min); tapping it opens the Community Update form with this `visit_id`.

**Errors:** 422 `TOO_FAR_FROM_PLACE`, 409 `VISIT_ALREADY_ACTIVE`.

### 10.2 `GET /visits/active` - User

Returns `visit` (shape above) or null. Feeds `active_visit` on Home (6.1).

### 10.3 `PATCH /visits/{visit_id}/end` - User ⚠

Marks the visit finished. Response: `visit` with `ended_at`.

## 11. Contribute

### 11.1 `GET /contribute/places?lat=&lng=&limit=` - User - Contribute screen

Response: `items[]` of PlaceCard (the card shows View Details and Update this place), ordered by how much each place needs a fresh update.

### 11.2 Media upload - Add Photo or Video

**`POST /media/uploads`** - User. Request: `kind` (`photo` or `video`), `content_type`, `size_bytes`. Limits: photo 10 MB, video 50 MB and 30 s ⚠. Response: `media_id`, `upload_url` (presigned), `upload_headers`, `expires_in_seconds`.

**`POST /media/{media_id}/complete`** - User. Called after the file is uploaded. Response: `status` (`processing` or `ready`), `thumbnail_url`.

### 11.3 `POST /places/{place_id}/community-updates` - User - Community Update form (Review screen is a client-side preview of these values)

| Field | Type | Req | UI element |
| --- | --- | --- | --- |
| `visit_id` | string | no | links the update to a verified visit |
| `crowd` | crowd\_level | no | Crowd chips |
| `noise` | noise\_level | no | Noise chips |
| `wait` | wait\_bucket | no | Wait chips |
| `parking` | parking\_level | no | Parking chips |
| `atmosphere` | atmosphere\[\] | no | Atmosphere chips, 1 to 3 ⚠ |
| `note` | string | no | Add a Note, max 500 ⚠ |
| `media_ids` | string\[\] | no | up to 4 ⚠ |

At least one of crowd, noise, wait, parking, atmosphere, or note is required.

**201 response:** `update` = { `id`, `status` (`submitted`), `created_at` }, `reward` = { `xp_awarded` (50), `xp_total`, `level`, `credibility.level`, `new_badges[]` }. Drives the Update Submitted screen (Thanks for helping others know before they go. +50 XP).

**Errors:** 422 `NO_SIGNAL_PROVIDED`, 429 `UPDATE_COOLDOWN` (same place within a short window, includes `retry_after_seconds`), 422 `MEDIA_NOT_READY`.

### 11.4 Follow place - Want updates from The Garden Kitchen?

**`PUT /places/{place_id}/follow`** - User. Request: `notifications` (bool, true when the user accepts). Response: `following` (true). **`DELETE /places/{place_id}/follow`** - 204. The two button labels on this screen are not in the design ⚠.

## 12. Saved

**`GET /me/saved-places?lat=&lng=&cursor=&limit=`** - User - Saved screen. Response: `items[]` of PlaceCard, `meta.next_cursor`. Each card supports Compare (Section 8), View (Section 7.1), NOT RIGHT NOW and WHY NOT HERE (Sections 6.4 and 7.4, session optional outside chat ⚠).

**`PUT /me/saved-places/{place_id}`** - User. Response: `is_saved` (true). **`DELETE /me/saved-places/{place_id}`** - 204.

## 13. Profile and Account

### 13.1 `GET /me/profile` - User - Profile screen

Response: `handle` (@city-guide22), `avatar`, `xp` (1,240), `level` (7, shown as LVL 7), `credibility.level` (High), `next_level_xp` (optional ⚠).

### 13.2 Badges and Rewards

**`GET /me/badges`** - User - Badges screen (design shows the title only ⚠). Items: `id`, `name`, `description`, `icon_url`, `earned`, `earned_at`, `progress`. **`GET /me/rewards`** - User. Items: `id`, `title`, `description`, `xp_required`, `unlocked`.

### 13.3 Your Activity

**`GET /me/favourites`** - alias of `GET /me/saved-places` ⚠.

**`GET /me/notifications?cursor=`** - User. Items: `id`, `type` (`place_update`, `community_update`, `reward`, `system`), `title`, `body`, `place_id`, `created_at`, `read`. `meta.unread_count`. **`POST /me/notifications/read`** with `ids[]` or `all` (bool). **`GET /me/notification-settings`** and **`PATCH /me/notification-settings`**: `push_enabled`, `place_updates`, `community_updates`, `rewards` ⚠. **`POST /me/devices`**: `platform`, `push_token`.

### 13.4 Personal Information (Personal Info, Email field)

**`GET /me`** - User. Response: `id`, `email`, `email_verified`, `handle`, `avatar`, `account_type`, `created_at`.

**`PATCH /me`** - User. Request: `email`. If the email changed, response is **202** with `pending_email` and `otp` object; then **`POST /me/email/verify`** with `otp` returns **200** and the updated `email`. Errors: 409 `EMAIL_ALREADY_REGISTERED`, 400 `OTP_INVALID`.

### 13.5 `POST /me/password` - User - Change Password screen

Request: `current_password`, `new_password`, `new_password_confirmation` (all required, same rules as 3.1). **204** no body; other sessions are revoked. Errors: 400 `CURRENT_PASSWORD_INCORRECT`, 422 `PASSWORD_MISMATCH`, 422 `PASSWORD_REUSED` ⚠.

### 13.6 Settings ⚠

**`GET /me/settings`**, **`PATCH /me/settings`**: `distance_unit` (`km` or `mi`), `language`. The Settings row has no designed screen.

### 13.7 Account Deletion

**`POST /me/account-deletion`** - User. Request: `password`, `reason` (optional). **202 response:** `scheduled_for` (30 days out ⚠). **`DELETE /me/account-deletion`** cancels during the grace period.

### 13.8 Log Out

See 3.9. Location Permission: see Section 5.

## 14. Legal content

**`GET /content/legal/{slug}`** - Public. `slug` is `privacy-policy` or `terms`. Response: `slug`, `title`, `version`, `updated_at`, `body_markdown`. Serves the Privacy Policy and Terms & conditions screens.

## 15. Error codes

| HTTP | Code | Meaning |
| --- | --- | --- |
| 400 | `OTP_INVALID` | Wrong code, includes `attempts_remaining` |
| 400 | `CURRENT_PASSWORD_INCORRECT` | Change Password current field wrong |
| 401 | `INVALID_CREDENTIALS` | Sign In failed |
| 401 | `TOKEN_EXPIRED` | Access token expired, client should refresh |
| 401 | `REFRESH_TOKEN_INVALID` | Force sign out |
| 401 | `SOCIAL_TOKEN_INVALID` | Google or Apple token rejected |
| 403 | `EMAIL_NOT_VERIFIED` | Route to OTP |
| 404 | `NOT_FOUND` | Place, session, or visit not found |
| 409 | `EMAIL_ALREADY_REGISTERED` |  |
| 409 | `VISIT_ALREADY_ACTIVE` |  |
| 410 | `OTP_EXPIRED`, `RESET_TOKEN_EXPIRED` |  |
| 422 | `VALIDATION_FAILED`, `PASSWORD_MISMATCH`, `PASSWORD_REUSED`, `NO_SIGNAL_PROVIDED`, `TOO_FAR_FROM_PLACE`, `MEDIA_NOT_READY` | `fields` populated where relevant |
| 423 | `ACCOUNT_LOCKED` |  |
| 429 | `OTP_RESEND_COOLDOWN`, `OTP_SEND_LIMIT`, `OTP_ATTEMPTS_EXCEEDED`, `UPDATE_COOLDOWN`, `RATE_LIMITED` | includes `retry_after_seconds` |

## 16. Endpoint index

| Method | Path | Screen |
| --- | --- | --- |
| POST | `/auth/register` | Register |
| POST | `/auth/login` | Sign In |
| POST | `/auth/social/google`, `/auth/social/apple` | Sign In, Register |
| POST | `/auth/forgot-password` | Forgot Pass |
| POST | `/auth/otp/verify`, `/auth/otp/resend` | OTP |
| POST | `/auth/reset-password` | Create New Password |
| POST | `/auth/refresh`, `/auth/logout` | Session, Profile |
| GET, POST, PUT | `/me/identity`, `/me/identity/regenerate`, `/me/account-type` | Identity ready |
| GET, PUT | `/content/onboarding`, `/me/onboarding` | Onboarding 4-6 |
| GET, PUT | `/me/location-permission` | Allow Location |
| GET | `/locations/search` | Add address manually |
| GET, PUT | `/me/location` | Home header |
| GET | `/home` | Home |
| POST, GET | `/chat/sessions`, `/chat/sessions/{id}/messages` | Chat, Result |
| POST | `/chat/sessions/{id}/places/{place_id}/not-right-now` | Not Right Now |
| GET | `/places/{id}`, `/places/{id}/community-updates`, `/places/{id}/reliability`, `/places/{id}/why-not-here` | Details, Reliability, Why not here |
| GET | `/places/compare` | Compare |
| GET | `/places/{id}/directions` | Get Direction |
| POST, PATCH, DELETE | `/navigation/sessions` | Start navigation |
| POST, GET, PATCH | `/visits`, `/visits/active`, `/visits/{id}/end` | Manual Arrival, Home after visit |
| GET | `/contribute/places` | Contribute |
| POST | `/media/uploads`, `/media/{id}/complete` | Add Photo or Video |
| POST | `/places/{id}/community-updates` | Community Update, Review, Updated |
| PUT, DELETE | `/places/{id}/follow` | After click done |
| GET, PUT, DELETE | `/me/saved-places`, `/me/saved-places/{id}` | Saved |
| GET | `/me/profile`, `/me/badges`, `/me/rewards`, `/me/favourites`, `/me/notifications` | Profile, Badges |
| GET, PATCH | `/me`, `/me/email/verify`, `/me/password`, `/me/settings`, `/me/notification-settings` | Personal Information, Change Password |
| POST, DELETE | `/me/account-deletion`, `/me/devices` | Account Deletion |
| GET | `/content/legal/{slug}` | Privacy Policy, Terms |

## 17. Open questions (please confirm)

1. **Business portal.** This Figma file has only the consumer app. The business portal you mentioned (Live Biz Update, Event, Offer authoring and Continue as Provider) is not in it. Please share its link so I can add the provider endpoints.
2. **Identity ready.** Does the refresh icon regenerate the handle and avatar, and what does Continue as Provider lead to?
3. **Terms checkbox** on Register: confirm it is terms and privacy consent.
4. **Atmosphere chips** on Community Update: single choice or multiple?
5. **Parking value Limited** appears in Why not here and Compare but not in the submission chips. Is it a server-computed label?
6. **Reliability medium.** Only High and Low appear in the design. Is there a Medium?
7. **Compare** screen shows 2 places while the button says Compare 3 places. Is the limit 2 to 3?
8. **Not Right Now and Why not here on Saved**: should they apply outside a chat session?
9. **Favourites vs Saved**: same list?
10. **After click done** (follow prompt) button labels, and the **Badges, Rewards, Notifications, Settings** screens, are not designed yet.
11. **Reset password flow**: return to Sign In, or auto-login?
12. **Limits** (password rules, note length, media size, arrival radius, deletion grace period) are my defaults, please adjust.

# Part 2 - Business Portal API

Source: the Business Portal page of the Figma file (28 frames: dashboard screens at 1440 px, sign-in and claim screens at 1920 px). Sections 1 (conventions) and 15 (error format) apply here too. All portal endpoints live under `/business`.

## 18. Portal conventions

| Topic | Rule |
| --- | --- |
| Accounts | The portal has no Register screen. A portal user is an account created in the phone app and set to `provider` (Identity ready > Continue as Provider) ⚠ |
| Roles | `owner` (the claimant) and `staff` (invited from Team) |
| Active location | One location per account, taken from the token. Header `X-Location-Id` is accepted for future multi-location use ⚠ |
| Claim gating | Until `claim_status` is `approved`, only Sections 19, 20 and 29 work. Everything else returns 403 `CLAIM_NOT_APPROVED` |
| Permissions | Staff only reach modules assigned to them (Section 27). Otherwise 403 `FORBIDDEN_MODULE`. Team and Settings are owner-only ⚠ |
| Dates | The UI uses MM/DD/YYYY plus a time. The API takes ISO 8601 with timezone offset, in the location timezone |
| Notify followers | Publishing an update, offer or calendar item sends a push to followers (Analytics tracks Delivered and Opened). Optional `notify_followers` (default true) on those endpoints ⚠ |

### 18.1 Portal enums

| Enum | Values | UI source |
| --- | --- | --- |
| `claim_status` | `none`, `draft`, `pending_review`, `approved`, `rejected` ⚠ | Pending Review banner |
| `permission_module` | `overview`, `live_updates`, `promotions_calendar`, `offers_coupons`, `analytics`, `location_profile`, `reports` | Assign Permissions |
| `publish_mode` | `now`, `schedule` | Publishing Options |
| `live_update_status` | `scheduled`, `published`, `expired` | Live Biz Updates list |
| `offer_status` | `scheduled`, `active`, `expired` ⚠ | Offers list |
| `calendar_item_type` | `update`, `special`, `event`, `coupon`, `reminder` | Schedule Item > Item Type |
| `report_status` | `pending_review`, `resolved` | Reports table |
| `dispute_status` | `open` ⚠, `in_progress`, `resolved` ⚠ | Disputes list |
| `team_status` | `active`, `invited` ⚠, `disabled` ⚠ | Team table |
| `weekday` | `monday` ... `sunday` | Location Profile hours |

## 19. Portal authentication

### 19.1 `POST /business/auth/login` - Public - Log in screen

Request: `email`, `password` (both required).

**200 response:** `access_token`, `refresh_token`, `expires_in`, `user` = { `id`, `name`, `email`, `role`, `permissions[]` }, `business` = { `location_id`, `name` (Garden Kitchen), `label` (Business Account), `claim_status` }.

Client routing from `claim_status`: `none` goes to Claim your business, `draft` resumes the claim, `pending_review` shows the Claim status screen, `approved` opens Overview.

**Errors:** 401 `INVALID_CREDENTIALS`, 403 `ACCOUNT_NOT_PROVIDER`, 423 `ACCOUNT_LOCKED`.

### 19.2 Password recovery - Forgot Password, Verify Email, Create New Password

| Endpoint | Request | Response |
| --- | --- | --- |
| `POST /business/auth/forgot-password` | `email` | `message`, `otp.expires_in_seconds`, `otp.resend_available_in_seconds` (always 200) |
| `POST /business/auth/otp/verify` | `email`, `otp` (6 digits, Verification Code), `purpose` = `password_reset` | `reset_token`, `reset_token_expires_in_seconds` |
| `POST /business/auth/otp/resend` | `email`, `purpose` | `otp` object (Resend Code) |
| `POST /business/auth/reset-password` | `reset_token`, `password` (New Password), `password_confirmation` (Confirm Password) | `password_reset` = true |

Errors are the same as 3.5 to 3.7. `POST /business/auth/refresh` and `POST /business/auth/logout` mirror 3.8 and 3.9 (no logout control is drawn in the portal ⚠).

## 20. Claim your business

### 20.1 `GET /business/claims/current` - User - Claim status screen

Response: `claim` or null. Fields: `id`, `status`, `location` = { `id`, `name` (The Garden Kitchen), `address_line` (214 Willow Ave, Dhaka) }, `submitted_at`, `message` (Your claim is currently being reviewed by our administration team. You will be notified once a decision is made.).

### 20.2 `GET /business/locations/search?q=` - User - Find your business (Search by business name)

Response: `items[]` = { `id`, `name`, `address_line`, `claimable` (bool) }.

### 20.3 `POST /business/claims` - User - Is this your business? (confirm)

Request: `location_id`. **201 response:** `claim` = { `id`, `status` (`draft`), `location` }. **Errors:** 409 `LOCATION_ALREADY_CLAIMED`, 409 `CLAIM_ALREADY_OPEN`.

### 20.4 `GET /business/claims/{claim_id}/requirements` - User - Verify Your Business

The admin configures what is asked, so the client renders it dynamically.

| Field | UI element |
| --- | --- |
| `fields[]` = { `key`, `label`, `placeholder`, `type`, `required`, `validation` } | CONFIGURABLE VERIFICATION FIELD (Enter required verification details) |
| `documents[]` = { `key`, `label`, `accepted_formats[]`, `max_size_mb`, `required` } | CONFIGURABLE DOCUMENT UPLOAD (Upload File, Format as configured by system) |

### 20.5 File uploads (claim documents and gallery media)

**`POST /business/uploads`** - User. Request: `purpose` (`claim_document` or `gallery`), `file_name`, `content_type`, `size_bytes`. Response: `upload_id`, `upload_url` (presigned), `upload_headers`, `expires_in_seconds`.

**`POST /business/uploads/{upload_id}/complete`** - User. Response: `status` (`ready`), `file_name`, `size_bytes`.

### 20.6 `PUT /business/claims/{claim_id}/verification` - User - Verify Your Business > Continue

Request: `fields` (object of key to value), `documents` (object of key to `upload_id`). **200 response:** the claim with the review summary (see 20.7). **Errors:** 422 `VERIFICATION_FIELDS_INVALID` (with `fields`), 422 `DOCUMENT_REQUIRED`.

### 20.7 `GET /business/claims/{claim_id}` - User - Review your claim

Response: `claim` = { `id`, `status`, `location` = { `name`, `address_line` }, `fields[]` = { `key`, `label`, `value` }, `documents[]` = { `key`, `label`, `file_name`, `size_bytes` } }.

### 20.8 `POST /business/claims/{claim_id}/submit` - User - Review your claim > Submit

No body. **202 response:** `status` (`pending_review`), `submitted_at`, `message` (Your business claim has been submitted successfully and is currently in the queue for review.). **Errors:** 422 `CLAIM_INCOMPLETE`.

The admin decision has no screen in this design ⚠. The owner is told by email or push, and the next login returns the new `claim_status`.

## 21. Overview

**`GET /business/overview`** - permission `overview`.

| Field | UI element |
| --- | --- |
| `business` = { `name`, `label` } | Garden Kitchen, Business Account |
| `right_now` = { `noise`, `wait_minutes`, `crowd`, `parking` } | Right Now: Quiet, 10 min wait, Moderate Crowd, Not Applicable (parking `not_applicable`) |
| `followers` = { `total` (12,450), `new_this_week` (5) } | Total Followers, +5 new this week |
| `recent_engagement[]` = { `id`, `author` (handle and avatar), `text`, `posted_at` } | Recent Engagement (3 items) |
| `upcoming_scheduled_posts[]` = { `id`, `title`, `scheduled_for`, `status` (`scheduled`) } | Upcoming Scheduled Posts: Today 4:00 PM, Happy Hour Specials Starting Soon! |
| `need_attention[]` = { `id`, `type` (`reported_content`), `title`, `message`, `report_id` } | Need Attention: Reported Content, A user reported a recent photo tagged at your location. Please review. |

**`GET /business/community-updates?cursor=&limit=`** - View All under Recent Engagement. Items have the same shape as `recent_engagement[]`, plus `signals` and `media[]` as in 7.2.

## 22. Live Biz Updates

### 22.1 `GET /business/live-updates?status=&cursor=&limit=` - permission `live_updates`

Response: `items[]` = { `id`, `text`, `category` (`environment_status` ⚠, the label above the list), `status`, `published_at`, `scheduled_for`, `expires_at`, `created_by` = { `id`, `name` } }. Example row: Outdoor seating is currently available and heaters are on., Today 2:14 PM, Published; older rows show Expired.

### 22.2 `POST /business/live-updates` - Create Live Biz Update

| Field | Type | Req | UI element |
| --- | --- | --- | --- |
| `text` | string | yes | Update (Tell customers what they need to know...), max 280 ⚠ |
| `publish_mode` | publish\_mode | yes | Publish Now / Schedule |
| `scheduled_for` | timestamp | if schedule | date and time of the schedule |
| `expires_at` | timestamp | no ⚠ | Expiration Date + Expiration Time |
| `notify_followers` | bool | no |  |

**201 response:** the item (shape above). **Errors:** 422 `VALIDATION_FAILED`, 422 `EXPIRY_BEFORE_PUBLISH`.

### 22.3 `DELETE /business/live-updates/{id}` ⚠

Ends an update early (the list needs an action, none is drawn). **204**.

## 23. Promotions Calendar

**`GET /business/calendar?from=&to=`** - permission `promotions_calendar`. Powers Month, Week and Schedule views (the view switch and Today button are client-side; September 2026 is the month header). Response: `items[]` = { `id`, `type`, `title`, `starts_at`, `status`, `offer_id` }.

**`POST /business/calendar/items`** - Schedule Item form.

| Field | Type | Req | UI element |
| --- | --- | --- | --- |
| `item_type` | calendar\_item\_type | yes | Item Type: Update, Special, Event, Coupon, Reminder |
| `title` | string | yes | Title (e.g. Live Music Friday) |
| `publish_mode` | publish\_mode | yes | Publish Now toggle |
| `scheduled_at` | timestamp | if schedule | Date + time |
| `notify_followers` | bool | no |  |

**201 response:** the item. **`DELETE /business/calendar/items/{id}`** cancels a scheduled item ⚠ (**204**).

## 24. Offers and coupons

### 24.1 `GET /business/offers?status=&cursor=` - permission `offers_coupons`

Response: `items[]` = { `id`, `title` (20% off Dinner), `terms`, `status`, `valid_from` (Validity, Sep 2, 2026 5:00 PM), `valid_to`, `code` (SWEET), `quantity_limit`, `quantity_available` (50 available) }.

### 24.2 `POST /business/offers` - Create New Offer

| Field | Type | Req | UI element |
| --- | --- | --- | --- |
| `title` | string | yes | Title (e.g. Live Music Friday) |
| `terms` | string | yes | Terms (e.g. 20% off selected dinner items.) |
| `publish_mode` | publish\_mode | yes | Publish Now / Schedule |
| `valid_from` | timestamp | yes | Start Date + Start Time |
| `valid_to` | timestamp | no | End Date + End Time |
| `code` | string | yes | Redemption Code (SWEET), unique per location |
| `quantity_limit` | int | no ⚠ | Quantity Limit (shown in list, not in the form) |
| `notify_followers` | bool | no |  |

**201 response:** the offer. **Errors:** 409 `CODE_ALREADY_USED`, 422 `VALIDITY_RANGE_INVALID`. **`PATCH /business/offers/{id}`** and **`DELETE /business/offers/{id}`** ⚠ end or edit an offer.

On the phone, `title` and `terms` appear as Offer title and description in Details (7.1).

## 25. Analytics

**`GET /business/analytics?range=30d`** - permission `analytics`. `range` is `7d`, `30d` or `90d` ⚠ (the design shows a 30-day axis, D1 to D30).

| Field | UI element |
| --- | --- |
| `follower_growth` = { `change` (589), `change_percent` (15.5), `series[]` = { `date`, `value` } } | Follower Growth: +589 followers, +15.5% this period |
| `notifications` = { `delivered` (12,130), `opened` (3,047), `open_rate` (25.1), `series[]` = { `date`, `delivered`, `opened` } } | Notification Delivery & Opens. Opens are counted on notification tap |
| `post_engagement` = { `avg_views_per_post` (679), `total_views` (20,361), `series[]` = { `date`, `views` } } | Post Engagement |
| `coupons` = { `views` (5,837), `redemptions` (1,497), `series[]` = { `date`, `views`, `redemptions` } } | Coupon Views & Redemptions |
| `community_updates` = { `total` (617), `avg_per_day` (21), `series[]` = { `date`, `count` } } | Community Update Volume |
| `aggregate` = { `followers`, `followers_change_percent`, `notifications_delivered`, `notification_open_rate`, `post_views`, `avg_views_per_post`, `coupon_views`, `coupon_redemptions` } | Aggregate Trends cards |

## 26. Location Profile

**`GET /business/location-profile`** - permission `location_profile`. Response:

| Field | UI element |
| --- | --- |
| `hours[]` = { `day`, `open_time`, `close_time`, `closed` } (7 entries) | Hours: your regular operating schedule |
| `contact` = { `phone`, `email`, `address` } | Contact details |
| `links` = { `website`, `instagram` } | Links |
| `description` | Description |
| `gallery[]` = { `id`, `url`, `thumbnail_url`, `status` (`approved` or `pending_review`) } | Approved business gallery |

**`PUT /business/location-profile`** - Edit Profile > Save Changes. Request: `hours[]`, `contact`, `links`, `description` (same shapes). Response: the updated profile. Cancel is client-side. Address changes may need re-verification ⚠ (`422 ADDRESS_CHANGE_REQUIRES_REVIEW`).

**`POST /business/location-profile/gallery`** - Add Media. Request: `upload_id` (purpose `gallery`). Response: media with `status` = `pending_review` until approved. **`DELETE /business/location-profile/gallery/{media_id}`** - **204**.

## 27. Team

**`GET /business/team`** - owner only. Response: `members[]` = { `id`, `name` (Maya Chowdhury), `email`, `status` (Active), `role`, `permissions[]` }.

**`POST /business/team/invitations`** - Invite Staff. Request: `email` (required), `name` (optional ⚠), `permissions[]` (permission\_module, optional). Response: member with status `invited`. Errors: 409 `ALREADY_A_MEMBER`.

**`PUT /business/team/{member_id}/permissions`** - Assign Permissions (Save). Request: `permissions[]` from Overview, Live Biz Updates, Promotions Calendar, Offers/Coupons, Analytics, Location Profile, Reports. Response: member.

**`DELETE /business/team/{member_id}`** ⚠ (Actions column). **204**.

## 28. Reports and Disputes

### 28.1 `GET /business/reports?status=&cursor=` - permission `reports`

Reports affecting business content. Response: `items[]` = { `id`, `affected_content` = { `type`, `label` (Live Biz Update), `excerpt` }, `reason` (Reported as misleading), `status`, `reported_at` }. **`GET /business/reports/{id}`** adds the full content (text or photo `media`) and `disputable` (bool). The table shows a STAFF MEMBER column header that looks like a copy error ⚠.

### 28.2 Disputes and support requests

**`GET /business/disputes?cursor=`** - Response: `items[]` = { `id`, `subject` (Dispute on branch menu report), `related_content` (Live Biz Update - Weekend brunch menu), `details`, `submitted_at`, `status` }.

**`POST /business/disputes`** - Submit dispute / support request.

| Field | Type | Req | UI element |
| --- | --- | --- | --- |
| `subject` | string | yes | Subject (Brief summary of the issue) |
| `related_content` | string | no | Related Content (e.g. Live Biz Update from sept 2) |
| `report_id` | string | no ⚠ | links to a row in 28.1 |
| `details` | string | yes | Details (Explain the issue and what you would like reviewed) |

**201 response:** the dispute with `status` and `submitted_at` (Submitted Sep 3, 2026).

## 29. Portal Settings

**`GET /business/me`** - Response: `name` (Sarah Jessie), `restaurant_name`, `email`, `phone`, `role`. **`PATCH /business/me`** - My Profile > Save Changes. Request: `name`, `restaurant_name`, `phone`, `email`. Changing `email` triggers an OTP check as in 13.4 ⚠; `restaurant_name` changes may go to review since the location name is verified ⚠.

**`POST /business/me/password`** - Security > Update Password. Request: `current_password`, `new_password`, `password_confirmation`. **204**. Errors as 13.5.

**`GET /content/legal/{slug}`** - Privacy & Policy and Terms & Conditions tabs (same endpoint as Section 14).

## 30. Portal error codes

| HTTP | Code | Meaning |
| --- | --- | --- |
| 403 | `CLAIM_NOT_APPROVED` | Claim still pending, module locked |
| 403 | `FORBIDDEN_MODULE` | Staff lacks the permission |
| 403 | `ACCOUNT_NOT_PROVIDER` | Not a provider account |
| 409 | `LOCATION_ALREADY_CLAIMED`, `CLAIM_ALREADY_OPEN`, `CODE_ALREADY_USED`, `ALREADY_A_MEMBER` |  |
| 422 | `VERIFICATION_FIELDS_INVALID`, `DOCUMENT_REQUIRED`, `CLAIM_INCOMPLETE`, `EXPIRY_BEFORE_PUBLISH`, `VALIDITY_RANGE_INVALID`, `ADDRESS_CHANGE_REQUIRES_REVIEW` | `fields` populated where relevant |

# Part 3 - Workflows (screen to request to response)

Each table follows the screen order in Figma. Section numbers point to the full field definitions. Order of the first screens (Splash, Onboarding, then Sign In or Register) follows the frame labels on the canvas ⚠.

## 31. Phone app workflows

### A1. Create account and identity

| Step | Screen | User action | Call | Request and response |
| --- | --- | --- | --- | --- |
| 1 | Splash 2 and 3 | wait | none | static |
| 2 | Onboarding 4, 5, 6 | swipe or tap Skip | `GET /content/onboarding`, later `PUT /me/onboarding` | slides\[\] in; status `completed` or `skipped` is kept on device and sent after login ⚠ |
| 3 | Register | fill email, password, retype password, tick terms, Register | `POST /auth/register` | 4 fields in; `otp` timers out (3.1) |
| 4 | Register | tap Google or Apple | `POST /auth/social/google` or `/apple` | `id_token` in; tokens, `is_new_user` out |
| 5 | OTP | enter 6 digits | `POST /auth/otp/verify` (`email_verification`) | tokens, `user`, `is_new_user` out |
| 6 | OTP | tap Send again | `POST /auth/otp/resend` | new `otp` timers |
| 7 | Identity ready | screen opens | `GET /me/identity` | `handle` (@city-guide22), `avatar` |
| 8 | Identity ready | tap refresh icon | `POST /me/identity/regenerate` ⚠ | new handle and avatar |
| 9 | Identity ready | Continue or Continue as Provider | `PUT /me/account-type` | `consumer` goes to the app, `provider` goes to the portal claim flow |

### A2. Set location

| Step | Screen | User action | Call | Request and response |
| --- | --- | --- | --- | --- |
| 1 | Allow Location | allow or deny | `PUT /me/location-permission` | `status` in; `status` out |
| 2a | Allow Location | allowed | `PUT /me/location` | `source: gps`, `lat`, `lng` in; `location.city` out |
| 2b | Add address manually | type address | `GET /locations/search?q=` | `items[]` out |
| 3b | Add address manually | pick a result | `PUT /me/location` | `source: manual`, `location_id` in; `location.city` out |

### A3. Sign in and password recovery

| Step | Screen | User action | Call | Request and response |
| --- | --- | --- | --- | --- |
| 1 | Sign In | email, password, Sign In | `POST /auth/login` | tokens and `user` flags out (routes to Onboarding, Allow Location or Home) |
| 2 | Sign In | Google or Apple | `POST /auth/social/*` | as A1 step 4 |
| 3 | Forgot Pass | email, submit | `POST /auth/forgot-password` | `otp` timers out |
| 4 | OTP | 6 digits | `POST /auth/otp/verify` (`password_reset`) | `reset_token` out |
| 5 | Create New Password | password, retype | `POST /auth/reset-password` | `password_reset: true`, back to Sign In |

### A4. Discover a place (ask the assistant)

| Step | Screen | User action | Call | Request and response |
| --- | --- | --- | --- | --- |
| 1 | Home | open | `GET /home` | `location.city`, `time_of_day`, `prompt_suggestions[]`, `active_visit` |
| 2 | Home | type or tap a suggestion | `POST /chat/sessions` | `text`, `suggestion_id` in; `session_id`, `messages[]` out |
| 3 | Chat | assistant asks how far; tap Walking Distance | `POST /chat/sessions/{id}/messages` | `quick_reply_id` in; assistant `message`, `results` out |
| 4 | Result | view cards | same response | `results.places[]` PlaceCard with Right Now, Reliability, Why this fits |
| 5 | Result | tap NOT RIGHT NOW, pick a reason | `POST /chat/sessions/{id}/places/{pid}/not-right-now` | `reason` in; refreshed `results` out |
| 6 | Result | tap WHY NOT HERE? | `GET /places/{id}/why-not-here` | `crowd`, `wait_minutes`, `parking`, `reliability` |

### A5. Evaluate, compare and save

| Step | Screen | User action | Call | Request and response |
| --- | --- | --- | --- | --- |
| 1 | Details | tap View | `GET /places/{id}` | full place object (7.1) |
| 2 | Details | tap Reliability | `GET /places/{id}/reliability` | counts, last update, explanation |
| 3 | Details | see all community updates | `GET /places/{id}/community-updates` | paged list |
| 4 | Result or Saved | select cards, Compare 3 places | `GET /places/compare?ids=` | comparison rows (Section 8) |
| 5 | Details or card | save or unsave | `PUT` or `DELETE /me/saved-places/{id}` | `is_saved` |
| 6 | Saved | open tab | `GET /me/saved-places` | PlaceCard list |

### A6. Navigate and arrive

| Step | Screen | User action | Call | Request and response |
| --- | --- | --- | --- | --- |
| 1 | Get Direction | tap directions | `GET /places/{id}/directions` | route, `arrive_at` (Arrive around 7:35 PM) |
| 2 | Start navigation | Start | `POST /navigation/sessions` | `place_id`, `origin`, `mode` in; `eta_minutes`, `remaining_distance_m`, `next_step` out |
| 3 | Start navigation | position changes | `PATCH /navigation/sessions/{id}` | `location` in; updated eta, step, `arrived` out |
| 4 | Manual Arrival | tap confirm (At The Garden Kitchen?) | `POST /visits` | `method: manual`, `location` in; `visit` with `contribution_prompt_at` out |
| 5 | Show it after 10 min | prompt appears 10 min later | none (timer from `contribution_prompt_at`) | opens the Community Update form with `visit_id` |
| 6 | Home after visit | open Home | `GET /home` | `active_visit` shows the YOU ARE VISITING card |

### A7. Contribute an update

| Step | Screen | User action | Call | Request and response |
| --- | --- | --- | --- | --- |
| 1 | Contribute | open tab | `GET /contribute/places` | PlaceCard list |
| 2 | Community Update | tap Add Photo or Video | `POST /media/uploads`, then `/complete` | `media_id` out |
| 3 | Community Update | pick chips, note, media | none until Review | state kept on device |
| 4 | Review | check summary, submit | `POST /places/{id}/community-updates` | signals, `note`, `media_ids`, `visit_id` in; `update`, `reward` out |
| 5 | Updated | Update Submitted, +50 XP | none | uses `reward.xp_awarded` |
| 6 | After click done | Want updates from this place? | `PUT /places/{id}/follow` | `notifications` in; `following` out |

### A8. Profile and account

| Step | Screen | User action | Call | Request and response |
| --- | --- | --- | --- | --- |
| 1 | Profile | open | `GET /me/profile` | handle, xp, level, credibility |
| 2 | Badges | open | `GET /me/badges` | badge list ⚠ |
| 3 | Notifications | open | `GET /me/notifications` | list, `unread_count` |
| 4 | Personal Information | edit email | `PATCH /me`, `POST /me/email/verify` | `email` in; OTP check |
| 5 | Change Password | three fields | `POST /me/password` | 204 |
| 6 | Location Permission | change | `PUT /me/location-permission` | `status` |
| 7 | Privacy Policy, Terms | open | `GET /content/legal/{slug}` | body text |
| 8 | Account Deletion | confirm | `POST /me/account-deletion` | 202, `scheduled_for` |
| 9 | Log Out | tap | `POST /auth/logout` | 204 |

## 32. Business portal workflows

### P1. Sign in and recover

| Step | Screen | User action | Call | Request and response |
| --- | --- | --- | --- | --- |
| 1 | Log in | email, password | `POST /business/auth/login` | tokens, `user`, `business.claim_status` |
| 2 | Forgot Password | email | `POST /business/auth/forgot-password` | `otp` timers |
| 3 | Verify Email | 6 digits, Resend Code | `POST /business/auth/otp/verify`, `/resend` | `reset_token` |
| 4 | Create New Password | new, confirm | `POST /business/auth/reset-password` | `password_reset: true` |

### P2. Claim your business

| Step | Screen | User action | Call | Request and response |
| --- | --- | --- | --- | --- |
| 1 | Claim your business | Start | `GET /business/claims/current` | null, so continue |
| 2 | Find your business | search by name | `GET /business/locations/search?q=` | `items[]` |
| 3 | Is this your business? | confirm | `POST /business/claims` | `location_id` in; `claim` (draft) out |
| 4 | Verify Your Business | form loads | `GET /business/claims/{id}/requirements` | `fields[]`, `documents[]` |
| 5 | Verify Your Business | Upload File | `POST /business/uploads`, `/complete` | `upload_id` |
| 6 | Verify Your Business | Continue | `PUT /business/claims/{id}/verification` | `fields`, `documents` in; claim summary out |
| 7 | Review your claim | check | `GET /business/claims/{id}` | summary |
| 8 | Review your claim | Submit | `POST /business/claims/{id}/submit` | 202, `pending_review` |
| 9 | Claim Submitted | done | none | message |
| 10 | Claim status | later logins | `GET /business/claims/current` | Pending Review banner, modules locked until `approved` |

### P3. Daily dashboard

| Step | Screen | User action | Call | Request and response |
| --- | --- | --- | --- | --- |
| 1 | Overview | open | `GET /business/overview` | right\_now, followers, recent\_engagement, scheduled posts, need\_attention |
| 2 | Overview | View All | `GET /business/community-updates` | paged list |
| 3 | Overview | open Reported Content alert | `GET /business/reports/{id}` | content and reason |

### P4. Post a live update

| Step | Screen | User action | Call | Request and response |
| --- | --- | --- | --- | --- |
| 1 | Live Biz Updates | open | `GET /business/live-updates` | list with status |
| 2 | Create Update | text, Publish Now or Schedule, expiration | `POST /business/live-updates` | body in; item out |
| 3 | Live Biz Updates | update expires | none | `status` becomes `expired` |

### P5. Plan promotions

| Step | Screen | User action | Call | Request and response |
| --- | --- | --- | --- | --- |
| 1 | Promotions Calendar | switch Month, Week, Schedule; change month | `GET /business/calendar?from=&to=` | `items[]` |
| 2 | Schedule Item | type, title, date and time | `POST /business/calendar/items` | item out |

### P6. Create an offer

| Step | Screen | User action | Call | Request and response |
| --- | --- | --- | --- | --- |
| 1 | Offers | open | `GET /business/offers` | list |
| 2 | Create Offer | title, terms, publishing, validity, code, Save | `POST /business/offers` | offer out |

### P7. Review analytics

| Step | Screen | User action | Call | Request and response |
| --- | --- | --- | --- | --- |
| 1 | Analytics | open | `GET /business/analytics?range=30d` | six chart blocks plus aggregate cards |

### P8. Edit the public profile

| Step | Screen | User action | Call | Request and response |
| --- | --- | --- | --- | --- |
| 1 | Location Profile | open | `GET /business/location-profile` | hours, contact, links, description, gallery |
| 2 | Edit Profile | change fields, Save Changes | `PUT /business/location-profile` | profile out |
| 3 | Edit Profile | Add Media | `POST /business/uploads`, `POST /business/location-profile/gallery` | media `pending_review` |

### P9. Manage the team

| Step | Screen | User action | Call | Request and response |
| --- | --- | --- | --- | --- |
| 1 | Team | open | `GET /business/team` | members |
| 2 | Team | Invite Staff | `POST /business/team/invitations` | email, permissions in |
| 3 | Assign Permissions | tick modules, Save | `PUT /business/team/{id}/permissions` | member out |

### P10. Handle a report or dispute

| Step | Screen | User action | Call | Request and response |
| --- | --- | --- | --- | --- |
| 1 | Reports | open | `GET /business/reports` | affected content, reason, status |
| 2 | Disputes | Submit Dispute | `POST /business/disputes` | subject, related\_content, details in; dispute out |
| 3 | Disputes | follow progress | `GET /business/disputes` | status `in_progress` |

### P11. Settings

| Step | Screen | User action | Call | Request and response |
| --- | --- | --- | --- | --- |
| 1 | My Profile | edit, Save Changes | `GET` and `PATCH /business/me` | name, restaurant\_name, phone, email |
| 2 | Security | Update Password | `POST /business/me/password` | 204 |

## 33. How the app and the portal connect

| Flow | Portal side | App side |
| --- | --- | --- |
| Live update to consumers | `POST /business/live-updates` | appears as `live_biz_update` in `GET /places/{id}` and cards, counts in `live_biz_updates_count` on Reliability |
| Offer to consumers | `POST /business/offers` | appears as `offer` in Details |
| Event to consumers | calendar item of type `event` | appears as `event` in Details and cards |
| Followers and push | Overview follower count, Analytics Delivered and Opened | `PUT /places/{id}/follow` (A7 step 6) |
| Community to business | Overview Recent Engagement, Analytics volume | `POST /places/{id}/community-updates` |
| Provider onboarding | Section 20 claim flow | Identity ready, Continue as Provider (A1 step 9) |
| Reported content | Overview Need Attention, Reports, Disputes | no report screen in the app design, so no endpoint yet ⚠ |
| Coupon redemption | Analytics Coupon Views and Redemptions | no redeem screen in the app design, so no endpoint yet ⚠ |

## 34. Portal open questions (please confirm)

1. **Reporting and redemption:** The portal shows reported content and coupon redemptions, but the phone app has no screen to report content or redeem a coupon. Do you want endpoints proposed for these?
2. **Admin back office:** Claim approval, report moderation and dispute handling are not designed. Do you want an admin API as well?
3. **Provider accounts:** Is a portal user always created in the phone app first, or will the portal get its own sign-up?
4. **Follower counts** disagree across screens (12,450 on Overview, 4,389 in Analytics aggregate, +589 growth). Which is the source of truth?
5. **Environment Status** label above Live Biz Updates: what does it mean?
6. **Quantity Limit** appears in the offers list but not in the Create Offer form. Should it be in the form?
7. **Team:** how does an invited staff member accept (email link or code), and can Team and Settings be delegated?
8. **Offer redemption code:** one shared code per offer (as drawn) or unique codes per user?
9. **Reports table** has a STAFF MEMBER column header that looks like a copy error.
10. **Multiple locations:** can one owner manage several locations?
