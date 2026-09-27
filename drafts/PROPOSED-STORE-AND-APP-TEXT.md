# Proposed corrections — App Store privacy label, App Review notes, and in-app copy

Companion to the privacy.html / terms.html / support.html / delete-account.html redraft in this
PR. These files live in `wheeldash`, so nothing here is applied there — this is the drafted text
for whoever picks up that PR. Every citation is `file:line` in `wheeldash` as of 2026-09-27.

## 1. `legal/app-store-privacy.md`

### 1a. Location usage string (currently says "when-in-use")

Current, under "Also confirm in App Store Connect":
> `Info.plist` usage strings must be present and match: location (when-in-use), Bluetooth,
> notifications, and photo library (if avatar is picked from Photos).

**Wrong.** The app requests Always: `INFOPLIST_KEY_NSLocationAlwaysAndWhenInUseUsageDescription`
(`WheelLife.xcodeproj/project.pbxproj:1349,1392`) and `LocationTracker.requestAlwaysAuthorization()`
(`WheelKit/Sources/WheelKit/Location/LocationTracker.swift:151`).

**Proposed replacement:**
> `Info.plist` usage strings must be present and match: location (**Always and When In Use** —
> the app requests Always so a ride keeps recording with the phone in a pocket, screen off),
> Bluetooth, notifications, and photo library (if avatar is picked from Photos).

### 1b. Diagnostics "Not linked to identity"

Current:
> Crash Data / Performance Data / Other Diagnostic Data — Not linked to identity

**Wrong.** `wheel_logs.user_id` and `crash_reports.user_id` are both `not null ... references
profiles (id) on delete cascade` (`supabase/31_wheel_logs.sql:16`, `supabase/30_crash_reports.sql:16`),
RLS restricts reads to `user_id = auth.uid()`, and migration `116_adopt_anonymous_data.sql`
exists specifically to re-key these rows to a new account id — the schema is built to preserve
the identity link, not to avoid it. The payload content (voltage/speed/temperature/raw packets,
stack traces) has no name, email or location in it, but the row itself is linked to the account.

**Proposed replacement:** move Crash Data, Performance Data, and Other Diagnostic Data from
"Not Linked to Identity" to **"Linked to Identity"**, and reword the note:
> Each diagnostic row is tied to the account's id (so we can act on it and so it is deleted when
> the account is deleted) even though its contents carry no name, email or location.

This also changes the résumé at the bottom of the file: "Data Not Linked to You" would then list
only whatever is left — as of this pass, nothing diagnostic qualifies. Re-check this section
before the next App Store Connect submission; "linked" changes what the printed privacy label says.

### 1c. New omission: HealthKit

Not mentioned anywhere in the file. `WatchAutoLaunchPolicy` (`WheelKit/Sources/WheelKit/Domain/WatchAutoLaunchPolicy.swift`)
calls `HKHealthStore.startWatchApp(with:)` to launch the paired watch app when the setting (off by
default) is on; it does not read or write any health record. Apple's own "Health" data type
question should stay "Not collected" for health *records*, but the entitlement itself and its
purpose should be named in this doc so a future reviewer doesn't have to rediscover it:
> **HealthKit entitlement** — used only to ask watchOS to launch the paired Wheel Life watch app
> (`HKHealthStore.startWatchApp`). No health or fitness record is read, written, or stored.

## 2. App Review notes (`docs/APP-REVIEW-NOTES.md`)

### 2a. Location

Current:
> We request **When In Use** authorisation only; we never ask for Always.

**False** — see 1a above.

**Proposed replacement:**
> We request **Always** authorisation (in addition to When In Use). A ride starts itself the
> moment your wheel connects, which is usually with the phone already in a pocket and the screen
> off — Always is what lets the route keep recording through that, instead of the ride coming
> back with no map. Location is used solely to record the rider's own route and stats and to
> power features the rider turns on (nearby riders, territory). It is never sold, never used for
> advertising, and a route never leaves the device unless the rider posts that ride, trimmed at
> both ends.

### 2b. "Never controls the wheel"

Current:
> **Safety framing.** The app displays what the wheel reports and never controls the wheel. Our
> Terms and in-app copy state that telemetry may be wrong or delayed and must not be relied on to
> decide whether it is safe to ride.

**False.** The app writes to supported wheels: saved settings presets (`WheelKit/Sources/WheelKit/Domain/WheelSettingsPreset.swift`),
headlight on/off (`AutoHeadlight.swift`, `AutoHeadlightSession.swift`), and command encoders
including horn (`WheelCommandEncoders.swift`). CLAUDE.md's own state section lists "Veteran (BMS,
settings writes...)" as hardware-verified.

**Proposed replacement:**
> **Safety framing.** The app displays what the wheel reports, and on supported models can send
> commands the rider asks for — a saved settings preset, or turning a headlight on or off — but
> it does not make the wheel safe and never overrides the wheel's own protections. Our Terms and
> in-app copy state that telemetry may be wrong or delayed and must not be relied on to decide
> whether it is safe to ride.

## 3. In-app sentences

All three of these are the exact sentence the rider who opened #1310 quoted as contradicting the
policy. Fix all three together or the discrepancy just moves to whichever one is missed.

### 3a. "...and nothing appears in the feed" (three occurrences, both platforms)

- iOS `AppSource/Features/Settings/SettingsView.swift:1184` (Sharing & Privacy footer)
- iOS `AppSource/Features/Settings/SettingsView.swift:690-691` ("Your Data" screen)
- Android `SettingsScreen.kt:1145-1147` (verbatim mirror of the first)

Current: "Leaderboards count every finished ride automatically — only the numbers (distance,
speed, airtime) sync, never your location, and nothing appears in the feed."

**False since #1105** (`supabase/191_stats_only_feed_cards.sql`): a leaderboard-only ride shows as
a stats card in the feed, to the rider and their accepted friends. It's also incomplete: leaderboard
sync also uploads the ride's coarse territory cells (~900 m), which is a location signal.

**Proposed replacement (all three sites, keep identical across both platforms):**
> "Leaderboards count every finished ride automatically — the numbers (distance, speed, airtime)
> sync, plus the coarse map cells it passed through for Territory, never your precise route. It
> shows as a numbers-only card in your feed and your friends' — never to a stranger."

### 3b. Wheel-side account-free list (several sites)

- privacy.html / support.html (fixed in this PR)
- iOS `AppSource/Features/Community/Auth/AuthView.swift` perks list and similar copy in
  `AccountGateView.swift`, `AppTourController.swift`
- Android equivalents in `AuthScreen.kt`, `AppTourUi.kt`

Wherever in-app copy lists "ride recording" or "History" alongside "connecting/live
dashboard/alarms" as working without an account, split it: those two need a free account
(`AccountGate.needsAccount`, `WheelKit/Sources/WheelKit/Community/AccountGate.swift:44-52` —
`.recordRide` and `.viewRides` both return `true`).

### 3c. Android GPS-calibration copy says "Always" (Android only, real bug — not just a truth-check nit)

`android/app/src/main/java/com/freespin/wheellife/ui/SettingsScreen.kt:505-509`:
> "...Needs location set to Always and Precise."

Android never declares `ACCESS_BACKGROUND_LOCATION` (confirmed absent from `AndroidManifest.xml`;
`AppTourUi.kt:165-169` documents this explicitly) — there is no "Always" grant on this build to
set. This line was copied from iOS without adapting.

**Proposed replacement:** "...Needs precise Location allowed and the app not restricted from
running in the background (Settings → Recording explains this)." — match whatever wording
Android's own permissions tour (`AppTourUi.kt:181-184`) already uses correctly.

### 3d. Android's two contradictory delete-account strings

- `SettingsScreen.kt:811-814` — "removes your profile, your shared rides, your club membership
  and any territory you hold... Rides recorded on this phone stay on this phone."
- `ProfileEditorScreen.kt:722-723,1404-1406` (`DELETE_CONSEQUENCE`) — "removes your profile,
  shared rides, and group rides" — no mention of club/territory or on-device rides, despite its
  own comment claiming it's "iOS's words."

Pick one sentence — recommend the first, since it's closer to what actually cascades (see
`ab2ce409d9153d166` finding set: profile, shared rides, group rides, territory (`hex_contributions`),
club **membership** all cascade; a club a rider *founded* has `created_by ... on delete set null`,
but that column is also `not null` — worth a test run before either string promises it, see the
PR body's defects list) — and reuse it at both call sites.

### 3e. iOS's own two delete-account dialogs are structurally inconsistent

`ProfileView.swift`'s `confirmationDialog` states the consequence only in the row's subtitle,
never restated in the dialog itself; `ProGating.swift`'s version of the same dialog restates it in
`message:`. Pick one pattern (restating in the dialog is safer — a rider can reach the row's
subtitle without reading it) and use it at both call sites.

### 3f. Push token wording, if it also exists in-app anywhere

Not found verbatim in-app (it's a privacy.html-only line, fixed in this PR), but if any settings
screen repeats "the token identifies the device, not you," correct it the same way: `device_tokens`
carries `user_id` (`supabase/20_push.sql:7-16`).

## 4. Not fixed here, flagged for a separate session

- The website (`wheellife-web`) `sharedRide()` in `src/lib/community.ts:70-71` does a raw
  `shared_rides` table select instead of calling the `shared_ride()` RPC, so it inherits
  `shared_rides_read`'s RLS (`supabase/02_rls.sql:36-40`), which has no `stats_only` check — a
  signed-out visitor who has a stats-only ride's id can currently read its numbers there, even
  though every app-facing RPC excludes strangers from stats-only rows. The owner has approved a
  fix (hide stats-only rows from that read path); the privacy.html redraft in this PR is written
  for the fixed behavior and says so — track that fix and re-check this page once it ships.
- `clubs.created_by` is declared `not null` **and** `references auth.users(id) on delete set
  null` (`supabase/43_clubs.sql:13`) — Postgres will refuse to satisfy both on a cascading delete.
  Worth a real test (create an account, found a club, delete the account) before any document
  promises what happens to a club founder's account deletion.
- Account deletion does not remove Storage files (avatar, ride GPX, ride photos) that a deleted
  row pointed to — `delete_account()` (`supabase/03_functions.sql:110-119`) is a single
  `delete from auth.users` statement. privacy.html and delete-account.html in this PR now say so
  as a known gap instead of promising removal; closing the gap is a wheeldash fix (call the same
  per-file storage-delete helpers `deleteRide`/`deleteRidePhoto`/`avatars_delete` already use for
  a single delete, from `delete_account()` too, or move storage cleanup into an Edge Function
  triggered by the account-deletion event).
