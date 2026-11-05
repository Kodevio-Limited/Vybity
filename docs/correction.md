remove this section:
4. Content & onboarding
Method	Path	Auth	Purpose
GET	/content/onboarding	Public	{slides[]}
PUT	/me/onboarding	User	`{status:"completed"
GET	/content/legal/{slug}	Public	{slug,title,version,updated_at,body_markdown}

5. location:
here you will take location as latitute & altitute. as well as in gps formate or string how things are standard

6:
remove this api:
POST	/chat/sessions/{id}/places/{pid}/not-right-now	{reason,other_text?} → {removed_place_id,results}

7. one queestion here:
in this url why you puted id?
	/places/{id}?lat=&lng=



GET	/places/{id}/community-updates?cursor=&limit=
question: what does curson means here?



Note:
there should be another dashboard for admin. who will approve each business panel. and he will confrol few stuff.

& i don't think there that many api needed:
7. Places
Method	Path	Purpose
GET	/places/{id}?lat=&lng=	full Details object (name, distance, reliability, opening_hours, address, last_updated_at, right_now, live_biz_update, event, offer, community_updates[first 3], is_saved, is_following)
GET	/places/{id}/community-updates?cursor=&limit=	{items:[{id,author,text,posted_at,media[],signals?}],meta}
GET	/places/{id}/reliability	{place_name,level,community_updates_count,live_biz_updates_count,most_recent_update_at,explanation}
GET	/places/{id}/why-not-here?session_id=	{crowd,wait_minutes,parking,reliability:{level}}




9. Directions & navigation
Method	Path	Purpose
GET	/places/{id}/directions?origin_lat=&origin_lng=&mode=	{place,right_now,reliability,travel:{mode,distance_m,duration_minutes,arrive_at},route:{polyline,bounds}}
POST	/navigation/sessions	{place_id,origin,mode} → 201 {session_id,route,eta_minutes,remaining_distance_m,arrive_at,next_step,place,right_now,reliability}
PATCH	/navigation/sessions/{id}	{location,heading?} → {eta_minutes,remaining_distance_m,next_step,arrived,visit?}
DELETE	/navigation/sessions/{id}	204

here i have few queswtion. how will you find route, eta_minutes?? do we will have info about all road? how will you measere that? in frontend it can be done? or its professional to handle all the stuff in our backedn or i should use map api key. i want things accurate.






11. Contribute
Method	Path	Purpose
GET	/contribute/places?lat=&:lng=&limit=	{items:[PlaceCard]}
POST	/media/uploads	{kind,content_type,size_bytes} → {media_id,upload_url,upload_headers,expires_in_seconds}
POST	/media/{id}/complete	→ {status,thumbnail_url}
POST	/places/{id}/community-updates	{visit_id?,crowd?,noise?,wait?,parking?,atmosphere[]?,note?,media_ids[]?} → 201 {update,reward}
PUT/DELETE	/places/{id}/follow	{notifications} → {following:true} / 204
At least one signal required. Errors 422 NO_SIGNAL_PROVIDED/MEDIA_NOT_READY, 429 UPDATE_COOLDOWN. reward = {xp_awarded,xp_total,level,credibility:{level},new_badges[]}.


does here slll the api needed based on ui?


12. Saved
Method	Path	Purpose
GET	/me/saved-places?lat=&lng=&cursor=&limit=	{items:[PlaceCard],meta}
PUT/DELETE	/me/saved-places/{id}	{is_saved:true} / 204

better to saved based on id, alsongside lat & long 


like each place will have a place id. under plaace id there should be a location_id. in that location_id thre should be location_name & and lat, lang  wait



13. here lots apis they don't need:

13. Profile & account
Method	Path	Purpose
GET	/me/profile	{handle,avatar,xp,level,credibility,next_level_xp?}
GET	/me/badges ⚠	{items:[{id,name,description,icon_url,earned,earned_at,progress}]}
GET	/me/rewards ⚠	{items:[{id,title,description,xp_required,unlocked}]}
GET	/me/favourites ⚠	alias of saved
GET	/me/notifications?cursor=	{items[],meta.unread_count}
POST	/me/notifications/read	`{ids[]
GET/PATCH	/me/notification-settings ⚠	{push_enabled,place_updates,community_updates,rewards}
POST	/me/devices	{platform,push_token}
GET/PATCH	/me	profile; email change → 202 {pending_email,otp}
POST	/me/email/verify	{otp} → 200 {email}
POST	/me/password	{current_password,new_password,new_password_confirmation} → 204
GET/PATCH	/me/settings ⚠	{distance_unit,language}
POST/DELETE	/me/account-deletion	{password,reason?} → 202 {scheduled_for} / cancel
Errors: 400 CURRENT_PASSWORD_INCORRECT/OTP_INVALID, 409 EMAIL_ALREADY_REGISTERED, 422 PASSWORD_MISMATCH/PASSWORD_REUSED⚠.

keep just requred endpontss


we don't need to warry about splash screen or onbording thats totally controlled by frontend


