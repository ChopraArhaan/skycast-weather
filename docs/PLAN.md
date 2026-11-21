# Phase 1: Technical Plan & Architecture Specification
**Project:** Production-Grade Android Weather App with Smart Auto-Updating Widget  
**Document Version:** 1.0  
**Status:** Pending Approval  

---

## 1. Technical Feasibility Assessment

### 1.1 Weather API: Open-Meteo
- **Feasibility:** High.
- **Key Advantages:**
  - Free non-commercial tier with up to 10,000 API requests/day without requiring an API key.
  - Returns raw, un-opinionated JSON for current conditions, hourly forecasts (up to 7 days), and daily forecasts (up to 16 days).
  - High accuracy using WMO weather interpretation codes (`weathercode`), temperature, feels-like (`apparent_temperature`), relative humidity, surface pressure, wind speed/direction, UV index, visibility, precipitation probability, and sunrise/sunset times.
  - No secrets or secret keys to leak or manage in repository.
- **Constraints & Mitigation:**
  - WMO weather codes require client-side mapping to localized condition strings and Material icons.
  - Requires reverse-geocoding (using Android System `Geocoder` API or Open-Meteo Geocoding API) to map latitude/longitude to city names.

### 1.2 Widget Framework: Jetpack Glance (`androidx.glance:glance-appwidget`)
- **Feasibility:** High.
- **Key Advantages:**
  - Modern Kotlin-first framework built on top of Jetpack Compose primitives to generate `RemoteViews`.
  - Simplifies widget UI development across different screen densities and dimensions.
  - Integrates seamlessly with Jetpack DataStore / Room cache via `GlanceStateDefinition` or direct cache queries inside `provideGlance`.
- **Constraints & Mitigation:**
  - Glance UI capabilities are a subset of standard Jetpack Compose (e.g. limited custom drawing/Canvas directly in RemoteViews).
  - Four distinct widget layout variants (Compact, Standard, Detailed, Minimal) will be defined using Glance `SizeMode.Responsive` or explicit layout definitions to handle flexible resizing cleanly.

### 1.3 Background Scheduling: WorkManager
- **Feasibility:** High.
- **Key Advantages:**
  - De-facto standard for deferrable, guaranteed background work on Android.
  - Respects system Doze mode, battery saver, and network connectivity constraints.
  - PeriodicWorkRequest enforces a minimum interval of 15 minutes, perfectly aligning with our ~30-minute target auto-refresh interval.

### 1.4 Location Handling: Fused Location Provider API
- **Feasibility:** High.
- **Key Advantages:**
  - Google Play Services Fused Location Provider API delivers fast, low-power location fixes.
  - Graceful fallback to user-managed saved locations when location permission is denied or device location is disabled.

---

## 2. Recommended Tech Stack & Minimum SDK

### 2.1 Minimum SDK Recommendation
- **Recommended `minSdk`:** **26 (Android 8.0 Oreo)**
- **Target / Compile `targetSdk`:** **35 (Android 15)**
- **Reasoning:**
  1. **Market Reach:** Android 8.0+ covers >95% of active Android devices worldwide.
  2. **System Capabilities:** Min SDK 26 guarantees native support for Notification Channels, modern `java.time` APIs (without heavy desugaring overhead), and stable JobScheduler/WorkManager background constraints.
  3. **Library Compatibility:** Full compatibility with Jetpack Compose 1.7+, Jetpack Glance 1.1+, Hilt 2.50+, and Material 3 without backward-compatibility workarounds required for legacy Android 7.x/6.x.

### 2.2 Tech Stack Matrix
| Component | Technology / Library | Purpose |
| :--- | :--- | :--- |
| **Language** | Kotlin 2.0+ | Modern, safe, idiomatic async programming |
| **UI Framework** | Jetpack Compose + Material 3 | Declarative UI, Dynamic Color (Material You) |
| **Widget Engine** | Jetpack Glance (`androidx.glance`) | Compose-style RemoteViews for home-screen widgets |
| **Architecture** | MVVM + Clean Architecture | Strict separation of Data, Domain, and Presentation layers |
| **Dependency Injection** | Hilt (`com.google.dagger:hilt-android`) | Standardized dependency management and scoping |
| **Networking** | Retrofit 2 + OkHttp 4 + kotlinx.serialization | REST API client & type-safe JSON parsing |
| **Database / Cache** | Room Database | Local weather cache and saved locations storage |
| **Preferences** | Jetpack DataStore (Preferences) | Storing user settings (temperature unit °C/°F, dark theme, active location) |
| **Async & Streams** | Kotlin Coroutines + Flow / StateFlow | Reactive data flow and asynchronous operations |
| **Background Tasks** | WorkManager (`androidx.work`) | Periodic and on-demand weather data fetching |
| **Location** | Fused Location Provider API | High-accuracy and low-power GPS location updates |
| **Async Image Loading** | Coil (`io.coil-kt:coil-compose`) | Efficient icon rendering and caching |

---

## 3. Architecture & Folder Structure

### 3.1 Clean Architecture Overview
The application strictly enforces a 3-layer architecture:
- **Presentation Layer:** Jetpack Compose UI (Dashboard, Forecast, Search, Settings), ViewModels with `StateFlow`, and Jetpack Glance Widget implementations.
- **Domain Layer:** Pure Kotlin modules containing Business Logic, Domain Models (`WeatherInfo`, `CurrentWeather`, `HourlyForecast`, `DailyForecast`), Use Cases, and Repository Interfaces. No Android framework dependencies.
- **Data Layer:** Room Entities, Retrofit API Services, OkHttp client, DataStore Manager, Geocoder Service, Mapper classes, and Repository Implementations.

```
+-----------------------------------------------------------------------+
|                          PRESENTATION LAYER                           |
|  [Compose Screens / ViewModels]          [Glance Widgets & Receivers] |
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
|  [Room Cache DB]   [Retrofit API]   [DataStore]   [Repositories Impl] |
+-----------------------------------------------------------------------+
```

### 3.2 Concrete Folder Structure
```
app/src/main/java/com/weatherapp/
├── WeatherApplication.kt            # Application entry point (@HiltAndroidApp)
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
│   ├── settings/                    # App Settings Screen (°C/°F, Theme, Battery guide)
│   │   ├── SettingsScreen.kt
│   │   └── SettingsViewModel.kt
│   ├── ui/theme/                    # Material 3 Theme, Color, Type, Shape
│   └── util/                        # UI helpers & formatters
├── widget/                          # Jetpack Glance Widgets
│   ├── WeatherGlanceWidget.kt       # Primary Glance AppWidget definition
│   ├── WeatherGlanceReceiver.kt     # GlanceAppWidgetReceiver
│   ├── variants/                    # Widget Layout Variants
│   │   ├── CompactWidgetLayout.kt   # Variant 1: Temp, icon, location
│   │   ├── StandardWidgetLayout.kt  # Variant 2: Temp, condition, high/low
│   │   ├── DetailedWidgetLayout.kt  # Variant 3: Temp, feels-like, humidity, wind
│   │   └── MinimalWidgetLayout.kt   # Variant 4: Large temp-focused
│   └── update/                      # Widget update helpers
│       └── WidgetUpdater.kt
└── worker/                          # Background WorkManager Workers
    ├── RefreshWeatherWorker.kt      # Periodic & One-Time Refresh Worker
    └── BootCompletedReceiver.kt    # System Boot Receiver
```

---

## 4. Detailed Widget Auto-Refresh Strategy

### 4.1 Single Source of Truth & Cache-Only Widgets
- **Core Principle:** The widget **never** makes network requests directly.
- **Flow:**
  `Open-Meteo API` → `WeatherRepositoryImpl` → `Room Database Cache` → `Glance Widget Update`.
- The widget re-renders solely by querying the Room Database cache (or DataStore). This ensures instantaneous rendering, zero network lag when rendering RemoteViews, and consistent state between app and widget.

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

### 4.2 WorkManager Scheduling Strategy
- **Periodic Refresh:**
  - `PeriodicWorkRequestBuilder<RefreshWeatherWorker>(30, TimeUnit.MINUTES)`
  - Constraints: `NetworkType.CONNECTED`.
  - Enqueued with `ExistingPeriodicWorkPolicy.KEEP` to ensure a stable single periodic worker.
- **One-Time Immediate Refresh:**
  - Triggered via `OneTimeWorkRequestBuilder<RefreshWeatherWorker>()`.
  - Executed on:
    1. Widget added to home screen (`GlanceAppWidget.provideGlance` / `onEnabled`).
    2. Widget resized (`onOptionChanged`).
    3. User clicks manual refresh button on widget.
    4. System reboot completion (`ACTION_BOOT_COMPLETED`).
    5. App installation update (`ACTION_MY_PACKAGE_REPLACED`).
    6. Active location updated in-app.
  - Policy: `ExistingWorkPolicy.REPLACE` to deduplicate overlapping requests.

### 4.3 Network Failures, Retries & Deduplication
- **Exponential Backoff:** Configured on WorkManager with `BackoffPolicy.EXPONENTIAL` starting at 5 minutes delay. Max retry limit set to 3 attempts.
- **Request Deduplication:** Network requests are batched per active location ID. Even if multiple widget variants exist on the home screen, only **one** API request is dispatched per location.

### 4.4 Stale Data Handling
- **Thresholds:**
  - **Fresh Data:** Data updated < 1 hour ago.
  - **Stale Data:** Data updated between 1 to 4 hours ago. Widget displays "Updated Xm ago" or "Updated Xh ago".
  - **Severe Stale Data (> 4 hours):** Widget displays a warning indicator icon next to last-updated timestamp (e.g. "Stale • 10:30 AM").
- The last-updated timestamp is clearly displayed across all widget variants.

---

## 5. Background Execution Limitations & Mitigation

### 5.1 Android System Constraints & Solutions
| Limitation | Impact | Mitigation Strategy |
| :--- | :--- | :--- |
| **Doze Mode & Standby** | Suspends background network operations during deep sleep. | WorkManager handles Doze execution windows. Accept that updates defer during sleep; update immediately when screen turns on or device wakes. Show clear "last updated" time. |
| **OEM Aggressive App Killing** | Manufacturers (Xiaomi, Samsung, OnePlus, Huawei) stop background workers aggressively. | 1. Implement an in-app "Battery Optimization Guide" screen directing users to disable power optimization for the app.<br>2. Provide explicit `Intent` triggers to open system auto-start / battery settings. |
| **Network Unavailability** | Device in Airplane mode or out of cellular coverage. | Worker checks network availability via WorkManager constraints. On failure, cached data is retained without overwriting valid data; UI flags offline status gracefully. |
| **Android 12+ Background Start Restrictions** | Prevents starting Foreground Services from the background. | Use standard WorkManager background workers instead of foreground services. |

---

## 6. GitHub Repository Setup & Branching Strategy

### 6.1 Branching Model (Git Flow Variant)
- **`main`:** Contains production-ready code. Tagged with semantic versions (e.g. `v1.0.0`).
- **`develop`:** Integration branch for active feature development and testing. All feature branches originate from and merge back into `develop`.
- **`feature/<feature-name>`:** Individual feature development branches (e.g., `feature/data-layer`, `feature/glance-widget`). Merged into `develop` via pull request or merge commit after build and test pass.

### 6.2 Commit Messages
- Enforce **Conventional Commits**:
  - `feat(data): add Open-Meteo Retrofit client and DTOs`
  - `fix(widget): resolve stale timestamp formatting bug`
  - `docs(plan): add architecture specification`
  - `test(worker): add unit test for RefreshWeatherWorker`

---

## 7. Milestones & Complexity Estimates

| Phase | Title | Scope & Deliverables | Complexity | Est. Effort |
| :---: | :--- | :--- | :---: | :---: |
| **Phase 1** | **Planning & Specs** | Architecture document (`docs/PLAN.md`), technical feasibility, Git setup. | Low / Med | 2 hrs |
| **Phase 2** | **Project Initialization** | Gradle Version Catalog (`libs.versions.toml`), Hilt setup, dependencies, `.gitignore`, initial empty build. | Low | 3 hrs |
| **Phase 3** | **Data Layer & Cache** | Open-Meteo Retrofit client, Room DB, Mappers, Repositories, DataStore, Unit tests for offline caching. | Medium | 8 hrs |
| **Phase 4** | **Main Application UI** | Compose Dashboard, 24h Hourly Forecast, 7-day Forecast Graph, Location Search & Saved Locations, Permissions, Settings, Theme Engine. | High | 12 hrs |
| **Phase 5** | **Widget Suite** | Jetpack Glance 4 widget variants (Compact, Standard, Detailed, Minimal), WorkManager refresh strategy, Boot receiver, Stale data indicators, Worker unit tests. | High | 10 hrs |
| **Phase 6** | **Testing & Hardening** | Integration testing, edge-case fixes, battery/network optimization, `docs/DEVICE_TEST_CHECKLIST.md`. | Medium | 5 hrs |
| **Phase 7** | **Release Preparation** | App icon, signed APK configuration, release build, updated README/CHANGELOG, GitHub Release tag. | Low / Med | 3 hrs |

---

## 8. Kotlin vs Java Overview (Key Language Concepts)

Below is a concise comparison of the core Kotlin features used in this project alongside their Java equivalents:

1. **Data Classes (`data class WeatherEntity(...)`)**
   - *Java Equivalent:* `public record WeatherEntity(...)` (Java 14+) or traditional POJO with getters, setters, `equals()`, `hashCode()`, and `toString()`.
2. **Coroutines & Suspended Functions (`suspend fun getWeather()`)**
   - *Java Equivalent:* Asynchronous execution via `ExecutorService`, `CompletableFuture<Weather>`, or RxJava `Single`/`Observable`.
3. **Reactive Streams (`Flow<WeatherInfo>` & `StateFlow`)**
   - *Java Equivalent:* `java.util.concurrent.Flow.Publisher`, RxJava `Flowable`, or `LiveData<WeatherInfo>`.
4. **Sealed Interfaces (`sealed interface Resource<out T>`)**
   - *Java Equivalent:* `public sealed interface Resource permits Success, Error` (Java 17+) or abstract class with private constructor.
5. **Null Safety (`String?` vs `String`)**
   - *Java Equivalent:* `@Nullable` / `@NonNull` annotations, `Optional<String>`, or explicit `if (str != null)` guards.
6. **Extension Functions (`fun Double.toCelsius(): String`)**
   - *Java Equivalent:* Public static utility methods: `public static String toCelsius(double temp)`.
7. **Default Parameter Values (`fun fetch(locationId: String = "default")`)**
   - *Java Equivalent:* Method overloading: `public void fetch()` calling `public void fetch(String locationId)`.
8. **Delegated Properties (`val viewModel: MainViewModel by viewModels()`)**
   - *Java Equivalent:* Explicit initialization helper methods: `new ViewModelProvider(this).get(MainViewModel.class)`.
