# Prompt: Implementar Scene List Details en App Compose Nativa

## Contexto
Este documento proporciona una guía completa para implementar el patrón **Scene List Details** en una aplicación nativa de Jetpack Compose utilizando Navigation 3.0, siguiendo las mejores prácticas de arquitectura MVVM y Kotlin Multiplatform.

> **Nota**: Este es un documento guía que muestra cómo **implementar nuevas funcionalidades** (Scene List/Details). Los nombres de clases y archivos mostrados (SceneListScreen, SceneDetailScreen, etc.) son ejemplos de lo que se debe crear, no referencias al código existente del proyecto.

## Arquitectura del Patrón List-Details

### 1. Estructura de Navegación

El patrón List-Details consiste en dos pantallas principales:
- **Lista (List Screen)**: Muestra una colección de elementos
- **Detalles (Details Screen)**: Muestra información detallada de un elemento seleccionado

### 2. Definición de Rutas (Routes)

```kotlin
// navigation/Route.kt
package com.plcoding.nav3_guide.navigation

import androidx.navigation3.runtime.NavKey
import kotlinx.serialization.Serializable

@Serializable
sealed interface Route: NavKey {

    @Serializable
    data object SceneList: Route, NavKey

    @Serializable
    data class SceneDetail(
        val sceneId: String,
        val sceneName: String
    ): Route, NavKey
}
```

**Puntos clave:**
- Usa `@Serializable` para permitir la serialización de los argumentos de navegación
- Hereda de `NavKey` para compatibilidad con Navigation 3.0
- La ruta de detalles contiene parámetros necesarios (ID, nombre, etc.)

### 3. Configuración de Navegación

```kotlin
// navigation/NavigationRoot.kt
package com.plcoding.nav3_guide.navigation

import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.lifecycle.viewmodel.navigation3.rememberViewModelStoreNavEntryDecorator
import androidx.navigation3.runtime.NavEntry
import androidx.navigation3.runtime.NavKey
import androidx.navigation3.runtime.rememberNavBackStack
import androidx.navigation3.runtime.rememberSaveableStateHolderNavEntryDecorator
import androidx.navigation3.ui.NavDisplay
import androidx.savedstate.serialization.SavedStateConfiguration
import com.plcoding.nav3_guide.screens.SceneDetailScreen
import com.plcoding.nav3_guide.screens.SceneListScreen
import kotlinx.serialization.modules.SerializersModule
import kotlinx.serialization.modules.polymorphic

@Composable
fun NavigationRoot(
    modifier: Modifier = Modifier
) {
    val backStack = rememberNavBackStack(
        configuration = SavedStateConfiguration {
            serializersModule = SerializersModule {
                polymorphic(NavKey::class) {
                    subclass(Route.SceneList::class, Route.SceneList.serializer())
                    subclass(Route.SceneDetail::class, Route.SceneDetail.serializer())
                }
            }
        },
        Route.SceneList // Pantalla inicial
    )
    
    NavDisplay(
        modifier = modifier,
        backStack = backStack,
        entryDecorators = listOf(
            rememberSaveableStateHolderNavEntryDecorator(),
            rememberViewModelStoreNavEntryDecorator()
        ),
        entryProvider = { key ->
            when(key) {
                is Route.SceneList -> {
                    NavEntry(key) {
                        SceneListScreen(
                            onSceneClick = { sceneId, sceneName ->
                                backStack.add(Route.SceneDetail(sceneId, sceneName))
                            }
                        )
                    }
                }
                is Route.SceneDetail -> {
                    NavEntry(key) {
                        SceneDetailScreen(
                            sceneId = key.sceneId,
                            sceneName = key.sceneName
                        )
                    }
                }
                else -> error("Unknown NavKey: $key")
            }
        }
    )
}
```

**Características importantes:**
- `rememberNavBackStack`: Gestiona el stack de navegación con estado guardado
- `SerializersModule`: Configura la serialización de rutas personalizadas
- `entryDecorators`: Proporciona ViewModel y SavedState scoping
- `entryProvider`: Define qué composable mostrar para cada ruta

### 4. ViewModel para la Lista

```kotlin
// viewmodels/SceneListViewModel.kt
package com.plcoding.nav3_guide.viewmodels

import androidx.lifecycle.ViewModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.asStateFlow

data class Scene(
    val id: String,
    val name: String,
    val description: String,
    val imageUrl: String? = null
)

class SceneListViewModel: ViewModel() {

    private val _scenes = MutableStateFlow<List<Scene>>(emptyList())
    val scenes = _scenes.asStateFlow()

    private val _isLoading = MutableStateFlow(false)
    val isLoading = _isLoading.asStateFlow()

    init {
        loadScenes()
    }

    private fun loadScenes() {
        _isLoading.value = true
        // Simular carga de datos
        _scenes.value = (1..20).map { index ->
            Scene(
                id = "scene_$index",
                name = "Scene $index",
                description = "Description for scene $index"
            )
        }
        _isLoading.value = false
    }

    override fun onCleared() {
        super.onCleared()
        println("SceneListViewModel cleared")
    }
}
```

**Patrón de estado:**
- Usa `StateFlow` para exponer estado reactivo
- Gestión de estado de carga
- Inicialización de datos en `init`
- Logging en `onCleared` para debugging

### 5. ViewModel para Detalles

```kotlin
// viewmodels/SceneDetailViewModel.kt
package com.plcoding.nav3_guide.viewmodels

import androidx.lifecycle.ViewModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.asStateFlow

data class SceneDetailState(
    val sceneId: String,
    val sceneName: String,
    val description: String = "",
    val details: List<String> = emptyList(),
    val isLoading: Boolean = false
)

class SceneDetailViewModel(
    private val sceneId: String,
    private val sceneName: String
): ViewModel() {

    private val _state = MutableStateFlow(
        SceneDetailState(
            sceneId = sceneId,
            sceneName = sceneName
        )
    )
    val state = _state.asStateFlow()

    init {
        loadSceneDetails()
        println("SceneDetailViewModel initialized for $sceneId")
    }

    private fun loadSceneDetails() {
        _state.value = _state.value.copy(isLoading = true)
        
        // Simular carga de detalles
        val details = (1..10).map { "Detail item $it for $sceneName" }
        
        _state.value = _state.value.copy(
            description = "Detailed information about $sceneName",
            details = details,
            isLoading = false
        )
    }

    override fun onCleared() {
        super.onCleared()
        println("SceneDetailViewModel cleared for $sceneId")
    }
}
```

**Características:**
- ViewModel recibe parámetros desde la ruta
- Estado inmutable con `copy()`
- Gestión de ciclo de vida con logging

### 6. Pantalla de Lista (List Screen)

```kotlin
// screens/SceneListScreen.kt
package com.plcoding.nav3_guide.screens

import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import com.plcoding.nav3_guide.viewmodels.Scene
import com.plcoding.nav3_guide.viewmodels.SceneListViewModel

@Composable
fun SceneListScreen(
    viewModel: SceneListViewModel = viewModel {
        SceneListViewModel()
    },
    onSceneClick: (String, String) -> Unit,
    modifier: Modifier = Modifier
) {
    val scenes by viewModel.scenes.collectAsStateWithLifecycle()
    val isLoading by viewModel.isLoading.collectAsStateWithLifecycle()

    Column(
        modifier = modifier.fillMaxSize()
    ) {
        // Título
        Text(
            text = "Scenes",
            style = MaterialTheme.typography.headlineMedium,
            modifier = Modifier.padding(16.dp)
        )

        if (isLoading) {
            Box(
                modifier = Modifier.fillMaxSize(),
                contentAlignment = Alignment.Center
            ) {
                CircularProgressIndicator()
            }
        } else {
            LazyColumn(
                modifier = Modifier.fillMaxSize(),
                contentPadding = PaddingValues(horizontal = 16.dp, vertical = 8.dp),
                verticalArrangement = Arrangement.spacedBy(8.dp)
            ) {
                items(scenes) { scene ->
                    SceneListItem(
                        scene = scene,
                        onClick = {
                            onSceneClick(scene.id, scene.name)
                        }
                    )
                }
            }
        }
    }
}

@Composable
private fun SceneListItem(
    scene: Scene,
    onClick: () -> Unit,
    modifier: Modifier = Modifier
) {
    Card(
        modifier = modifier
            .fillMaxWidth()
            .clickable(onClick = onClick),
        elevation = CardDefaults.cardElevation(defaultElevation = 2.dp)
    ) {
        Column(
            modifier = Modifier.padding(16.dp)
        ) {
            Text(
                text = scene.name,
                style = MaterialTheme.typography.titleMedium
            )
            Spacer(modifier = Modifier.height(4.dp))
            Text(
                text = scene.description,
                style = MaterialTheme.typography.bodyMedium,
                color = MaterialTheme.colorScheme.onSurfaceVariant
            )
        }
    }
}
```

**Elementos de UI:**
- `LazyColumn` para listas eficientes
- `Card` para elementos visuales atractivos
- Estado de carga con `CircularProgressIndicator`
- Callbacks para navegación

### 7. Pantalla de Detalles (Details Screen)

```kotlin
// screens/SceneDetailScreen.kt
package com.plcoding.nav3_guide.screens

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import com.plcoding.nav3_guide.viewmodels.SceneDetailViewModel

@Composable
fun SceneDetailScreen(
    sceneId: String,
    sceneName: String,
    viewModel: SceneDetailViewModel = viewModel {
        SceneDetailViewModel(sceneId, sceneName)
    },
    modifier: Modifier = Modifier
) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    Column(
        modifier = modifier
            .fillMaxSize()
            .background(MaterialTheme.colorScheme.surface)
    ) {
        // Header con información principal
        Surface(
            modifier = Modifier.fillMaxWidth(),
            color = MaterialTheme.colorScheme.primaryContainer
        ) {
            Column(
                modifier = Modifier.padding(24.dp)
            ) {
                Text(
                    text = state.sceneName,
                    style = MaterialTheme.typography.headlineLarge,
                    color = MaterialTheme.colorScheme.onPrimaryContainer
                )
                Spacer(modifier = Modifier.height(8.dp))
                Text(
                    text = "ID: ${state.sceneId}",
                    style = MaterialTheme.typography.bodyMedium,
                    color = MaterialTheme.colorScheme.onPrimaryContainer.copy(alpha = 0.7f)
                )
            }
        }

        if (state.isLoading) {
            Box(
                modifier = Modifier.fillMaxSize(),
                contentAlignment = Alignment.Center
            ) {
                CircularProgressIndicator()
            }
        } else {
            // Contenido de detalles
            LazyColumn(
                modifier = Modifier.fillMaxSize(),
                contentPadding = PaddingValues(16.dp),
                verticalArrangement = Arrangement.spacedBy(12.dp)
            ) {
                item {
                    Card(
                        modifier = Modifier.fillMaxWidth()
                    ) {
                        Column(
                            modifier = Modifier.padding(16.dp)
                        ) {
                            Text(
                                text = "Description",
                                style = MaterialTheme.typography.titleMedium
                            )
                            Spacer(modifier = Modifier.height(8.dp))
                            Text(
                                text = state.description,
                                style = MaterialTheme.typography.bodyMedium
                            )
                        }
                    }
                }

                item {
                    Text(
                        text = "Details",
                        style = MaterialTheme.typography.titleLarge,
                        modifier = Modifier.padding(vertical = 8.dp)
                    )
                }

                items(state.details) { detail ->
                    Card(
                        modifier = Modifier.fillMaxWidth()
                    ) {
                        Text(
                            text = detail,
                            modifier = Modifier.padding(16.dp),
                            style = MaterialTheme.typography.bodyMedium
                        )
                    }
                }
            }
        }
    }
}
```

**Características de diseño:**
- Header destacado con información clave
- Uso de Material 3 design system
- Secciones organizadas en cards
- Layout responsive

## Dependencias Necesarias

Asegúrate de tener estas dependencias en tu `build.gradle.kts`:

```kotlin
commonMain.dependencies {
    implementation(compose.runtime)
    implementation(compose.foundation)
    implementation(compose.material3)
    implementation(compose.ui)
    implementation(compose.components.resources)
    implementation(compose.components.uiToolingPreview)
    
    // Lifecycle y ViewModel
    implementation(libs.androidx.lifecycle.viewmodelCompose)
    implementation(libs.androidx.lifecycle.runtimeCompose)
    
    // Navigation 3.0
    implementation(libs.jetbrains.navigation3.ui)
    implementation(libs.jetbrains.lifecycle.viewmodel.nav3)
    implementation(libs.jetbrains.lifecycle.viewmodel)
    
    // Serialización
    implementation(libs.kotlinx.serialization.json)
}
```

## Integración en la App Principal

```kotlin
// App.kt
package com.plcoding.nav3_guide

import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Scaffold
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import com.plcoding.nav3_guide.navigation.NavigationRoot

@Composable
fun App() {
    MaterialTheme {
        Scaffold { innerPadding ->
            NavigationRoot(
                modifier = Modifier.padding(innerPadding)
            )
        }
    }
}
```

## Mejores Prácticas

### 1. Gestión de Estado
- ✅ Usa `StateFlow` para estado reactivo
- ✅ Mantén el estado inmutable
- ✅ Usa `copy()` para actualizar estados
- ✅ Expón solo flows de solo lectura (`.asStateFlow()`)

### 2. Navegación
- ✅ Define rutas con `@Serializable`
- ✅ Pasa solo datos necesarios en rutas
- ✅ Usa type-safe navigation con clases selladas
- ✅ Configura correctamente el `SerializersModule`

### 3. ViewModels
- ✅ Inyecta parámetros necesarios en el constructor
- ✅ Inicializa datos en `init {}`
- ✅ Limpia recursos en `onCleared()`
- ✅ Usa scoping adecuado con decorators

### 4. Composables
- ✅ Separa lógica de UI
- ✅ Usa `collectAsStateWithLifecycle()` para flows
- ✅ Implementa estados de carga
- ✅ Mantén composables pequeños y reutilizables

### 5. Performance
- ✅ Usa `LazyColumn` para listas largas
- ✅ Implementa keys estables en listas
- ✅ Evita recomposiciones innecesarias
- ✅ Usa `remember` y `derivedStateOf` cuando sea apropiado

## Estructura de Archivos Recomendada

```
composeApp/src/commonMain/kotlin/
├── com.plcoding.nav3_guide/
│   ├── App.kt
│   ├── navigation/
│   │   ├── Route.kt
│   │   └── NavigationRoot.kt
│   ├── screens/
│   │   ├── SceneListScreen.kt
│   │   └── SceneDetailScreen.kt
│   ├── viewmodels/
│   │   ├── SceneListViewModel.kt
│   │   └── SceneDetailViewModel.kt
│   └── models/
│       └── Scene.kt (si se necesita)
```

## Testing

### ViewModel Test Example
```kotlin
class SceneListViewModelTest {
    @Test
    fun `scenes are loaded on init`() = runTest {
        val viewModel = SceneListViewModel()
        val scenes = viewModel.scenes.first()
        
        assertTrue(scenes.isNotEmpty())
        assertEquals(20, scenes.size)
    }
}
```

## Conclusión

Este patrón List-Details es fundamental en aplicaciones móviles y proporciona una navegación intuitiva entre una colección de elementos y sus detalles. La implementación con Navigation 3.0 y Jetpack Compose permite crear UIs declarativas, reactivas y type-safe.

**Características clave implementadas:**
- ✅ Navegación type-safe con Navigation 3.0
- ✅ Arquitectura MVVM con ViewModels
- ✅ UI declarativa con Jetpack Compose
- ✅ Gestión de estado con StateFlow
- ✅ Serialización de argumentos de navegación
- ✅ Material 3 design system
- ✅ Kotlin Multiplatform compatible

## Referencias

- [Navigation 3.0 Documentation](https://www.jetbrains.com/help/kotlin-multiplatform-dev/compose-navigation-routing.html)
- [Jetpack Compose](https://developer.android.com/jetpack/compose)
- [Kotlin Serialization](https://kotlinlang.org/docs/serialization.html)
- [Material 3 Design](https://m3.material.io/)
