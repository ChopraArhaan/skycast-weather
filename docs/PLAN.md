# Phase 1: Technical Plan & Architecture Specification
**Project:** Production-Grade Android Weather App with Smart Auto-Updating Widget  
**Document Version:** 1.1  
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
- **API Efficiency Rules:**
  - **Single Request per Location:** Batch and deduplicate requests so that only one API network request is dispatched per unique location ID, regardless of how many widget instances exist.
  - **Cache First:** Always query Room cache before making a network call. If cached data for a location is less than 15 minutes old, return the cache without triggering a network request.
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

### 1.3 Widget Framework: Jetpack Glance (`androidx.glance:glance-appwidget`)
- **Feasibility:** High.
- **Architecture Decision (Single Responsive Widget vs 4 Separate Widgets):**
  - **Decision:** Implement **a single `GlanceAppWidget`** using `SizeMode.Responsive` rather than 4 separate widget items in the launcher widget picker.
  - **Justification:** Providing a single responsive widget simplifies the user experience in the home-screen widget selector while allowing a single placed widget to dynamically switch between 4 layout variants (`Compact`, `Standard`, `Detailed`, `Minimal`) based on `LocalSize.current` width and height bounds as the user resizes it on their home screen.
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
  - Each widget instance can be configured via a Widget Configuration Activity (or global preference) to target either **"Current Location"** or a specific **"Saved Location"** (e.g., London).
  - For "Current Location", the app/worker uses the **last-known coordinates** captured when the user last opened the main app (stored in DataStore / Room).
  - **Background Location (`ACCESS_BACKGROUND_LOCATION`) is OPTIONAL in v1.** The app does not require background location permission. If location is unavailable or denied, the widget seamlessly falls back to the user's primary saved location.

---

## 2. Recommended Tech Stack & Verified Versions

### 2.1 Minimum SDK & Target SDK
- **Recommended `minSdk`:** **26 (Android 8.0 Oreo)** — Covers ~95%+ of active devices globally. Guarantees native support for Notification Channels, `java.time` APIs without desugaring overhead, and modern WorkManager constraints.
- **Target / Compile `targetSdk`:** **35 (Android 15)** — Complies with Google Play's mandatory `targetSdk` requirements for 2024/2025.

### 2.2 Verified Dependency Stack (as of 2024/2025)
| Component | Library | Verified Version | Purpose |
| :--- | :--- | :--- | :--- |
| **Language** | Kotlin | `2.0.21` | Language & Compiler |
| **Build Tool** | Android Gradle Plugin (AGP) | `8.7.3` | Build system |
| **Annotation Processing** | KSP (`com.google.devtools.ksp`) | `2.0.21-1.0.27` | High-performance code generation (replaces kapt) |
| **UI Framework** | Jetpack Compose BOM | `2024.10.01` | Material 3, Canvas, Dynamic Color |
| **Widget Engine** | Jetpack Glance AppWidget | `1.1.1` | Declarative Compose-style RemoteViews |
| **Background Processing** | WorkManager KTX | `2.10.0` | Background task scheduling |
| **Dependency Injection** | Hilt | `2.52` | Dependency injection & Hilt Worker integration |
| **Database / Cache** | Room | `2.6.1` | Local SQLite weather cache & saved locations |
| **Preferences** | DataStore Preferences | `1.1.1` | User settings (°C/°F, dark theme, active location) |
| **Networking** | Retrofit 2 + OkHttp 4 | `2.11.0` / `4.12.0` | Type-safe REST client |
| **Serialization** | kotlinx.serialization | `1.7.3` | Pure Kotlin JSON parsing |
| **Location** | Play Services Location | `21.3.0` | Fused Location Provider API |

---

## 3. Architecture & Dependency Injection Wiring

### 3.1 Clean Architecture Layers
```
+-----------------------------------------------------------------------+
|                          PRESENTATION LAYER                           |
|  [Compose Screens / ViewModels]       [Glance Widget & Receivers]     |
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
         fun userPreferencesRepository(): UserPreferencesRepository
     }
     
     // Inside provideGlance / Receiver:
     val repository = EntryPointAccessors.fromApplication(
         context.applicationContext,
         GlanceDependencies::class.java
     ).weatherRepository()
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
│   │   ├── dao/ WeatherDao.kt, LocationDao.kt
│   │   ├── entity/ WeatherEntity.kt, LocationEntity.kt
│   │   └── preferences/ UserPreferencesRepository.kt
│   ├── location/                    # Location service implementation
│   │   └── DefaultLocationTracker.kt
│   ├── mapper/                      # DTO <-> Entity <-> Domain Mappers
│   │   ├── WeatherMapper.kt
│   │   └── LocationMapper.kt
│   └── repository/                  # Data Repository Implementations
│       ├── WeatherRepositoryImpl.kt
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
│   ├── WeatherGlanceReceiver.kt     # GlanceAppWidgetReceiver
│   ├── GlanceDependencies.kt        # Hilt EntryPoint interface
│   └── layouts/                     # Responsive size breakpoint layouts
│       ├── CompactWidgetLayout.kt   # Variant 1: Temp, icon, location
│       ├── StandardWidgetLayout.kt  # Variant 2: Temp, condition, high/low
│       ├── DetailedWidgetLayout.kt  # Variant 3: Temp, feels-like, humidity, wind
│       └── MinimalWidgetLayout.kt   # Variant 4: Large temp-focused
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
- **Migration Strategy:** Version 1 baseline. Future updates will implement explicit `Migration` scripts or Room `AutoMigration`.

---

## 5. Detailed Widget Auto-Refresh Strategy

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

### 5.2 WorkManager Enqueue Policies & Retries
- **Periodic Refresh Worker:**
  - `PeriodicWorkRequestBuilder<RefreshWeatherWorker>(30, TimeUnit.MINUTES)`
  - Constraints: `NetworkType.CONNECTED`.
  - Enqueue Policy: `ExistingPeriodicWorkPolicy.UPDATE` (WorkManager 2.8+) to keep the schedule updated without resetting interval timings.
- **One-Time Manual / Immediate Refresh:**
  - Triggered on: Widget added to home screen, user clicks manual refresh button on widget, active location changed in-app.
  - Enqueue Policy: `ExistingWorkPolicy.KEEP` (or `REPLACE` for explicit manual button tap) to prevent redundant duplicate network requests if a worker is already running.
- **Resize Policy:** Widget resizing (`onOptionChanged` / `LocalSize.current`) re-evaluates UI layout constraints and re-renders from the existing Room cache. **Resizing does NOT trigger a network fetch.**
- **Boot Receiver Note:** Dropped as unnecessary. WorkManager automatically re-schedules periodic work across system reboots.
- **Retry Policy & Max Attempt Limit:**
  - Backoff policy: `BackoffPolicy.EXPONENTIAL` (initial delay 5 minutes).
  - Max retry tracking: Enforced manually inside `RefreshWeatherWorker.doWork()` by inspecting `runAttemptCount`. If `runAttemptCount >= 3`, return `Result.failure()` to stop retrying until the next 30-minute periodic cycle.

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
| **Phase 5** | **Widget Suite** | Single responsive Jetpack Glance widget with 4 size layouts (`Compact`, `Standard`, `Detailed`, `Minimal`), Hilt EntryPoint, WorkManager worker with failure re-render & retry limits, Worker tests. | High | 12 hrs |
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
