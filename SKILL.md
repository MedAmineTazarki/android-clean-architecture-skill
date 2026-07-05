---
name: android-clean-architecture
description: Build production-quality Android/KMP applications following Clean Architecture, MVVM/MVI, and modern Kotlin best practices. Use when creating Android projects, feature modules, screens, ViewModels, repositories, data layers, or when asked about architecture patterns. Triggers on Kotlin Multiplatform, Compose Multiplatform, Koin DI, Ktor networking, Room/SQLDelight, or multi-module project setup.
---

# Android Clean Architecture — KMP Stack

Build Android/KMP applications following Clean Architecture with the project's established stack:
**Kotlin Multiplatform · Compose Multiplatform · Koin · Ktor · Room KMP / SQLDelight · Coroutines/Flow**

## Quick Reference

| Task | Reference File |
|------|----------------|
| Architecture layers (UI, Domain, Data) | [architecture.md](references/architecture.md) |
| Jetpack/Compose Multiplatform patterns | [compose-patterns.md](references/compose-patterns.md) |
| Project structure & modules | [modularization.md](references/modularization.md) |
| Gradle & build configuration | [gradle-setup.md](references/gradle-setup.md) |
| Testing approach | [testing.md](references/testing.md) |

## Workflow Decision Tree

**Creating a new project?**
→ Read [modularization.md](references/modularization.md) for structure
→ Use `commonMain` as default code location

**Adding a new feature?**
→ Follow patterns in [architecture.md](references/architecture.md)
→ Create: domain model → repository interface → data impl → ViewModel → Screen

**Building UI screens?**
→ Read [compose-patterns.md](references/compose-patterns.md)
→ Create Screen (stateless) + ViewModel (state holder) + UiState (sealed)

**Setting up data layer?**
→ Read data layer section in [architecture.md](references/architecture.md)
→ Create: DTO → Mapper → Repository impl → Domain model

**Need platform-specific code?**
→ Use `expect`/`actual` ONLY when unavoidable
→ Keep in `commonMain` by default

## Core Principles

1. **Domain-centric**: Domain layer has ZERO framework dependencies
2. **Unidirectional data flow (UDF)**: Events↓ State↑
3. **Reactive streams**: Kotlin `Flow` (StateFlow/SharedFlow) for all data
4. **Dependency inversion**: UI and Data depend on Domain, never the reverse
5. **KMP-first**: Maximum code in `commonMain`, `expect`/`actual` only when strictly necessary
6. **Testable by design**: Interfaces + constructor injection via Koin

## Architecture Layers

```
┌─────────────────────────────────────────┐
│              UI Layer (Presentation)      │
│  Compose Screens (stateless)             │
│  + ViewModels (state holders)            │
│  + UiState sealed classes                │
├─────────────────────────────────────────┤
│           Domain Layer (Business)        │
│  Pure Kotlin models (data class)         │
│  Repository interfaces                   │
│  Use Cases (optional, for shared logic)  │
├─────────────────────────────────────────┤
│            Data Layer (Infrastructure)   │
│  Repository implementations              │
│  DTOs (@Serializable)                    │
│  Mappers (DTO ↔ Domain)                  │
│  API services (Ktor)                     │
│  Local storage (Room/SQLDelight)         │
└─────────────────────────────────────────┘
```

### Dependency Rule (STRICT)

```
UI → Domain ← Data
      ↑
   (Domain knows NOTHING about UI or Data)
```

## Package Structure (Single Module)

```
com.example.app/
├── domain/
│   ├── model/          # Pure Kotlin data classes (NO framework imports)
│   │   ├── User.kt
│   │   └── Match.kt
│   ├── repository/     # Interfaces only
│   │   ├── UserRepository.kt
│   │   └── MatchRepository.kt
│   └── usecase/        # Optional — when logic is shared across ViewModels
│       └── GetLiveMatchesUseCase.kt
├── data/
│   ├── remote/
│   │   ├── ApiService.kt          # Ktor HttpClient calls
│   │   ├── dto/                   # @Serializable DTOs
│   │   │   ├── UserDto.kt
│   │   │   └── MatchDto.kt
│   │   └── mapper/                # DTO → Domain mappers
│   │       ├── UserMapper.kt
│   │       └── MatchMapper.kt
│   ├── local/
│   │   ├── dao/                   # Room DAOs / SQLDelight queries
│   │   └── entity/                # DB entities
│   ├── repository/                # Concrete implementations
│   │   ├── UserRepositoryImpl.kt
│   │   └── MatchRepositoryImpl.kt
│   └── MockData.kt               # Dev/preview data (if needed)
├── ui/
│   ├── screens/                   # @Composable screens (stateless)
│   ├── viewmodel/                 # ViewModels (state holders)
│   ├── model/                     # UiState sealed classes
│   ├── components/                # Reusable composables
│   └── navigation/                # Nav graph
├── di/
│   └── AppModules.kt             # Koin modules
└── theme/                         # Colors, Typography, Theme
```

## Standard Patterns

### Domain Model
```kotlin
// domain/model/User.kt — ZERO imports, pure Kotlin
data class User(
    val id: String,
    val name: String,
    val email: String,
    val avatarUrl: String = "",
)
```

### Repository Interface (Domain)
```kotlin
// domain/repository/UserRepository.kt
interface UserRepository {
    fun getUsers(): Flow<List<User>>
    suspend fun getUserById(id: String): User
    suspend fun updateUser(user: User)
}
```

### DTO (Data)
```kotlin
// data/remote/dto/UserDto.kt
@Serializable
data class UserDto(
    val id: String,
    val name: String,
    val email: String,
    @SerialName("avatar_url") val avatarUrl: String? = null,
)
```

### Mapper (Data)
```kotlin
// data/remote/mapper/UserMapper.kt
// Style 1: Object mapper (grouped, testable)
object UserMapper {
    fun toDomain(dto: UserDto): User = User(
        id = dto.id,
        name = dto.name,
        email = dto.email,
        avatarUrl = dto.avatarUrl ?: "",
    )
    fun toDomainList(dtos: List<UserDto>): List<User> = dtos.map(::toDomain)
}

// Style 2: Extension functions (concise, near data models)
fun UserDto.toDomain() = User(
    id = id, name = name, email = email,
    avatarUrl = avatarUrl ?: "",
)
fun UserEntity.toDomain() = User(
    id = id, name = name, email = email,
    avatarUrl = avatarUrl,
)
fun User.toEntity() = UserEntity(
    id = id, name = name, email = email,
    avatarUrl = avatarUrl,
)
```

### Repository Implementation (Data)
```kotlin
// data/repository/UserRepositoryImpl.kt
class UserRepositoryImpl(
    private val apiService: ApiService,
    private val userDao: UserDao,
) : UserRepository {

    // runCatching wraps exceptions into Result<T>
    override fun getUsers(): Flow<List<User>> =
        userDao.getAll().map { entities -> entities.map { it.toDomain() } }

    override suspend fun getUserById(id: String): User {
        val dto = apiService.getUser(id)
        return dto.toDomain()
    }

    override suspend fun updateUser(user: User) {
        userDao.upsert(user.toEntity())
    }
}
```

### UiState (Presentation)
```kotlin
// ui/model/UsersUiState.kt
sealed interface UsersUiState {
    data object Loading : UsersUiState
    data class Success(val users: List<User>) : UsersUiState
    data class Error(val message: String) : UsersUiState
}
```

### ViewModel (Presentation)
```kotlin
// ui/viewmodel/UsersViewModel.kt
class UsersViewModel(
    private val repository: UserRepository, // domain interface
) : ViewModel() {

    private val _uiState = MutableStateFlow<UsersUiState>(UsersUiState.Loading)
    val uiState: StateFlow<UsersUiState> = _uiState.asStateFlow()

    init { loadUsers() }

    fun loadUsers() {
        viewModelScope.launch {
            _uiState.value = UsersUiState.Loading
            try {
                repository.getUsers().collect { users ->
                    _uiState.value = UsersUiState.Success(users)
                }
            } catch (e: Exception) {
                _uiState.value = UsersUiState.Error(e.localizedMessage ?: "Error")
            }
        }
    }
}
```

### Screen (Presentation — STATELESS)
```kotlin
// ui/screens/UsersScreen.kt
@Composable
fun UsersScreen(
    modifier: Modifier = Modifier,
    viewModel: UsersViewModel = koinViewModel(),
) {
    val uiState by viewModel.uiState.collectAsState()

    when (val state = uiState) {
        is UsersUiState.Loading -> ShimmerLoadingList()
        is UsersUiState.Success -> UsersList(state.users)
        is UsersUiState.Error -> ErrorMessage(state.message)
    }
}

// ALWAYS separate Route (connected) from Screen (pure/preview-able)
@Composable
private fun UsersList(users: List<User>) { /* ... */ }
```

### Koin DI Modules
```kotlin
// di/AppModules.kt
val networkModule = module {
    single { /* Ktor HttpClient setup */ }
    single<ApiService> { ApiServiceImpl(get()) }
}

val repositoryModule = module {
    single<UserRepository> { UserRepositoryImpl(get(), get()) }
    single<MatchRepository> { MatchRepositoryImpl(get()) }
}

val viewModelModule = module {
    viewModel { UsersViewModel(get()) }
    viewModel { MatchesViewModel(get()) }
}

val appModules = listOf(networkModule, repositoryModule, viewModelModule)
```

## Anti-Patterns to AVOID

| ❌ Don't | ✅ Do |
|----------|-------|
| Import `data.*` in UI layer | Import only `domain.model.*` |
| Put business logic in `@Composable` | Extract to ViewModel or UseCase |
| Use DTOs as domain models | Create separate domain models + mappers |
| Use `expect`/`actual` for everything | Only when native API access is unavoidable |
| Use Hilt/Dagger in KMP | Use Koin (multiplatform compatible) |
| Use Retrofit in KMP | Use Ktor Client |
| Put ViewModel in `screens/` | Separate into `viewmodel/` package |
| Hardcode API URLs in services | Configure via DI module constants |
| Skip Loading/Error states | Always implement Loading/Success/Error sealed class |
| Use `GlobalScope` | Use `viewModelScope` or structured concurrency |
| Fat repository with all logic | Split into focused DataSources |
| Circular module dependencies | If A→B, then B must NOT→A |
| Import Android framework in `domain/` | Keep domain as pure Kotlin |
| Use try/catch without mapping errors | Use `runCatching` or `Result<T>` pattern |

## UseCase Pattern

UseCases encapsulate **one business operation**. Use `operator fun invoke` for clean call sites.
Only create a UseCase when logic is **shared across ≥2 ViewModels** or **complex enough to warrant isolation**.

```kotlin
// domain/usecase/GetLiveMatchesUseCase.kt
class GetLiveMatchesUseCase(
    private val matchRepository: MatchRepository,
) {
    // operator fun invoke → allows: useCase() instead of useCase.execute()
    suspend operator fun invoke(): Result<List<CompetitionGroup>> {
        return runCatching {
            matchRepository.getLiveMatches()
        }
    }
}

// Flow-based UseCase (reactive)
class ObserveLiveMatchesUseCase(
    private val matchRepository: MatchRepository,
) {
    operator fun invoke(): Flow<List<CompetitionGroup>> {
        return matchRepository.observeLiveMatches()
            .map { groups -> groups.sortedBy { it.priority } }
    }
}

// Usage in ViewModel
class MatchesViewModel(
    private val getLiveMatches: GetLiveMatchesUseCase,
) : ViewModel() {
    fun loadLive() {
        viewModelScope.launch {
            _uiState.value = MatchesUiState.Loading
            getLiveMatches()  // ← clean call thanks to operator invoke
                .onSuccess { _uiState.value = MatchesUiState.Success(it) }
                .onFailure { _uiState.value = MatchesUiState.Error(it.message ?: "Error") }
        }
    }
}
```

## Error Handling

### Option 1: Kotlin `Result<T>` + `runCatching`
```kotlin
// In Repository
override suspend fun getMatchesForDate(date: LocalDate): Result<List<CompetitionGroup>> {
    return runCatching {
        val response = apiService.getFixturesByDate(date.toString())
        MatchMapper.mapToCompetitionGroups(response.data)
    }
}

// In ViewModel
repository.getMatchesForDate(date)
    .onSuccess { groups -> _uiState.value = UiState.Success(groups) }
    .onFailure { error -> _uiState.value = UiState.Error(error.message ?: "Unknown") }
```

### Option 2: Custom `AppResult<T>` sealed type (richer errors)
```kotlin
sealed interface AppResult<out T> {
    data class Success<T>(val data: T) : AppResult<T>
    data class Failure(val error: AppError) : AppResult<Nothing>
}

sealed interface AppError {
    data class Network(val message: String, val code: Int? = null) : AppError
    data class Database(val message: String) : AppError
    data object Unauthorized : AppError
    data object NotFound : AppError
}

// In ViewModel — map error to user-facing message
when (val result = getMatches(date)) {
    is AppResult.Success -> _uiState.value = UiState.Success(result.data)
    is AppResult.Failure -> _uiState.value = UiState.Error(result.error.toMessage())
}

fun AppError.toMessage(): String = when (this) {
    is AppError.Network -> "Erreur réseau : $message"
    is AppError.Database -> "Erreur locale : $message"
    AppError.Unauthorized -> "Session expirée"
    AppError.NotFound -> "Non trouvé"
}
```

## Checklist: New Feature

- [ ] Domain model created (pure Kotlin, no framework deps)
- [ ] Repository interface in `domain/repository/`
- [ ] UseCase (optional — if logic shared across ViewModels)
- [ ] DTOs in `data/remote/dto/` with `@Serializable`
- [ ] Mapper (object or extension functions) in `data/remote/mapper/`
- [ ] Repository implementation in `data/repository/` with `runCatching`
- [ ] UiState sealed class in `ui/model/`
- [ ] ViewModel in `ui/viewmodel/` — injects domain interface
- [ ] Screen in `ui/screens/` — stateless composable
- [ ] Koin bindings updated in `di/AppModules.kt`
- [ ] Loading + Success + Error states implemented
- [ ] Error handling with `Result<T>` or `AppResult<T>`
