# Vybity — Spec Validation, Assumptions & Open Questions

This document does two things:

1. **Validates** the provided `Vybity - REST API Specification v1.md` against the screens actually
   present in the Figma file (`App Ui` `0:1` and `Business Portal` `671:2529`).
2. Collects **assumptions** (things not readable from a design) and **open questions** needing your
   confirmation. Nothing here has been silently invented.

Legend: ✅ matches Figma · ⚠️ assumption · ❌ mismatch/needs decision.

---

## Part A — Screen coverage validation

| Spec area | Screens in Figma | Verdict |
| --- | --- | --- |
| §3 Auth (register/login/social/forgot/OTP/reset) | Sign In, Register, Forgot Pass, OTP, Create New Password | ✅ all present |
| §3.10 Identity | Identity ready `208:336` | ✅ present |
| §4 Onboarding | Onboarding 4/5/6 | ✅ (Splash 1 absent — only Splash 2/3) |
| §5 Location | Allow Location, Add address manually | ✅ |
| §6 Home & Chat | Home, Chat, Home after visit | ✅ |
| §7 Places | Details ×2, Reliability, Why not here? | ✅ |
| §8 Compare | Compare, After click Compare | ✅ |
| §9 Directions/nav | Get Direction, Start navigation | ✅ |
| §10 Visits | Manual Arrival, Show it after 10 min | ✅ |
| §11 Contribute | Contribute, Community Update ×2, Review ×2, Updated ×2, After click done ×2 | ✅ |
| §12 Saved | Saved | ✅ |
| §13 Profile/account | Profile, Personal Information, Badges (hidden/empty), Change Password | ✅ (Badges empty → §13.2 assumed) |
| §14 Legal | Privacy Policy, Terms & conditions | ✅ |
| §18–29 Portal | Log in, Forgot Password, Verify Email, Create New Password, Claim flow ×6, Claim status, Overview, Live Biz Updates, Promotions Calendar, Offers, Analytics, Location Profile ×2, Team ×2, Settings ×2, Create Update, Create Offer, Schedule Item, Assign Permissions, Submit dispute | ✅ all present |
| §28 Reports/Disputes | **No frame named "Reports"**, but the screen exists inside the frame mis-named **`Team` (`808:9892`)** | ✅ UI present (see A1) |

### A1. Reports/Disputes screen mis-named `Team` ❌→✅
The portal sidebar has a **Reports** entry and the spec §28 describes a Reports table plus a Disputes
list. There is no frame called "Reports"; that UI lives in frame **`808:9892` named `Team`** (a copy of
the sidebar + heading, with the wrong frame name). The real Team screen is `801:9377`.
→ **Spec §28 is valid**; note the duplicate frame name for traceability.

### A2. Reports table header "STAFF MEMBER" ✅ (confirmed copy error)
Frame `808:9892` shows columns `STAFF MEMBER | AFFECTED CONTENT | REASON | STATUS`. "STAFF MEMBER" is
clearly wrong for a reports table. Needs a corrected label (proposal: **"CONTENT TYPE"** or **"REPORTER"**).

### A3. "Splash 1" ❌
Only **Splash 2** and **Splash 3** exist. Either Splash 1 was deleted or Splash 2 is the first. No API
impact; confirm ordering.

---

## Part B — Point-by-point validation of spec assumptions (⚠ items)

| Spec ref | Spec says (⚠) | What the Figma actually shows | Verdict |
| --- | --- | --- | --- |
| §3.1 | Terms checkbox = `accepted_terms` | Register has a `Check box` component whose label text is **inside a nested component** (not exposed in metadata) | ⚠ need label |
| §3.10 | Refresh icon regenerates handle+avatar | The refresh icon is **inside the "Continue as Provider" button**, beside its label — not a standalone regenerate control | ⚠ re-examine |
| §2.1 | `reliability_level` = high/medium/low | Only **High** and **Low** ever appear (Details, Result, Saved, Compare, Reliability sheet, Review) | ⚠ medium unconfirmed |
| §2.1 | Responses may return parking `limited` | `Limited` appears in **Why not here?** and **Compare**; submission chips are Easy/Moderate/Difficult/Full/Not Applicable | ⚠ server-computed label |
| §2.1 | `atmosphere` multi-select | 6 chips (Cozy/Lively/Romantic/Casual/Trendy/Loud Music); no visual single/multi indicator in static design | ⚠ ask |
| §6.3 | quick replies e.g. "Walking Distance" | Chat shows exactly **Walking Distance**; assistant "Got it. How far are you willing to travel?" | ✅ |
| §6.4 | Not Right Now reasons | 7 reasons match verbatim (Too Crowded, Too Loud, Too expensive, Too far, Not the right atmosphere, Changed my mind, Other) | ✅ |
| §7.1 | Community updates show "first 3" | Details shows **3** community cards | ✅ |
| §7.4/§8 | Parking `limited` | "Parking: Limited" in Why not here?; "Parking: Easy / Limited" in Compare | ✅ |
| §8 | 2 or 3 ids; button "Compare 3 places" | Compare shows **2 columns**; selection button says **Compare 3 places** | ⚠ confirm max |
| §11.3 | Atmosphere 1–3, note ≤500, media ≤4 | Not shown in static design (no counters) | ⚠ defaults |
| §11.4 | Follow button labels not designed | "After click done" shows title/body only; the actual Yes/No controls are `Button` instances (labels not exposed) | ⚠ |
| §12 | NOT RIGHT NOW / WHY NOT HERE on Saved | **Both buttons appear on Saved cards** | ✅ confirmed |
| §13.2 | Badges screen only has a title | Badges frame `1009:254` is **hidden and empty** | ⚠ |
| §13.2/13.3 | Rewards / Notifications / Settings screens not designed | Profile lists **Rewards, Notifications, Favourites, Settings** rows, but **no frames** exist for them | ⚠ |
| §13.3 | Favourites = Saved alias | Profile lists **Favourites** as its own row | ⚠ confirm alias |
| §13.6 | Settings `distance_unit`, `language` | No Settings screen frame; Profile has a "Settings" row | ⚠ |
| §18 | Portal has no Register screen | Portal has only Log in / Forgot Password / Verify Email / Create New Password | ✅ |
| §19.1 | `claim_status` routing | Claim status `871:2023` shows **Pending Review**; Claim your business `868:1605` is the entry | ✅ |
| §22.1 | `category` = `environment_status` | Table shows label **"Environment Status"** above each update | ✅ |
| §22.2 | Live update text ≤280 | No counter shown | ⚠ default |
| §24.2 | `quantity_limit` in list, not in form | Offers list shows **"50 available"**; Create Offer modal has no Quantity field | ✅ confirmed |
| §24.2 | Offer `title`, `terms` | Offers shows "20% off Dinner" + Terms; Create Offer has Title + Terms fields | ✅ |
| §25 | `range` 7d/30d/90d | Analytics axis is **D1…D30** (30-day); no range switcher drawn | ⚠ |
| §25 / §21 | Follower counts disagree | Overview **12,450**; Analytics aggregate **4,389**, growth **+589**; §34 Q4 | ❌ data inconsistency |
| §26 | `hours[]` 7 entries | Location Profile shows Mon–Sun with "Closed" | ✅ |
| §28.1 | Reports `status` pending_review/resolved | Shows "Pending Review" and "Resolved" | ✅ |
| §28.2 | Disputes status `in_progress` | Shows "In Progress" | ✅ |
| §29 | `restaurant_name` change may need review | Settings has "Restaurant Name" field | ⚠ |

---

## Part C — Additional findings not in the spec

1. ⚠ **`Contribute` card has a save/heart control** (`641:1072`) — the spec's PlaceCard on Contribute
   doesn't mention a save toggle. Add `is_saved`/save action to that screen.
2. ✅ **`Open` chip is on Contribute and Result/Saved cards** — `is_open` is exposed everywhere.
3. ⚠ **`Needs Update` tag** appears on Result and Saved cards — the spec mentions it only as a
   `tags[]` value; confirm whether it also drives ordering on Contribute (spec §11.1 says ordered by
   "how much each place needs a fresh update") — likely the same signal.
4. ⚠ **`Limited Current Information`** appears as a card state (Result, Saved) — matches
   `RightNow.status = limited_current_information`. Confirm the card still shows Reliability + Live Biz
   Update + Event when Right Now is limited (design suggests yes).
5. ⚠ **`Why this fits`** appears on Contribute and Saved too, not only after a chat search. Confirm
   whether it is personalised per-search only, or a generic blurb elsewhere.
6. ⚠ **Address/portrait placeholders are inconsistent**: claim address "214 Willow Ave, Dhaka",
   Location Profile "214 willow ave, Dhaka", but product launches in **Kansas City** and app header
   shows Kansas City. Confirm real data model supports arbitrary cities.
7. ⚠ **Portal phone format** "(415) 4579 83475" is malformed; treat as placeholder.
8. ⚠ **Overview "Need Attention" repeats the same row ×3** and **"Upcoming Scheduled Posts" repeats one
   item ×5** — placeholders, not a data requirement.
9. ⚠ **Analytics legend** includes "Community updates" and "Open rate %" toggles — confirm these are
   display toggles only.
10. ⚠ **No admin/moderation screens** exist anywhere in the file, yet the product needs claim approval,
    report moderation, and dispute handling. The spec §0.5/§34 Q2 flags this; still unresolved.

---

## Part D — Open questions (grouped, ready to answer)

### Product / identity
1. **Identity refresh** — does the refresh icon on "Identity ready" regenerate the handle+avatar, and
   what exactly does **Continue as Provider** do next (portal claim flow)?
2. **Terms checkbox** — exact label on Register (terms + privacy consent?).
3. **Provider accounts** — is a portal user always created in the phone app first, or will the portal
   get its own sign-up? (Spec assumes app-first.)

### Data model / enums
4. **Reliability** — is there a **Medium** level, or only High/Low?
5. **Parking `Limited`** — server-computed label, or a real stored value? Present in Why-not-here and
   Compare but absent from the submission chips.
6. **Atmosphere** — single choice or multi-select (spec assumes 1–3)?
7. **Compare limit** — 2 or 3 places? (Design shows 2 columns; button says "Compare 3 places".)
8. **Grouping / environment** — `environment` (Outdoors/Indoors) appears only in Compare. Where is it
   captured on submit? (Not in the Community Update chips.)

### Contribute & visits
9. **Contribution window** — is the "Show it after 10 min" prompt the only path to contribute, and can a
   place be updated without a verified visit?
10. **Cooldown** — what is the "short window" for `UPDATE_COOLDOWN`?
11. **Media limits** — photo/video size & duration, max attachments per update?
12. **XP/badges/rewards** — where are the rules defined? (No screen.)

### Business portal
13. **Follower count source of truth** — Overview 12,450 vs Analytics 4,389 vs +589 growth.
14. **Reports table header** — correct the "STAFF MEMBER" column.
15. **Report/redemption** — the portal *receives* reports and coupon redemptions, but the app has **no
    screens to report content or redeem a coupon**. Add those endpoints/proposals?
16. **Admin back-office** — should I propose an admin API (claim approval, report moderation, dispute
    resolution)?
17. **Team invite** — how does an invited staff member accept (email link or code)? Can Team/Settings be
    delegated to staff?
18. **Offer code** — one shared code per offer (as drawn: "SWEET") or per-user unique codes?
19. **Multiple locations** — can one owner manage several locations? (Portal assumes one active
    location from the token.)
20. **Notification attribution** — publishing sends a push to followers; confirm `notify_followers`
    default and rate limits.

### Non-functional
21. **Auth token lifetimes** (15 min / 30 day), password rules, OTP limits, page size, base URL — spec
    placeholders to confirm.
22. **Backend stack** — the brief says Node/Express + MongoDB, but §1 of the task mentions Django/DRF.
    Which is authoritative for the implementation phase? (Documentation here is stack-agnostic.)

---

## Part E — Answers needed before implementation

| # | Question | Blocks |
| --- | --- | --- |
| D2 | Register terms label | Auth schema |
| D4 | Reliability Medium? | enum |
| D6 | Atmosphere multi-select? | Community Update schema |
| D7 | Compare 2 vs 3 | Compare validation |
| D13 | Follower source of truth | Overview/Analytics schema |
| D15/D16 | Report + redemption + admin APIs | full-surface completeness |
| D22 | Django vs Node backend | implementation |

Everything else is documented with a clearly marked assumption and can proceed.
