# 4 Práctica incremental

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 252 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Aplicar |

## Descripción general

En esta práctica ampliarás la versión de **CampusExplorer** que incluye mapas y framework, incorporando una estrategia de calidad automatizada. Crearás pruebas unitarias deterministas para `PlacesViewModel` con Combine, pruebas de snapshot para los estados visuales de la lista de lugares y un flujo de integración continua en GitHub Actions.

La práctica evita dependencias no deterministas: no se usará red real, ubicación real, permisos del sistema ni temporizadores arbitrarios. El resultado será un pull request cuya compilación y pruebas se ejecutarán automáticamente.

## Objetivos de aprendizaje

- [ ] Crear los targets `CampusExplorerTests` y `CampusExplorerSnapshotTests`.
- [ ] Probar las transiciones de estado de `PlacesViewModel` usando `XCTestExpectation`, Combine y dependencias inyectadas.
- [ ] Implementar mocks de `PlacesRepositoryProtocol` y `LocationProviding`.
- [ ] Registrar y validar snapshots de los estados cargando, resultados, vacío y error.
- [ ] Configurar GitHub Actions para compilar y ejecutar las pruebas en cada `push` a `main` y en cada pull request.

## Prerrequisitos

### Conocimientos necesarios

- Haber completado la práctica `04-00-01` y disponer de la etiqueta Git `lab-04-complete`.
- Conocer la estructura básica de pruebas con XCTest.
- Comprender los conceptos de `Publisher`, `sink`, `AnyCancellable` y `@Published`.
- Conocer operaciones básicas con Git: ramas, commits, push y pull requests.
- Comprender la inyección de dependencias mediante protocolos.

### Accesos necesarios

- Cuenta de GitHub con permisos de escritura en el repositorio o en un fork de `CampusExplorer`.
- Acceso a Internet para resolver dependencias de Swift Package Manager y ejecutar GitHub Actions.
- Permisos locales para ejecutar Xcode y el simulador de iOS.

## Entorno de laboratorio

### Hardware recomendado

| Recurso | Requisito |
|---|---|
| Equipo | Mac Apple Silicon o Intel compatible |
| Sistema operativo | macOS Sequoia 15.5 |
| Memoria | 16 GB recomendados; 8 GB mínimos |
| Espacio libre | 40 GB recomendados |
| Dispositivo físico | Opcional: iPhone con iOS 18.5 |

### Software requerido

| Herramienta | Versión |
|---|---|
| Xcode | 16.4 |
| Swift | 6.1 |
| iOS SDK | 18.5 |
| iOS Simulator | iPhone 16, iOS 18.5 |
| Git | 2.46.0 o compatible |
| Swift Snapshot Testing | 1.17.0 |
| `actions/checkout` | 4.2.2 |
| `maxim-lobanov/setup-xcode` | 1.6.0 |

### Preparación inicial

El directorio de trabajo obligatorio es `~/Developer/CampusExplorer`. Ejecuta los siguientes comandos en Terminal:

```bash
mkdir -p ~/Developer
cd ~/Developer

git clone <URL_DE_TU_REPOSITORIO> CampusExplorer
cd CampusExplorer

git checkout main
git pull origin main
git tag --list lab-04-complete
```

Se debe mostrar la etiqueta:

```text
lab-04-complete
```

Crea una rama específica para la práctica:

```bash
git checkout -b feature/lab-05-testing-ci
```

Comprueba la versión de Xcode activa y los destinos disponibles:

```bash
xcodebuild -version

xcrun simctl list devices available | grep "iPhone 16"
```

Salida esperada aproximada:

```text
Xcode 16.4
Build version 16F6

iPhone 16 (... ) (Booted)
```

Abre el proyecto:

```bash
open CampusExplorer.xcodeproj
```

Confirma en Xcode lo siguiente:

- El proyecto se llama `CampusExplorer.xcodeproj`.
- El esquema compartido se llama `CampusExplorer`.
- El deployment target global es iOS 18.0.
- El destino de ejecución es **iPhone 16 (iOS 18.5)**.
- Existen los paquetes locales:
  - `Packages/CampusDomain`
  - `Packages/CampusNetworking`
  - `Packages/CampusDesignSystem`

## Procedimiento paso a paso

### Paso 1. Verificar el punto de partida y compartir el esquema

**Objetivo:** asegurar que el proyecto base compila y que el esquema será visible para GitHub Actions.

**Instrucciones:**

1. En Xcode, selecciona el esquema `CampusExplorer`.
2. Selecciona el destino **iPhone 16 (iOS 18.5)**.
3. Ejecuta la aplicación con `⌘R`.
4. Verifica que la pantalla de lugares y el mapa siguen funcionando como en el laboratorio anterior.
5. Abre **Product > Scheme > Manage Schemes…**.
6. Marca la casilla **Shared** para el esquema `CampusExplorer`.
7. Cierra la ventana de esquemas.
8. Comprueba que Xcode haya creado o actualizado el archivo compartido:

```bash
find . -path "*xcshareddata/xcschemes*" -type f
```

9. Compila desde Terminal usando el destino obligatorio:

```bash
xcodebuild build \
  -project CampusExplorer.xcodeproj \
  -scheme CampusExplorer \
  -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5'
```

**Salida esperada:**

La compilación termina con:

```text
** BUILD SUCCEEDED **
```

Debe existir un archivo similar a:

```text
CampusExplorer.xcodeproj/xcshareddata/xcschemes/CampusExplorer.xcscheme
```

**Verificación:**

Ejecuta:

```bash
git status --short
```

Si el esquema no estaba compartido, debe aparecer como archivo nuevo o modificado. No continúes hasta que el esquema sea compartido, porque GitHub Actions no podrá localizar un esquema privado de usuario.

---

### Paso 2. Revisar y preparar el contrato comprobable de `PlacesViewModel`

**Objetivo:** asegurar que `PlacesViewModel` recibe dependencias abstraídas y publica estados observables.

**Instrucciones:**

1. Localiza `PlacesViewModel`, `PlacesRepositoryProtocol` y `LocationProviding`.
2. Verifica que el ViewModel no cree directamente:
   - Un cliente HTTP.
   - Un `CLLocationManager`.
   - Un repositorio concreto.
3. Si fuese necesario, adapta el ViewModel para recibir las dependencias en su inicializador.
4. Usa un estado de presentación explícito y comparable. El contrato de referencia para esta práctica es el siguiente; conserva los nombres existentes si tu proyecto ya los utiliza, pero mantén el mismo comportamiento funcional:

```swift
import Combine
import CoreLocation
import Foundation

enum PlacesViewState: Equatable {
    case idle
    case loading
    case loaded([Place])
    case empty
    case error(String)
}

protocol PlacesRepositoryProtocol {
    func fetchPlaces(
        near coordinate: CLLocationCoordinate2D
    ) -> AnyPublisher<[Place], Error>
}

protocol LocationProviding {
    func requestCurrentLocation() -> AnyPublisher<CLLocationCoordinate2D, Error>
}
```

5. Implementa o revisa una variante equivalente de `PlacesViewModel`:

```swift
import Combine
import CoreLocation
import Foundation

final class PlacesViewModel: ObservableObject {
    @Published private(set) var state: PlacesViewState = .idle

    private let repository: PlacesRepositoryProtocol
    private let locationProvider: LocationProviding
    private var cancellables = Set<AnyCancellable>()

    init(
        repository: PlacesRepositoryProtocol,
        locationProvider: LocationProviding
    ) {
        self.repository = repository
        self.locationProvider = locationProvider
    }

    func loadPlaces() {
        state = .loading

        locationProvider.requestCurrentLocation()
            .flatMap { [repository] coordinate in
                repository.fetchPlaces(near: coordinate)
            }
            .map { places -> PlacesViewState in
                places.isEmpty ? .empty : .loaded(places)
            }
            .catch { error in
                Just(.error(error.localizedDescription))
            }
            .receive(on: DispatchQueue.main)
            .sink { [weak self] newState in
                self?.state = newState
            }
            .store(in: &cancellables)
    }
}
```

6. Si el tipo `Place` no implementa `Equatable`, añádelo en el paquete o módulo donde se declara. Por ejemplo:

```swift
struct Place: Equatable, Identifiable {
    let id: String
    let name: String
    let address: String
    let latitude: Double
    let longitude: Double
}
```

7. Asegúrate de que los mensajes de error usados por el ViewModel sean estables y aptos para pruebas. Evita comparar directamente errores técnicos de `URLSession`.

**Salida esperada:**

El ViewModel:

- Expone `@Published private(set) var state`.
- Recibe un repositorio y un proveedor de ubicación por inyección.
- Publica `.loading` antes del resultado.
- Convierte una lista vacía en `.empty`.
- Convierte errores en `.error(String)`.

**Verificación:**

Compila el proyecto:

```bash
xcodebuild build \
  -project CampusExplorer.xcodeproj \
  -scheme CampusExplorer \
  -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5'
```

La salida debe contener:

```text
** BUILD SUCCEEDED **
```

---

### Paso 3. Crear el target de pruebas unitarias

**Objetivo:** crear el target `CampusExplorerTests` y conectarlo al código de la aplicación.

**Instrucciones:**

1. En Xcode, selecciona el proyecto `CampusExplorer` en el navegador.
2. Pulsa el botón **+** en la sección **TARGETS**.
3. Selecciona **Unit Testing Bundle**.
4. Configura:
   - Product Name: `CampusExplorerTests`
   - Testing System: `XCTest`
   - Target to be Tested: `CampusExplorer`
5. Pulsa **Finish**.
6. Si Xcode solicita activar el esquema, conserva el esquema actual `CampusExplorer`.
7. Selecciona el target `CampusExplorerTests`.
8. En **Build Settings**, verifica:
   - `iOS Deployment Target`: `18.0`.
   - `Swift Language Version`: `Swift 6`.
9. En **Build Phases > Target Dependencies**, confirma que el target principal `CampusExplorer` se encuentre disponible como host de pruebas.
10. Elimina el archivo de prueba generado por Xcode si no vas a utilizarlo y crea el archivo:

```text
CampusExplorerTests/PlacesViewModelTests.swift
```

11. Importa el módulo de la aplicación mediante `@testable import`. Si el nombre del módulo difiere del nombre del proyecto, usa el nombre indicado en **Build Settings > Product Module Name**.

```swift
import Combine
import CoreLocation
import XCTest
@testable import CampusExplorer
```

**Salida esperada:**

El navegador del proyecto debe mostrar un grupo y target denominados:

```text
CampusExplorerTests
```

**Verificación:**

Ejecuta las pruebas desde Xcode con `⌘U`. Aunque todavía no existan pruebas funcionales, la compilación del bundle de pruebas debe finalizar correctamente.

Desde Terminal:

```bash
xcodebuild test \
  -project CampusExplorer.xcodeproj \
  -scheme CampusExplorer \
  -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5' \
  -only-testing:CampusExplorerTests
```

La salida esperada incluye:

```text
** TEST SUCCEEDED **
```

---

### Paso 4. Crear dobles de prueba deterministas

**Objetivo:** simular ubicación y repositorio sin usar Core Location, red, permisos ni temporizadores.

**Instrucciones:**

1. Crea el archivo:

```text
CampusExplorerTests/Support/PlacesTestDoubles.swift
```

2. Añade un error estable para las pruebas:

```swift
import Combine
import CoreLocation
import Foundation
@testable import CampusExplorer

enum PlacesTestError: LocalizedError {
    case networkUnavailable
    case locationDenied

    var errorDescription: String? {
        switch self {
        case .networkUnavailable:
            return "No hay conexión disponible."
        case .locationDenied:
            return "No fue posible obtener la ubicación."
        }
    }
}
```

3. Implementa un stub de ubicación basado en `Result`:

```swift
final class LocationProviderStub: LocationProviding {
    var result: Result<CLLocationCoordinate2D, Error>?
    private(set) var requestCount = 0

    func requestCurrentLocation() -> AnyPublisher<CLLocationCoordinate2D, Error> {
        requestCount += 1

        guard let result else {
            fatalError("Configura LocationProviderStub.result antes de ejecutar la prueba.")
        }

        return result.publisher.eraseToAnyPublisher()
    }
}
```

4. Implementa un stub de repositorio:

```swift
final class PlacesRepositoryStub: PlacesRepositoryProtocol {
    var result: Result<[Place], Error>?
    private(set) var requestedCoordinate: CLLocationCoordinate2D?
    private(set) var requestCount = 0

    func fetchPlaces(
        near coordinate: CLLocationCoordinate2D
    ) -> AnyPublisher<[Place], Error> {
        requestCount += 1
        requestedCoordinate = coordinate

        guard let result else {
            fatalError("Configura PlacesRepositoryStub.result antes de ejecutar la prueba.")
        }

        return result.publisher.eraseToAnyPublisher()
    }
}
```

5. Añade un repositorio controlado manualmente para comprobar que `.loading` se publica antes de recibir datos:

```swift
final class ControlledPlacesRepository: PlacesRepositoryProtocol {
    let subject = PassthroughSubject<[Place], Error>()
    private(set) var requestedCoordinate: CLLocationCoordinate2D?

    func fetchPlaces(
        near coordinate: CLLocationCoordinate2D
    ) -> AnyPublisher<[Place], Error> {
        requestedCoordinate = coordinate
        return subject.eraseToAnyPublisher()
    }
}
```

6. Añade una fábrica de lugares. Ajusta los parámetros al inicializador real de tu tipo `Place`:

```swift
func makePlace(
    id: String = "library",
    name: String = "Biblioteca Central",
    address: String = "Avenida del Campus, 1",
    latitude: Double = 40.4168,
    longitude: Double = -3.7038
) -> Place {
    Place(
        id: id,
        name: name,
        address: address,
        latitude: latitude,
        longitude: longitude
    )
}
```

**Salida esperada:**

Los dobles de prueba deben devolver publicadores configurables con:

- Éxito inmediato: `.success(valor)`.
- Error inmediato: `.failure(error)`.
- Emisión manual: `PassthroughSubject`.

**Verificación:**

Comprueba que ningún doble use:

```swift
CLLocationManager()
URLSession.shared
DispatchQueue.asyncAfter
sleep(...)
```

Los tests deben controlar completamente el momento de cada respuesta.

---

### Paso 5. Implementar pruebas de estados de `PlacesViewModel`

**Objetivo:** validar de forma determinista los estados de éxito, vacío, error y carga.

**Instrucciones:**

1. Abre `CampusExplorerTests/PlacesViewModelTests.swift`.
2. Crea la estructura base de las pruebas:

```swift
import Combine
import CoreLocation
import XCTest
@testable import CampusExplorer

final class PlacesViewModelTests: XCTestCase {
    private var repository: PlacesRepositoryStub!
    private var locationProvider: LocationProviderStub!
    private var viewModel: PlacesViewModel!
    private var cancellables: Set<AnyCancellable>!

    private let coordinate = CLLocationCoordinate2D(
        latitude: 40.4168,
        longitude: -3.7038
    )

    override func setUp() {
        super.setUp()

        repository = PlacesRepositoryStub()
        locationProvider = LocationProviderStub()
        cancellables = []

        viewModel = PlacesViewModel(
            repository: repository,
            locationProvider: locationProvider
        )
    }

    override func tearDown() {
        cancellables = nil
        viewModel = nil
        locationProvider = nil
        repository = nil

        super.tearDown()
    }
}
```

3. Añade la prueba de éxito. Esta prueba verifica el estado final, la secuencia y las llamadas a dependencias:

```swift
func testLoadPlaces_whenRepositorySucceeds_publishesLoadingAndLoadedStates() {
    let expectedPlaces = [
        makePlace(),
        makePlace(id: "cafeteria", name: "Cafetería Norte")
    ]

    locationProvider.result = .success(coordinate)
    repository.result = .success(expectedPlaces)

    let expectation = expectation(
        description: "El ViewModel publica el estado loaded con los lugares esperados."
    )

    var receivedStates: [PlacesViewState] = []

    viewModel.$state
        .sink { state in
            receivedStates.append(state)

            if state == .loaded(expectedPlaces) {
                expectation.fulfill()
            }
        }
        .store(in: &cancellables)

    viewModel.loadPlaces()

    wait(for: [expectation], timeout: 1.0)

    XCTAssertEqual(locationProvider.requestCount, 1)
    XCTAssertEqual(repository.requestCount, 1)
    XCTAssertEqual(repository.requestedCoordinate?.latitude, coordinate.latitude)
    XCTAssertEqual(repository.requestedCoordinate?.longitude, coordinate.longitude)
    XCTAssertEqual(viewModel.state, .loaded(expectedPlaces))
    XCTAssertEqual(
        receivedStates,
        [.idle, .loading, .loaded(expectedPlaces)]
    )
}
```

4. Añade la prueba de lista vacía:

```swift
func testLoadPlaces_whenRepositoryReturnsEmptyList_publishesEmptyState() {
    locationProvider.result = .success(coordinate)
    repository.result = .success([])

    let expectation = expectation(
        description: "El ViewModel publica el estado empty."
    )

    viewModel.$state
        .dropFirst()
        .sink { state in
            if state == .empty {
                expectation.fulfill()
            }
        }
        .store(in: &cancellables)

    viewModel.loadPlaces()

    wait(for: [expectation], timeout: 1.0)

    XCTAssertEqual(viewModel.state, .empty)
    XCTAssertEqual(repository.requestCount, 1)
}
```

5. Añade la prueba de error del repositorio:

```swift
func testLoadPlaces_whenRepositoryFails_publishesErrorState() {
    locationProvider.result = .success(coordinate)
    repository.result = .failure(PlacesTestError.networkUnavailable)

    let expectedMessage = "No hay conexión disponible."
    let expectation = expectation(
        description: "El ViewModel publica un estado de error de red."
    )

    viewModel.$state
        .dropFirst()
        .sink { state in
            if state == .error(expectedMessage) {
                expectation.fulfill()
            }
        }
        .store(in: &cancellables)

    viewModel.loadPlaces()

    wait(for: [expectation], timeout: 1.0)

    XCTAssertEqual(viewModel.state, .error(expectedMessage))
    XCTAssertEqual(repository.requestCount, 1)
}
```

6. Añade la prueba de error de ubicación:

```swift
func testLoadPlaces_whenLocationFails_publishesErrorAndDoesNotRequestRepository() {
    locationProvider.result = .failure(PlacesTestError.locationDenied)

    let expectedMessage = "No fue posible obtener la ubicación."
    let expectation = expectation(
        description: "El ViewModel publica un estado de error de ubicación."
    )

    viewModel.$state
        .dropFirst()
        .sink { state in
            if state == .error(expectedMessage) {
                expectation.fulfill()
            }
        }
        .store(in: &cancellables)

    viewModel.loadPlaces()

    wait(for: [expectation], timeout: 1.0)

    XCTAssertEqual(viewModel.state, .error(expectedMessage))
    XCTAssertEqual(repository.requestCount, 0)
}
```

7. Implementa la prueba de carga usando el repositorio controlado. Como el test necesita un tipo de repositorio distinto, crea el ViewModel dentro del método:

```swift
func testLoadPlaces_beforeRepositoryResponds_keepsLoadingState() {
    let controlledRepository = ControlledPlacesRepository()
    locationProvider.result = .success(coordinate)

    let controlledViewModel = PlacesViewModel(
        repository: controlledRepository,
        locationProvider: locationProvider
    )

    let loadingExpectation = expectation(
        description: "El ViewModel publica loading antes de recibir lugares."
    )

    var receivedStates: [PlacesViewState] = []

    controlledViewModel.$state
        .sink { state in
            receivedStates.append(state)

            if state == .loading {
                loadingExpectation.fulfill()
            }
        }
        .store(in: &cancellables)

    controlledViewModel.loadPlaces()

    wait(for: [loadingExpectation], timeout: 1.0)

    XCTAssertEqual(controlledViewModel.state, .loading)
    XCTAssertEqual(receivedStates, [.idle, .loading])

    controlledRepository.subject.send(completion: .finished)
}
```

8. Si el compilador indica que `CLLocationCoordinate2D` no es comparable, no compares el valor completo; compara latitud y longitud como se muestra en la prueba de éxito.

**Salida esperada:**

La clase contiene al menos cinco pruebas:

```text
testLoadPlaces_whenRepositorySucceeds_publishesLoadingAndLoadedStates
testLoadPlaces_whenRepositoryReturnsEmptyList_publishesEmptyState
testLoadPlaces_whenRepositoryFails_publishesErrorState
testLoadPlaces_whenLocationFails_publishesErrorAndDoesNotRequestRepository
testLoadPlaces_beforeRepositoryResponds_keepsLoadingState
```

**Verificación:**

Ejecuta únicamente las pruebas del ViewModel:

```bash
xcodebuild test \
  -project CampusExplorer.xcodeproj \
  -scheme CampusExplorer \
  -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5' \
  -only-testing:CampusExplorerTests/PlacesViewModelTests
```

La salida debe terminar con:

```text
** TEST SUCCEEDED **
```

Las pruebas no deben contener `sleep`, esperas activas ni `asyncAfter`. El límite de un segundo de `XCTestExpectation` es un mecanismo de detección de fallo, no una forma de sincronización normal.

---

### Paso 6. Crear el target de snapshots y añadir Swift Snapshot Testing

**Objetivo:** configurar `CampusExplorerSnapshotTests` con Swift Snapshot Testing 1.17.0.

**Instrucciones:**

1. En Xcode, crea un nuevo target mediante **File > New > Target…**.
2. Selecciona **Unit Testing Bundle**.
3. Configura:
   - Product Name: `CampusExplorerSnapshotTests`
   - Testing System: `XCTest`
   - Target to be Tested: `CampusExplorer`
4. En **File > Add Package Dependencies…**, añade la dependencia:

```text
https://github.com/pointfreeco/swift-snapshot-testing.git
```

5. Selecciona la versión exacta:

```text
1.17.0
```

6. Añade el producto `SnapshotTesting` exclusivamente al target:

```text
CampusExplorerSnapshotTests
```

7. Verifica que `SnapshotTesting` no se añada al target de producción si no es necesario.
8. En el target `CampusExplorerSnapshotTests`, configura:
   - Deployment target: iOS 18.0.
   - Host application: `CampusExplorer`.
9. Crea el archivo:

```text
CampusExplorerSnapshotTests/PlacesListSnapshotTests.swift
```

10. Comprueba en el navegador de paquetes que Swift Package Manager haya resuelto la versión solicitada.

**Salida esperada:**

En Xcode debe aparecer la dependencia:

```text
swift-snapshot-testing 1.17.0
```

Y el proyecto debe contener:

```text
CampusExplorerSnapshotTests
```

**Verificación:**

Ejecuta:

```bash
xcodebuild -resolvePackageDependencies \
  -project CampusExplorer.xcodeproj \
  -scheme CampusExplorer
```

La resolución debe finalizar sin errores. Comprueba los cambios pendientes:

```bash
git status --short
```

Es normal que aparezcan cambios en:

```text
Package.resolved
CampusExplorer.xcodeproj/project.pbxproj
```

---

### Paso 7. Implementar snapshots de la lista de lugares

**Objetivo:** registrar y comparar las pantallas de lista en estado cargando, con resultados, vacía y con error.

**Instrucciones:**

1. Asegúrate de que el controlador de lista pueda recibir o mostrar un estado conocido sin iniciar automáticamente una consulta de red o ubicación.
2. Si tu controlador actual inicia `loadPlaces()` en `viewDidLoad`, añade una opción para inyectar el ViewModel y controla explícitamente cuándo se carga. Por ejemplo:

```swift
final class PlacesListViewController: UIViewController {
    private let viewModel: PlacesViewModel
    private var shouldLoadOnAppear: Bool

    init(
        viewModel: PlacesViewModel,
        shouldLoadOnAppear: Bool = true
    ) {
        self.viewModel = viewModel
        self.shouldLoadOnAppear = shouldLoadOnAppear
        super.init(nibName: nil, bundle: nil)
    }

    required init?(coder: NSCoder) {
        fatalError("init(coder:) no está implementado")
    }

    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)

        if shouldLoadOnAppear {
            viewModel.loadPlaces()
        }
    }
}
```

3. Expón una forma controlada de actualizar la interfaz según el estado. Por ejemplo, mantén la vinculación Combine de producción y usa un ViewModel construido con stubs en las pruebas.
4. Crea una utilidad de pruebas para generar el controlador. Ajusta el nombre de tu controlador real si se denomina de otra forma:

```swift
import Combine
import CoreLocation
import XCTest
import SnapshotTesting
@testable import CampusExplorer

final class PlacesListSnapshotTests: XCTestCase {
    private let coordinate = CLLocationCoordinate2D(
        latitude: 40.4168,
        longitude: -3.7038
    )

    override func setUp() {
        super.setUp()

        isRecording = false
    }

    private func makeViewController(
        repositoryResult: Result<[Place], Error>
    ) -> PlacesListViewController {
        let repository = PlacesRepositoryStub()
        repository.result = repositoryResult

        let locationProvider = LocationProviderStub()
        locationProvider.result = .success(coordinate)

        let viewModel = PlacesViewModel(
            repository: repository,
            locationProvider: locationProvider
        )

        return PlacesListViewController(
            viewModel: viewModel,
            shouldLoadOnAppear: false
        )
    }

    private func prepareForSnapshot(
        _ viewController: PlacesListViewController
    ) {
        viewController.loadViewIfNeeded()
        viewController.view.frame = CGRect(
            x: 0,
            y: 0,
            width: 390,
            height: 844
        )
    }
}
```

5. Si los dobles están definidos únicamente en `CampusExplorerTests`, no serán visibles desde `CampusExplorerSnapshotTests`. Puedes:
   - Duplicar los dobles mínimos en el target de snapshots, o
   - Crear un grupo `TestSupport` y marcar sus archivos como miembros de ambos targets.

6. Implementa el snapshot del estado cargando usando `ControlledPlacesRepository`. Como el estado debe permanecer en carga, no envíes ningún valor al `subject`:

```swift
func testPlacesList_loading() {
    let repository = ControlledPlacesRepository()

    let locationProvider = LocationProviderStub()
    locationProvider.result = .success(coordinate)

    let viewModel = PlacesViewModel(
        repository: repository,
        locationProvider: locationProvider
    )

    let viewController = PlacesListViewController(
        viewModel: viewModel,
        shouldLoadOnAppear: false
    )

    prepareForSnapshot(viewController)

    let loadingExpectation = expectation(
        description: "La pantalla recibe el estado loading."
    )

    var cancellable: AnyCancellable?
    cancellable = viewModel.$state
        .dropFirst()
        .sink { state in
            if state == .loading {
                loadingExpectation.fulfill()
            }
        }

    viewModel.loadPlaces()
    wait(for: [loadingExpectation], timeout: 1.0)

    assertSnapshot(
        matching: viewController,
        as: .image(on: .iPhone13)
    )

    cancellable?.cancel()
}
```

7. Implementa el snapshot con resultados:

```swift
func testPlacesList_loaded() {
    let places = [
        makePlace(),
        makePlace(
            id: "cafeteria",
            name: "Cafetería Norte",
            address: "Plaza de Ingeniería, 2"
        )
    ]

    let viewController = makeViewController(
        repositoryResult: .success(places)
    )

    prepareForSnapshot(viewController)

    let expectation = expectation(
        description: "La pantalla muestra los lugares cargados."
    )

    viewController.viewModel.loadPlaces()

    DispatchQueue.main.async {
        expectation.fulfill()
    }

    wait(for: [expectation], timeout: 1.0)

    assertSnapshot(
        matching: viewController,
        as: .image(on: .iPhone13)
    )
}
```

8. Si `viewModel` es privado en el controlador, expón un método de carga para pruebas o inyecta un closure de arranque. No fuerces el acceso a propiedades privadas desde el target de pruebas.
9. Implementa el snapshot vacío:

```swift
func testPlacesList_empty() {
    let viewController = makeViewController(
        repositoryResult: .success([])
    )

    prepareForSnapshot(viewController)

    viewController.viewModel.loadPlaces()

    let expectation = expectation(
        description: "La pantalla muestra el estado vacío."
    )

    DispatchQueue.main.async {
        expectation.fulfill()
    }

    wait(for: [expectation], timeout: 1.0)

    assertSnapshot(
        matching: viewController,
        as: .image(on: .iPhone13)
    )
}
```

10. Implementa el snapshot de error:

```swift
func testPlacesList_error() {
    let viewController = makeViewController(
        repositoryResult: .failure(PlacesTestError.networkUnavailable)
    )

    prepareForSnapshot(viewController)

    viewController.viewModel.loadPlaces()

    let expectation = expectation(
        description: "La pantalla muestra el estado de error."
    )

    DispatchQueue.main.async {
        expectation.fulfill()
    }

    wait(for: [expectation], timeout: 1.0)

    assertSnapshot(
        matching: viewController,
        as: .image(on: .iPhone13)
    )
}
```

11. En una implementación de producción, es preferible que el controlador publique una API interna para iniciar la carga o que el ViewModel se inyecte desde el Coordinator. Si necesitas acceder al ViewModel desde un test, no conviertas una propiedad `private` en pública solo para probarla; usa una API de intención como `loadPlacesForTesting()` o controla la carga durante la inicialización.
12. Antes de grabar, revisa que la interfaz no dependa de:
    - La fecha u hora actuales.
    - Posición GPS real.
    - Animaciones activas.
    - Datos remotos.
    - Tipografías descargadas dinámicamente.
    - Un mapa de Apple cargado por red dentro de la vista capturada.

**Salida esperada:**

Se han definido cuatro pruebas de snapshot:

```text
testPlacesList_loading
testPlacesList_loaded
testPlacesList_empty
testPlacesList_error
```

**Verificación:**

Compila el target de snapshots:

```bash
xcodebuild test \
  -project CampusExplorer.xcodeproj \
  -scheme CampusExplorer \
  -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5' \
  -only-testing:CampusExplorerSnapshotTests
```

En la primera ejecución los snapshots aún no existen, por lo que es normal que falle indicando que faltan referencias. Regístralas en el paso siguiente.

---

### Paso 8. Registrar y confirmar las imágenes de referencia

**Objetivo:** generar los archivos de referencia que usarán las pruebas de snapshot.

**Instrucciones:**

1. En `PlacesListSnapshotTests.swift`, cambia temporalmente:

```swift
isRecording = false
```

por:

```swift
isRecording = true
```

2. Ejecuta solamente el target de snapshots desde Xcode con `⌘U`, o desde Terminal:

```bash
xcodebuild test \
  -project CampusExplorer.xcodeproj \
  -scheme CampusExplorer \
  -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5' \
  -only-testing:CampusExplorerSnapshotTests
```

3. Localiza los directorios de snapshots generados:

```bash
find . -type d -name "__Snapshots__"
find . -type f -name "*.png" | grep Snapshots
```

4. Revisa visualmente las imágenes generadas. Deben representar de forma clara:
   - Indicador de carga.
   - Lista con dos lugares.
   - Mensaje de lista vacía.
   - Mensaje de error con una acción de reintento, si tu interfaz la proporciona.

5. Restaura el modo de comparación:

```swift
isRecording = false
```

6. Ejecuta las pruebas otra vez.

**Salida esperada:**

Los directorios `__Snapshots__` contienen imágenes PNG de referencia y las pruebas finalizan correctamente en modo comparación.

**Verificación:**

Ejecuta:

```bash
xcodebuild test \
  -project CampusExplorer.xcodeproj \
  -scheme CampusExplorer \
  -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5' \
  -only-testing:CampusExplorerSnapshotTests
```

La salida debe contener:

```text
** TEST SUCCEEDED **
```

Comprueba que los PNG se versionarán:

```bash
git status --short
```

Los snapshots son artefactos de prueba obligatorios y deben incluirse en el commit.

---

### Paso 9. Ejecutar la batería local completa

**Objetivo:** validar conjuntamente las pruebas unitarias y de snapshot antes de configurar la integración continua.

**Instrucciones:**

1. Ejecuta todas las pruebas del esquema compartido:

```bash
xcodebuild test \
  -project CampusExplorer.xcodeproj \
  -scheme CampusExplorer \
  -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5' \
  -derivedDataPath build/DerivedData
```

2. Revisa el resumen de pruebas en la salida.
3. Si quieres abrir el resultado de pruebas generado por Xcode, localiza el `.xcresult`:

```bash
find build/DerivedData -name "*.xcresult" -maxdepth 6
```

4. Ejecuta también la aplicación en el simulador para comprobar que los cambios de inyección no han roto el flujo real:

```bash
xcodebuild build \
  -project CampusExplorer.xcodeproj \
  -scheme CampusExplorer \
  -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5'
```

**Salida esperada:**

La batería completa termina con:

```text
** TEST SUCCEEDED **
```

**Verificación:**

Confirma que:

- Todas las pruebas unitarias son verdes.
- Los cuatro snapshots son verdes.
- La aplicación sigue compilando.
- No se requieren permisos de ubicación ni conectividad para ejecutar los tests.

---

### Paso 10. Configurar GitHub Actions

**Objetivo:** ejecutar compilación y pruebas automáticamente en `push` a `main` y pull requests.

**Instrucciones:**

1. Crea el directorio del workflow:

```bash
mkdir -p .github/workflows
```

2. Crea el archivo `.github/workflows/ios-ci.yml`:

```yaml
name: iOS CI

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  test:
    name: Build and Test
    runs-on: macos-15
    timeout-minutes: 30

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4.2.2

      - name: Select Xcode 16.4
        uses: maxim-lobanov/setup-xcode@v1.6.0
        with:
          xcode-version: "16.4"

      - name: Show environment
        run: |
          xcodebuild -version
          xcrun simctl list devices available

      - name: Resolve Swift packages
        run: |
          xcodebuild -resolvePackageDependencies \
            -project CampusExplorer.xcodeproj \
            -scheme CampusExplorer

      - name: Build and run tests
        run: |
          set -o pipefail

          xcodebuild test \
            -project CampusExplorer.xcodeproj \
            -scheme CampusExplorer \
            -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5' \
            -derivedDataPath build/DerivedData \
            CODE_SIGNING_ALLOWED=NO \
            | xcpretty
```

3. Si `xcpretty` no está instalado en el runner, elimina `| xcpretty` para evitar una dependencia adicional. El bloque final alternativo es:

```yaml
      - name: Build and run tests
        run: |
          xcodebuild test \
            -project CampusExplorer.xcodeproj \
            -scheme CampusExplorer \
            -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5' \
            -derivedDataPath build/DerivedData \
            CODE_SIGNING_ALLOWED=NO
```

4. Verifica la sintaxis YAML, especialmente la indentación de dos espacios.
5. Confirma que el workflow referencia:
   - `actions/checkout@v4.2.2`
   - `maxim-lobanov/setup-xcode@v1.6.0`
   - Xcode `16.4`
   - El destino `platform=iOS Simulator,name=iPhone 16,OS=18.5`
6. Añade los cambios al área de preparación:

```bash
git add .
git status
```

**Salida esperada:**

Debe aparecer el nuevo workflow:

```text
.github/workflows/ios-ci.yml
```

**Verificación:**

Comprueba el contenido:

```bash
cat .github/workflows/ios-ci.yml
```

Asegúrate de que no se usen ramas distintas de `main` para los eventos de integración continua.

---

### Paso 11. Crear el commit, publicar la rama y abrir el pull request

**Objetivo:** demostrar la ejecución correcta del pipeline de integración continua.

**Instrucciones:**

1. Revisa los archivos modificados:

```bash
git status --short
```

2. Confirma que se incluyen:
   - Targets y configuración de proyecto.
   - Esquema compartido.
   - Código de `PlacesViewModel` actualizado, si fue necesario.
   - Dobles de prueba.
   - Pruebas unitarias.
   - Pruebas de snapshot.
   - Imágenes de referencia.
   - `Package.resolved`.
   - Workflow `.github/workflows/ios-ci.yml`.

3. Realiza el commit:

```bash
git commit -m "test: add view model snapshots and iOS CI"
```

4. Publica la rama:

```bash
git push -u origin feature/lab-05-testing-ci
```

5. Abre GitHub en el repositorio.
6. Selecciona **Compare & pull request**.
7. Configura:
   - Base: `main`
   - Compare: `feature/lab-05-testing-ci`
8. Usa una descripción de pull request similar a esta:

```markdown
## Cambios

- Añade pruebas unitarias deterministas para PlacesViewModel.
- Añade mocks de repositorio y ubicación.
- Añade snapshots para carga, resultados, vacío y error.
- Añade pipeline iOS CI con Xcode 16.4 e iPhone 16 iOS 18.5.

## Validación local

- [x] xcodebuild test ejecutado correctamente.
- [x] Snapshots registrados y comparados.
- [x] Aplicación compilada en iPhone 16 iOS 18.5.
```

9. Crea el pull request.
10. Abre la pestaña **Checks** o **Actions** y espera la finalización del workflow.

**Salida esperada:**

GitHub muestra una ejecución llamada:

```text
iOS CI / Build and Test
```

con estado correcto:

```text
Success
```

**Verificación:**

El pull request debe mostrar una marca verde para el workflow. Abre los logs y confirma que contienen:

```text
Xcode 16.4
** TEST SUCCEEDED **
```

## Validación y pruebas

Ejecuta la validación final completa desde el directorio raíz del repositorio:

```bash
cd ~/Developer/CampusExplorer

xcodebuild test \
  -project CampusExplorer.xcodeproj \
  -scheme CampusExplorer \
  -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5' \
  -derivedDataPath build/DerivedData
```

Lista de comprobación final:

- [ ] El esquema `CampusExplorer` está compartido y versionado.
- [ ] Existe el target `CampusExplorerTests`.
- [ ] Existe el target `CampusExplorerSnapshotTests`.
- [ ] `PlacesViewModel` recibe `PlacesRepositoryProtocol` y `LocationProviding`.
- [ ] Las pruebas unitarias validan éxito, carga, lista vacía, error de repositorio y error de ubicación.
- [ ] Las pruebas mantienen las suscripciones en un `Set<AnyCancellable>`.
- [ ] Las pruebas usan `XCTestExpectation` y no usan `sleep`.
- [ ] Existen snapshots para cargando, resultados, vacío y error.
- [ ] `isRecording` está configurado como `false` antes del commit final.
- [ ] Los directorios `__Snapshots__` y sus PNG están incluidos en Git.
- [ ] Existe `.github/workflows/ios-ci.yml`.
- [ ] El workflow usa `actions/checkout@v4.2.2`.
- [ ] El workflow usa `maxim-lobanov/setup-xcode@v1.6.0`.
- [ ] El workflow selecciona Xcode 16.4.
- [ ] El workflow ejecuta pruebas contra iPhone 16 con iOS 18.5.
- [ ] El pull request hacia `main` muestra el pipeline en verde.

## Resolución de problemas

### Problema 1: una prueba de Combine expira y muestra “Asynchronous wait failed”

**Síntomas:**

- XCTest muestra un mensaje similar a:

```text
Asynchronous wait failed - Exceeded timeout of 1.0 seconds
```

- El estado esperado no se recibe.
- La prueba falla de forma intermitente o solo en integración continua.

**Causa:**

La suscripción a `viewModel.$state` se crea después de llamar a `loadPlaces()`, o el `AnyCancellable` se libera antes de que llegue la emisión. Con `Result.publisher`, los valores pueden emitirse inmediatamente, por lo que una suscripción tardía pierde el evento.

**Solución:**

1. Instala la suscripción antes de ejecutar `viewModel.loadPlaces()`.
2. Conserva la suscripción en `cancellables`:

```swift
viewModel.$state
    .sink { state in
        // Verificación
    }
    .store(in: &cancellables)
```

3. Inicializa `cancellables = []` en `setUp()`.
4. No uses `sleep` ni aumentes el timeout como solución principal.
5. Para verificar `.loading`, usa `PassthroughSubject` y emite la respuesta solo cuando el test lo decida.

### Problema 2: GitHub Actions no encuentra el destino iPhone 16 con iOS 18.5

**Síntomas:**

El job falla con un mensaje similar a:

```text
Unable to find a destination matching the provided destination specifier
```

o:

```text
iPhone 16 is not available
```

**Causa:**

El runner no tiene el runtime de simulador solicitado, el nombre del destino no coincide exactamente o el workflow está usando una versión de Xcode distinta de la requerida.

**Solución:**

1. Revisa el paso `Show environment` del workflow y busca los dispositivos disponibles:

```bash
xcrun simctl list devices available
```

2. Confirma que el workflow usa:

```yaml
uses: maxim-lobanov/setup-xcode@v1.6.0
with:
  xcode-version: "16.4"
```

3. Verifica que el destino coincida exactamente con la especificación del laboratorio:

```yaml
-destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5'
```

4. Si GitHub actualiza temporalmente sus imágenes y el runtime no está disponible, revisa el log, valida el runtime publicado para `macos-15` y actualiza la imagen del runner únicamente si el entorno docente lo autoriza. No sustituyas silenciosamente el simulador requerido por un destino genérico.

## Limpieza

Después de confirmar el pull request y el pipeline:

1. Conserva los archivos de pruebas, snapshots y workflow en el repositorio.
2. Elimina únicamente los artefactos locales derivados:

```bash
cd ~/Developer/CampusExplorer
rm -rf build
```

3. Si necesitas liberar espacio adicional de Xcode, puedes limpiar los datos derivados del proyecto desde Xcode:

```text
Xcode > Settings > Locations > Derived Data > Delete
```

4. No elimines:
   - `Package.resolved`.
   - `__Snapshots__`.
   - `.github/workflows/ios-ci.yml`.
   - El esquema compartido.
   - Los paquetes locales dentro de `Packages`.

5. Cuando el pull request sea aceptado, actualiza tu rama principal:

```bash
git checkout main
git pull origin main
```

## Resumen

En esta práctica has aplicado una estrategia de calidad automatizada para CampusExplorer. Las dependencias de ubicación y repositorio se desacoplaron mediante protocolos, permitiendo pruebas unitarias rápidas y deterministas para `PlacesViewModel`.

También incorporaste snapshots que protegen la representación visual de los estados principales de la lista y configuraste un pipeline de GitHub Actions que ejecuta compilación y pruebas para cada pull request y cada cambio integrado en `main`. El proyecto queda preparado para evolucionar hacia una arquitectura modular y VIPER con una base de pruebas reutilizable.

### Recursos opcionales

- [Documentación de XCTest](https://developer.apple.com/documentation/xctest)
- [Documentación de Combine](https://developer.apple.com/documentation/combine)
- [Swift Snapshot Testing](https://github.com/pointfreeco/swift-snapshot-testing)
- [GitHub Actions para iOS](https://docs.github.com/actions)
- [Documentación de xcodebuild](https://developer.apple.com/library/archive/technotes/tn2339/_index.html)
