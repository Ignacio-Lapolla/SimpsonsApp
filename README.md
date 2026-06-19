# 2do Parcial — Parte Práctica

**Consigna original:**

> El código tiene 10 errores. Recae en usted analizar que es un error dentro del código. Los alumnos tendrán que forkear este repo como propio, hacer un issue desde Github con comentarios refiriendo en qué línea está el error y cómo se debe solucionar.
>
> https://github.com/ExBattou/SimpsonsApp

**Fork:** https://github.com/Ignacio-Lapolla/SimpsonsApp  
**Alumno:** Ignacio Lapolla

---

## Resultados del análisis

Se identificaron **16 errores** en total (los 10 requeridos + 6 adicionales encontrados mediante análisis estático y recorrida del grafo de conocimiento del proyecto).

Cada error tiene su issue correspondiente en el fork con la descripción del problema, el código con error y la corrección propuesta.

---

## Tabla de errores

| # | Archivo | Línea(s) | Descripción | Issue |
|---|---------|----------|-------------|-------|
| 1 | `data/local/entity/EpisodeEntity.kt` | 7 | `class` en lugar de `data class` — Room necesita `data class` para `equals()`/`hashCode()` y diffing en Paging | [#1](https://github.com/Ignacio-Lapolla/SimpsonsApp/issues/1) |
| 2 | `domain/model/Episode.kt` | 13–15 | Bloque `init` flotante fuera de la clase, sintaxis inválida en Kotlin — no compila | [#2](https://github.com/Ignacio-Lapolla/SimpsonsApp/issues/2) |
| 3 | `di/DataModule.kt` | 34–38 | Retrofit construido sin `.baseUrl()` → `IllegalStateException` en runtime | [#3](https://github.com/Ignacio-Lapolla/SimpsonsApp/issues/3) |
| 4 | `domain/usecase/GetEpisodesUseCase.kt` | 13 | Llama a `repository.get_episodes()` (snake_case) pero el método se llama `getEpisodes()` — no compila | [#4](https://github.com/Ignacio-Lapolla/SimpsonsApp/issues/4) |
| 5 | `main/MainScreenViewModel.kt` | 12 | Falta `@HiltViewModel` + `@Inject constructor` — ViewModel no integrado con el grafo de Hilt | [#5](https://github.com/Ignacio-Lapolla/SimpsonsApp/issues/5) |
| 6 | `main/MainScreen.kt` | 119 | `LazyRow` en lugar de `LazyColumn` para la lista principal de episodios — scroll horizontal incorrecto | [#6](https://github.com/Ignacio-Lapolla/SimpsonsApp/issues/6) |
| 7 | `main/MainScreen.kt` | 51–53 | Side effect (`viewModel.refreshSeasons()`) directo en el cuerpo del composable sin `LaunchedEffect` — se ejecuta en cada recomposición | [#7](https://github.com/Ignacio-Lapolla/SimpsonsApp/issues/7) |
| 8 | `AppNavigation.kt` | 9–10 | Wildcard imports `import androidx.navigation.*` e `import androidx.compose.*` — el segundo es inválido y no resuelve ningún símbolo | [#8](https://github.com/Ignacio-Lapolla/SimpsonsApp/issues/8) |
| 9 | `AndroidManifest.xml` | 13 | `android:windowSoftInputMode` declarado en `<application>` en lugar de `<activity>` — el atributo es ignorado por el sistema | [#9](https://github.com/Ignacio-Lapolla/SimpsonsApp/issues/9) |
| 10 | `MainScreenViewModelTest.kt` | 21–24 | Test `uiState_onItemSaved_isDisplayed` es copia exacta del test anterior — no verifica lo que su nombre dice | [#10](https://github.com/Ignacio-Lapolla/SimpsonsApp/issues/10) |
| 11 | `domain/repository/EpisodeRepository.kt` | 8 | La interfaz declara `get_episodes()` pero `EpisodeRepositoryImpl` implementa `getEpisodes()` — contrato roto, no compila | [#11](https://github.com/Ignacio-Lapolla/SimpsonsApp/issues/11) |
| 12 | `MainActivity.kt` | 17 | Usa `MaterialTheme {}` genérico ignorando `SimpsonsAppTheme` — el colorScheme, dark mode y dynamic color del proyecto no se aplican | [#12](https://github.com/Ignacio-Lapolla/SimpsonsApp/issues/12) |
| 13 | `data/local/entity/RemoteKeyEntity.kt` | 7 | Misma clase `class` en lugar de `data class` — mismo problema que Error 1, afecta a la entidad de claves remotas | [#13](https://github.com/Ignacio-Lapolla/SimpsonsApp/issues/13) |
| 14 | `app/build.gradle.kts` | 27 | `isMinifyEnabled = false` en release — R8 desactivado, APK sin shrinking, sin optimización y sin ofuscación | [#14](https://github.com/Ignacio-Lapolla/SimpsonsApp/issues/14) |
| 15 | `MainScreenTest.kt` | 18 | Test de UI llama `MainScreen(FAKE_DATA)` con `List<String>` pero la firma espera `(Int) -> Unit` — no compila | [#15](https://github.com/Ignacio-Lapolla/SimpsonsApp/issues/15) |
| 16 | `di/DataModule.kt` | 27–29 | `HttpLoggingInterceptor.Level.BODY` sin guarda de `BuildConfig.DEBUG` — loguea bodies HTTP en release, expone datos sensibles | [#16](https://github.com/Ignacio-Lapolla/SimpsonsApp/issues/16) |

---

## Detalle de cada error

### Error 1 — `EpisodeEntity` declarada como `class` en lugar de `data class`

**Archivo:** `app/src/main/java/com/example/simpsonsapp/data/local/entity/EpisodeEntity.kt` — línea 7

```kotlin
// ❌ Error
class EpisodeEntity(
    @PrimaryKey val id: Int, ...
)

// ✅ Corrección
data class EpisodeEntity(
    @PrimaryKey val id: Int, ...
)
```

Las entidades de Room deben ser `data class` para que se generen automáticamente `equals()`, `hashCode()` y `copy()`. Sin esto, Paging 3 no puede hacer diffing correcto entre ítems.

---

### Error 2 — Bloque `init` flotante fuera de la clase en `Episode.kt`

**Archivo:** `app/src/main/java/com/example/simpsonsapp/domain/model/Episode.kt` — líneas 13–15

```kotlin
// ❌ Error — el init está fuera de la clase, a nivel de package
data class Episode(...) { }

init {
    return Episode; //NO BORRAR
}
```

Un bloque `init` a nivel top-level no es Kotlin válido. Además `return Episode` es sintaxis inválida dentro de un `init`. Provoca error de compilación. El bloque debe eliminarse.

---

### Error 3 — Retrofit sin `baseUrl` en `DataModule.kt`

**Archivo:** `app/src/main/java/com/example/simpsonsapp/di/DataModule.kt` — líneas 34–38

```kotlin
// ❌ Error
return Retrofit.Builder()
    .client(client)
    .addConverterFactory(GsonConverterFactory.create())
    .build()
    .create(SimpsonsApi::class.java)

// ✅ Corrección
return Retrofit.Builder()
    .baseUrl("https://thesimpsonsapi.com/")
    .client(client)
    .addConverterFactory(GsonConverterFactory.create())
    .build()
    .create(SimpsonsApi::class.java)
```

Retrofit lanza `IllegalStateException: Base URL required` en runtime si no se provee `baseUrl`.

---

### Error 4 — Nombre de método snake_case en `GetEpisodesUseCase.kt`

**Archivo:** `app/src/main/java/com/example/simpsonsapp/domain/usecase/GetEpisodesUseCase.kt` — línea 13

```kotlin
// ❌ Error
return repository.get_episodes()

// ✅ Corrección
return repository.getEpisodes()
```

El método en el repositorio se llama `getEpisodes()` (camelCase). `get_episodes()` no existe — no compila. Los métodos en Kotlin siempre usan camelCase.

---

### Error 5 — `MainScreenViewModel` sin `@HiltViewModel` ni `@Inject`

**Archivo:** `app/src/main/java/com/example/simpsonsapp/main/MainScreenViewModel.kt` — línea 12

```kotlin
// ❌ Error
class MainScreenViewModel(dataRepository: DataRepository) : ViewModel()

// ✅ Corrección
@HiltViewModel
class MainScreenViewModel @Inject constructor(
    private val dataRepository: DataRepository
) : ViewModel()
```

Sin `@HiltViewModel`, Hilt no puede construir ni inyectar este ViewModel. Rompe el patrón de DI del proyecto y lo hace no testeable.

---

### Error 6 — `LazyRow` en lugar de `LazyColumn` en `MainScreen.kt`

**Archivo:** `app/src/main/java/com/example/simpsonsapp/main/MainScreen.kt` — línea 119

```kotlin
// ❌ Error
LazyRow(state = listState, modifier = Modifier.fillMaxSize()) { ... }

// ✅ Corrección
LazyColumn(state = listState, modifier = Modifier.fillMaxSize()) { ... }
```

Una lista principal de episodios debe ser vertical. `LazyRow` produce scroll horizontal, incorrecto para este caso y confuso al combinarse con el otro `LazyRow` de selección de episodios.

---

### Error 7 — Side effect directo en el cuerpo del Composable

**Archivo:** `app/src/main/java/com/example/simpsonsapp/main/MainScreen.kt` — líneas 51–53

```kotlin
// ❌ Error — se ejecuta en cada recomposición
if (episodes.loadState.refresh is LoadState.NotLoading && seasons.isEmpty()) {
    viewModel.refreshSeasons()
}

// ✅ Corrección
val refreshState = episodes.loadState.refresh
LaunchedEffect(refreshState, seasons.isEmpty()) {
    if (refreshState is LoadState.NotLoading && seasons.isEmpty()) {
        viewModel.refreshSeasons()
    }
}
```

Los composables deben ser idempotentes y sin efectos secundarios en su cuerpo. Los side effects van en `LaunchedEffect`.

---

### Error 8 — Wildcard imports inválidos en `AppNavigation.kt`

**Archivo:** `app/src/main/java/com/example/simpsonsapp/AppNavigation.kt` — líneas 9–10

```kotlin
// ❌ Error
import androidx.navigation.*
import androidx.compose.*   // ← no existe como package unificado, no resuelve nada
```

`import androidx.compose.*` no es un import válido. Los componentes de Compose están en subpackages específicos. Todos los símbolos necesarios ya están importados individualmente en las líneas anteriores. Deben eliminarse ambas líneas.

---

### Error 9 — `windowSoftInputMode` en `<application>` en lugar de `<activity>`

**Archivo:** `app/src/main/AndroidManifest.xml` — línea 13

```xml
<!-- ❌ Error -->
<application android:windowSoftInputMode="adjustResize" ...>

<!-- ✅ Corrección -->
<application ...>
    <activity android:windowSoftInputMode="adjustResize" ...>
```

`android:windowSoftInputMode` es exclusivo de `<activity>`. En `<application>` es ignorado silenciosamente — el teclado virtual puede tapar la UI sin que el desarrollador lo note.

---

### Error 10 — Test duplicado que no verifica lo que dice

**Archivo:** `app/src/test/java/com/example/simpsonsapp/ui/main/MainScreenViewModelTest.kt` — líneas 21–24

```kotlin
// ❌ Error — idéntico al test anterior, nunca verifica Success
@Test
fun uiState_onItemSaved_isDisplayed() = runTest {
    val viewModel = MainScreenViewModel(FakeMyModelRepository())
    assertEquals(viewModel.aux.first(), MainScreenUiState.Loading)
}

// ✅ Corrección — verificar que los datos se muestran
@Test
fun uiState_onItemSaved_isDisplayed() = runTest {
    val viewModel = MainScreenViewModel(FakeMyModelRepository())
    val states = mutableListOf<MainScreenUiState>()
    val job = launch { viewModel.aux.collect { states.add(it) } }
    advanceUntilIdle()
    job.cancel()
    assert(states.any { it is MainScreenUiState.Success && it.data == listOf("Sample") })
}
```

El nombre promete verificar que los datos se muestran pero solo verifica `Loading`. Falsa sensación de cobertura.

---

### Error 11 — Contrato roto entre `EpisodeRepository` y `EpisodeRepositoryImpl`

**Archivos:** `domain/repository/EpisodeRepository.kt:8` / `data/repository/EpisodeRepositoryImpl.kt:23`

```kotlin
// ❌ Error — interfaz usa snake_case
interface EpisodeRepository {
    fun get_episodes(): Flow<PagingData<Episode>>
}

// La implementación usa camelCase — nombres diferentes, contrato roto
override fun getEpisodes(): Flow<PagingData<Episode>> { ... }

// ✅ Corrección — unificar en camelCase en la interfaz
interface EpisodeRepository {
    fun getEpisodes(): Flow<PagingData<Episode>>
}
```

`EpisodeRepositoryImpl` no implementa realmente el método de la interfaz porque los nombres no coinciden → error de compilación. Este es el error raíz del Error 4.

---

### Error 12 — `MainActivity` usa `MaterialTheme` en lugar de `SimpsonsAppTheme`

**Archivo:** `app/src/main/java/com/example/simpsonsapp/MainActivity.kt` — línea 17

```kotlin
// ❌ Error
setContent {
    MaterialTheme {
        Surface(...) { AppNavigation() }
    }
}

// ✅ Corrección
setContent {
    SimpsonsAppTheme {
        Surface(...) { AppNavigation() }
    }
}
```

El proyecto define `SimpsonsAppTheme` con colorScheme personalizado, dynamic color (Android 12+) y dark mode. Usar `MaterialTheme {}` directo los bypasea completamente.

---

### Error 13 — `RemoteKeyEntity` declarada como `class` en lugar de `data class`

**Archivo:** `app/src/main/java/com/example/simpsonsapp/data/local/entity/RemoteKeyEntity.kt` — línea 7

```kotlin
// ❌ Error
class RemoteKeyEntity(@PrimaryKey val episodeId: Int, val prevKey: Int?, val nextKey: Int?)

// ✅ Corrección
data class RemoteKeyEntity(@PrimaryKey val episodeId: Int, val prevKey: Int?, val nextKey: Int?)
```

Mismo problema que el Error 1. Todas las entidades de Room deben ser `data class`.

---

### Error 14 — `isMinifyEnabled = false` en release (R8 desactivado)

**Archivo:** `app/build.gradle.kts` — línea 27

```kotlin
// ❌ Error
release {
    isMinifyEnabled = false
    proguardFiles(...)
}

// ✅ Corrección
release {
    isMinifyEnabled = true
    isShrinkResources = true
    proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
}
```

R8 desactivado en release produce un APK sin shrinking, sin optimización y sin ofuscación. El APK es más grande, más lento y más fácil de descompilar.

---

### Error 15 — `MainScreenTest` con firma incompatible: el test no compila

**Archivo:** `app/src/androidTest/java/com/example/simpsonsapp/ui/main/MainScreenTest.kt` — línea 18

```kotlin
// ❌ Error — List<String> no es compatible con (Int) -> Unit
composeTestRule.setContent { MainScreen(FAKE_DATA) }

// ✅ Corrección
composeTestRule.setContent {
    MainScreen(onNavigateToDetail = { })
}
```

`MainScreen` espera `(Int) -> Unit` pero el test pasa `List<String>`. Tipos incompatibles — el test no compila y no cubre nada.

---

### Error 16 — `HttpLoggingInterceptor.Level.BODY` sin guarda de debug

**Archivo:** `app/src/main/java/com/example/simpsonsapp/di/DataModule.kt` — líneas 27–29

```kotlin
// ❌ Error — activo en release
val logging = HttpLoggingInterceptor().apply {
    level = HttpLoggingInterceptor.Level.BODY
}

// ✅ Corrección
val logging = HttpLoggingInterceptor().apply {
    level = if (BuildConfig.DEBUG) HttpLoggingInterceptor.Level.BODY
            else HttpLoggingInterceptor.Level.NONE
}
```

`Level.BODY` loguea el cuerpo completo de cada request/response HTTP en Logcat. Sin la guarda de `BuildConfig.DEBUG`, esto ocurre también en producción, exponiendo datos sensibles.

---

*Análisis realizado por Ignacio Lapolla — Issues: https://github.com/Ignacio-Lapolla/SimpsonsApp/issues*
