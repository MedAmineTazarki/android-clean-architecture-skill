# Android Clean Architecture Skill

A production-ready AI agent skill for building Android and Kotlin Multiplatform (KMP) applications following Clean Architecture patterns.

> **What is this?** A set of structured reference files that AI coding assistants (Gemini, Claude, etc.) can use to produce consistent, production-quality Android code.

## 🎯 Overview

This skill teaches AI agents to build Android/KMP applications with:

- **Clean Architecture** — 3 strict layers (Domain, Data, UI)
- **MVVM/MVI** — Unidirectional data flow with sealed UiStates
- **Kotlin Multiplatform** — Maximum shared code in `commonMain`
- **Dependency Injection** — Koin (KMP-compatible)
- **Networking** — Ktor Client
- **Persistence** — Room KMP / SQLDelight
- **Async** — Coroutines + Flow (StateFlow/SharedFlow)

## 📁 Structure

```
android-clean-architecture-skill/
├── SKILL.md                          # Main skill — patterns & quick reference
├── references/
│   ├── architecture.md               # 3 layers, DataSource, offline-first, expect/actual
│   ├── compose-patterns.md           # State hoisting, shimmer, performance, theme
│   ├── modularization.md             # Single vs multi-module, KMP structure
│   ├── gradle-setup.md               # Version catalog, Ktor config, bundles
│   └── testing.md                    # Fakes, ViewModel/Mapper/Repo tests, Turbine
└── showcase.html                     # Interactive visual showcase page
```

## 🚀 Installation

### For Gemini Code Assist / Antigravity IDE

Copy the skill folder into your global skills directory:

```bash
# Windows
xcopy /E /I . "%USERPROFILE%\.gemini\config\skills\android-clean-architecture"

# macOS/Linux
cp -r . ~/.gemini/config/skills/android-clean-architecture
```

### For Claude Code

Copy into your Claude skills directory:

```bash
cp -r . ~/.claude/skills/android-clean-architecture
```

The skill auto-activates when you work on Android/KMP projects.

## 🏗️ Architecture

```
┌─────────────────────────────────────────┐
│              UI Layer                    │
│  Compose Screens (stateless)             │
│  + ViewModels (state holders)            │
│  + UiState sealed classes                │
├─────────────────────────────────────────┤
│           Domain Layer                   │
│  Pure Kotlin models (data class)         │
│  Repository interfaces                   │
│  Use Cases (optional, for shared logic)  │
├─────────────────────────────────────────┤
│            Data Layer                    │
│  Repository implementations              │
│  DTOs (@Serializable)                    │
│  Mappers (DTO ↔ Domain)                  │
│  API services (Ktor)                     │
│  Local storage (Room/SQLDelight)         │
└─────────────────────────────────────────┘
```

**Dependency Rule:** `UI → Domain ← Data` — Domain knows nothing about UI or Data.

## 📦 Package Structure

```
com.example.app/
├── domain/
│   ├── model/          # Pure Kotlin data classes
│   ├── repository/     # Interfaces only
│   └── usecase/        # Optional — shared business logic
├── data/
│   ├── remote/
│   │   ├── dto/        # @Serializable DTOs
│   │   └── mapper/     # DTO → Domain
│   ├── local/
│   │   ├── dao/        # Room DAOs / SQLDelight
│   │   └── entity/     # DB entities
│   └── repository/     # Concrete implementations
├── ui/
│   ├── screens/        # @Composable (stateless)
│   ├── viewmodel/      # State holders
│   ├── model/          # UiState sealed classes
│   └── components/     # Reusable composables
├── di/                 # Koin modules
└── theme/              # Colors, Typography
```

## ⚡ Key Patterns

### ViewModel
```kotlin
class MatchesViewModel(
    private val repository: MatchRepository,  // domain interface
) : ViewModel() {
    val uiState = MutableStateFlow<UiState>(Loading)

    fun loadMatches(date: LocalDate) {
        viewModelScope.launch {
            repository.getMatches(date)
                .onSuccess { uiState.value = Success(it) }
                .onFailure { uiState.value = Error(it.message) }
        }
    }
}
```

### UseCase
```kotlin
class GetLiveMatchesUseCase(
    private val repository: MatchRepository,
) {
    suspend operator fun invoke(): Result<List<Match>> {
        return runCatching { repository.getLiveMatches() }
    }
}
```

### Koin DI
```kotlin
val repositoryModule = module {
    single<MatchRepository> { MatchRepositoryImpl(get()) }
}
val viewModelModule = module {
    viewModel { MatchesViewModel(get()) }
}
```

## ❌ Anti-Patterns to Avoid

| Don't | Do |
|---|---|
| Import `data.*` in UI layer | Import only `domain.model.*` |
| Business logic in `@Composable` | Extract to ViewModel or UseCase |
| Use DTOs as domain models | Separate models + mappers |
| Hilt/Dagger in KMP | Koin (multiplatform) |
| Retrofit in KMP | Ktor Client |
| `GlobalScope` | `viewModelScope` |
| ViewModel in `screens/` | Separate `viewmodel/` package |

## 🎨 Showcase

Open `showcase.html` in your browser for an interactive visual reference of all patterns, with:
- Architecture diagram with hover effects
- 12 pattern cards with Feather Icons
- Syntax-highlighted Kotlin code examples
- UDF data flow visualization
- Interactive feature checklist

## 📄 License

MIT — Use freely in personal and commercial projects.

## 🙏 Credits

Patterns inspired by:
- [NowInAndroid](https://github.com/android/nowinandroid) by Google
- [Android Architecture Guide](https://developer.android.com/topic/architecture)
- [Kotlin Multiplatform](https://kotlinlang.org/docs/multiplatform.html)
