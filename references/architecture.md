# Architecture — Clean Architecture for KMP

## The Three Layers

### 1. Domain Layer (Core)

The **innermost layer**. Contains business logic and contracts.

**Rules:**
- ZERO framework dependencies (no Android, no Compose, no Ktor, no Room)
- Only pure Kotlin + Coroutines/Flow
- Other layers depend on this — it depends on nothing

**Contents:**
- **Models**: Pure `data class` objects representing business entities
- **Repository interfaces**: Contracts that the data layer must implement
- **Use Cases** (optional): Encapsulate reusable business logic shared across ViewModels

```kotlin
// domain/model/Match.kt
data class Match(
    val id: String,
    val homeTeam: Team,
    val awayTeam: Team,
    val score: Score,
    val status: MatchStatus,
    val startTime: Instant,
)

enum class MatchStatus { SCHEDULED, LIVE, FINISHED, POSTPONED }

data class Score(val home: Int, val away: Int)
```

```kotlin
// domain/repository/MatchRepository.kt
interface MatchRepository {
    fun getLiveMatches(): Flow<List<Match>>
    suspend fun getMatchById(id: String): Match
    suspend fun getMatchesForDate(date: LocalDate): List<Match>
}
```

```kotlin
// domain/usecase/GetLiveMatchesUseCase.kt
// Use only when logic is shared across multiple ViewModels
class GetLiveMatchesUseCase(
    private val matchRepository: MatchRepository,
) {
    operator fun invoke(): Flow<List<Match>> =
        matchRepository.getLiveMatches()
            .map { matches -> matches.sortedBy { it.startTime } }
}
```

### 2. Data Layer (Infrastructure)

Implements domain contracts. Handles networking, persistence, caching.

**Rules:**
- Depends on Domain layer only
- DTOs are internal — never exposed to UI
- Mappers convert between DTO/Entity ↔ Domain models
- Repository implementations are the **only** public API

**Structure:**
```
data/
├── remote/
│   ├── ApiService.kt              # Ktor HTTP calls
│   ├── dto/
│   │   └── MatchDto.kt            # @Serializable — matches API JSON
│   └── mapper/
│       └── MatchMapper.kt         # DTO → Domain
├── local/
│   ├── dao/
│   │   └── MatchDao.kt            # Room DAO or SQLDelight queries
│   └── entity/
│       └── MatchEntity.kt         # DB schema representation
├── repository/
│   └── MatchRepositoryImpl.kt     # Implements domain interface
└── MockData.kt                    # Preview/development data
```

**Ktor API Service:**
```kotlin
class ApiService(private val client: HttpClient) {

    suspend fun getLiveFixtures(): List<MatchDto> {
        val response: ApiResponse = client.get("api/fixtures/live").body()
        return response.data
    }

    suspend fun getFixturesByDate(date: String): List<MatchDto> {
        val response: ApiResponse = client.get("api/fixtures") {
            parameter("date", date)
        }.body()
        return response.data
    }
}
```

**Mapper pattern:**
```kotlin
object MatchMapper {
    fun toDomain(dto: MatchDto): Match = Match(
        id = dto.fixture.id.toString(),
        homeTeam = Team(name = dto.teams.home.name, logoUrl = dto.teams.home.logo ?: ""),
        awayTeam = Team(name = dto.teams.away.name, logoUrl = dto.teams.away.logo ?: ""),
        score = Score(home = dto.goals.home ?: 0, away = dto.goals.away ?: 0),
        status = mapStatus(dto.fixture.status.short),
        startTime = Instant.fromEpochSeconds(dto.fixture.timestamp),
    )

    private fun mapStatus(code: String): MatchStatus = when (code) {
        "1H", "2H", "HT", "ET", "P", "LIVE" -> MatchStatus.LIVE
        "FT", "AET", "PEN" -> MatchStatus.FINISHED
        "PST", "CANC" -> MatchStatus.POSTPONED
        else -> MatchStatus.SCHEDULED
    }
}
```

**Repository implementation:**
```kotlin
class MatchRepositoryImpl(
    private val apiService: ApiService,
) : MatchRepository {

    override fun getLiveMatches(): Flow<List<Match>> = flow {
        val dtos = apiService.getLiveFixtures()
        emit(dtos.map(MatchMapper::toDomain))
    }

    override suspend fun getMatchById(id: String): Match {
        val dto = apiService.getFixture(id)
        return MatchMapper.toDomain(dto)
    }

    override suspend fun getMatchesForDate(date: LocalDate): List<Match> {
        val dtos = apiService.getFixturesByDate(date.toString())
        return dtos.map(MatchMapper::toDomain)
    }
}
```

### 3. UI Layer (Presentation)

Displays data and handles user interactions.

**Rules:**
- Depends on Domain layer only (never imports `data.*`)
- ViewModels inject domain interfaces (not implementations)
- Screens are stateless composables — state hoisted to ViewModel
- UiState is a sealed class/interface: `Loading | Success | Error`

**Structure:**
```
ui/
├── screens/        # @Composable functions — stateless
├── viewmodel/      # State holders — inject domain interfaces
├── model/          # UiState sealed classes
├── components/     # Reusable UI building blocks
└── navigation/     # Navigation graph
```

**Data flow:**
```
User action → Screen → ViewModel.onEvent()
                              ↓
                    Repository (domain interface)
                              ↓
                    RepositoryImpl (data layer)
                              ↓
                    API / DB (data sources)
                              ↓
                    Domain model returned
                              ↓
                    ViewModel updates StateFlow<UiState>
                              ↓
                    Screen recomposes with new state
```

## Offline-First Pattern (Advanced)

When local DB is the source of truth:

```kotlin
class OfflineFirstMatchRepository(
    private val localDao: MatchDao,
    private val remoteApi: ApiService,
) : MatchRepository {

    override fun getLiveMatches(): Flow<List<Match>> =
        localDao.observeAll()                          // 1. Emit from DB
            .map { entities -> entities.map { it.toDomain() } }
            .onStart { syncFromRemote() }              // 2. Trigger remote sync

    private suspend fun syncFromRemote() {
        try {
            val remote = remoteApi.getLiveFixtures()
            localDao.upsertAll(remote.map { it.toEntity() })
        } catch (_: Exception) { /* Silent fail — DB has stale data */ }
    }
}
```

## expect/actual Guidelines

Use `expect`/`actual` ONLY for:
- Platform-specific APIs (e.g., `Context` on Android, `NSUserDefaults` on iOS)
- Platform-specific libraries with no KMP alternative
- File system access, Bluetooth, sensors

**NEVER** use for:
- Business logic
- Data transformations
- UI components (use Compose Multiplatform)
- Networking (use Ktor)
- Serialization (use kotlinx.serialization)

## DataSource Pattern

For complex data layers, split the Repository impl into focused DataSources:

```kotlin
// data/remote/MatchRemoteDataSource.kt
class MatchRemoteDataSource(private val client: HttpClient) {
    suspend fun fetchLiveFixtures(): List<MatchDto> {
        return client.get("api/fixtures/live").body<ApiResponse>().data
    }
    suspend fun fetchFixturesByDate(date: String): List<MatchDto> {
        return client.get("api/fixtures") {
            parameter("date", date)
        }.body<ApiResponse>().data
    }
}

// data/local/MatchLocalDataSource.kt
class MatchLocalDataSource(private val dao: MatchDao) {
    fun observeAll(): Flow<List<MatchEntity>> = dao.observeAll()
    suspend fun upsert(entities: List<MatchEntity>) = dao.upsert(entities)
    suspend fun getByDate(date: String): List<MatchEntity> = dao.getByDate(date)
}

// data/repository/MatchRepositoryImpl.kt — coordinates DataSources
class MatchRepositoryImpl(
    private val remote: MatchRemoteDataSource,
    private val local: MatchLocalDataSource,
) : MatchRepository {
    override suspend fun getMatchesForDate(date: LocalDate): Result<List<Match>> {
        return runCatching {
            val dtos = remote.fetchFixturesByDate(date.toString())
            local.upsert(dtos.map { it.toEntity() })
            local.getByDate(date.toString()).map { it.toDomain() }
        }
    }
}
```

## Room DAO (Android)

```kotlin
@Entity(tableName = "matches")
data class MatchEntity(
    @PrimaryKey val id: String,
    val homeTeamName: String,
    val awayTeamName: String,
    val homeScore: Int,
    val awayScore: Int,
    val status: String,
    val date: String,
    val leagueName: String,
)

@Dao
interface MatchDao {
    @Query("SELECT * FROM matches WHERE date = :date")
    suspend fun getByDate(date: String): List<MatchEntity>

    @Upsert
    suspend fun upsert(matches: List<MatchEntity>)

    @Query("SELECT * FROM matches")
    fun observeAll(): Flow<List<MatchEntity>>

    @Query("DELETE FROM matches WHERE date < :date")
    suspend fun deleteOlderThan(date: String)
}
```

## SQLDelight (KMP)

```sql
-- Match.sq
CREATE TABLE MatchEntity (
    id TEXT NOT NULL PRIMARY KEY,
    home_team_name TEXT NOT NULL,
    away_team_name TEXT NOT NULL,
    home_score INTEGER NOT NULL DEFAULT 0,
    away_score INTEGER NOT NULL DEFAULT 0,
    status TEXT NOT NULL,
    date TEXT NOT NULL,
    league_name TEXT NOT NULL
);

getByDate:
SELECT * FROM MatchEntity WHERE date = ?;

upsert:
INSERT OR REPLACE INTO MatchEntity
(id, home_team_name, away_team_name, home_score, away_score, status, date, league_name)
VALUES (?, ?, ?, ?, ?, ?, ?, ?);

observeAll:
SELECT * FROM MatchEntity;

deleteOlderThan:
DELETE FROM MatchEntity WHERE date < ?;
```

## Convention Plugins (build-logic/)

For multi-module KMP projects, centralize build config:

```
build-logic/
├── convention/
│   ├── build.gradle.kts
│   └── src/main/kotlin/
│       ├── KmpLibraryConventionPlugin.kt
│       ├── AndroidComposeConventionPlugin.kt
│       └── AndroidFeatureConventionPlugin.kt
└── settings.gradle.kts
```

```kotlin
// build-logic/convention/src/main/kotlin/KmpLibraryConventionPlugin.kt
class KmpLibraryConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) = with(target) {
        with(pluginManager) {
            apply("org.jetbrains.kotlin.multiplatform")
        }
        extensions.configure<KotlinMultiplatformExtension> {
            androidTarget()
            iosX64(); iosArm64(); iosSimulatorArm64()
            sourceSets {
                commonMain.dependencies { /* shared deps */ }
                commonTest.dependencies { implementation(kotlin("test")) }
            }
        }
    }
}

// Apply in modules:
// domain/build.gradle.kts
plugins { id("kmp-library") }
// No other config needed — convention plugin handles everything
```
