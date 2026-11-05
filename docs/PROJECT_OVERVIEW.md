# Vybity — Project Overview

> Derived **independently of the existing API spec**, by walking every frame in the Figma file
> `SbnpZCb45gYYPhmshOWC8D` (page **App Ui** `0:1`, page/section **Business Portal** `671:2529`).
> Purpose: list every screen, every input, every piece of displayed data, and every user action,
> so the API surface can be derived screen-first. Cross-check against the spec is in
> `ASSUMPTIONS_AND_QUESTIONS.md`.

---

## 1. What the product is


Vybity basically an ai application . here user can talk to ai. & ai give user suggesstion about place, and stuff trhat user need. and let user interact with places.
Vybity tells a person **what a physical place is like right now, before they go**, from conditions
contributed by people who were actually there. It launches in Kansas City. The user talks to the app,
not to other people. Data describes the **place** (crowd, noise, wait, parking, atmosphere), never
other people. Public identity is system-generated (handle + avatar).

Three surfaces:

| Surface | Users | In this Figma file? | Screens |
| --- | --- | --- | --- |
| Consumer mobile app | Visitors | Yes — `App Ui` (`0:1`) | 47 real screens + 7 section labels |
| Business portal | Owners & staff | Yes — `Business Portal` (`671:2529`) | 27 screens (incl. modals) |
| Admin / moderation | Vybity staff | **No screens** | n/a |

**Proposed stack (from the brief, not visible in Figma):** Flutter (app), Next.js with tailwine css for admin dashbaord & business portal. & django(backend)

---

## 2. Screen inventory (every frame found)

### 2.1 Consumer app — `App Ui` (`0:1`)

Frames listed in canvas order. Node IDs are Figma IDs.

| # | Screen (Figma frame) | Node ID | Group |
| --- | --- | --- | --- |
| 1 | Splash 2 | `390:113` | Splash (static, images only) |
| 2 | Splash 3 | `390:152` | Splash (static, images only) |
| 3 | Onboarding 4 | `78:561` | Onboarding |
| 4 | Onboarding 5 | `78:859` | Onboarding |
| 5 | Onboarding 6 | `78:899` | Onboarding |
| 6 | Sign In | `74:518` | Auth |
| 7 | Register | `76:32` | Auth |
| 8 | Forgot Pass | `76:173` | Auth |
| 9 | OTP | `76:409` | Auth |
| 10 | Create New Password | `76:475` | Auth |
| 11 | Identity ready | `208:336` | Auth / identity |
| 12 | Allow Location | `847:961` | Location |
| 13 | Add address manually | `847:865` | Location |
| 14 | Home | `225:1012` | Home |
| 15 | Chat | `225:1209` | Home / chat |
| 16 | Result | `139:1912` | Discover |
| 17 | After click Compare | `251:1502` | Compare (selection mode) |
| 18 | Compare | `251:1871` | Compare |
| 19 | NOT RIGHT NOW | `287:183` | Discover (bottom sheet) |
| 20 | Why not here? | `293:323` | Discover (bottom sheet) |
| 21 | Reliability | `380:661` | Place (bottom sheet) |
| 22 | Details | `847:474` | Place |
| 23 | Details (2nd copy) | `631:517` | Place |
| 24 | Get Direction | `470:199` | Navigation |
| 25 | Start navigation | `479:1076` | Navigation |
| 26 | Manual Arrival | `522:375` | Visit |
| 27 | Show it after 10 min | `518:74` | Visit (OS/banner prompt) |
| 28 | Home after visit the restaurant | `536:394` | Home (active visit) |
| 29 | Contribute | `641:1072` | Contribute |
| 30 | Community Update | `847:149` | Contribute |
| 31 | Community Update (2nd copy) | `747:263` | Contribute |
| 32 | Review | `847:368` | Contribute (client preview) |
| 33 | Review (2nd copy) | `763:664` | Contribute (client preview) |
| 34 | Updated | `561:1969` | Contribute (success) |
| 35 | Updated (2nd copy) | `641:1009` | Contribute (success) |
| 36 | After click done | `771:782` | Follow prompt |
| 37 | After click done (2nd copy) | `847:760` | Follow prompt |
| 38 | Saved | `631:97` | Saved |
| 39 | Profile | `849:1094` | Profile |
| 40 | Personal Information | `666:2257` | Profile |
| 41 | Change Password | `666:2385` | Profile |
| 42 | Badges | `1009:254` | Profile (**hidden**, empty) |
| 43 | Privacy Policy | `666:2455` | Legal |
| 44 | Terms & conditions | `666:2478` | Legal |
| 45 | `6337000920293511251 2` | `1229:156` | Stray component (no content) |

Section headers (text labels only, no UI): Authentication `74:491`, Onboarding `78:556`, Home `90:234`,
Saved `620:484`, Contribute `641:731`, Profile `641:1430`, Splash `90:77`.

> **Note:** there is **no "Splash 1"** frame — only Splash 2 and Splash 3. (The spec says "Splash 1–3".)

### 2.2 Business portal — `Business Portal` (`671:2529`)

| # | Screen (Figma frame) | Node ID | Type |
| --- | --- | --- | --- |
| 1 | Log in | `820:10473` | Auth (1920px) |
| 2 | Forgot Password | `820:10505` | Auth |
| 3 | Verify Email | `820:10524` | Auth |
| 4 | Create New Password | `820:10553` | Auth |
| 5 | Claim your business | `868:1605` | Claim |
| 6 | Find your business | `868:1645` | Claim |
| 7 | Is this your business? | `868:1765` | Claim |
| 8 | Verify Your Business | `869:1840` | Claim |
| 9 | Review your claim | `871:1920` | Claim |
| 10 | Claim Submitted | `871:1978` | Claim |
| 11 | Calim status (Claim status) | `871:2023` | Claim |
| 12 | Overview | `925:328` | Dashboard |
| 13 | Live Biz Updates | `691:3361` | Dashboard |
| 14 | Promotions Calendar | `700:3748` | Dashboard |
| 15 | Offers | `710:5700` | Dashboard |
| 16 | Analytics | `702:5049` | Dashboard |
| 17 | Location Profile | `791:8570` | Dashboard |
| 18 | Location Profile (edit Profile) | `799:8923` | Dashboard |
| 19 | Team | `801:9377` | Dashboard |
| 20 | Team *(actually Reports + Disputes)* | `808:9892` | Dashboard |
| 21 | Settings — My Profile | `927:610` | Dashboard |
| 22 | Settings — Security | `951:642` | Dashboard |
| 23 | Create Update (modal) | `698:3585` | Modal |
| 24 | Create Offer (modal) | `710:5618` | Modal |
| 25 | Schedule Item (modal) | `702:4841` | Modal |
| 26 | Assign Permissions (modal) | `807:9668` | Modal |
| 27 | Submit dispute / support request (modal) | `817:10251` | Modal |

> **Finding:** The frame named **`Team` (`808:9892`)** actually contains the **Reports + Disputes**
> screen. There are two frames named `Team`; the real Team screen is `801:9377`. There is **no frame
> named "Reports"**, but the Reports/Disputes UI does exist here.

---

## 3. Per-screen inputs and displayed data (screen-first derivation)

Legend: **IN** = input the user supplies · **OUT** = data the screen displays (must come from an API) ·
**ACT** = user action (navigation is client-side unless noted).

### 3.1 Splash & Onboarding

| Screen | IN | OUT | ACT |
| --- | --- | --- | --- |
| Splash 2 `390:113` | — | image only (no text) | auto-advance |
| Splash 3 `390:152` | — | image only | auto-advance |
| Onboarding 4 `78:561` | swipe | title "Know Before You Go", body text, image, "Skip" | Skip / swipe |
| Onboarding 5 `78:859` | swipe | title "Information from the people and places that know", body | Skip / swipe |
| Onboarding 6 `78:899` | swipe | title "Explore freely. Stay anonymous.", body | Skip / swipe → Auth |

No API needed for Splash. Onboarding copy is static (endpoint optional).

### 3.2 Authentication & identity

| Screen | IN | OUT | ACT |
| --- | --- | --- | --- |
| Sign In `74:518` | Email, Password | — | Sign In; Forgot password?; Google; Apple; "Register" link |
| Register `76:32` | Email, Password, Retype Password, **Check box** (terms) | — | Register; Google; Apple; "Sign in" link |
| Forgot Pass `76:173` | Email | — | submit → OTP |
| OTP `76:409` | 6 digits (Verify OTP) | "We've sent 6 digits code on your email" | Verify; "Send again" |
| Create New Password `76:475` | Password, Retype Password | — | submit |
| Identity ready `208:336` | — | `@city-guide22`, 3D avatar | **Continue as Provider** (button carries a refresh icon), **Continue** |

> The refresh icon sits **inside the "Continue as Provider" button**, next to its label — it does not
> read as a standalone "regenerate my identity" control. Flagged for confirmation.

### 3.3 Location

| Screen | IN | OUT | ACT |
| --- | --- | --- | --- |
| Allow Location `847:961` | Allow / Deny (buttons) | rationale copy ("Vybity uses your location…") | allow → GPS; deny → manual |
| Add address manually `847:865` | "Address Details" text | results list (**Kansas City** shown) | pick result |

### 3.4 Home & Chat

| Screen | IN | OUT | ACT |
| --- | --- | --- | --- |
| Home `225:1012` | free-text "Tell me what you're looking for…" | header "Kansas City", "Good Afternoon", "How can I help?", suggestion chips: *Somewhere quiet for dinner / What's good right now? / Something fun tonight / Somewhere relaxed nearby* | tap chip or type → Chat |
| Chat `225:1209` | text, quick-reply **Walking Distance** | user msg "I want to go for a dinner"; assistant "Got it. How far are you willing to travel?" | send / tap chip |
| Home after visit `536:394` | — | **YOU'RE VISITING** card: "Garden Kitchen", buttons **View Details**, **Update this Place**; same chips | open details / contribute |

### 3.5 Discover, Compare, Place

| Screen | IN | OUT | ACT |
| --- | --- | --- | --- |
| Result `139:1912` | — | assistant summary "I found a few places…"; place cards with: name (The Garden Kitchen), Reliability High/Low, distance 1.5 km, Updated 3 mins ago, Right Now (Quiet / 10 min wait / Moderate Crowd), Why this fits, **Open** chip, Live Biz Update (**"Outdoor seating is currently available"**, 3 mins ago), Event (**Live Music, Today at 8 PM**), tags **Needs Update** / **Limited Current Information** | Compare, View, NOT RIGHT NOW, WHY NOT HERE? |
| After click Compare `251:1502` | card selection | same cards + **Cancel**, **Compare 3 places** | select → Compare |
| Compare `251:1871` | — | rows: Environment (Outdoors/Indoors), Crowd (Moderate/Busy), Noise (Quiet/Loud), Wait (10 min / 25 min), Distance (1.2 km / 2.4 km), Parking (Easy/Limited), Amenities (Patio, WiFi / Bar, WiFi), Freshness (8 min / 4 min ago), Reliability (High); columns Garden Kitchen, Downtown Bistro | back |
| NOT RIGHT NOW `287:183` | reason (Too Crowded / Too Loud / Too expensive / Too far / Not the right atmosphere / Changed my mind / Other) | "This helps refine results for this search only." | pick reason |
| Why not here? `293:323` | — | Crowd Moderate, Wait 10 min, Parking **Limited**, Reliability High | close |
| Reliability `380:661` | — | Garden Kitchen, **High**, Community Updates **4**, Live Biz Updates **0**, "Most recent update: 24 min ago", explanation text | close |
| Details `847:474` / `631:517` | — | name, 1.5 km, Reliability High, Opening Hours 8:00 AM–10:00 PM, Address Kansas City, Last Updated 8 mins ago, Right Now (Quiet / 10 min wait / Moderate Crowd), Live Biz Update, Event (Live Music, Today–8:00 PM), Offer (**"20% off dinner"** / "Enjoy 20% off selected dinner items tonight."), Community Update feed (@city-guide22, 3 min, quoted text + avatars) | View, Reliability, follow, save |

### 3.6 Navigation, visit, contribute, follow

| Screen | IN | OUT | ACT |
| --- | --- | --- | --- |
| Get Direction `470:199` | — | Garden Kitchen, Kansas City, Right Now, Reliability High, **Arrive around 7:35 PM** | start |
| Start navigation `479:1076` | — | "6 min", "1.5 km remaining", "IN 200M", "Turn Left", Arrive around 7:35 PM | drive (live position) |
| Show it after 10 min `518:74` | — | "You're at the Garden Kitchen — What's it like there right now?" | tap → Community Update |
| Manual Arrival `522:375` | confirm | "Confirm you're here… At The Garden Kitchen?" | confirm |
| Contribute `641:1072` | — | place card: Garden Kitchen, Reliability High, 1.5 km, Updated 3 mins ago, Right Now, Why this fits, Open, save-heart; buttons **View Details**, **Update this place** | open / update |
| Community Update `847:149` / `747:263` | Crowd (Light/Moderate/Busy/Packed), Noise (Quiet/Moderate/Loud), Wait (None/5 min/10 min/20+ min), Parking (Easy/Moderate/Difficult/Full/Not Applicable), Atmosphere (Cozy/Lively/Romantic/Casual/Trendy/Loud Music), Note ("Tell others what you noticed…"), Media ("Add Photo or Video") | — | next → Review |
| Review `847:368` / `763:664` | (preview of above) | Garden Kitchen, Reliability High "Based on current sources", **Your Update**: Crowd Moderate, Noise, Atmosphere Cozy, Wait 10 min, Parking Difficult, Your Note "Somewhere relaxed nearby" | submit |
| Updated `561:1969` / `641:1009` | — | "Update Submitted", "Thanks for helping others know before they go.", **+50 XP** | done |
| After click done `771:782` / `847:760` | follow yes/no | "Want updates from The Garden Kitchen? We'll let you know when things change…" | confirm/deny |

### 3.7 Saved & Profile

| Screen | IN | OUT | ACT |
| --- | --- | --- | --- |
| Saved `631:97` | — | list of place cards (same card shape as Result) incl. Live Biz Update, Event, Reliability Low, **Limited Current Information**, **OFFFER** label; buttons Compare / View / NOT RIGHT NOW / WHY NOT HERE? | open |
| Profile `849:1094` | — | @city-guide22, avatar, XP 1,240, CREDIBILITY High, LEVEL LVL 7; rows: **Badges, Rewards**; YOUR ACTIVITY: **Favourites, Notifications**; ACCOUNT: Personal Information (subtitle "Profile management"), Change Password, Location Permission, Privacy Policy, Settings, Account Deletion, Log Out | navigate |
| Personal Information `666:2257` | Email ("Enter your email…") | — | save |
| Change Password `666:2385` | Current Password, New Password, Retype Password | — | save |
| Badges `1009:254` | — | (frame hidden/empty) | — |
| Privacy Policy `666:2455` | — | (title only; body from CMS) | — |
| Terms & conditions `666:2478` | — | (title only) | — |

### 3.8 Business portal screens

| Screen | IN | OUT | ACT |
| --- | --- | --- | --- |
| Log in `820:10473` | Email, Password | — | Sign In; Forgot password? |
| Forgot Password `820:10505` | Email | — | submit |
| Verify Email `820:10524` | Verification Code (6 digits) | "We sent a 6-digit code to your email" | verify; Resend Code |
| Create New Password `820:10553` | New Password, Confirm Password | — | submit |
| Claim your business `868:1605` | — | explanatory copy | Start |
| Find your business `868:1645` | search by business name | result: The Garden Kitchen, 214 Willow Ave, Dhaka | select |
| Is this your business? `868:1765` | confirm | The Garden Kitchen, 214 Willow Ave, Dhaka | confirm |
| Verify Your Business `869:1840` | configurable verification fields, configurable document upload | "CONFIGURABLE VERIFICATION FIELD", "CONFIGURABLE DOCUMENT UPLOAD", "Format as configured by system" | upload; Continue |
| Review your claim `871:1920` | — | The Garden Kitchen, address, entered fields, uploaded docs | Submit |
| Claim Submitted `871:1978` | — | "Your business claim has been submitted successfully…" | Done |
| Claim status `871:2023` | — | The Garden Kitchen, 214 Willow Ave Dhaka, **Pending Review**, review copy | — |
| Overview `925:328` | — | Right Now (Quiet / 10 min wait / Moderate Crowd / **Not Applicable**), **Total Followers 12,450**, "+5 new this week", Recent Engagement (3 items: @city-guide22, 3 min, quoted), Upcoming Scheduled Posts ("Today, 4:00 PM / Happy Hour Specials Starting Soon! / Scheduled"), **Need Attention** ("Reported Content" ×3) | View All; open report |
| Live Biz Updates `691:3361` | — | rows: "Environment Status", Published/Expired, "Outdoor seating is currently available and heaters are on.", "Today, 2:14 PM" | Create Update |
| Promotions Calendar `700:3748` | month/week switch, Today | "September 2026", weekday grid, scheduled items | Schedule |
| Offers `710:5700` | — | "20% off Dinner", Terms "20% off selected dinner items for dine-in customers only.", Validity "Sep 2, 2026 - 5:00 PM To", Code **SWEET**, Quantity Limit "50 available" | Create Offer |
| Analytics `702:5049` | range (chart axis D1–D30) | Follower Growth "+589 · +15.5%", Notification Delivery & Opens "12,130 delivered · 3,047 opened · 25.1%", Post Engagement "Avg 679 views per post · 20,361 total", Coupon Views & Redemptions "5,837 views · 1,497 redemptions", Community Update Volume "617 · avg 21 per day", Aggregate Trends (Followers **4,389**, Notifications 12,130, Post Views 20,361, Coupon Views 5,837, Community Updates 617) | range switch |
| Location Profile `791:8570` | — | Hours (Mon–Sun, "Closed", 9:00 AM/5:00 PM etc.), Contact (Phone (415) 4579 83475, Email hello@example.com, Address 214 willow ave Dhaka), Links (Website, Instagram), Description, **Approved business gallery** | Edit Profile |
| Location Profile (edit) `799:8923` | editable Hours/Contact/Links/Description, Add Media | same | Save Changes; Cancel |
| Team `801:9377` | — | table **STAFF MEMBER / STATUS / ACTIONS**, Maya Chowdhury, maya@gmail.com, Active | Invite Staff |
| Team→Reports `808:9892` | — | "Reports affecting business content…", table **STAFF MEMBER / AFFECTED CONTENT / REASON / STATUS** (Live Biz Update, "Weekend brunch menu is back…", Reported as misleading, Pending Review / Resolved); "Disputes / support requests" (Dispute on branch menu report, related content, details, "Submitted Sep 3, 2026", In Progress) | Submit Dispute |
| Settings–My Profile `927:610` | Name (Sarah Jessie), Restaurant Name, Email, Phone Number (+00888888) | — | Save Changes |
| Settings–Security `951:642` | Current Password, New Password, Confirm Password | Privacy & Policy, Terms & Conditions links | Update Password |
| Create Update modal `698:3585` | Update (text), Publishing Options Publish Now/Schedule, Expiration Date 08/16/2026, Expiration Time 9:50 AM | — | Cancel; save |
| Create Offer modal `710:5618` | Title, Terms, Publishing Options, Validity Start/End Date+Time, Redemption Code SWEET | — | Cancel; Save |
| Schedule Item modal `702:4841` | Item Type (Update/Special/Event/Coupon/Reminder), Title, Publish Now, Date, Time | — | Cancel; Schedule |
| Assign Permissions modal `807:9668` | checkboxes: Overview, Live Biz Updates, Promotions Calendar, Offers/Coupons, Analytics, Location Profile, Reports | — | Cancel; Save |
| Submit dispute modal `817:10251` | Subject, Related Content (Optional), Details | — | Cancel; Submit Request |

---

## 4. The single workflow loop

Everything is one loop; the app and the portal are its two sides.

1. **Ask** — user opens Home, chats ("somewhere quiet for dinner"); app may ask one follow-up (distance).
2. **Get places** — app returns place cards with a Right Now line.
3. **Open a place** — Details shows Right Now, Live Biz Update, Event, Offer, Community Updates.
4. **Go** — start navigation; app detects arrival.
5. **Contribute** — after arriving, the app asks what it's like; user submits a Community Update, earns XP, is offered to follow the place.
6. **Loop closes** — the update feeds Right Now verdicts + Community Updates the next user sees, and feeds the business's Overview/Analytics.

Portal side: owner signs in → verifies → claims location → manages Live Biz Updates, Promotions Calendar, Offers, Location Profile, Team, Reports, Analytics.

**Content ownership**

| Content on Details | Written by | Endpoint area |
| --- | --- | --- |
| Right Now verdicts | Computed from contributions | app §7 |
| Community Updates | Visitors present at the place | app §11 |
| Live Biz Update | Business via portal | portal §22 |
| Event | Business via Promotions Calendar | portal §23 |
| Offers/coupons | Business via portal | portal §24 |
| Location Profile | Business via portal | portal §26 |
| Analytics | Derived, read-only | portal §25 |

---

## 5. Workflow index (used throughout the docs)

**App:** A1 account+identity · A2 location · A3 sign-in+recovery · A4 discover · A5 evaluate/compare/save ·
A6 navigate+arrive · A7 contribute · A8 profile/account.
**Portal:** P1 sign-in/recover · P2 claim · P3 dashboard · P4 live update · P5 promotions · P6 offer ·
P7 analytics · P8 profile · P9 team · P10 report/dispute · P11 settings.

Full request/response per step: `SCREEN_TO_API_MAPPING.md`.
