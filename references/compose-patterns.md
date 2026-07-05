# Compose Patterns — Best Practices

## State Hoisting

**Always separate connected (Route) from pure (Screen):**

```kotlin
// Route — connected to ViewModel, handles navigation
@Composable
fun SettingsRoute(
    onNavigateBack: () -> Unit,
    viewModel: SettingsViewModel = koinViewModel(),
) {
    val uiState by viewModel.uiState.collectAsState()
    SettingsScreen(
        uiState = uiState,
        onAction = viewModel::onAction,
        onNavigateBack = onNavigateBack,
    )
}

// Screen — pure, testable, preview-able
@Composable
fun SettingsScreen(
    uiState: SettingsUiState,
    onAction: (SettingsAction) -> Unit,
    onNavigateBack: () -> Unit,
    modifier: Modifier = Modifier,
) {
    // UI implementation — NO ViewModel reference here
}
```

## UiState Sealed Classes

```kotlin
sealed interface SettingsUiState {
    data object Loading : SettingsUiState
    data class Success(
        val darkMode: Boolean,
        val notifications: Boolean,
        val language: String,
    ) : SettingsUiState
    data class Error(val message: String) : SettingsUiState
}

sealed interface SettingsAction {
    data class ToggleDarkMode(val enabled: Boolean) : SettingsAction
    data class ToggleNotifications(val enabled: Boolean) : SettingsAction
    data class ChangeLanguage(val code: String) : SettingsAction
}
```

## Loading States — Skeleton/Shimmer

Always use shimmer effects for loading, never blank screens:

```kotlin
@Composable
fun ShimmerBox(
    modifier: Modifier = Modifier,
    shape: Shape = RoundedCornerShape(8.dp),
) {
    val transition = rememberInfiniteTransition(label = "shimmer")
    val alpha by transition.animateFloat(
        initialValue = 0.3f,
        targetValue = 0.7f,
        animationSpec = infiniteRepeatable(
            animation = tween(800, easing = LinearEasing),
            repeatMode = RepeatMode.Reverse,
        ),
        label = "shimmerAlpha",
    )

    Box(
        modifier = modifier
            .clip(shape)
            .background(OnSurfaceVariant.copy(alpha = alpha)),
    )
}
```

## Performance Optimization

### remember / derivedStateOf
```kotlin
@Composable
fun MatchesList(matches: List<Match>) {
    // ✅ Derive expensive computation
    val liveMatches by remember(matches) {
        derivedStateOf { matches.filter { it.status == MatchStatus.LIVE } }
    }

    // ✅ Remember stable keys
    val groupedByLeague = remember(matches) {
        matches.groupBy { it.league }
    }
}
```

### Stable keys in LazyColumn
```kotlin
LazyColumn {
    items(
        items = matches,
        key = { match -> match.id }, // ✅ Stable unique key
    ) { match ->
        MatchRow(match)
    }
}
```

### Avoid recomposition leaks
```kotlin
// ❌ BAD — lambda created every recomposition
items(matches) { match ->
    MatchRow(
        match = match,
        onClick = { viewModel.onMatchClicked(match.id) },
    )
}

// ✅ GOOD — stable lambda reference
items(matches) { match ->
    MatchRow(
        match = match,
        onClick = viewModel::onMatchClicked,
    )
}
```

## Theme & Design Tokens

Use a centralized design system:

```kotlin
// theme/Color.kt
val BackgroundDark = Color(0xFF0A0A1A)    // Deep dark
val Accent = Color(0xFF00FF85)             // Neon green
val PrimaryPurple = Color(0xFF3D195B)      // Premier League purple
val OnSurface = Color(0xFFFFFFFF)
val OnSurfaceVariant = Color(0xFFB0B0B0)
val SurfaceElevated = Color(0xFF1A1A2E)
val LiveRed = Color(0xFFFF4444)

// theme/Type.kt
val SoraFont = FontFamily(/* ... */)
val JetBrainsMonoFont = FontFamily(/* ... */)
```

## Side Effects

```kotlin
// LaunchedEffect — run suspend code when key changes
LaunchedEffect(selectedDate) {
    viewModel.fetchMatchesForDate(selectedDate)
}

// DisposableEffect — cleanup when leaving composition
DisposableEffect(Unit) {
    val listener = registerListener()
    onDispose { listener.unregister() }
}

// rememberCoroutineScope — for event-driven coroutines
val scope = rememberCoroutineScope()
Button(onClick = { scope.launch { viewModel.save() } })
```

## Component Design Rules

1. **Stateless by default**: Components receive data + callbacks
2. **Modifier as first optional param**: Always accept `modifier: Modifier = Modifier`
3. **Preview functions**: Every component should have `@Preview`
4. **No business logic in composables**: Delegate to ViewModel
5. **Use `remember` for expensive operations**: Layout calculations, filtering, sorting
