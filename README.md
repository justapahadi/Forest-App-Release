# Forest-App-Release
# Forest App — v1.4.0

A field-work diary app for Forest Van Mitras, Forest Guards, and other field staff, built with Expo (managed workflow) + React Native. Tracks daily From/To/Remarks entries per month, attaches GPS-stamped photos, and exports a completed month as a Word document. Requires an account and a paid subscription; a companion backend handles login, email verification, and subscription payments.

## Features

**Account & subscription**
- Sign up with name, email, and password; a verification email is sent (via the backend) with a link to confirm the address before the app can be used
- Login / logout, forgot-password and reset-password flows
- Change password from within the app
- One-time profile setup after signup: designation, profile picture
- Paid subscription (Monthly / Yearly) via Razorpay in-app checkout; access is controlled by a signed entitlement token verified on-device, so the app keeps working offline once entitled

**Diary**
- Current-month diary, auto-created on first open
- Create a diary for any historical month/year (leap-year aware)
- Daily entries: From / To / Remarks, 3 entries per page with Previous/Next
- Edit any existing entry (updates in place — never duplicates a row)
- Future dates locked in the *current* month only; historical months are fully editable
- Missed days stay empty until filled — never auto-filled or deleted
- Progress tracking and a "My Diaries" list with completion %
- Word (.docx) export via the native share sheet, blocked until the diary is 100% complete

**Profile**
- Setup form (name, designation, title, DOB, usual tour-start location) required before any diary can be created
- "My Profile" tab opens as a read-only summary card with an **Edit Profile** button that reveals the same form (Save Changes / Cancel)
- The usual tour-start location pre-fills each new entry's "From" field — always read live from the current profile, never baked in at diary-creation time

**Camera & GPS-stamped photos**
- Live camera preview with a real-time GPS/time overlay, flip camera, and flash toggle
- Preview framing matches the device's actual best available picture size (closest to 16:9) so what's on screen is what gets captured
- On capture, GPS/time (and an optional note) are burned onto the photo in a single compositing pass
- **Retake** button on the confirm screen, alongside Confirm, to discard the shot and go straight back to the camera
- A small thumbnail of the most recent gallery photo sits on the camera screen; tapping it opens the device's default Gallery/Photos viewer directly via a native `ACTION_VIEW` intent on Android, or a full-preview share sheet on iOS
- Captured photos save straight to the device's public Gallery (via `expo-media-library`) — there is no private in-app copy — and can optionally attach to today's diary entry
- If today's entry is already completed, the user is asked to **Replace** its From/To/Remarks with the new photo's, or just **Save to Gallery** instead
- On a diary entry, "See Photo" shows the full, uncropped photo (so the GPS/time stamp in the corner is never cropped off) in a small preview box; tapping it opens a full-screen pinch-to-zoom/pan viewer
- If a photo attached to an entry has since been deleted from the device's Gallery, the entry shows "Photo not available" rather than a broken image

**Storage**
- Diary and profile data stored locally in SQLite (`expo-sqlite`) on the device
- Account, subscription and entitlement data live on the backend; the diary content itself is never uploaded

## How to install

This app is not yet published on the Play Store. Download the latest APK from the **Releases** section of this repository and install it on an Android phone.

- Android will likely warn about installing from an unknown source — allow it for this file.
- If the app is already installed, installing the new APK **updates it in place** and keeps existing data, as long as it's the same signing key and the version code has increased.

### Using the app
- On first launch, sign up with an email and password.
- Check that inbox for a verification email and tap the link (opens a page with a button back into the app).
- Complete the one-time profile setup.
- Subscribe (Monthly ₹59 or Yearly ₹599) via the in-app Razorpay checkout to unlock full access.

## Backend

The app talks to a separate Node/Express backend (`TourDiary-backend`) for auth, email and subscriptions. `src/config/api.js` holds the single base URL the app points at:
```js
export const API_BASE_URL = 'https://forestapp-backend.onrender.com';
```
Change that one line to point the app at a different environment (local dev, staging, production).

## Architecture

```
src/
  api/            authApi.js, userApi.js, subscriptionApi.js — backend HTTP calls
  config/         api.js (backend URL + deep-link scheme)
  context/        AuthContext.js — session/auth state
  database/       database.js (SQLite connection), migrations.js (schema, v1-v3)
  repositories/   diaryRepository.js, diaryEntryRepository.js, profileRepository.js — raw SQL only
  services/       diaryService.js (business rules), exportService.js (docx),
                   profileService.js (profile validation/save), photoService.js
                   (camera/location/GPS-stamp/gallery/viewer logic),
                   entitlementService.js (on-device subscription-token verification)
  screens/
    auth/         LoginScreen, SignupScreen, ForgotPasswordScreen, ResetPasswordScreen,
                   VerifyEmailScreen, DesignationSetupScreen, ProfilePicSetupScreen
    HomeScreen, DiaryDetailsScreen, MyDiariesScreen, CreateDiaryScreen,
    CameraCaptureScreen, PhotoDetailsFormScreen, ProfileScreen,
    SubscriptionScreen, ChangePasswordScreen
  components/     DiaryEntryCard, EntryPhotoState, MonthSelector, ProgressBar,
                   NavigationControls, PhotoCompositor, AppTabBar
  navigation/     AppNavigator.js (root stack: auth / setup / main app),
                   MainTabNavigator.js (Home / My Diaries / Camera / Profile tabs),
                   HomeStackNavigator.js, MyDiariesStackNavigator.js
  utils/          dateUtils.js (local-date-only helpers), validation.js
  constants/      colors.js, dimensions.js, locations.js (empty by default)
```

Screens only ever call `diaryService` / `profileService` / `photoService` / the `api/*` modules — never SQLite, `expo-camera`, `expo-location`, or `expo-media-library` directly.

**Note:** `src/screens/CurrentDiaryScreen.js` exists in the tree but isn't wired into any navigator — the Home tab renders `HomeScreen` directly. Harmless, but worth knowing if you're tracing navigation.

### Date handling

All diary dates are stored and compared as local `YYYY-MM-DD` strings built from `getFullYear()/getMonth()/getDate()`. `Date.toISOString()` is never used for the diary `date` field, since that converts to UTC and can shift a date near midnight in timezones ahead of UTC (e.g. IST). See `src/utils/dateUtils.js` for details. (Audit-only `created_at`/`updated_at` timestamps do use ISO strings — that's just bookkeeping metadata, not the date-locking logic.)

### Photo handling

- One photo per diary entry max (`diary_entries.photo_path`, nullable) — no separate photos table.
- The raw capture and the GPS/time-plus-note stamp are composited together in a single pass on `PhotoDetailsFormScreen` (via `PhotoCompositor`, built on `react-native-view-shot`) — avoiding a double JPEG re-encode.
- The finished, stamped photo is saved once to the device's public Gallery; the app just remembers that Gallery path. There's no separate app-private storage copy, so deleting a photo from the device's Gallery is reflected back in the app ("Photo not available").
- `photoService.openInViewer()` opens photos in the device's actual Gallery/Photos app on Android (native `ACTION_VIEW` intent via `expo-intent-launcher`, with a graceful fallback to the share sheet if that fails); iOS uses the share sheet, which shows a full image preview before any action is chosen.

### Subscription & entitlement

- `SubscriptionScreen` opens Razorpay's in-app checkout (`react-native-razorpay`) for the Monthly/Yearly plan.
- A successful checkout doesn't itself grant access; the backend confirms the payment (via webhook) and issues a signed entitlement token, which the app polls for.
- `entitlementService.js` verifies that token fully offline using `@noble/curves` (P-256 / ES256), so a field worker with no signal keeps working, while the paywall can't be spoofed by flipping a local boolean.

## Database schema (v3)

- **diaries** — `id, month, year, created_at, updated_at`, unique on `(month, year)`
- **diary_entries** — `id, diary_id, serial_number, date, from_location, to_location, remarks, status (EMPTY|COMPLETED), photo_path, created_at, updated_at`
- **profile** — single row (`id = 1`): `salutation, name, designation, dob, default_from_location, created_at, updated_at`

Migrations run automatically on every launch via `PRAGMA user_version` and are a no-op once already applied.

## Tech stack

- Expo SDK 54 (managed workflow), React Native 0.81, React 19
- `expo-sqlite` — local diary/profile storage
- `expo-camera`, `expo-location`, `expo-media-library`, `expo-intent-launcher` — camera capture, GPS, gallery save, native photo viewer
- `expo-sharing` — Word export share sheet, iOS photo preview fallback
- `react-native-view-shot` — GPS/time stamp compositing onto photos
- `docx` — Word (.docx) generation
- `@react-navigation` (native-stack + bottom-tabs) — navigation
- `react-native-razorpay` — in-app subscription checkout
- `@noble/curves` — offline verification of the signed entitlement token
- `expo-secure-store` — secure storage of auth tokens on-device
- `expo-linking` — deep links for email verification / password reset (scheme: `forestapp`)

## Excluded (by design)

Notifications, AI-generated remarks, attendance, and analytics remain out of scope for now. The architecture (service/repository separation, empty `LOCATIONS` config, swappable export layer) is set up so these can be added later without a rewrite.
