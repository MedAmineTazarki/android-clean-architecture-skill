# Gradle Setup — KMP Build Configuration

## Version Catalog (libs.versions.toml)

```toml
[versions]
kotlin = "2.1.0"
agp = "8.7.3"
compose-multiplatform = "1.7.3"
compose-bom = "2024.12.01"
koin-bom = "4.0.4"
ktor = "3.0.3"
room = "2.7.1"
sqldelight = "2.0.2"
coroutines = "1.9.0"
serialization = "1.7.3"
coil = "3.0.4"
lifecycle = "2.8.7"
navigation = "2.8.5"

[libraries]
# Compose BOM
androidx-compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "compose-bom" }
androidx-compose-material3 = { group = "androidx.compose.material3", name = "material3" }
androidx-compose-ui-tooling-preview = { group = "androidx.compose.ui", name = "ui-tooling-preview" }

# Lifecycle
androidx-lifecycle-viewmodel = { group = "androidx.lifecycle", name = "lifecycle-viewmodel-compose", version.ref = "lifecycle" }
androidx-lifecycle-runtime-compose = { group = "androidx.lifecycle", name = "lifecycle-runtime-compose", version.ref = "lifecycle" }

# Navigation
androidx-navigation-compose = { group = "androidx.navigation", name = "navigation-compose", version.ref = "navigation" }

# Koin (DI)
koin-bom = { group = "io.insert-koin", name = "koin-bom", version.ref = "koin-bom" }
koin-core = { group = "io.insert-koin", name = "koin-core" }
koin-android = { group = "io.insert-koin", name = "koin-android" }
koin-compose = { group = "io.insert-koin", name = "koin-androidx-compose" }

# Ktor (Networking)
ktor-client-core = { group = "io.ktor", name = "ktor-client-core", version.ref = "ktor" }
ktor-client-okhttp = { group = "io.ktor", name = "ktor-client-okhttp", version.ref = "ktor" }
ktor-client-content-negotiation = { group = "io.ktor", name = "ktor-client-content-negotiation", version.ref = "ktor" }
ktor-serialization-json = { group = "io.ktor", name = "ktor-serialization-kotlinx-json", version.ref = "ktor" }
ktor-client-logging = { group = "io.ktor", name = "ktor-client-logging", version.ref = "ktor" }

# Serialization
kotlinx-serialization-json = { group = "org.jetbrains.kotlinx", name = "kotlinx-serialization-json", version.ref = "serialization" }

# Room (KMP)
room-runtime = { group = "androidx.room", name = "room-runtime", version.ref = "room" }
room-compiler = { group = "androidx.room", name = "room-compiler", version.ref = "room" }
room-ktx = { group = "androidx.room", name = "room-ktx", version.ref = "room" }

# Coil (Image Loading)
coil-compose = { group = "io.coil-kt.coil3", name = "coil-compose", version.ref = "coil" }
coil-network-okhttp = { group = "io.coil-kt.coil3", name = "coil-network-okhttp", version.ref = "coil" }

# Coroutines
kotlinx-coroutines-core = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-core", version.ref = "coroutines" }
kotlinx-coroutines-android = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-android", version.ref = "coroutines" }

# Testing
junit = { group = "junit", name = "junit", version = "4.13.2" }
kotlinx-coroutines-test = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-test", version.ref = "coroutines" }
turbine = { group = "app.cash.turbine", name = "turbine", version = "1.2.0" }

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
android-library = { id = "com.android.library", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-multiplatform = { id = "org.jetbrains.kotlin.multiplatform", version.ref = "kotlin" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
compose-compiler = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
compose-multiplatform = { id = "org.jetbrains.compose", version.ref = "compose-multiplatform" }
room = { id = "androidx.room", version.ref = "room" }

[bundles]
compose = [
    "androidx-compose-material3",
    "androidx-compose-ui-tooling-preview",
    "androidx-lifecycle-viewmodel",
    "androidx-lifecycle-runtime-compose",
]
ktor = [
    "ktor-client-core",
    "ktor-client-okhttp",
    "ktor-client-content-negotiation",
    "ktor-serialization-json",
    "ktor-client-logging",
]
koin = [
    "koin-core",
    "koin-android",
    "koin-compose",
]
```

## App build.gradle.kts

```kotlin
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.kotlin.serialization)
    alias(libs.plugins.compose.compiler)
}

android {
    namespace = "com.example.app"
    compileSdk = 35

    defaultConfig {
        applicationId = "com.example.app"
        minSdk = 26
        targetSdk = 35
        versionCode = 1
        versionName = "1.0"
    }

    buildFeatures { compose = true }

    kotlin { jvmToolchain(17) }
}

dependencies {
    // Compose
    implementation(platform(libs.androidx.compose.bom))
    implementation(libs.bundles.compose)

    // Koin
    implementation(platform(libs.koin.bom))
    implementation(libs.bundles.koin)

    // Ktor
    implementation(libs.bundles.ktor)

    // Serialization
    implementation(libs.kotlinx.serialization.json)

    // Image loading
    implementation(libs.coil.compose)
    implementation(libs.coil.network.okhttp)

    // Navigation
    implementation(libs.androidx.navigation.compose)

    // Testing
    testImplementation(libs.junit)
    testImplementation(libs.kotlinx.coroutines.test)
    testImplementation(libs.turbine)
}
```

## Ktor Client Setup

```kotlin
// di/AppModules.kt — networkModule
single {
    HttpClient(OkHttp) {
        install(ContentNegotiation) {
            json(Json {
                ignoreUnknownKeys = true
                coerceInputValues = true
                prettyPrint = false
            })
        }
        install(Logging) {
            level = LogLevel.HEADERS
        }
        defaultRequest {
            url("https://your-api-base-url.com/")
            contentType(ContentType.Application.Json)
        }
    }
}
```

## Retrofit Alternative (Android-only projects)

If not using KMP and preferring Retrofit:

```kotlin
single {
    Retrofit.Builder()
        .baseUrl(BASE_URL)
        .client(get<OkHttpClient>())
        .addConverterFactory(get<Json>().asConverterFactory("application/json".toMediaType()))
        .build()
}

single<ApiService> {
    get<Retrofit>().create(ApiService::class.java)
}
```
