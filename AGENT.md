# Project: Production-Grade Android Weather App with Smart Auto-Updating Widget

## Role
Act as a senior Android architect, Kotlin developer, and UI/UX engineer. Treat this as a real software project, not a tutorial.

## Goal
Build a polished weather app whose main value is a home-screen widget that stays reasonably fresh without the user opening the app. Android cannot guarantee exact-minute background updates. Never promise them. The widget must always show the latest successfully fetched data with a clear "last updated" time.

## Tech Stack
- Kotlin, Android Studio, Jetpack Compose + Material 3
- MVVM + Clean Architecture (data / domain / presentation)
- Retrofit + OkHttp, Open-Meteo API (no API key needed; justify any alternative)
- WorkManager, Jetpack Glance for widgets (justify if using something else)
- Room + DataStore, Coroutines + Flow, Hilt
- Android Location Services (fused location)
- Recommend and justify a minimum SDK in the Phase 1 document.

## Features
**Widget (top priority):** periodic auto-refresh, current temp, condition, icon, location, high/low, feels-like, last-updated time, manual refresh, tap-to-open app, offline cached data. Four variants: Compact (temp, icon, location), Standard (temp, condition, high/low), Detailed (temp, feels-like, humidity, wind), Minimal (large temperature-focused). Must resize gracefully.

**App:** current weather (humidity, wind, pressure, UV, visibility, sunrise/sunset, rain probability), 24-hour hourly forecast, 7-day forecast with temperature graph and precipitation, auto location + city search + multiple saved locations, permission handling, °C/°F, light/dark theme, alerts only where the API supports them, offline cache. Premium minimalist Material You look with dynamic weather backgrounds and smooth animations.

## Widget Refresh Strategy (must be designed around these constraints)
- WorkManager periodic work, 15-minute minimum interval. Default to about 30 minutes, with network constraint and exponential backoff.
- Immediate one-time refresh when a widget is added, resized, or manually refreshed, and after reboot or app update.
- Weather API → Repository → Room cache → Worker → Widget update. The widget reads only from the cache.
- Handle Doze, battery saver, background limits, network failures, retries, stale data. Show a visible "stale" state if data is older than a defined threshold.
- Batch and deduplicate API requests (one request per location, not per widget).
- Add an in-app note guiding users to exempt the app from battery optimization on aggressive OEMs (Xiaomi, Samsung, OnePlus, etc.).

## Hard Rules
- Never fabricate weather data or alerts.
- No placeholder or stub implementations presented as finished.
- No deprecated APIs or unnecessary dependencies.
- Never commit secrets, API keys, signing keys, `local.properties`, or keystores. Use an Android-appropriate `.gitignore`.
- Diagnose errors from logs and stack traces. Do not apply random fixes.
- Keep all files consistent with previously created files. Read existing code before changing it.
- Explain important technical decisions before implementing them.

## Workflow Rules
1. Work **one phase per session**, in order. Do not start the next phase until I say "proceed to Phase N".
2. Within a phase, work in small milestones. After each milestone:
    - Run `./gradlew assembleDebug` and `./gradlew test` and fix all failures before continuing.
    - Report: files created/modified, implementation summary, how to test, and Language Notes (below).
    - Commit with a meaningful message (Conventional Commits, e.g. `feat(data): add weather repository with Room cache`).
3. Stop at the end of each phase and wait for my review.

## Git Rules
- Branches: `main` (stable releases only), `develop` (integration), `feature/<name>` per feature. Merge features into `develop` via pull request or merge commit.
- Commit after each logically complete milestone. Keep the repo buildable at every commit.
- Push only if git/`gh` is authenticated. **Never claim a push succeeded unless the command output confirms it.** If a push fails, show me the error.
- Maintain `README.md` (overview, architecture, setup, build instructions), `docs/ARCHITECTURE.md`, and a `CHANGELOG.md`.

## Language Notes (for my learning)
I am learning Kotlin and Java side by side. At the end of each milestone report, add a "Language Notes" section (max 10 lines): 2–3 Kotlin concepts used in that milestone, each with a one-line Java equivalent. Keep it an overview, not a tutorial.

## Phases and Acceptance Criteria
1. **Planning** (document only, no code): feasibility, final stack, architecture and folder structure, widget refresh strategy, Android background limits, Git strategy, milestones with complexity estimates. *Done when:* `docs/PLAN.md` exists and I approve it.
2. **Initialization:** Android Studio project, Gradle version catalog, Hilt setup, dependencies, `.gitignore`, GitHub repo, branches. *Done when:* the empty app builds and runs, and `main`/`develop` exist on GitHub.
3. **Data layer:** Open-Meteo integration, models, mappers, repositories, Room + DataStore, offline caching, error handling. *Done when:* build and unit tests pass, and the repository returns cached data when the network is unavailable (proven by a test).
4. **App UI:** dashboard, hourly/7-day forecasts, graphs, location management, permissions, settings, themes. *Done when:* build passes, all screens work with real data, and permission-denied and offline states are handled.
5. **Widget:** four layouts, WorkManager refresh, cache reads, manual refresh, reboot handling, last-updated display. *Done when:* build passes, all four widgets render from cache, and worker logic is unit-tested.
6. **Testing and hardening:** fix bugs, optimize battery and network, add instrumented and worker tests, and write `docs/DEVICE_TEST_CHECKLIST.md`.
7. **Release:** app icon, signing configuration (keystore stays out of Git), release APK, final docs, tagged GitHub release.

## Testing Responsibilities
- You (the agent) write automated tests and the checklist.
- **I run device tests myself** on a physical phone: app closed, screen off, Doze, battery saver, airplane mode, location permission denied, reboot, location change, widget resize, manual refresh, failed-then-retried updates. Do not claim these were verified. Give me exact steps (including `adb` commands such as `adb shell dumpsys deviceidle force-idle`) and wait for my results.