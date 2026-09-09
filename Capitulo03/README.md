# 4 Práctica incremental

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 288 minutos |
| Complejidad | Alta |
| Nivel de Bloom | Aplicar |

## Descripción general

En esta práctica migrarás operaciones seleccionadas de `CampusExplorer` desde un flujo reactivo basado en Combine hacia `async/await`. Implementarás una carga concurrente de lugares, categorías y recursos visuales, protegerás una caché mediante un `actor` y mantendrás el estado visual aislado en `MainActor`.

También analizarás el comportamiento antes y después de la optimización con Instruments y documentarás una mejora observable relacionada con CPU, bloqueos de interfaz o asignaciones de memoria.

## Objetivos de aprendizaje

Al finalizar la práctica podrás:

- [ ] Migrar operaciones de red y transformación de datos a `async/await`.
- [ ] Aplicar concurrencia estructurada mediante `async let` y `withThrowingTaskGroup`.
- [ ] Proteger datos compartidos con un actor `CampusCache`.
- [ ] Evitar actualizaciones inseguras de interfaz mediante `@MainActor`.
- [ ] Identificar con Instruments operaciones costosas y documentar una mejora técnica verificable.

## Requisitos previos

### Conocimientos

- Haber completado el laboratorio `02-00-01`.
- Comprender publishers de Combine, `sink`, `AnyCancellable` y propagación de errores.
- Conocer los fundamentos de `async/await`, `Task`, `TaskGroup`, `actor`, `Sendable` y `MainActor`.
- Saber ejecutar pruebas desde Xcode y desde línea de comandos.

### Acceso y recursos

- macOS Sequoia 15.5 o compatible.
- Xcode 16.4 con Swift 6.1 e iOS SDK 18.5.
- Runtime de iOS Simulator 18.5.
- Simulador `iPhone 16`.
- Git 2.46.0 o superior.
- Acceso al repositorio local `CampusExplorer`.
- Etiqueta Git obligatoria `lab-02-complete`.

## Entorno del laboratorio

### Configuración esperada

| Elemento | Valor obligatorio |
|---|---|
| Directorio de trabajo | `~/Developer/CampusExplorer` |
| Repositorio | `CampusExplorer` |
| Rama principal | `main` |
| Proyecto | `CampusExplorer.xcodeproj` |
| Esquema compartido | `CampusExplorer` |
| Deployment target | iOS 18.0 |
| SDK de compilación | iOS 18.5 |
| Simulador | iPhone 16, iOS 18.5 |
| Destino `xcodebuild` | `platform=iOS Simulator,name=iPhone 16,OS=18.5` |
| Paquetes locales | `Packages/CampusDomain`, `Packages/CampusNetworking`, `Packages/CampusDesignSystem` |

### Preparación inicial

1. Abre Terminal y verifica el repositorio y la etiqueta de partida:

   ```bash
   cd ~/Developer/CampusExplorer
   git status
   git tag --list lab-02-complete
   ```

2. Si necesitas restaurar el punto de inicio del laboratorio, crea una rama de trabajo desde la etiqueta:

   ```bash
   git checkout main
   git pull --ff-only
   git checkout -b lab-03-concurrency lab-02-complete
   ```

3. Abre el proyecto:

   ```bash
   open CampusExplorer.xcodeproj
   ```

4. En Xcode, selecciona:

   - Esquema: `CampusExplorer`
   - Destino: `iPhone 16`
   - Runtime: `iOS 18.5`

5. Compila el estado inicial:

   ```bash
   xcodebuild \
     -project CampusExplorer.xcodeproj \
     -scheme CampusExplorer \
     -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5' \
     build
   ```

## Desarrollo paso a paso

### Paso 1. Crear una línea base de rendimiento

**Objetivo:** Identificar el comportamiento inicial de la pantalla de exploración antes de aplicar concurrencia estructurada.

**Instrucciones:**

1. Ejecuta la aplicación desde Xcode con `⌘R`.

2. Navega a la pantalla que carga lugares del campus, categorías y miniaturas o recursos visuales.

3. Realiza una interacción repetible:
   - Abre la pantalla de exploración.
   - Espera a que termine la carga.
   - Cambia de categoría tres veces.
   - Regresa a la lista principal.
   - Repite el flujo dos veces.

4. Abre Instruments desde Xcode:

   ```text
   Xcode > Product > Profile
   ```

5. Selecciona la plantilla **Time Profiler**.

6. Inicia la grabación y repite el flujo de interacción anterior.

7. Detén la grabación y localiza:
   - Métodos de ordenación.
   - Métodos de filtrado.
   - Decodificación de imágenes.
   - Transformaciones de colecciones.
   - Trabajo ejecutado en el hilo principal.

8. Crea una carpeta para evidencias:

   ```bash
   mkdir -p ~/Developer/CampusExplorer/Documentation/Lab03
   ```

9. Registra una medición inicial en el archivo `Documentation/Lab03/mediciones.md`:

   ```markdown
   # Mediciones Lab 03

   ## Línea base

   - Instrumento: Time Profiler
   - Dispositivo: iPhone 16, iOS 18.5
   - Flujo: abrir Exploración, cambiar categorías y volver a la lista
   - Observación:
   - Función costosa detectada:
   - Evidencia visual o captura:
   ```

**Resultado esperado:**

Dispones de una observación inicial sobre una operación costosa o un tramo de trabajo relevante en el hilo principal.

**Verificación:**

- Existe el archivo `Documentation/Lab03/mediciones.md`.
- El archivo incluye una observación de la ejecución inicial.
- Puedes identificar al menos una función de procesamiento, ordenación, filtrado o decodificación en Time Profiler.

---

### Paso 2. Definir contratos asíncronos y tipos transferibles

**Objetivo:** Definir interfaces compatibles con Swift Concurrency y tipos seguros para el intercambio entre tareas.

**Instrucciones:**

1. En el paquete `CampusDomain`, crea o actualiza el archivo:

   ```text
   Packages/CampusDomain/Sources/CampusDomain/Explorer/CampusExplorerModels.swift
   ```

2. Define modelos inmutables conformes a `Sendable`. Adapta las propiedades a los modelos existentes del proyecto si ya dispones de nombres equivalentes.

   ```swift
   import Foundation

   public struct CampusPlace: Codable, Identifiable, Hashable, Sendable {
       public let id: UUID
       public let name: String
       public let categoryID: UUID
       public let latitude: Double
       public let longitude: Double
       public let imageURL: URL?

       public init(
           id: UUID,
           name: String,
           categoryID: UUID,
           latitude: Double,
           longitude: Double,
           imageURL: URL?
       ) {
           self.id = id
           self.name = name
           self.categoryID = categoryID
           self.latitude = latitude
           self.longitude = longitude
           self.imageURL = imageURL
       }
   }

   public struct CampusCategory: Codable, Identifiable, Hashable, Sendable {
       public let id: UUID
       public let name: String

       public init(id: UUID, name: String) {
           self.id = id
           self.name = name
       }
   }

   public struct CampusPlaceGroup: Identifiable, Sendable {
       public let id: UUID
       public let category: CampusCategory
       public let places: [CampusPlace]

       public init(category: CampusCategory, places: [CampusPlace]) {
           self.id = category.id
           self.category = category
           self.places = places
       }
   }

   public struct ExplorerSnapshot: Sendable {
       public let groups: [CampusPlaceGroup]
       public let imageDataByURL: [URL: Data]

       public init(
           groups: [CampusPlaceGroup],
           imageDataByURL: [URL: Data]
       ) {
           self.groups = groups
           self.imageDataByURL = imageDataByURL
       }
   }
   ```

3. Crea contratos asíncronos en:

   ```text
   Packages/CampusDomain/Sources/CampusDomain/Explorer/CampusExplorerRepository.swift
   ```

   ```swift
   import Foundation

   public protocol CampusExplorerRepository: Sendable {
       func fetchPlaces() async throws -> [CampusPlace]
       func fetchCategories() async throws -> [CampusCategory]
       func fetchImageData(from url: URL) async throws -> Data
   }
   ```

4. Si tu repositorio del laboratorio anterior usa Combine, conserva temporalmente su implementación existente. En pasos posteriores crearás un adaptador para exponer métodos `async`.

5. Compila el paquete o el proyecto completo:

   ```bash
   xcodebuild \
     -project CampusExplorer.xcodeproj \
     -scheme CampusExplorer \
     -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5' \
     build
   ```

**Resultado esperado:**

El dominio define entidades inmutables y un contrato asíncrono que no depende de UIKit, Combine ni de View Controllers.

**Verificación:**

- Los modelos relevantes conforman a `Sendable`.
- No se utiliza `@unchecked Sendable`.
- El protocolo `CampusExplorerRepository` contiene funciones `async throws`.

---

### Paso 3. Migrar el repositorio de Combine a async/await

**Objetivo:** Exponer una API asíncrona para el repositorio sin bloquear el hilo principal.

**Instrucciones:**

1. Localiza el repositorio reactivo creado en el laboratorio anterior. Su nombre puede ser similar a:

   ```text
   CampusAPIRepository
   CampusPlacesRepository
   CampusNetworkingRepository
   ```

2. Si el repositorio actual devuelve publishers, implementa un adaptador que espere el primer valor usando una continuación. Crea el archivo:

   ```text
   Packages/CampusNetworking/Sources/CampusNetworking/CombineAsyncAdapter.swift
   ```

3. Añade la siguiente extensión:

   ```swift
   import Combine
   import Foundation

   public extension Publisher where Failure: Error {
       func firstValue() async throws -> Output {
           var cancellable: AnyCancellable?

           return try await withTaskCancellationHandler {
               try await withCheckedThrowingContinuation { continuation in
                   var didResume = false

                   cancellable = self
                       .first()
                       .sink(
                           receiveCompletion: { completion in
                               guard !didResume else { return }
                               didResume = true

                               switch completion {
                               case .finished:
                                   continuation.resume(
                                       throwing: URLError(.badServerResponse)
                                   )
                               case .failure(let error):
                                   continuation.resume(throwing: error)
                               }

                               cancellable?.cancel()
                               cancellable = nil
                           },
                           receiveValue: { value in
                               guard !didResume else { return }
                               didResume = true
                               continuation.resume(returning: value)
                               cancellable?.cancel()
                               cancellable = nil
                           }
                       )
               }
           } onCancel: {
               cancellable?.cancel()
           }
       }
   }
   ```

4. Implementa el protocolo `CampusExplorerRepository` en el repositorio de red existente. El siguiente ejemplo presupone publishers existentes llamados `placesPublisher()`, `categoriesPublisher()` e `imageDataPublisher(url:)`. Sustituye esos nombres por los de tu proyecto.

   ```swift
   import CampusDomain
   import Foundation

   public final class CampusAPIRepository: CampusExplorerRepository, @unchecked Sendable {
       private let service: CampusAPIService

       public init(service: CampusAPIService) {
           self.service = service
       }

       public func fetchPlaces() async throws -> [CampusPlace] {
           try Task.checkCancellation()
           return try await service.placesPublisher().firstValue()
       }

       public func fetchCategories() async throws -> [CampusCategory] {
           try Task.checkCancellation()
           return try await service.categoriesPublisher().firstValue()
       }

       public func fetchImageData(from url: URL) async throws -> Data {
           try Task.checkCancellation()
           return try await service.imageDataPublisher(url: url).firstValue()
       }
   }
   ```

5. **Importante:** si `CampusAPIRepository` contiene estado mutable, no uses `@unchecked Sendable`. En ese caso, aplica una de estas alternativas:

   - Convierte el repositorio en `actor`.
   - Haz inmutable el estado interno.
   - Crea un adaptador `actor` que encapsule el servicio no seguro.
   - Usa directamente `URLSession` con tipos `Sendable`.

6. Una implementación preferible basada en `URLSession` puede seguir esta estructura:

   ```swift
   public struct CampusAPIRepository: CampusExplorerRepository {
       private let session: URLSession
       private let baseURL: URL

       public init(session: URLSession = .shared, baseURL: URL) {
           self.session = session
           self.baseURL = baseURL
       }

       public func fetchPlaces() async throws -> [CampusPlace] {
           let url = baseURL.appending(path: "places")
           let (data, response) = try await session.data(from: url)
           try validate(response)
           return try JSONDecoder().decode([CampusPlace].self, from: data)
       }

       public func fetchCategories() async throws -> [CampusCategory] {
           let url = baseURL.appending(path: "categories")
           let (data, response) = try await session.data(from: url)
           try validate(response)
           return try JSONDecoder().decode([CampusCategory].self, from: data)
       }

       public func fetchImageData(from url: URL) async throws -> Data {
           let (data, response) = try await session.data(from: url)
           try validate(response)
           return data
       }

       private func validate(_ response: URLResponse) throws {
           guard let httpResponse = response as? HTTPURLResponse,
                 (200...299).contains(httpResponse.statusCode) else {
               throw URLError(.badServerResponse)
           }
       }
   }
   ```

**Resultado esperado:**

El módulo de acceso a datos expone métodos `async throws` para lugares, categorías e imágenes.

**Verificación:**

- Las llamadas de red no usan `DispatchQueue.main.sync`.
- La API de dominio no expone `AnyPublisher`.
- La compilación no presenta advertencias de aislamiento ignoradas.
- Las cancelaciones se propagan mediante `Task.checkCancellation()` o APIs cancelables de `URLSession`.

---

### Paso 4. Implementar el actor CampusCache

**Objetivo:** Evitar condiciones de carrera al almacenar respuestas de lugares, categorías y recursos visuales.

**Instrucciones:**

1. En `CampusDomain`, crea el archivo:

   ```text
   Packages/CampusDomain/Sources/CampusDomain/Explorer/CampusCache.swift
   ```

2. Implementa el actor:

   ```swift
   import Foundation

   public actor CampusCache {
       private var places: [CampusPlace]?
       private var categories: [CampusCategory]?
       private var imageDataByURL: [URL: Data] = [:]

       public init() {}

       public func cachedPlaces() -> [CampusPlace]? {
           places
       }

       public func cachedCategories() -> [CampusCategory]? {
           categories
       }

       public func cachedImageData(for url: URL) -> Data? {
           imageDataByURL[url]
       }

       public func store(places: [CampusPlace]) {
           self.places = places
       }

       public func store(categories: [CampusCategory]) {
           self.categories = categories
       }

       public func store(imageData: Data, for url: URL) {
           imageDataByURL[url] = imageData
       }

       public func removeAll() {
           places = nil
           categories = nil
           imageDataByURL.removeAll()
       }

       public func statistics() -> (places: Int, categories: Int, images: Int) {
           (
               places?.count ?? 0,
               categories?.count ?? 0,
               imageDataByURL.count
           )
       }
   }
   ```

3. Añade una prueba unitaria para demostrar accesos concurrentes seguros. Crea o actualiza:

   ```text
   CampusExplorerTests/CampusCacheTests.swift
   ```

   ```swift
   import XCTest
   @testable import CampusDomain

   final class CampusCacheTests: XCTestCase {
       func testStoresImagesConcurrently() async {
           let cache = CampusCache()
           let urls = (0..<20).map {
               URL(string: "https://example.org/\($0).jpg")!
           }

           await withTaskGroup(of: Void.self) { group in
               for url in urls {
                   group.addTask {
                       await cache.store(
                           imageData: Data(url.absoluteString.utf8),
                           for: url
                       )
                   }
               }
           }

           let statistics = await cache.statistics()

           XCTAssertEqual(statistics.images, 20)
       }
   }
   ```

4. Ejecuta las pruebas:

   ```bash
   xcodebuild \
     test \
     -project CampusExplorer.xcodeproj \
     -scheme CampusExplorer \
     -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5'
   ```

**Resultado esperado:**

La caché serializa los accesos a su estado mutable y la prueba concurrente finaliza correctamente.

**Verificación:**

- `CampusCache` está declarado como `actor`.
- Las lecturas y escrituras desde fuera del actor usan `await`.
- La prueba `testStoresImagesConcurrently` pasa.
- No existen diccionarios mutables compartidos sin aislamiento para la caché.

---

### Paso 5. Crear el caso de uso de carga concurrente

**Objetivo:** Cargar lugares y categorías en paralelo, procesar datos fuera del actor principal y descargar imágenes con un grupo de tareas.

**Instrucciones:**

1. Crea el archivo:

   ```text
   Packages/CampusDomain/Sources/CampusDomain/Explorer/LoadCampusExplorerUseCase.swift
   ```

2. Implementa el caso de uso. `async let` se utiliza porque lugares y categorías son dos dependencias independientes y conocidas. `withThrowingTaskGroup` se utiliza porque el número de imágenes es dinámico.

   ```swift
   import Foundation

   public struct LoadCampusExplorerUseCase: Sendable {
       private let repository: any CampusExplorerRepository
       private let cache: CampusCache

       public init(
           repository: any CampusExplorerRepository,
           cache: CampusCache
       ) {
           self.repository = repository
           self.cache = cache
       }

       public func execute() async throws -> ExplorerSnapshot {
           try Task.checkCancellation()

           async let placesTask: [CampusPlace] = loadPlaces()
           async let categoriesTask: [CampusCategory] = loadCategories()

           let (places, categories) = try await (placesTask, categoriesTask)

           try Task.checkCancellation()

           let groups = process(
               places: places,
               categories: categories
           )

           let imageDataByURL = try await loadImages(for: places)

           return ExplorerSnapshot(
               groups: groups,
               imageDataByURL: imageDataByURL
           )
       }

       private func loadPlaces() async throws -> [CampusPlace] {
           if let cached = await cache.cachedPlaces() {
               return cached
           }

           let places = try await repository.fetchPlaces()
           try Task.checkCancellation()
           await cache.store(places: places)
           return places
       }

       private func loadCategories() async throws -> [CampusCategory] {
           if let cached = await cache.cachedCategories() {
               return cached
           }

           let categories = try await repository.fetchCategories()
           try Task.checkCancellation()
           await cache.store(categories: categories)
           return categories
       }

       private func process(
           places: [CampusPlace],
           categories: [CampusCategory]
       ) -> [CampusPlaceGroup] {
           let placesByCategory = Dictionary(
               grouping: places,
               by: \.categoryID
           )

           return categories.compactMap { category in
               guard let categoryPlaces = placesByCategory[category.id] else {
                   return nil
               }

               let sortedPlaces = categoryPlaces.sorted {
                   simulatedDistance(for: $0) < simulatedDistance(for: $1)
               }

               return CampusPlaceGroup(
                   category: category,
                   places: sortedPlaces
               )
           }
       }

       private func simulatedDistance(for place: CampusPlace) -> Double {
           let campusLatitude = 40.4168
           let campusLongitude = -3.7038

           let latitudeDelta = place.latitude - campusLatitude
           let longitudeDelta = place.longitude - campusLongitude

           return (latitudeDelta * latitudeDelta) +
               (longitudeDelta * longitudeDelta)
       }

       private func loadImages(
           for places: [CampusPlace]
       ) async throws -> [URL: Data] {
           let urls = Array(
               Set(places.compactMap(\.imageURL))
           )

           return try await withThrowingTaskGroup(
               of: (URL, Data).self,
               returning: [URL: Data].self
           ) { group in
               for url in urls {
                   group.addTask {
                       try Task.checkCancellation()

                       if let cached = await cache.cachedImageData(for: url) {
                           return (url, cached)
                       }

                       let data = try await repository.fetchImageData(from: url)

                       try Task.checkCancellation()

                       await cache.store(imageData: data, for: url)

                       return (url, data)
                   }
               }

               var result: [URL: Data] = [:]

               for try await (url, data) in group {
                   try Task.checkCancellation()
                   result[url] = data
               }

               return result
           }
       }
   }
   ```

3. Observa las decisiones aplicadas:

   | Decisión | Motivo |
   |---|---|
   | `async let` para lugares y categorías | El número de tareas es fijo y ambas solicitudes son independientes. |
   | `withThrowingTaskGroup` para imágenes | La cantidad de URLs se conoce en tiempo de ejecución. |
   | `Task.checkCancellation()` | Permite detener trabajo adicional cuando la vista desaparece o inicia una nueva carga. |
   | `CampusCache` como actor | Evita carreras al consultar y almacenar datos desde tareas concurrentes. |
   | Procesamiento dentro del caso de uso | Evita realizar ordenación, agrupación y cálculo en el `ViewModel` o View Controller. |

4. Si el volumen de imágenes es elevado, limita la concurrencia. Como primera implementación del laboratorio, descarga todas las URLs disponibles. Como mejora opcional, implementa un límite de cuatro descargas simultáneas.

5. Compila el proyecto.

**Resultado esperado:**

El caso de uso obtiene datos independientes en paralelo, agrupa y ordena lugares fuera del `MainActor`, y descarga recursos visuales mediante un grupo de tareas.

**Verificación:**

- `async let` se utiliza para lugares y categorías.
- `withThrowingTaskGroup` se utiliza para imágenes.
- El orden de los lugares se conserva mediante una ordenación explícita.
- La caché se consulta y actualiza mediante `await`.
- No se crea un `Task.detached` para resolver el procesamiento de la interfaz.

---

### Paso 6. Actualizar el estado visual con MainActor

**Objetivo:** Mantener las mutaciones de estado de presentación en el actor principal y permitir cancelación al abandonar la pantalla.

**Instrucciones:**

1. Localiza el ViewModel, Presenter o controlador de presentación de la pantalla de exploración.

2. Si el proyecto utiliza UIKit con VIPER, el Presenter puede estar aislado en `MainActor`. Si utiliza un ViewModel, aplica el mismo criterio.

3. Implementa una versión equivalente a la siguiente:

   ```swift
   import CampusDomain
   import Foundation
   import UIKit

   @MainActor
   final class CampusExplorerViewModel: ObservableObject {
       @Published private(set) var groups: [CampusPlaceGroup] = []
       @Published private(set) var imagesByURL: [URL: UIImage] = [:]
       @Published private(set) var isLoading = false
       @Published private(set) var errorMessage: String?

       private let loadExplorerUseCase: LoadCampusExplorerUseCase
       private var loadTask: Task<Void, Never>?

       init(loadExplorerUseCase: LoadCampusExplorerUseCase) {
           self.loadExplorerUseCase = loadExplorerUseCase
       }

       func load() {
           loadTask?.cancel()

           loadTask = Task { [weak self] in
               guard let self else { return }

               isLoading = true
               errorMessage = nil

               defer {
                   isLoading = false
               }

               do {
                   let snapshot = try await loadExplorerUseCase.execute()

                   try Task.checkCancellation()

                   groups = snapshot.groups
                   imagesByURL = decodeImages(snapshot.imageDataByURL)
               } catch is CancellationError {
                   // No se muestra un error al usuario por una cancelación esperada.
               } catch {
                   errorMessage = "No fue posible cargar los lugares del campus."
               }
           }
       }

       func cancelLoading() {
           loadTask?.cancel()
           loadTask = nil
       }

       private func decodeImages(_ dataByURL: [URL: Data]) -> [URL: UIImage] {
           dataByURL.reduce(into: [:]) { result, element in
               let (url, data) = element
               result[url] = UIImage(data: data)
           }
       }
   }
   ```

4. Para evitar que la decodificación de imágenes ocurra en `MainActor`, mueve la decodificación a un componente no aislado. Crea una representación visual intermedia si tu arquitectura lo requiere.

   En proyectos UIKit, utiliza un servicio dedicado para decodificar datos antes de actualizar la vista. Un ejemplo simplificado es:

   ```swift
   import Foundation
   import ImageIO
   import UIKit

   enum ImageDecoder {
       nonisolated static func thumbnail(
           from data: Data,
           maxPixelSize: Int = 300
       ) -> UIImage? {
           let options: [CFString: Any] = [
               kCGImageSourceCreateThumbnailFromImageAlways: true,
               kCGImageSourceThumbnailMaxPixelSize: maxPixelSize,
               kCGImageSourceCreateThumbnailWithTransform: true
           ]

           guard let source = CGImageSourceCreateWithData(data as CFData, nil),
                 let image = CGImageSourceCreateThumbnailAtIndex(
                    source,
                    0,
                    options as CFDictionary
                 ) else {
               return nil
           }

           return UIImage(cgImage: image)
       }
   }
   ```

5. Si Swift 6 muestra diagnósticos de aislamiento al devolver `UIImage` desde un contexto no aislado, no los silencies con `@unchecked Sendable`. En ese caso:
   - conserva `Data` como resultado transferible;
   - realiza una decodificación limitada por demanda en la capa de UI;
   - o encapsula el uso de UIKit en una capa específicamente aislada, midiendo su impacto.

6. En el View Controller, inicia y cancela la carga según el ciclo de vida:

   ```swift
   override func viewDidLoad() {
       super.viewDidLoad()
       viewModel.load()
   }

   override func viewDidDisappear(_ animated: Bool) {
       super.viewDidDisappear(animated)
       viewModel.cancelLoading()
   }
   ```

7. Si utilizas SwiftUI, usa una tarea vinculada al ciclo de vida:

   ```swift
   .task {
       viewModel.load()
   }
   .onDisappear {
       viewModel.cancelLoading()
   }
   ```

**Resultado esperado:**

El estado visual se modifica únicamente desde `MainActor`, mientras que la red, agrupación, ordenación y carga concurrente se realizan fuera de la interfaz.

**Verificación:**

- El ViewModel o Presenter está marcado con `@MainActor`.
- La tarea anterior se cancela antes de iniciar una carga nueva.
- `CancellationError` no se presenta como error visible.
- No se usan `DispatchQueue.main.async` como sustituto del aislamiento de Swift Concurrency.
- La pantalla continúa respondiendo mientras se cargan datos.

---

### Paso 7. Añadir pruebas unitarias del caso de uso

**Objetivo:** Verificar la agrupación, el uso de datos y el comportamiento concurrente sin depender de servicios de red reales.

**Instrucciones:**

1. Crea un repositorio falso en el target de pruebas:

   ```swift
   import CampusDomain
   import Foundation

   final class CampusExplorerRepositorySpy: CampusExplorerRepository, @unchecked Sendable {
       var places: [CampusPlace] = []
       var categories: [CampusCategory] = []
       var imageData: [URL: Data] = [:]

       private(set) var fetchPlacesCallCount = 0
       private(set) var fetchCategoriesCallCount = 0

       func fetchPlaces() async throws -> [CampusPlace] {
           fetchPlacesCallCount += 1
           return places
       }

       func fetchCategories() async throws -> [CampusCategory] {
           fetchCategoriesCallCount += 1
           return categories
       }

       func fetchImageData(from url: URL) async throws -> Data {
           imageData[url] ?? Data()
       }
   }
   ```

2. Para un proyecto con comprobación estricta de concurrencia, convierte el spy en `actor`:

   ```swift
   actor CampusExplorerRepositorySpy: CampusExplorerRepository {
       var places: [CampusPlace] = []
       var categories: [CampusCategory] = []
       var imageData: [URL: Data] = [:]

       func fetchPlaces() async throws -> [CampusPlace] {
           places
       }

       func fetchCategories() async throws -> [CampusCategory] {
           categories
       }

       func fetchImageData(from url: URL) async throws -> Data {
           imageData[url] ?? Data()
       }

       func configure(
           places: [CampusPlace],
           categories: [CampusCategory],
           imageData: [URL: Data]
       ) {
           self.places = places
           self.categories = categories
           self.imageData = imageData
       }
   }
   ```

3. Añade una prueba de agrupación y ordenación:

   ```swift
   import XCTest
   @testable import CampusDomain

   final class LoadCampusExplorerUseCaseTests: XCTestCase {
       func testExecuteGroupsPlacesByCategory() async throws {
           let category = CampusCategory(
               id: UUID(),
               name: "Bibliotecas"
           )

           let place = CampusPlace(
               id: UUID(),
               name: "Biblioteca Central",
               categoryID: category.id,
               latitude: 40.4170,
               longitude: -3.7040,
               imageURL: nil
           )

           let repository = CampusExplorerRepositorySpy()
           await repository.configure(
               places: [place],
               categories: [category],
               imageData: [:]
           )

           let useCase = LoadCampusExplorerUseCase(
               repository: repository,
               cache: CampusCache()
           )

           let snapshot = try await useCase.execute()

           XCTAssertEqual(snapshot.groups.count, 1)
           XCTAssertEqual(snapshot.groups.first?.category.name, "Bibliotecas")
           XCTAssertEqual(
               snapshot.groups.first?.places.first?.name,
               "Biblioteca Central"
           )
       }
   }
   ```

4. Ejecuta las pruebas desde Xcode o Terminal:

   ```bash
   xcodebuild \
     test \
     -project CampusExplorer.xcodeproj \
     -scheme CampusExplorer \
     -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5'
   ```

**Resultado esperado:**

Las pruebas validan que la lógica de agrupación no depende de la interfaz ni de la red.

**Verificación:**

- Existe al menos una prueba asíncrona `async throws`.
- El caso de uso se prueba con un repositorio falso.
- Las pruebas pasan en el simulador configurado.
- La lógica de negocio no requiere instanciar un View Controller.

---

### Paso 8. Analizar la versión optimizada con Instruments

**Objetivo:** Comparar el comportamiento de la versión optimizada frente a la línea base y registrar una decisión técnica basada en datos.

**Instrucciones:**

1. Ejecuta la aplicación optimizada.

2. Repite exactamente el flujo definido en el Paso 1.

3. Abre **Time Profiler**:

   ```text
   Xcode > Product > Profile > Time Profiler
   ```

4. Graba el flujo de carga y revisa:

   - Actividad del hilo principal.
   - Tiempo acumulado en ordenaciones y agrupaciones.
   - Tiempo invertido en decodificación de imágenes.
   - Presencia de trabajo prolongado durante la interacción.
   - Métodos propios del caso de uso y la caché.

5. Abre **Allocations** y repite el flujo:

   ```text
   Xcode > Product > Profile > Allocations
   ```

6. Observa:

   - Número de objetos `UIImage`, `Data` y colecciones.
   - Crecimiento sostenido de memoria tras abrir y cerrar la pantalla.
   - Recursos visuales duplicados.
   - Evidencia de reutilización de datos almacenados por `CampusCache`.

7. Activa Main Thread Checker:

   ```text
   Product > Scheme > Edit Scheme > Run > Diagnostics > Main Thread Checker
   ```

8. Ejecuta la aplicación y confirma que no aparecen advertencias relacionadas con actualizaciones de UIKit fuera del hilo principal.

9. Completa `Documentation/Lab03/mediciones.md`:

   ```markdown
   ## Versión optimizada

   - Instrumento: Time Profiler y Allocations
   - Dispositivo: iPhone 16, iOS 18.5
   - Flujo: abrir Exploración, cambiar categorías y volver a la lista
   - Resultado observado:
   - Trabajo desplazado fuera del actor principal:
   - Uso de caché observado:
   - Advertencias de Main Thread Checker: ninguna / describir

   ## Comparación

   - Antes:
   - Después:
   - Mejora observable:
   - Decisión técnica adoptada:
   - Justificación:
   ```

10. Documenta al menos una decisión técnica concreta. Ejemplo válido:

   ```markdown
   - Mejora observable: la ordenación y agrupación ya no aparecen como trabajo
     dominante en el hilo principal durante la transición a la pantalla de
     exploración.
   - Decisión técnica adoptada: usar `async let` para lugares y categorías,
     y `withThrowingTaskGroup` para recursos visuales.
   - Justificación: las dos primeras peticiones tienen cardinalidad fija e
     independencia funcional; las imágenes dependen de una lista dinámica de URLs.
   ```

**Resultado esperado:**

Existe una comparación antes/después con al menos una mejora observable y una decisión técnica justificada.

**Verificación:**

- `Documentation/Lab03/mediciones.md` contiene mediciones iniciales y finales.
- Se usaron Time Profiler y Allocations.
- Main Thread Checker no reporta actualizaciones de UI fuera del contexto permitido.
- La decisión técnica menciona una evidencia obtenida durante el análisis.

## Validación y pruebas

Ejecuta la validación completa antes de entregar:

```bash
cd ~/Developer/CampusExplorer

xcodebuild \
  clean \
  -project CampusExplorer.xcodeproj \
  -scheme CampusExplorer

xcodebuild \
  test \
  -project CampusExplorer.xcodeproj \
  -scheme CampusExplorer \
  -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5'
```

Verifica además los siguientes criterios:

| Criterio | Evidencia esperada |
|---|---|
| Migración a async/await | Repositorio con métodos `async throws`. |
| Concurrencia estructurada | Uso de `async let` y `withThrowingTaskGroup`. |
| Cancelación cooperativa | Uso de `Task.checkCancellation()` o comprobación de `Task.isCancelled`. |
| Estado compartido seguro | `CampusCache` declarado como `actor`. |
| Interfaz segura | ViewModel o Presenter con `@MainActor`. |
| Procesamiento fuera de UI | Agrupación, ordenación y carga de datos fuera del estado visual. |
| Pruebas | Pruebas unitarias asíncronas aprobadas. |
| Rendimiento | Documento con mediciones de Time Profiler y Allocations. |

Registra los cambios en Git:

```bash
git status
git add \
  Packages/CampusDomain \
  Packages/CampusNetworking \
  CampusExplorerTests \
  Documentation/Lab03

git commit -m "feat: migrate explorer loading to Swift Concurrency"
git status
```

## Resolución de problemas

### Problema 1: Swift 6 muestra errores de concurrencia o de `Sendable`

**Síntoma:** Aparecen errores similares a `Task or actor isolated value cannot be sent` o advertencias al pasar una clase, `UIImage` o un servicio de Combine a una tarea concurrente.

**Causa:** Se está transfiriendo un tipo de referencia mutable entre dominios de concurrencia sin aislamiento o sin conformidad segura a `Sendable`.

**Solución:**

1. No agregues `@unchecked Sendable` únicamente para eliminar el diagnóstico.
2. Convierte el estado mutable compartido en un `actor`.
3. Transfiere tipos inmutables como `struct`, `String`, `UUID`, `URL`, `Data` y arreglos de modelos `Sendable`.
4. Mantén objetos UIKit en la capa de presentación o utiliza datos intermedios como `Data`.
5. Revisa que el repositorio no conserve propiedades mutables no protegidas.

### Problema 2: La pantalla sigue bloqueándose o Time Profiler muestra trabajo intenso en el hilo principal

**Síntoma:** La lista tarda en responder, el desplazamiento se vuelve irregular o Time Profiler muestra `sorted`, `Dictionary(grouping:)`, decodificación de imágenes o filtrado en el hilo principal.

**Causa:** El procesamiento pesado se está ejecutando desde un método aislado en `MainActor`, normalmente dentro del ViewModel, Presenter o View Controller.

**Solución:**

1. Mueve agrupación, filtrado, ordenación y cálculo de distancias al caso de uso no aislado.
2. Mantén en `@MainActor` únicamente las mutaciones del estado visual.
3. Reduce el tamaño de las miniaturas durante la decodificación.
4. Comprueba la cancelación dentro de bucles largos.
5. Evita crear una tarea independiente por cada actualización visual; actualiza la interfaz cuando el resultado completo o una unidad de presentación esté preparada.

## Limpieza

1. Detén cualquier sesión activa de Instruments.
2. Cierra el simulador si no lo necesitas:

   ```bash
   xcrun simctl shutdown "iPhone 16"
   ```

3. Revisa que no existan archivos temporales o capturas sin documentar:

   ```bash
   git status
   ```

4. Conserva en el repositorio:

   - Código fuente de `CampusCache`.
   - Caso de uso asíncrono.
   - Pruebas unitarias.
   - `Documentation/Lab03/mediciones.md`.

5. No elimines paquetes locales obligatorios ni modifiques el deployment target global de iOS 18.0.

## Resumen

En esta práctica migraste una carga de datos de `CampusExplorer` a Swift Concurrency. Aplicaste `async/await` para expresar operaciones asíncronas, `async let` para dependencias fijas, `withThrowingTaskGroup` para recursos visuales dinámicos y cancelación cooperativa para evitar trabajo innecesario.

También protegiste respuestas compartidas mediante el actor `CampusCache`, mantuviste el estado de presentación bajo `MainActor` y utilizaste Instruments para justificar una mejora técnica basada en evidencia. Estos elementos constituyen una base reutilizable para módulos VIPER que necesiten separar presentación, lógica de negocio, acceso a datos y rendimiento de interfaz.
