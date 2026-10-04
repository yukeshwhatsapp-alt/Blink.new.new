# BLINK Android wrapper

This project wraps the existing BLINK web application in Android WebView. The original extracted web files are bundled in `app/src/main/assets/www/`; `index.html`, `blink.html`, its Firebase/auth bundle, local-storage keys, and navigation are kept in place. The Android bridge is additive and does not replace the web UI.

## Build

Install JDK 17 or newer supported by Android Gradle Plugin 8.13, Android SDK Platform 35, and Android Build Tools 35.0.0. Then run:

```sh
./gradlew assembleDebug
```

The debug build produces one standalone arm64 APK (rather than an APK split),
including only the matching OCR native library. Build an optimized distribution
with:

```sh
./gradlew assembleRelease
```

Release builds enable code and resource shrinking. This APK requires a 64-bit
ARM device.

The app uses Gradle 8.13 through the checked-in wrapper. The Android SDK is not included in the repository; install Platform 35 and Build Tools 35.0.0 locally before building.

BLINK's Firebase web configuration is bundled in `blink.html` and `blink-firebase-config.js`. Firebase Analytics initializes when the SDK reports WebView support. The Android Google Connect flow uses the native Google account chooser and passes its ID token to Firebase Web Auth; it does not run Google OAuth in the embedded WebView.

For the Android APK, enable Google in Firebase Authentication and register package `com.yukeshworks.blink` plus the APK signing certificate in both the Firebase Android app and its Google OAuth Android client. The native request uses the Web OAuth client ID as its server client ID. Its project number (`807734030240`) matches the Firebase web config's `messagingSenderId`. The supplied Firebase Android client is registered with SHA-1 `CC:BF:55:04:45:12:C0:65:BF:2D:9E:8E:E2:46:B5:0F:8C:F5:D5:8E` (certificate SHA-256 `05:FF:43:C0:D7:35:0E:C3:0D:01:FE:40:A2:7D:66:EA:EB:61:09:EF:53:E1:65:B8:E3:48:77:BE:12:48:54:E9`), but the local debug APK uses a different key: SHA-1 `34:B5:1D:E2:E1:61:84:A6:73:FB:EF:9E:B2:AE:D1:DD:91:4F:E0:81` and SHA-256 `C7:8B:71:78:99:E6:D8:38:56:BA:F5:D9:F3:06:25:6C:F3:BC:7F:4A:CA:2B:7A:E5:55:5E:C5:D9:A8:A5:C0:0C`. Add the local debug fingerprints to both Android OAuth registrations to use this APK; register the release signing certificate separately. This project intentionally has no `google-services.json`: its WebView Firebase Auth uses the checked-in web config, and the native Google flow reads the Web client ID from `res/values/styles.xml`. Always verify the actual APK certificate with `apksigner verify --print-certs`.

`MainActivity` loads `https://appassets.androidplatform.net/assets/www/index.html`; the embedded BLINK iframe has the same origin. Keep `appassets.androidplatform.net` (host, without scheme or path) in Firebase Authentication > Settings > Authorized domains. A separately hosted web/development build must authorize its own actual browser hostname; do not add a Codespaces domain unless that is where that build is actually running.

## Native tracking

- **Cardio:** A user-started Android location foreground service uses Google Play services fused location updates. It filters inaccurate/stale/impossible fixes and calculates distance from real GPS fixes. Walking/running durations use reported GPS speed; step counts come from `TYPE_STEP_DETECTOR` only when the optional activity permission and sensor are available. Steps are displayed separately and never added to GPS distance.
- **Gym detection:** The user selects a point and radius on an OpenStreetMap-backed map. Android geofencing runs against that saved location and records presence only from 5:00 PM to 7:30 PM local time. It never creates a workout. Background location is requested separately and explained in context. OpenStreetMap attribution is shown with the map.
- **Sleep:** Android Usage Access events provide a phone-activity-based estimate using the 11:00 PM expected window and the next meaningful morning app-activity event. It is labeled estimated, requires Usage Access, and is never written over manual recovery entries.
- **Storage/sync:** Native records are persisted in app-private preferences and mirrored as `blinkCardioSessions`, `blinkStepHistory`, `blinkGymSessions`, `blinkSleepEstimates`, and `blinkTrackerSettings` local-storage entries. Their `blink` prefix lets the existing Firebase sync code handle them. Existing manual food, workout, cardio, recovery, and preference keys are not reset or substituted.
- **Food:** Existing Fuel foods, quantities, nutrition, editing, and sync are preserved. The supplied app has no dated food-consumption history to safely auto-mark: `blinkFoods` is the editable current food list and `blinkDietHistory` stores undo snapshots. The wrapper does not infer that planned foods were eaten.

Location and step tracking require suitable device permissions/sensors. Fused location and geofencing require Google Play services. Estimates are only generated when Android provides sufficient app-activity observations; this is not medical sleep or activity tracking.

Firebase authentication and Firestore sync remain in the existing WebView app. It syncs each local data key under `users/{uid}/blinkData`, with chunked records, merge handling, offline recovery, and a real-time listener. A separate Android `FirebaseAuth` instance would not share the WebView's signed-in session, and writing an `appData` field directly to `users/{uid}` would bypass this data format. Native tracking data is therefore mirrored into the existing local-storage sync path instead of being written by a second Firebase client.

## College schedule manager

The existing Today and Plan timetable views now read versioned schedule records from the synced `blinkCollegeSchedules` local-storage key. On first launch, the bundled Biotechnology schedule is migrated once as initial data; subsequent timetable changes are stored locally and, when the existing Firebase account sync is configured and signed in, sync through its existing `blink*` data path. Offline edits remain on-device and are queued by the existing sync layer. The APK does not need rebuilding for schedule updates.

Open the existing college timetable from the Today or Plan surface, then choose **Schedule Manager**. Paste timetable text to parse locally, inspect and edit every entry, then choose **Save All**. Schedule types are regular, exam, special/changed, and lab. Effective dates, history/restore, JSON backup/import, course-code dictionary matching, overlap/duplicate warnings, and exam visibility on the existing Today view use deterministic local code. No timetable content is sent to an AI service.

On Android, image and PDF import uses the bundled Google ML Kit Latin text-recognition model on-device; PDFs are rendered locally with Android `PdfRenderer`, with no OCR network request. OCR output is only parsed into a review draft and is never saved automatically. Class and exam reminders use the existing notification bridge, are checked while BLINK is open, and request Android notification permission only once.

Cloud sync requires the configured Firebase project's sign-in provider and authorized WebView origin; local schedules, Today, and offline edits work without it. The local parser tests can be run with `node --test tests/college-schedule.test.js`.
