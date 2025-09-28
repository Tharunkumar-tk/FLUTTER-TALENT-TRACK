# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

Project layout
- talent_track/ — Flutter app (primary)
- backend/ — FastAPI microservice used as server fallback for video analysis

Common commands (run from talent_track/ unless noted)
- Install deps: flutter pub get
- Analyze (lints): flutter analyze
- Format code (dry-run / write):
  - dart format .
  - dart format -o write .
- Run on a device/emulator: flutter run
  - Windows desktop: flutter run -d windows
  - Web (debug): flutter run -d chrome
- Build Android APK: flutter build apk --release --dart-define=API_BASE_URL=https://your.server
  - The app reads API_BASE_URL from --dart-define (defaults to http://10.0.2.2:8000 for Android emulator)
- Build Android App Bundle (Play): flutter build appbundle --release --dart-define=API_BASE_URL=https://your.server
- Tests:
  - Run all tests: flutter test
  - Run a single test file: flutter test test/some_test.dart
  - Filter by test name: flutter test -r expanded -n "partial test name"

Backend (run from backend/)
- Create venv + install: python -m venv .venv; .venv\Scripts\Activate.ps1; pip install -r requirements.txt
- Run dev server: uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
- Docker (optional):
  - docker build -t talenttrack-backend:latest .
  - docker run -p 8000:8000 talenttrack-backend:latest

Platform notes
- Windows + PowerShell:
  - Use quotes around values when needed: --dart-define=API_BASE_URL="http://localhost:8000"
- Assets are declared in pubspec.yaml under assets/config/* and assets/images/.

Configuration
- API base URL: lib/services/config.dart
  - AppConfig.apiBaseUrl is read from --dart-define API_BASE_URL, defaulting to http://10.0.2.2:8000 (Android emulator -> host).
- Challenge and badge catalogs are loaded from JSON assets (assets/config/challenges.json, assets/config/badges.json) via ConfigLoader.

High-level architecture
- State management: flutter_riverpod with StateNotifier
  - lib/state/gamification_provider.dart — coins/xp, badge unlocks, challenge progress
  - lib/state/profile_provider.dart — current UserProfile (role, name, body stats, para flags, goals)
  - lib/state/weight_provider.dart — simple weight history list for sparkline
- App entry: lib/main.dart
  - ProviderScope -> MaterialApp (light theme) -> OnboardingFlow as the initial home screen
- First-run onboarding: lib/screens/onboarding/onboarding_flow.dart
  - Steps: role + name -> gender/goals/focus -> para-athlete + impairment -> body stats (activity level, weight, height, full-body photo upload)
  - On completion shows a violet “crafting” loading animation (animated circular progress with staged messages), then navigates to HomeShell
- Role-aware home shell: lib/screens/shell/home_shell.dart
  - Reads profile role
    - athlete -> bottom tabs: Training, Discover, Report, Roadmap
    - coach -> lib/screens/coach/coach_shell.dart (Dashboard, Leaderboard, Events, Reports)
    - sai -> lib/screens/sai/sai_shell.dart (Dashboard, Top Player, Organise Event, System Settings)
  - AppBar titles adapt per tab; profile and settings icons are present
- Feature screens (athlete role)
  - Training: lib/screens/training/training_screen.dart
    - Search + suggestions, Weekly Goal strip, Challenge cards (Full Body, Calisthenics, Kegel, Lower Body), Activity Focus chips -> grid of activities
    - Para-athlete adaptations and Stretch & Warm-up sections (UI)
    - Activity detail page (inline in file) offers “Proceed to Workout” -> consent/mode chooser -> record or upload flow
  - Discover: lib/screens/discover/discover_screen.dart
    - Loads challenge categories and items from assets via ConfigLoader; navigates to ChallengeDetailScreen
  - Report: lib/screens/report/report_screen.dart
    - Summary tiles, weight sparkline, BMI scale computed from profile’s weight/height
    - Edit opens a bottom sheet enforcing full-body photo before updating weight/height
  - Roadmap: lib/screens/roadmap/roadmap_screen.dart
    - Shows badge grid from assets; unlocked state sourced from gamification provider
- Workout capture and analysis
  - Consent/mode chooser: lib/screens/workout/consent_and_mode_sheet.dart
  - Record video (camera plugin): lib/screens/workout/record_video_page.dart
  - Upload video (gallery picker): lib/screens/workout/upload_video_flow.dart
  - Processing service: lib/services/process_service.dart
    - If mode == 'server': POST {file, activity, user_id, mode} to <API_BASE_URL>/api/v1/process
    - Returns ProcessResponse with annotated video URL and CSV URL (backend may return relative paths; client prefixes API base)
    - Stub path returns a local CSV file for demo
  - Results UI: lib/screens/workout/result_page.dart
    - Plays annotated video (network or local), fetches CSV (network or local) and shows a snippet
    - Awards default coins/xp; increments challenge progress and unlocks milestone/mastery badges per attempts

CI and signing (from talent_track/README.md)
- GitHub Actions run analyze/test on develop and produce APK artifacts on main/tags.
- Android signing in CI uses GitHub Secrets (ANDROID_KEYSTORE_BASE64, ANDROID_KEYSTORE_PASSWORD, ANDROID_KEY_ALIAS, ANDROID_KEY_PASSWORD). Without secrets, debug APK is built.

What to verify when extending
- Add new activities/challenges/badges by editing assets/config/*.json; keep challengeId consistent across challenges and badges for progress/unlocks.
- If changing backend response schema, reflect updates in ProcessService._processOnServer and ProcessSummary.
- Camera/gallery permissions and platform configs are standard (Flutter templates present under android/ios/*). If new plugins are added, re-run flutter pub get and validate manifests.
