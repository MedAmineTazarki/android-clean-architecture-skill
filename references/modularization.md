# Modularization — Project Structure

## Single Module (Small/Medium Projects)

For projects up to ~30 screens, a single app module with clean package structure is sufficient:

```
app/
└── src/main/java/com/example/app/
    ├── domain/
    │   ├── model/
    │   └── repository/
    ├── data/
    │   ├── remote/
    │   │   ├── dto/
    │   │   └── mapper/
    │   ├── local/
    │   │   ├── dao/
    │   │   └── entity/
    │   └── repository/
    ├── ui/
    │   ├── screens/
    │   ├── viewmodel/
    │   ├── model/
    │   ├── components/
    │   └── navigation/
    ├── di/
    └── theme/
```

## Multi-Module (Large Projects)

For larger projects, split into Gradle modules:

```
settings.gradle.kts:
  include(":app")
  include(":core:model")
  include(":core:data")
  include(":core:database")
  include(":core:network")
  include(":core:ui")
  include(":core:designsystem")
  include(":core:common")
  include(":feature:matches")
  include(":feature:news")
  include(":feature:standings")
  include(":feature:settings")
```

### Module Types

| Module | Contains | Depends On |
|--------|----------|------------|
| `app` | Navigation, Application class, DI root | All feature modules, core modules |
| `core:model` | Domain models (pure Kotlin) | Nothing |
| `core:data` | Repository impls, mappers | `core:model`, `core:network`, `core:database` |
| `core:network` | Ktor client, DTOs, API service | `core:model` |
| `core:database` | Room/SQLDelight, DAOs, entities | `core:model` |
| `core:ui` | Reusable composables | `core:designsystem`, `core:model` |
| `core:designsystem` | Theme, colors, typography, icons | Nothing |
| `core:common` | Shared utilities, extensions | Nothing |
| `feature:*` | Screen + ViewModel + UiState | `core:model`, `core:data`, `core:ui` |

### Dependency Rules

```
feature:matches ──→ core:model
                ──→ core:data
                ──→ core:ui

core:data ──→ core:model
          ──→ core:network
          ──→ core:database

core:network ──→ core:model
core:database ──→ core:model
core:ui ──→ core:designsystem
        ──→ core:model
```

**NEVER:**
- Feature → Feature (use navigation instead)
- core:model → anything (it's the innermost layer)
- core:network → core:database (they're parallel, not stacked)

## KMP Module Structure

For Kotlin Multiplatform projects:

```
shared/
└── src/
    ├── commonMain/kotlin/     # 95%+ of code goes here
    │   └── com/example/app/
    │       ├── domain/
    │       ├── data/
    │       └── di/
    ├── androidMain/kotlin/     # Android-specific (expect/actual)
    ├── iosMain/kotlin/         # iOS-specific (expect/actual)
    └── commonTest/kotlin/      # Shared tests

androidApp/
└── src/main/
    └── com/example/app/
        ├── ui/                 # Android Compose UI
        └── MainActivity.kt

iosApp/                         # Xcode project
```

## When to Extract a Module

Extract when:
- A feature has >10 files
- Code is reused across 2+ features
- You need to enforce compile-time dependency boundaries
- Build times become a bottleneck (parallel compilation)

Keep as package when:
- <30 screens total
- Team is small (1-3 devs)
- No need for strict compile-time boundaries
