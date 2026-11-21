# Weather App with Smart Auto-Updating Widget

A production-grade Android Weather Application featuring a smart home-screen widget, built using Kotlin, Jetpack Compose, Material 3, Clean Architecture, and WorkManager.

---

## Architecture & Tech Stack
- **Architecture:** MVVM + Clean Architecture (Presentation, Domain, Data)
- **UI Framework:** Jetpack Compose + Material 3
- **Widget Engine:** Jetpack Glance AppWidget (`androidx.glance`)
- **Background Tasks:** WorkManager (`androidx.work`) with `@HiltWorker`
- **Dependency Injection:** Hilt (`com.google.dagger:hilt-android`) with KSP
- **Database / Local Cache:** Room Database (`androidx.room`) with Room schema export
- **Preferences:** Jetpack DataStore Preferences
- **Networking:** Retrofit 2 + OkHttp 4 + kotlinx.serialization
- **Location Services:** Google Play Services Fused Location Provider
- **Weather API:** Open-Meteo API (Free tier, no API key required)

---

## Setup & Build Instructions

### Prerequisites
- **Android Studio:** Android Studio Ladybug (2024.2) or newer.
- **JDK:** Java 17 or higher.
- **Android SDK:** `compileSdk = 36`, `targetSdk = 36`, `minSdk = 26`.

### Building the App
1. Clone the repository:
   ```bash
   git clone https://github.com/<username>/weather-app.git
   cd weather-app
   ```
2. Build Debug APK:
   ```bash
   ./gradlew assembleDebug
   ```
3. Run Unit Tests:
   ```bash
   ./gradlew test
   ```

---

## Open-Meteo Attribution
Weather data provided by [Open-Meteo.com](https://open-meteo.com/) under the CC BY 4.0 license.
