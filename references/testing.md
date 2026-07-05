# Testing — Strategies & Patterns

## Testing Pyramid

```
        /  UI Tests  \        ← Few, slow, high confidence
       / Integration  \       ← Medium, test layers together
      / ─────────────── \
     /    Unit Tests      \   ← Many, fast, isolated
    /______________________\
```

## Unit Testing ViewModels

```kotlin
class MatchesViewModelTest {

    private lateinit var viewModel: MatchesViewModel
    private val fakeRepository = FakeMatchRepository()

    @Before
    fun setup() {
        viewModel = MatchesViewModel(fakeRepository)
    }

    @Test
    fun `initial state is Loading`() = runTest {
        viewModel.uiState.test {
            assertEquals(MatchesUiState.Loading, awaitItem())
        }
    }

    @Test
    fun `fetchMatches emits Success with data`() = runTest {
        fakeRepository.setMatches(listOf(testMatch()))

        viewModel.fetchMatchesForDate(LocalDate.now())

        viewModel.uiState.test {
            skipItems(1) // Skip Loading
            val state = awaitItem()
            assertIs<MatchesUiState.Success>(state)
            assertEquals(1, state.groups.flatMap { it.matches }.size)
        }
    }

    @Test
    fun `fetchMatches emits Error on exception`() = runTest {
        fakeRepository.setShouldThrow(true)

        viewModel.fetchMatchesForDate(LocalDate.now())

        viewModel.uiState.test {
            skipItems(1) // Skip Loading
            assertIs<MatchesUiState.Error>(awaitItem())
        }
    }
}
```

## Fake Repository (Test Double)

```kotlin
class FakeMatchRepository : MatchRepository {

    private var matches = listOf<CompetitionGroup>()
    private var shouldThrow = false

    fun setMatches(groups: List<CompetitionGroup>) {
        matches = groups
    }

    fun setShouldThrow(value: Boolean) {
        shouldThrow = value
    }

    override suspend fun getMatchesForDate(date: LocalDate): List<CompetitionGroup> {
        if (shouldThrow) throw IOException("Test error")
        return matches
    }

    override suspend fun getLiveMatches(): List<CompetitionGroup> {
        if (shouldThrow) throw IOException("Test error")
        return matches.filter { group ->
            group.matches.any { it.status == MatchStatus.LIVE }
        }
    }
}
```

## Testing Mappers

```kotlin
class MatchMapperTest {

    @Test
    fun `maps live match correctly`() {
        val dto = createFixtureResponse(
            statusShort = "1H",
            elapsed = 35,
            homeGoals = 1,
            awayGoals = 0,
        )

        val result = MatchMapper.mapToMatch(dto)

        assertEquals(MatchStatus.LIVE, result.status)
        assertEquals(35, result.minute)
        assertEquals(1, result.homeScore)
        assertEquals(0, result.awayScore)
    }

    @Test
    fun `maps finished match correctly`() {
        val dto = createFixtureResponse(statusShort = "FT")
        val result = MatchMapper.mapToMatch(dto)
        assertEquals(MatchStatus.FINISHED, result.status)
    }

    @Test
    fun `maps scheduled match correctly`() {
        val dto = createFixtureResponse(statusShort = "NS")
        val result = MatchMapper.mapToMatch(dto)
        assertEquals(MatchStatus.NOT_STARTED, result.status)
        assertEquals(0, result.minute)
    }
}
```

## Testing Repository

```kotlin
class MatchRepositoryImplTest {

    private val fakeApi = FakeApiService()
    private val repository = MatchRepositoryImpl(fakeApi)

    @Test
    fun `getMatchesForDate maps and groups correctly`() = runTest {
        fakeApi.fixtures = listOf(
            createFixture(league = "Premier League", home = "Arsenal", away = "Chelsea"),
            createFixture(league = "Premier League", home = "Liverpool", away = "Man City"),
            createFixture(league = "La Liga", home = "Barcelona", away = "Real Madrid"),
        )

        val groups = repository.getMatchesForDate(LocalDate.now())

        assertEquals(2, groups.size) // 2 leagues
        assertEquals(2, groups.first { "Premier League" in it.name }.matches.size)
        assertEquals(1, groups.first { "La Liga" in it.name }.matches.size)
    }
}
```

## Key Testing Libraries

```kotlin
// libs.versions.toml
testImplementation(libs.junit)
testImplementation(libs.kotlinx.coroutines.test)  // runTest, TestDispatcher
testImplementation(libs.turbine)                    // Flow testing
```

## Testing Rules

1. **No mocking frameworks** (Mockito, MockK) — use Fakes
2. **Test behavior, not implementation** — test public API only
3. **One assertion per concept** — but multiple asserts for one logical check is fine
4. **Given-When-Then** naming: `fun \`given X when Y then Z\`()`
5. **Test the mapper separately** from the repository
6. **Use `runTest`** for all coroutine tests
7. **Use Turbine** for Flow testing (`flow.test { }`)
