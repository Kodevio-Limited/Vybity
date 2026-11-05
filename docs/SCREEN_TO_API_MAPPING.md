# Vybity — Screen-to-API Mapping (workflow request/response)

Every screen and action, mapped to its endpoint, request and response, in workflow order, with a short
description. Conventions, enums and full field rules live in `API_SPECIFICATION.md`. Screens without an
API call (pure UI/state) are marked **client-side**.

Base URL: `https://api.vybity.com/v1` (placeholder). Auth: `Authorization: Bearer <access_token>`.
Success envelope `{ status(success | fail), status_code, message, data, meta? }`; error envelope `{status(success | fail), status_code, message, error: { code, message, fields? } }`.

---

## Part 1 — Consumer app workflows

### A1. Create account and identity

| # | Screen | Action | Endpoint | Request | Response | Description |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Splash 2 / 3 | wait | — | — | — | Static images; auto-advance. |
| 2 | Onboarding 4/5/6 | swipe / Skip | `GET /content/onboarding` | — | `slides[] {id,order,title,body,image_url}` | Copy is bundled; endpoint lets CMS override. Client stores `completed`/`skipped`. |
| 3 | Onboarding 4/5/6 | (after login) | `PUT /me/onboarding` | `{status:"completed"|"skipped"}` | `{onboarding_completed:true}` | Sends the stored choice once the user is authenticated. |
| 4 | Register | fill + Register | `POST /auth/register` | `{email,password,password_confirmation,accepted_terms,device?}` | `{user_id,email,email_verified:false,otp:{expires_in_seconds,resend_available_in_seconds}}` | Creates the account, emails a 6-digit code, returns OTP timers. Error `409 EMAIL_ALREADY_REGISTERED`, `422 PASSWORD_MISMATCH`. |
| 5 | Register | Google / Apple | `POST /auth/social/google` \| `/auth/social/apple` | `{id_token, authorization_code?, full_name?, device?}` | `{access_token,refresh_token,expires_in,user,is_new_user}` | `is_new_user:true` → Identity ready. Error `401 SOCIAL_TOKEN_INVALID`. |
| 6 | OTP | enter 6 digits | `POST /auth/otp/verify` | `{email,otp,purpose:"email_verification"}` | `{access_token,refresh_token,expires_in,user,is_new_user:true}` | Verifies email, signs the user in. Errors `400 OTP_INVALID`(+`attempts_remaining`), `410 OTP_EXPIRED`, `429 OTP_ATTEMPTS_EXCEEDED`. |
| 7 | OTP | Send again | `POST /auth/otp/resend` | `{email,purpose}` | `{otp:{...}}` | New code + timers. Errors `429 OTP_RESEND_COOLDOWN`, `429 OTP_SEND_LIMIT`. |
| 8 | Identity ready | open | `GET /me/identity` | — | `{handle,avatar:{id,url},created_at}` | Shows the system-generated public identity (`@city-guide22`). |
| 9 | Identity ready | (refresh icon) | `POST /me/identity/regenerate` ⚠ | — | `{handle,avatar}` | **Assumption** (icon sits inside "Continue as Provider" button — confirm). Limit 5/day. |
| 10 | Identity ready | Continue / Continue as Provider | `PUT /me/account-type` | `{account_type:"consumer"|"provider"}` | `{account_type,next:"app"|"business_onboarding"}` | `consumer` → app; `provider` → portal claim flow. |

### A2. Set location

| # | Screen | Action | Endpoint | Request | Response | Description |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Allow Location | Allow / Deny | `PUT /me/location-permission` | `{status:"granted"|"while_using"|"denied"}` | `{status}` | Records the OS permission outcome. |
| 2a | Allow Location | allowed | `PUT /me/location` | `{source:"gps",lat,lng}` | `{location:{city,label,lat,lng,source}}` | Stores GPS location; header shows "Kansas City". |
| 2b | Add address manually | type | `GET /locations/search?q=&lat=&lng=` | — | `{items:[{id,label,address_line,city,lat,lng}]}` | Geocoder suggestions (design shows "Kansas City"). |
| 3b | Add address manually | pick | `PUT /me/location` | `{source:"manual",location_id}` | `{location:{...}}` | Stores the chosen place as current location. |
| — | Profile > Location Permission | change | `GET`/`PUT /me/location-permission` | `{status}` | `{status}` | Same object as step 1. |

### A3. Sign in and password recovery

| # | Screen | Action | Endpoint | Request | Response | Description |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Sign In | email+password | `POST /auth/login` | `{email,password,device?}` | `{access_token,refresh_token,expires_in,user:{id,handle,avatar,account_type,email_verified,onboarding_completed,location_permission}}` | User flags route to Onboarding / Allow Location / Home. Errors `401 INVALID_CREDENTIALS`, `403 EMAIL_NOT_VERIFIED`, `423 ACCOUNT_LOCKED`. |
| 2 | Sign In | Google / Apple | `POST /auth/social/*` | as A1.5 | as A1.5 | — |
| 3 | Forgot Pass | email + submit | `POST /auth/forgot-password` | `{email}` | `{message,otp:{...}}` | Always 200 (no enumeration). Goes to OTP (`purpose:"password_reset"`). |
| 4 | OTP | 6 digits | `POST /auth/otp/verify` | `{email,otp,purpose:"password_reset"}` | `{reset_token,reset_token_expires_in_seconds}` | Yields a short-lived reset token (600 s). |
| 5 | Create New Password | password + retype | `POST /auth/reset-password` | `{reset_token,password,password_confirmation}` | `{password_reset:true}` | Revokes sessions; back to Sign In ⚠. Errors `410 RESET_TOKEN_EXPIRED`, `422 PASSWORD_MISMATCH`. |

### A4. Discover a place (ask the assistant)

| # | Screen | Action | Endpoint | Request | Response | Description |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Home | open | `GET /home?lat=&lng=` | — | `{location:{city},time_of_day,prompt_suggestions:[{id,text}],active_visit}` | Header "Kansas City", "Good Afternoon", TRY ASKING chips; `active_visit` drives the YOU'RE VISITING card. |
| 2 | Home | type / tap chip | `POST /chat/sessions` | `{text, suggestion_id?, location?}` | `{session_id,messages:[user,assistant]}` | Starts a search. Assistant may ask a follow-up ("How far are you willing to travel?"). |
| 3 | Chat | tap "Walking Distance" | `POST /chat/sessions/{id}/messages` | `{quick_reply_id}` | `{message:{id,role:"assistant",text,quick_replies[],created_at},results}` | `quick_replies` empty when `results` are returned. |
| 4 | Result | view cards | (same response) | — | `results:{summary_text,places:[PlaceCard]}` | Cards show Right Now, Reliability, distance, Updated, Why this fits, Open, Live Biz Update, Event, tags. |
| 5 | Result | NOT RIGHT NOW → reason | `POST /chat/sessions/{id}/places/{pid}/not-right-now` | `{reason, other_text?}` | `{removed_place_id,results}` | Re-ranks results for this search only. Reasons: Too Crowded/Loud/expensive/far/Not the right atmosphere/Changed my mind/Other. |
| 6 | Result | WHY NOT HERE? | `GET /places/{id}/why-not-here?session_id=` | — | `{crowd,wait_minutes,parking,reliability:{level}}` | Quick comparison sheet (Parking may be `limited`). |
| 7 | Chat | scroll back | `GET /chat/sessions/{id}/messages?cursor=` | — | `{messages[],results}` | Message history, oldest first. |

### A5. Evaluate, compare and save

| # | Screen | Action | Endpoint | Request | Response | Description |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Details | View | `GET /places/{id}?lat=&lng=` | — | full place object (see spec §7.1) | Right Now, Opening Hours, Address, Last Updated, Live Biz Update, Event, Offer, first 3 Community Updates, `is_saved`, `is_following`. |
| 2 | Details | Reliability | `GET /places/{id}/reliability` | — | `{place_name,level,community_updates_count,live_biz_updates_count,most_recent_update_at,explanation}` | Design: Community Updates 4, Live Biz Updates 0, "24 min ago". |
| 3 | Details | see all updates | `GET /places/{id}/community-updates?cursor=&limit=` | — | `{items:[{id,author,text,posted_at,media[],signals?}], meta.next_cursor}` | Paged feed behind the section. |
| 4 | Compare | select + Compare 3 places | `GET /places/compare?ids=a,b[,c]&lat=&lng=` | — | `{places:[{id,name,environment,crowd,noise,wait_minutes,distance_m,parking,amenities[],updated_at,reliability}]}` | Rows: Environment/Crowd/Noise/Wait/Distance/Parking/Amenities/Freshness/Reliability. 2 or 3 ids ⚠. |
| 5 | Details/Card | save toggle | `PUT`/`DELETE /me/saved-places/{id}` | — | `{is_saved:true}` / `204` | Heart icon (also on Contribute cards). |
| 6 | Saved | open tab | `GET /me/saved-places?lat=&lng=&cursor=` | — | `{items:[PlaceCard],meta.next_cursor}` | Cards include Live Biz Update, Event, Reliability Low, Limited Current Information; Compare/View/NOT RIGHT NOW/WHY NOT HERE?. |

### A6. Navigate and arrive

| # | Screen | Action | Endpoint | Request | Response | Description |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Get Direction | directions | `GET /places/{id}/directions?origin_lat=&origin_lng=&mode=` | — | `{place,right_now,reliability,travel:{mode,distance_m,duration_minutes,arrive_at},route:{polyline,bounds}}` | "Arrive around 7:35 PM". |
| 2 | Start navigation | Start | `POST /navigation/sessions` | `{place_id,origin:{lat,lng},mode:"walking"}` | `{session_id,route,eta_minutes,remaining_distance_m,arrive_at,next_step:{instruction,maneuver,distance_m},place,right_now,reliability}` | "6 min", "1.5 km remaining", "IN 200M Turn Left". |
| 3 | Start navigation | position update | `PATCH /navigation/sessions/{id}` | `{location:{lat,lng},heading?}` | `{eta_minutes,remaining_distance_m,next_step,arrived,visit?}` | `arrived:true` (≤100 m ⚠) also returns `visit`. |
| 4 | Manual Arrival | confirm | `POST /visits` | `{place_id,method:"manual",location:{lat,lng},navigation_session_id?}` | `{visit:{id,place,arrived_at,verified,contribution_prompt_at}}` | `contribution_prompt_at` = arrived_at + 10 min. Errors `422 TOO_FAR_FROM_PLACE`, `409 VISIT_ALREADY_ACTIVE`. |
| 5 | Show it after 10 min | timer fires | — | — | — | **Client-side** timer from `contribution_prompt_at`; opens Community Update with `visit_id`. |
| 6 | Home after visit | open Home | `GET /home` | — | `active_visit:{visit_id,place:{id,name,city}}` | YOU'RE VISITING card. |
| — | (end) | — | `PATCH /visits/{id}/end` ⚠ | — | `{visit:{...,ended_at}}` | Marks visit finished. |

### A7. Contribute an update

| # | Screen | Action | Endpoint | Request | Response | Description |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Contribute | open tab | `GET /contribute/places?lat=&lng=&limit=` | — | `{items:[PlaceCard]}` | Ordered by how much each place needs a fresh update; cards show Open, Reliability, Why this fits, View Details, Update this place, save. |
| 2 | Community Update | Add Photo/Video | `POST /media/uploads` then `POST /media/{id}/complete` | `{kind,content_type,size_bytes}` / — | `{media_id,upload_url,upload_headers,expires_in_seconds}` then `{status,thumbnail_url}` | Presigned direct-to-storage upload. |
| 3 | Community Update | pick chips/note | — | — | — | **Client-side** until Review. |
| 4 | Review | submit | `POST /places/{id}/community-updates` | `{visit_id?,crowd?,noise?,wait?,parking?,atmosphere[]?,note?,media_ids[]?}` | `{update:{id,status:"submitted",created_at},reward:{xp_awarded,xp_total,level,credibility:{level},new_badges[]}}` | At least one signal required. Errors `422 NO_SIGNAL_PROVIDED`, `429 UPDATE_COOLDOWN`, `422 MEDIA_NOT_READY`. |
| 5 | Updated | show success | — | — | — | **Client-side** uses `reward.xp_awarded` ("+50 XP"). |
| 6 | After click done | follow? | `PUT /places/{id}/follow` | `{notifications:true}` | `{following:true}` | "Want updates from The Garden Kitchen?"; `DELETE` to unfollow (204). |

### A8. Profile and account

| # | Screen | Action | Endpoint | Request | Response | Description |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Profile | open | `GET /me/profile` | — | `{handle,avatar,xp,level,credibility:{level},next_level_xp?}` | XP 1,240, LVL 7, Credibility High. |
| 2 | Badges ⚠ | open | `GET /me/badges` | — | `{items:[{id,name,description,icon_url,earned,earned_at,progress}]}` | Screen frame is empty/hidden → shape assumed. |
| — | Rewards ⚠ | open | `GET /me/rewards` | — | `{items:[{id,title,description,xp_required,unlocked}]}` | Listed on Profile; no frame. |
| 3 | Favourites ⚠ | open | `GET /me/favourites` | — | alias of saved places | Confirm alias vs separate list. |
| 4 | Notifications ⚠ | open | `GET /me/notifications?cursor=` | — | `{items:[{id,type,title,body,place_id,created_at,read}],meta.unread_count}` | `POST /me/notifications/read {ids[]|all}`. |
| 5 | Personal Information | edit email | `PATCH /me` | `{email}` | `202 {pending_email,otp}` then `POST /me/email/verify {otp}` → `200 {email}` | Errors `409 EMAIL_ALREADY_REGISTERED`, `400 OTP_INVALID`. |
| 6 | Change Password | 3 fields | `POST /me/password` | `{current_password,new_password,new_password_confirmation}` | `204` | Errors `400 CURRENT_PASSWORD_INCORRECT`, `422 PASSWORD_MISMATCH`. |
| 7 | Location Permission | change | `PUT /me/location-permission` | `{status}` | `{status}` | — |
| 8 | Privacy Policy / Terms | open | `GET /content/legal/{slug}` | — | `{slug,title,version,updated_at,body_markdown}` | `slug` = `privacy-policy` \| `terms`. |
| 9 | Settings ⚠ | change | `GET`/`PATCH /me/settings` | `{distance_unit?,language?}` | settings | No Settings frame; shape assumed. |
| 10 | Account Deletion | confirm | `POST /me/account-deletion` | `{password,reason?}` | `202 {scheduled_for}` | 30-day grace ⚠; `DELETE /me/account-deletion` cancels. |
| 11 | Log Out | tap | `POST /auth/logout` | `{refresh_token,all_devices?}` | `204` | — |
| — | (session) | token refresh | `POST /auth/refresh` | `{refresh_token}` | `{access_token,refresh_token,expires_in}` | Error `401 REFRESH_TOKEN_INVALID`. |

---

## Part 2 — Business portal workflows

Portal conventions: no Register; a portal user is a `provider` account. Roles `owner`/`staff`.
Everything is under `/business`. Until `claim_status = approved`, only auth + claim + `/business/me`
work; others return `403 CLAIM_NOT_APPROVED`. Staff limited to assigned modules (`403 FORBIDDEN_MODULE`).

### P1. Sign in and recover

| # | Screen | Action | Endpoint | Request | Response | Description |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Log in | email+password | `POST /business/auth/login` | `{email,password}` | `{access_token,refresh_token,expires_in,user:{id,name,email,role,permissions[]},business:{location_id,name,label,claim_status}}` | `claim_status` routes: `none`→Claim, `draft`→resume, `pending_review`→status, `approved`→Overview. Errors `401`, `403 ACCOUNT_NOT_PROVIDER`, `423`. |
| 2 | Forgot Password | email | `POST /business/auth/forgot-password` | `{email}` | `{message,otp:{...}}` | Always 200. |
| 3 | Verify Email | 6 digits / Resend | `POST /business/auth/otp/verify` \| `/resend` | `{email,otp,purpose:"password_reset"}` | `{reset_token,reset_token_expires_in_seconds}` | — |
| 4 | Create New Password | new+confirm | `POST /business/auth/reset-password` | `{reset_token,password,password_confirmation}` | `{password_reset:true}` | — |
| — | (session) | refresh / logout | `POST /business/auth/refresh` \| `/logout` | — | as app | No logout control drawn ⚠. |

### P2. Claim your business

| # | Screen | Action | Endpoint | Request | Response | Description |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Claim your business | Start | `GET /business/claims/current` | — | `{claim|null}` | If null, continue. |
| 2 | Find your business | search | `GET /business/locations/search?q=` | — | `{items:[{id,name,address_line,claimable}]}` | — |
| 3 | Is this your business? | confirm | `POST /business/claims` | `{location_id}` | `201 {claim:{id,status:"draft",location}}` | Errors `409 LOCATION_ALREADY_CLAIMED`, `409 CLAIM_ALREADY_OPEN`. |
| 4 | Verify Your Business | form load | `GET /business/claims/{id}/requirements` | — | `{fields:[{key,label,placeholder,type,required,validation}],documents:[{key,label,accepted_formats[],max_size_mb,required}]}` | Admin-configured; client renders dynamically. |
| 5 | Verify Your Business | Upload File | `POST /business/uploads` + `/complete` | `{purpose:"claim_document",file_name,content_type,size_bytes}` | `{upload_id,upload_url,upload_headers,expires_in_seconds}` → `{status:"ready",file_name,size_bytes}` | — |
| 6 | Verify Your Business | Continue | `PUT /business/claims/{id}/verification` | `{fields:{key:value},documents:{key:upload_id}}` | claim + review summary | Errors `422 VERIFICATION_FIELDS_INVALID`, `422 DOCUMENT_REQUIRED`. |
| 7 | Review your claim | open | `GET /business/claims/{id}` | — | `{claim:{id,status,location,fields[],documents[]}}` | — |
| 8 | Review your claim | Submit | `POST /business/claims/{id}/submit` | — | `202 {status:"pending_review",submitted_at,message}` | Error `422 CLAIM_INCOMPLETE`. |
| 9 | Claim Submitted | Done | — | — | — | Static confirmation. |
| 10 | Claim status | later login | `GET /business/claims/current` | — | claim with `status`, `message` | Modules locked until `approved`; admin decision has no screen ⚠. |

### P3. Daily dashboard

| # | Screen | Action | Endpoint | Request | Response | Description |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Overview | open | `GET /business/overview` | — | `{business,right_now,followers:{total,new_this_week},recent_engagement[],upcoming_scheduled_posts[],need_attention[]}` | Right Now incl. parking "Not Applicable"; Total Followers 12,450. |
| 2 | Overview | View All | `GET /business/community-updates?cursor=&limit=` | — | `{items:[{id,author,text,posted_at,signals?,media[]}]}` | Paged recent engagement. |
| 3 | Overview | open report | `GET /business/reports/{id}` | — | `{report:{...content,reason,disputable}}` | From Need Attention. |

### P4. Post a live update

| # | Screen | Action | Endpoint | Request | Response | Description |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Live Biz Updates | open | `GET /business/live-updates?status=&cursor=` | — | `{items:[{id,text,category,status,published_at,scheduled_for,expires_at,created_by}]}` | Rows show Environment Status + Published/Expired. |
| 2 | Create Update | save | `POST /business/live-updates` | `{text,publish_mode,scheduled_for?,expires_at?,notify_followers?}` | `201 item` | Errors `422 VALIDATION_FAILED`, `422 EXPIRY_BEFORE_PUBLISH`. |
| 3 | Live Biz Updates | expiry | — | — | — | Server flips `status`→`expired`. |
| — | Live Biz Updates | end early ⚠ | `DELETE /business/live-updates/{id}` | — | `204` | No action drawn. |

### P5. Plan promotions

| # | Screen | Action | Endpoint | Request | Response | Description |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Promotions Calendar | load month/week | `GET /business/calendar?from=&to=` | — | `{items:[{id,type,title,starts_at,status,offer_id}]}` | View switch + Today are client-side. |
| 2 | Schedule Item | schedule | `POST /business/calendar/items` | `{item_type,title,publish_mode,scheduled_at?,notify_followers?}` | `201 item` | Item Type: Update/Special/Event/Coupon/Reminder. |
| — | Calendar | cancel ⚠ | `DELETE /business/calendar/items/{id}` | — | `204` | — |

### P6. Create an offer

| # | Screen | Action | Endpoint | Request | Response | Description |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Offers | open | `GET /business/offers?status=&cursor=` | — | `{items:[{id,title,terms,status,valid_from,valid_to,code,quantity_limit,quantity_available}]}` | "20% off Dinner", code SWEET, "50 available". |
| 2 | Create Offer | Save | `POST /business/offers` | `{title,terms,publish_mode,valid_from,valid_to?,code,quantity_limit?,notify_followers?}` | `201 offer` | Errors `409 CODE_ALREADY_USED`, `422 VALIDITY_RANGE_INVALID`. |
| — | Offers | edit/end ⚠ | `PATCH`/`DELETE /business/offers/{id}` | — | offer / `204` | — |

### P7. Review analytics

| # | Screen | Action | Endpoint | Request | Response | Description |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Analytics | open / change range | `GET /business/analytics?range=30d` | — | `{follower_growth,notifications,post_engagement,coupons,community_updates,aggregate}` | Six charts + aggregate cards; axis D1–D30. Range 7d/30d/90d ⚠. |

### P8. Edit the public profile

| # | Screen | Action | Endpoint | Request | Response | Description |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Location Profile | open | `GET /business/location-profile` | — | `{hours[],contact,links,description,gallery[]}` | Hours/Contact/Links/Description/Gallery. |
| 2 | Edit Profile | Save Changes | `PUT /business/location-profile` | `{hours[],contact,links,description}` | updated profile | Error `422 ADDRESS_CHANGE_REQUIRES_REVIEW` ⚠. |
| 3 | Edit Profile | Add Media | `POST /business/uploads` + `POST /business/location-profile/gallery` | `{upload_id}` | media `status:"pending_review"` | Gallery approves later. |
| — | Edit Profile | remove media | `DELETE /business/location-profile/gallery/{media_id}` | — | `204` | — |

### P9. Manage the team

| # | Screen | Action | Endpoint | Request | Response | Description |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Team | open | `GET /business/team` | — | `{members:[{id,name,email,status,role,permissions[]}]}` | Owner-only ⚠. |
| 2 | Team | Invite Staff | `POST /business/team/invitations` | `{email,name?,permissions[]?}` | member `status:"invited"` | Error `409 ALREADY_A_MEMBER`. |
| 3 | Assign Permissions | Save | `PUT /business/team/{id}/permissions` | `{permissions[]}` | member | Modules: Overview, Live Biz Updates, Promotions Calendar, Offers/Coupons, Analytics, Location Profile, Reports. |
| — | Team | remove ⚠ | `DELETE /business/team/{id}` | — | `204` | Actions column. |

### P10. Handle a report or dispute

| # | Screen | Action | Endpoint | Request | Response | Description |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Reports (`808:9892`) | open | `GET /business/reports?status=&cursor=` | — | `{items:[{id,affected_content:{type,label,excerpt},reason,status,reported_at}]}` | Table columns: AFFECTED CONTENT / REASON / STATUS. Header "STAFF MEMBER" is a copy error ⚠. |
| 2 | Disputes | open | `GET /business/disputes?cursor=` | — | `{items:[{id,subject,related_content,details,submitted_at,status}]}` | "In Progress" etc. |
| 3 | Submit dispute modal | Submit Request | `POST /business/disputes` | `{subject,related_content?,report_id?,details}` | `201 dispute` | — |

### P11. Settings

| # | Screen | Action | Endpoint | Request | Response | Description |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Settings — My Profile | Save Changes | `GET`/`PATCH /business/me` | `{name,restaurant_name,phone,email}` | `{name,restaurant_name,email,phone,role}` | Email change triggers OTP ⚠. |
| 2 | Settings — Security | Update Password | `POST /business/me/password` | `{current_password,new_password,password_confirmation}` | `204` | — |
| 3 | Settings — Security | legal tabs | `GET /content/legal/{slug}` | — | legal body | Same as app §14. |

---

## Part 3 — How the two sides connect

| Flow | Portal action | App effect |
| --- | --- | --- |
| Live update | `POST /business/live-updates` | `live_biz_update` on Details + cards; `live_biz_updates_count` on Reliability |
| Offer | `POST /business/offers` | `offer` in Details |
| Event | calendar item `type:"event"` | `event` in Details + cards |
| Followers/push | Overview count, Analytics Delivered/Opened | `PUT /places/{id}/follow` |
| Community → business | Overview Recent Engagement, Analytics volume | `POST /places/{id}/community-updates` |
| Provider onboarding | Claim flow | Identity ready → Continue as Provider |
| Reported content | Reports/Disputes | **no app report endpoint yet** ⚠ |
| Coupon redemption | Analytics Redemptions | **no app redeem endpoint yet** ⚠ |
