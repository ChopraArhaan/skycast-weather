# Phase 1: Technical Plan & Architecture Specification
**Project:** Production-Grade Android Weather App with Smart Auto-Updating Widget  
**Document Version:** 1.4  
**Status:** Revised & Pending Approval  

---

## 1. Technical Feasibility Assessment

### 1.1 Weather API: Open-Meteo
- **Feasibility:** High.
- **Key Advantages:**
  - Free non-commercial tier with up to 10,000 API requests/day without requiring an API key.
  - Returns raw, un-opinionated JSON for current conditions, hourly forecasts (up to 7 days), and daily forecasts (up to 16 days).
  - High accuracy using WMO weather interpretation codes (`weathercode`), temperature, feels-like (`apparent_temperature`), relative humidity, surface pressure, wind speed/direction, UV index, visibility, precipitation probability, and sunrise/sunset times.
  - Zero API keys or secrets to store or expose.
- **API Efficiency & Deduplication Rules:**
  - **Single Fetch per Distinct Location:** `RefreshWeatherWorker` queries Room `WidgetConfigDao` to get all configured widgets, extracts the unique set of location coordinates, and fetches weather from Open-Meteo **exactly once per distinct location**, regardless of how many widget instances point to the same city.
  - **Cache First:** Always query Room cache before making a network call. If cached data for a location is less than 15 minutes old, return the cache without triggering a network request (unless explicitly forced by manual refresh).
- **Weather Alerts Note & v1 Decision:**
  - Open-Meteo's standard core forecast endpoint does **not** supply severe weather alerts/warnings.
  - *v1 Decision:* Weather alerts are explicitly marked **out of scope for v1** in compliance with the rule: *"Never fabricate weather data or alerts."* If alert endpoints are added by Open-Meteo or another provider in the future, they will be integrated in subsequent releases.
- **Open-Meteo Attribution Requirement:**
  - Open-Meteo requires visible attribution under the CC BY 4.0 license (*"Weather data by Open-Meteo.com"*).
  - Mandatory attribution text will be displayed in the App Settings screen, Dashboard footer, and About dialog.

### 1.2 Reverse Geocoding & City Search
- **Open-Meteo Limitations:** Open-Meteo's weather forecast API takes `latitude` and `longitude` coordinates and returns weather metrics; it does **not** perform reverse-geocoding (converting coordinates to a city name).
- **Geocoding Strategy:**
  - **City Search:** Use Open-Meteo's dedicated Geocoding API (`geocoding-api.open-meteo.com/v1/search`) for text-based city search.
  - **Reverse Geocoding (Lat/Lon -> City Name):** Use Android's system `android.location.Geocoder` API (with fallback to Open-Meteo search API or saved coordinate label) to resolve human-readable city and region names for GPS location fixes.

### 1.3 Widget Framework & Architecture (`androidx.glance:glance-appwidget`)
- **Feasibility:** High.
- **Widget Architecture (Single `GlanceAppWidget` with Declared `SizeMode.Responsive` & Minimal Style Option):**
  - **Single Widget Definition:** Implement **one `GlanceAppWidget`** registered in `AndroidManifest.xml` rather than multiple widget components in the launcher picker.
  - **Declared `SizeMode.Responsive` Sizes:** The widget specifies exact declared size targets:
    ```kotlin
    override val sizeMode = SizeMode.Responsive(
        setOf(
            DpSize(100.dp, 100.dp), // Resolves to Compact Layout
            DpSize(200.dp, 120.dp), // Resolves to Standard Layout
            DpSize(280.dp, 200.dp)  // Resolves to Detailed Layout
        )
    )
    ```
  - **Deterministic Layout Mapping:** In `provideGlance()`, evaluating `LocalSize.current` matches declared bounds to exactly one layout variant:
    1. `DpSize(100.dp, 100.dp)` -> `CompactWidgetLayout` (Temperature, Weather Icon, City Name).
    2. `DpSize(200.dp, 120.dp)` -> `StandardWidgetLayout` (Temperature, Weather Icon, City Name, Condition Description, High/Low Range).
    3. `DpSize(280.dp, 200.dp)` -> `DetailedWidgetLayout` (Temperature, Weather Icon, City Name, Condition, Feels-Like, Humidity, Wind Speed, High/Low Range, Absolute Last-Updated Time).
  - **Per-Widget "Minimal" Style Option:** Users can select a "Minimal" style option in the configuration activity. When `isMinimalStyle == true` for a given `appWidgetId`, the widget overrides size responsive layouts and renders `MinimalWidgetLayout` (large temperature-focused display).
  - **Configuration Activity (`WeatherWidgetConfigActivity`) & `RESULT_CANCELED` Handling:**
    - Declared via `android:configure` in `appwidget-provider` XML.
    - Receives `AppWidgetManager.EXTRA_APPWIDGET_ID`.
    - **Default Result:** Activity invokes `setResult(RESULT_CANCELED)` immediately in `onCreate()`. If the user presses the back button or closes the activity before saving, Android cancels widget placement completely so no unconfigured or orphaned widget is added to the home screen.
    - When confirmed by user: Saves `WidgetConfigEntity` to Room database, triggers initial Glance widget rendering via `WeatherGlanceWidget().update(context, glanceId)`, sets `setResult(RESULT_OK, intent)`, and finishes.
  - **Per-Widget Storage Strategy (Room `widget_config` Table):**
    - Widget configurations are stored centrally in the Room database in a `widget_config` table (`WidgetConfigEntity` with `appWidgetId`, `locationId`, `isMinimalStyle`).
    - **How the Worker Reads It:**
      1. `RefreshWeatherWorker` executes `doWork()`.
      2. It queries `WidgetConfigDao.getAllWidgetConfigs()` to retrieve all active widget configurations.
      3. It extracts the unique list of `locationId`s across all configured widgets.
      4. It fetches weather from Open-Meteo API **once per unique `locationId`** and writes updated data to `WeatherEntity` in Room cache.
      5. It invokes `WeatherGlanceWidget().updateAll(applicationContext)` so each widget instance queries `WidgetConfigDao.getConfigForWidget(appWidgetId)` and re-renders its layout from the cached weather data.
  - **`onDeleted` Cleanup:**
    - Override `GlanceAppWidgetReceiver.onDeleted(context, appWidgetIds)` to execute `WidgetConfigDao.deleteConfigs(appWidgetIds)` in a background coroutine, removing deleted widget configuration rows from Room and preventing database leaks.
- **Icons & Graphics:**
  - **No Coil Required:** Weather condition icons are mapped directly from WMO weather codes to local vector drawables (`R.drawable.ic_weather_*`) rendered using Glance vector primitives. External image loading libraries (such as Coil) are unnecessary and removed from the dependency stack.

### 1.4 Background Scheduling: WorkManager
- **Feasibility:** High.
- **Execution Rules:**
  - Minimum periodic interval: 15 minutes (default configured to ~30 minutes).
  - Native integration with Android Doze mode and background execution constraints.
  - Periodic work automatically survives device reboots via WorkManager's internal SQLite database storage.

### 1.5 Location Handling & Widget Location Model
- **Location Model:**
  - Each widget instance tracks either **"Current Location"** or a specific **"Saved Location"** (configured via `WeatherWidgetConfigActivity` and stored in `WidgetConfigEntity`).
  - For "Current Location", the background worker uses the **last-known coordinates** captured when the user last opened the main app (stored in DataStore / Room).
  - **Background Location (`ACCESS_BACKGROUND_LOCATION`) is OPTIONAL in v1.** The app does not require background location permission. If location is unavailable or denied, the widget seamlessly falls back to the user's primary saved location.

---

## 2. Recommended Tech Stack & Target API 36

### 2.1 Google Play Target API 36 Requirement & Target SDK
- **Google Play Requirement:** Target API 36 (Android 16) requirement starting August 31, 2026.
  - `compileSdk = 36`
  - `targetSdk = 36`
  - `minSdk = 26` (Android 8.0 Oreo)

### 2.2 Official Android 16 (API 36) Behavior Changes (Quoted from Official Documentation)

Directly quoted from developer.android.com ([developer.android.com/about/versions/16/behavior-changes-16](https://developer.android.com/about/versions/16/behavior-changes-16)):

1. **Edge-to-Edge Opt-Out Disabled (`targetSdk 36`):**
   > *"The attribute `R.attr#windowOptOutEdgeToEdgeEnforcement` is deprecated and ignored when running on Android 16. Apps targeting Android 16 can no longer opt out of edge-to-edge layouts."*
   - *Impact on Weather App:* All screens (Dashboard, Forecast, Settings, Location Search) must handle system bar insets (`WindowInsetsCompat` / Compose `insets`) properly to prevent UI content from overlapping status or navigation bars.
2. **Predictive Back Navigation Enabled by Default (`targetSdk 36`):**
   > *"System animations (back-to-home, cross-task, cross-activity) are enabled by default (`android:enableOnBackInvokedCallback` defaults to `true`). `onBackPressed()` and `KeyEvent.KEYCODE_BACK` are no longer dispatched unless explicitly intercepted using `OnBackInvokedCallback` or temporarily opted out in the manifest."*
   - *Impact on Weather App:* Compose navigation and custom back handling must use `BackHandler` or `OnBackInvokedCallback` to handle back navigation cleanly without unexpected activity exit.
3. **JobScheduler Quotas & Abandoned Jobs (All Apps on Android 16):**
   > *"Active standby bucket quotas, top-state execution duration, and jobs running alongside Foreground Services are subject to updated quota enforcement. Abandoned jobs (garbage-collected `JobParameters` without `jobFinished()`) receive the new stop reason `STOP_REASON_TIMEOUT_ABANDONED`. `JobInfo.Builder.setImportantWhileForeground()` is fully deprecated and ignored."*
   - *Impact on Weather App:* WorkManager workers must complete background work promptly and return explicit `Result` statuses without relying on deprecated foreground flags.
4. **Fixed Rate Work Scheduling Optimization (`targetSdk 36`):**
   > *"For `ScheduledExecutorService.scheduleAtFixedRate`, if multiple task executions are missed while outside a valid process lifecycle, at most one missed execution is executed immediately when returning to a valid lifecycle."*
   - *Impact on Weather App:* Periodic background execution avoids burst repetitions after long sleep intervals.
5. **Adaptive Layouts on Large Screens (`targetSdk 36`):**
   > *"On displays with a smallest width $\ge 600\text{dp}$, screen orientation, resizability, and aspect ratio restrictions (`screenOrientation`, `resizableActivity`, `minAspectRatio`, `maxAspectRatio`, `setRequestedOrientation()`) are ignored by default. Apps will fill the display window without pillarboxing."*
   - *Impact on Weather App:* Dashboard and Forecast screens must handle multi-pane responsive layouts on tablets and foldables.

### 2.3 Dependency Stack Matrix (Versions TBD)

Version specifications will be supplied by the repository owner via `libs.versions.toml` during Phase 2:

| Component | Library | Version | Purpose |
| :--- | :--- | :---: | :--- |
| **Language** | Kotlin | TBD, supplied by owner | Language & Compiler |
| **Build Tool** | Android Gradle Plugin (AGP) | TBD, supplied by owner | Build system (compatible with `compileSdk 36`) |
| **Annotation Processing** | KSP (`com.google.devtools.ksp`) | TBD, supplied by owner | High-performance code generation |
| **UI Framework** | Jetpack Compose BOM | TBD, supplied by owner | Material 3, Dynamic Color, UI primitives |
| **Widget Engine** | Jetpack Glance AppWidget | TBD, supplied by owner | Declarative Compose-style RemoteViews |
| **Background Processing** | WorkManager KTX | TBD, supplied by owner | Persistent task scheduling & constraints |
| **Dependency Injection** | Hilt | TBD, supplied by owner | Dependency injection & Hilt Worker integration |
| **Database / Cache** | Room | TBD, supplied by owner | Local SQLite database (weather cache & widget config) |
| **Preferences** | DataStore Preferences | TBD, supplied by owner | User settings (°C/°F, dark theme) |
| **Networking** | Retrofit 2 | TBD, supplied by owner | Type-safe REST client |
| **HTTP Client** | OkHttp 4 | TBD, supplied by owner | HTTP networking & logging interceptor |
| **Serialization** | kotlinx.serialization | TBD, supplied by owner | Pure Kotlin JSON parsing |
| **Location** | Play Services Location | TBD, supplied by owner | Fused Location Provider API |

---

## 3. Architecture & Dependency Injection Wiring

### 3.1 Clean Architecture Layers
```
+-----------------------------------------------------------------------+
|                          PRESENTATION LAYER                           |
|  [Compose Screens / ViewModels]       [Glance Widget & Config Act]    |
+-----------------------------------++----------------------------------+
                                    ||
                                    \/
+-----------------------------------------------------------------------+
|                             DOMAIN LAYER                              |
|  [Use Cases]        [Domain Models]       [Repository Interfaces]     |
+-----------------------------------++----------------------------------+
                                    ||
                                    \/
+-----------------------------------------------------------------------+
|                              DATA LAYER                               |
|  [Room DB Cache]   [Retrofit API]   [DataStore]   [Repositories Impl] |
+-----------------------------------------------------------------------+
```

### 3.2 Hilt Injection Wiring (WorkManager & Glance)
1. **WorkManager Hilt Wiring:**
   - Annotate `RefreshWeatherWorker` with `@HiltWorker` and `@AssistedInject`.
   - Provide `HiltWorkerFactory` in `WeatherApplication` implementing `Configuration.Provider`.
2. **Glance Widget Hilt Wiring:**
   - Glance `AppWidgetReceiver` instances are instantiated by the Android OS framework, not Hilt.
   - Access repositories inside Glance receivers/widgets using a custom `@EntryPoint`:
     ```kotlin
     @EntryPoint
     @InstallIn(SingletonComponent::class)
     interface GlanceDependencies {
         fun weatherRepository(): WeatherRepository
         fun widgetConfigDao(): WidgetConfigDao
         fun userPreferencesRepository(): UserPreferencesRepository
     }
     
     // Inside provideGlance / Receiver:
     val dependencies = EntryPointAccessors.fromApplication(
         context.applicationContext,
         GlanceDependencies::class.java
     )
     ```

### 3.3 Concrete Folder Structure
```
app/src/main/java/com/weatherapp/
├── WeatherApplication.kt            # Application entry point (@HiltAndroidApp, Configuration.Provider)
├── di/                              # Dependency Injection modules
│   ├── NetworkModule.kt             # Retrofit, OkHttp, Open-Meteo API
│   ├── DatabaseModule.kt            # Room DB, DAOs
│   ├── RepositoryModule.kt          # Repository interface bindings
│   └── LocationModule.kt            # FusedLocationProviderClient
├── data/                            # Data Layer
│   ├── api/                         # Retrofit Interfaces & Data Transfer Objects (DTOs)
│   │   ├── OpenMeteoApi.kt
│   │   └── dto/ WeatherResponseDto.kt
│   ├── local/                       # Room Database & DataStore
│   │   ├── WeatherDatabase.kt
│   │   ├── dao/ WeatherDao.kt, LocationDao.kt, WidgetConfigDao.kt
│   │   ├── entity/ WeatherEntity.kt, LocationEntity.kt, WidgetConfigEntity.kt
│   │   └── preferences/ UserPreferencesRepository.kt
│   ├── location/                    # Location service implementation
│   │   └── DefaultLocationTracker.kt
│   ├── mapper/                      # DTO <-> Entity <-> Domain Mappers
│   │   ├── WeatherMapper.kt
│   │   └── LocationMapper.kt
│   └── repository/                  # Data Repository Implementations
│       ├── WeatherRepositoryImpl.kt # Centrally throttles manual refresh requests
│       └── LocationRepositoryImpl.kt
├── domain/                          # Domain Layer (Pure Business Logic)
│   ├── model/                       # Domain Data Models
│   │   ├── WeatherInfo.kt
│   │   ├── CurrentWeather.kt
│   │   ├── HourlyForecast.kt
│   │   ├── DailyForecast.kt
│   │   └── LocationModel.kt
│   ├── repository/                  # Repository Interfaces
│   │   ├── WeatherRepository.kt
│   │   └── LocationRepository.kt
│   └── usecase/                     # Use Cases / Interactors
│       ├── GetWeatherUseCase.kt
│       ├── RefreshWeatherUseCase.kt
│       ├── GetSavedLocationsUseCase.kt
│       └── SearchLocationUseCase.kt
├── presentation/                    # Presentation Layer (Compose UI)
│   ├── dashboard/                   # Weather Dashboard Screen
│   │   ├── DashboardScreen.kt
│   │   ├── DashboardViewModel.kt
│   │   └── DashboardState.kt
│   ├── forecast/                    # Detailed 24h & 7-day Forecast Screen
│   │   ├── ForecastScreen.kt
│   │   └── components/ TempGraph.kt, HourlyItem.kt
│   ├── location/                    # Location Search & Management
│   │   ├── LocationSearchScreen.kt
│   │   └── LocationViewModel.kt
│   ├── settings/                    # App Settings Screen (°C/°F, Theme, Battery guide, Attribution)
│   │   ├── SettingsScreen.kt
│   │   └── SettingsViewModel.kt
│   ├── ui/theme/                    # Material 3 Theme, Color, Type, Shape
│   └── util/                        # UI helpers & formatters
├── widget/                          # Jetpack Glance Responsive Widget
│   ├── WeatherGlanceWidget.kt       # Single responsive GlanceAppWidget definition
│   ├── WeatherGlanceReceiver.kt     # GlanceAppWidgetReceiver (with onDeleted cleanup)
│   ├── WeatherWidgetConfigActivity.kt # Configuration Activity (with RESULT_CANCELED default)
│   ├── GlanceDependencies.kt        # Hilt EntryPoint interface
│   └── layouts/                     # Responsive size breakpoint & style layouts
│       ├── CompactWidgetLayout.kt   # Size Variant 1 (100dp x 100dp)
│       ├── StandardWidgetLayout.kt  # Size Variant 2 (200dp x 120dp)
│       ├── DetailedWidgetLayout.kt  # Size Variant 3 (280dp x 200dp)
│       └── MinimalWidgetLayout.kt   # Style Option Override: Large temp-focused
└── worker/                          # Background WorkManager Workers
    └── RefreshWeatherWorker.kt      # Periodic & One-Time Refresh Worker (@HiltWorker)
```

---

## 4. Room Database Schema & Management

### 4.1 Room Schema Export & Versioning
- **Schema Export Enabled:** `ksp { arg("room.schemaLocation", "$projectDir/schemas") }` configured in `app/build.gradle.kts`.
- **Entity Schemas:**
  - `WeatherEntity`: Stores cached weather metrics, location ID, `lastUpdatedMillis` (epoch timestamp), hourly JSON payload, daily JSON payload. Index on `locationId`.
  - `LocationEntity`: Stores saved locations (`id`, `cityName`, `country`, `latitude`, `longitude`, `isCurrentLocation`).
  - `WidgetConfigEntity`: Stores per-widget configurations (`appWidgetId` PK, `locationId`, `isMinimalStyle`).
- **Migration Strategy:** Version 1 baseline. Future updates will implement explicit `Migration` scripts or Room `AutoMigration`.

---

## 5. Detailed Widget Auto-Refresh Strategy & WorkManager Behavior

### 5.1 Single Source of Truth & Cache-Only Rendering
- **Core Principle:** The widget **never** executes network calls directly.
- **Flow:**
  `Open-Meteo API` → `WeatherRepositoryImpl` → `Room DB Cache` → `Glance Widget Re-render`.
- The widget re-renders solely by querying the Room Database cache.

```
+------------------+     Network      +-----------------------+
|  Open-Meteo API  | ---------------> | WeatherRepositoryImpl |
+------------------+                  +-----------------------+
                                                  |
                                                  | Persist
                                                  v
+------------------+   Re-render UI   +-----------------------+
|  Glance Widget   | <--------------- |   Room Database Cache |
+------------------+                  +-----------------------+
```

### 5.2 WorkManager Scheduling, Repository Manual Refresh Throttle & Retry Behavior
- **Periodic Refresh Worker:**
  - `PeriodicWorkRequestBuilder<RefreshWeatherWorker>(30, TimeUnit.MINUTES)`
  - Constraints: `NetworkType.CONNECTED`.
  - Enqueue Policy: `ExistingPeriodicWorkPolicy.UPDATE` (WorkManager 2.8+) to keep the schedule updated without resetting interval timings.
- **Centralized Manual Refresh Throttle (Repository Layer):**
  - **Single Throttle Point:** The manual refresh throttle (e.g. 10-second debounce window) is implemented directly inside **`WeatherRepositoryImpl`**.
  - **Coverage:** When a user taps the widget's refresh button (triggering a Glance `actionRunCallback<RefreshAction>()`), pulls-to-refresh in the app, or calls `viewModel.refresh()`, all calls pass through `WeatherRepositoryImpl.requestManualRefresh(locationId, force = true)`.
  - `WeatherRepositoryImpl` checks the last manual refresh timestamp in memory/DataStore. If `currentTimeMillis() - lastRefreshMillis < 10_000L`, the refresh request is dropped immediately without enqueuing a WorkManager job or calling the API.
  - **Enqueue Policy:** If allowed, WorkManager enqueues a one-time refresh job using **`ExistingWorkPolicy.KEEP` strictly** (retaining an active worker if one is currently executing).
- **Resize Policy:** Resizing a widget (`onOptionChanged` / `LocalSize.current`) re-evaluates UI layout constraints and re-renders from existing Room cache. **Resizing does NOT trigger a network fetch.**
- **WorkManager Periodic Work Retry & Failure Behavior (Confirmed via Official Docs):**
  - According to official Android WorkManager documentation ([developer.android.com/develop/background-work/background-tasks/persistent/how-to/states](https://developer.android.com/develop/background-work/background-tasks/persistent/how-to/states)):
    - Periodic Work has only **one** terminal state: `CANCELLED`. Terminal states `SUCCEEDED` and `FAILED` apply **only** to one-time work.
    - When `RefreshWeatherWorker` encounters a network failure during a periodic execution and returns `Result.retry()`, WorkManager reschedules that attempt using exponential backoff *within that execution period*.
    - If retry attempts are exhausted or if the worker returns `Result.failure()` (when `runAttemptCount >= 3`), **the current run attempt ends, but the Periodic Work Request itself remains active in `ENQUEUED` state**.
    - WorkManager automatically reschedules the periodic task for the **next periodic interval cycle** (30 minutes later).
    - Periodic work is **never terminated** by `Result.failure()` or exhausted retries.

### 5.3 Absolute Timestamp & Failure Re-Rendering
- **Absolute Timestamp Display:** The widget displays an **absolute time format** (e.g. `10:42 AM` or `Oct 24, 14:15`), avoiding relative strings like "15m ago" which would require continuous UI updates.
- **Re-Rendering on Fetch Failure:** Even if a network fetch attempt fails (e.g., server timeout or no internet connection), `RefreshWeatherWorker` MUST trigger a Glance widget update (`WeatherGlanceWidget().updateAll(context)`). The widget re-reads the Room cache, displays the previous valid weather metrics with its original successful timestamp, and presents a discrete offline/error badge.

---

## 6. Background Execution Limitations & Mitigation

| Limitation | Impact | Mitigation Strategy |
| :--- | :--- | :--- |
| **Doze Mode & App Standby** | Suspends background network calls during deep sleep. | Managed natively by WorkManager maintenance windows. Screen-on triggers are removed to prevent battery drain. Updates defer during sleep; widget displays absolute timestamp cleanly. |
| **OEM Aggressive App Killing** | Manufacturers (Xiaomi, Samsung, OnePlus) kill background workers. | 1. Implement an in-app "Battery Optimization Guide" screen directing users to disable power optimization.<br>2. Provide explicit `Intent` triggers to open system auto-start / battery settings. |
| **Network Unavailability** | Airplane mode or out of cellular range. | Worker checks network constraints. On failure, Room cache is retained, widget re-renders to show offline badge alongside cached metrics. |

---

## 7. GitHub Repository & CI Strategy

### 7.1 Branching Model
- `main`: Production-ready code (tagged e.g. `v1.0.0`).
- `develop`: Integration branch.
- `feature/<name>`: Feature branches merged via Pull Requests or merge commits.
- Enforce **Conventional Commits** (`feat(...)`, `fix(...)`, `docs(...)`, `test(...)`).

### 7.2 GitHub Actions CI Pipeline
- Add `.github/workflows/ci.yml` in Phase 2 / Phase 6.
- Automates `./gradlew assembleDebug` and `./gradlew test` on every PR and commit to `develop` and `main`.

---

## 8. Milestones & Realistic Complexity Estimates

| Phase | Title | Scope & Deliverables | Complexity | Est. Effort |
| :---: | :--- | :--- | :---: | :---: |
| **Phase 1** | **Planning & Specs** | Architecture & technical plan specification (`docs/PLAN.md`). | Low / Med | 3 hrs |
| **Phase 2** | **Project Initialization** | Version catalog, KSP, Hilt, Room schema config, dependencies, `.gitignore`, GitHub CI workflow (`.github/workflows/ci.yml`). | Low / Med | 5 hrs |
| **Phase 3** | **Data Layer & Cache** | Open-Meteo Retrofit client, Geocoder, Room DB, Mappers, Repositories, DataStore, Unit tests for offline caching & repository. | Medium | 10 hrs |
| **Phase 4** | **Main Application UI** | Compose Dashboard, 24h Hourly Forecast, 7-day Forecast Graph, Location Search & Saved Locations, Settings with Battery Guide & Open-Meteo Attribution. | High | 14 hrs |
| **Phase 5** | **Widget Suite** | Single responsive Jetpack Glance widget (`Compact`, `Standard`, `Detailed` layouts + `Minimal` style option), `WeatherWidgetConfigActivity` with `RESULT_CANCELED` default, Room `WidgetConfigEntity`, `onDeleted` cleanup, Hilt EntryPoint, WorkManager worker with failure re-render & retry limits, Worker tests. | High | 12 hrs |
| **Phase 6** | **Testing & Hardening** | Integration testing, edge-case fixes, battery/network optimization, `docs/DEVICE_TEST_CHECKLIST.md`. | Medium | 6 hrs |
| **Phase 7** | **Release Preparation** | App icon, signed APK configuration, release build, updated README/CHANGELOG, GitHub Release tag. | Low / Med | 4 hrs |
| **Total** | | | | **~54 hrs** |

---

## 9. Kotlin vs Java Overview (Key Language Concepts)

Below is a concise comparison of the core Kotlin features used in this project alongside their Java equivalents (max 15 lines):

1. **Data Classes (`data class WeatherEntity(...)`)**
   - *Java Equivalent:* `public record WeatherEntity(...)` (Java 14+) or POJO with getters, setters, `equals()`, `hashCode()`, `toString()`.
2. **Coroutines & Suspended Functions (`suspend fun getWeather()`)**
   - *Java Equivalent:* Asynchronous execution via `ExecutorService`, `CompletableFuture<Weather>`, or RxJava `Single`.
3. **Reactive Streams (`Flow<WeatherInfo>` & `StateFlow`)**
   - *Java Equivalent:* `java.util.concurrent.Flow.Publisher`, RxJava `Flowable`, or `LiveData<WeatherInfo>`.
4. **Sealed Interfaces (`sealed interface Resource<out T>`)**
   - *Java Equivalent:* `public sealed interface Resource permits Success, Error` (Java 17+) or abstract class with private constructor.
5. **Null Safety (`String?` vs `String`)**
   - *Java Equivalent:* `@Nullable` / `@NonNull` annotations, `Optional<String>`, or explicit `if (str != null)` guards.
6. **Extension Functions (`fun Double.toCelsius(): String`)**
   - *Java Equivalent:* Public static utility methods: `public static String toCelsius(double temp)`.
7. **Delegated Properties (`val viewModel: MainViewModel by viewModels()`)**
   - *Java Equivalent:* Explicit initialization helper methods: `new ViewModelProvider(this).get(MainViewModel.class)`.
