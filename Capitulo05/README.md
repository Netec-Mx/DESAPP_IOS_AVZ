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
8. Si tu proyecto viene de los laboratorios 3 y 4, no reescribas el repositorio ni la ubicación: implementa este contrato sobre el código existente con los adaptadores del **Anexo A.2 a A.4**. Las pruebas de los pasos 4 a 7 no usan los adaptadores; usan dobles de prueba.

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
4. Crea una utilidad de pruebas para generar el controlador. Como `PlacesListViewController` guarda su ViewModel como `private`, la fábrica `makeScreen` devuelve la pantalla **y** el ViewModel por separado; así las pruebas inician la carga sin acceder a una propiedad privada. Ajusta el nombre de tu controlador real si se denomina de otra forma:

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

    private func makeScreen(
        repositoryResult: Result<[Place], Error>
    ) -> (screen: PlacesListViewController, viewModel: PlacesViewModel) {
        let repository = PlacesRepositoryStub()
        repository.result = repositoryResult

        let locationProvider = LocationProviderStub()
        locationProvider.result = .success(coordinate)

        let viewModel = PlacesViewModel(
            repository: repository,
            locationProvider: locationProvider
        )

        let screen = PlacesListViewController(
            viewModel: viewModel,
            shouldLoadOnAppear: false
        )

        return (screen, viewModel)
    }

    private func snapshotAfterLoading(
        _ result: Result<[Place], Error>,
        file: StaticString = #filePath,
        testName: String = #function,
        line: UInt = #line
    ) {
        let (screen, viewModel) = makeScreen(repositoryResult: result)

        prepareForSnapshot(screen)
        viewModel.loadPlaces()

        let rendered = expectation(description: "La pantalla recibe el estado final.")
        DispatchQueue.main.async { rendered.fulfill() }
        wait(for: [rendered], timeout: 1.0)

        assertSnapshot(
            of: screen,
            as: .image(on: .iPhone13),
            file: file,
            testName: testName,
            line: line
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

5. Los dobles del paso 4 están en `CampusExplorerTests` y no son visibles desde `CampusExplorerSnapshotTests`. Selecciona `CampusExplorerTests/Support/PlacesTestDoubles.swift` y, en **Target Membership**, marca también `CampusExplorerSnapshotTests`. Compila el target de snapshots y confirma que no aparece `Cannot find 'PlacesRepositoryStub' in scope`.

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

7. Implementa los snapshots con resultados, vacío y error usando `snapshotAfterLoading`:

```swift
func testPlacesList_loaded() {
    snapshotAfterLoading(.success([
        makePlace(),
        makePlace(
            id: "cafeteria",
            name: "Cafetería Norte",
            address: "Plaza de Ingeniería, 2"
        )
    ]))
}

func testPlacesList_empty() {
    snapshotAfterLoading(.success([]))
}

func testPlacesList_error() {
    snapshotAfterLoading(.failure(PlacesTestError.networkUnavailable))
}
```

   `testName: #function` hace que cada prueba conserve su propio archivo de referencia aunque todas usen la misma función auxiliar. `DispatchQueue.main.async` espera una vuelta del hilo principal para que el estado publicado con `receive(on:)` llegue a la pantalla antes de tomar la imagen.

8. No conviertas la propiedad `private` del ViewModel en pública solo para probarla. En producción, el Coordinator crea el ViewModel y lo inyecta en el controlador; en las pruebas, `makeScreen` hace lo mismo y conserva una referencia propia.
9. Antes de grabar, revisa que la interfaz no dependa de:
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

      - name: Install xcpretty
        run: gem install xcpretty --no-document

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

3. El paso `Install xcpretty` instala el formateador de salida, que no viene garantizado en el runner. `set -o pipefail` es obligatorio: sin él, un fallo de `xcodebuild` quedaría oculto detrás de `xcpretty` y el job aparecería en verde. Si prefieres la salida completa de `xcodebuild`, elimina el paso `Install xcpretty` y la línea `| xcpretty` (junto con la barra invertida de la línea anterior).

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

---

## Anexo. Continuidad con las prácticas anteriores

Este anexo conecta la práctica con el código de los laboratorios 3 y 4 sin cambiar su contrato. `PlacesViewModel`, `PlacesRepositoryProtocol`, `LocationProviding` y las pruebas de los pasos 4 a 7 se conservan tal como están.

### A.1 Por qué cambia el contrato respecto a las prácticas anteriores

| Elemento | Laboratorios 2 a 4 | Laboratorio 5 | Motivo |
|---|---|---|---|
| Fuente de lugares | `CampusExplorerRepository` (`async throws`, laboratorio 3) | `PlacesRepositoryProtocol` (`AnyPublisher`) | El capítulo prueba flujos de Combine |
| Ubicación | `LocationManaging` con delegado (laboratorio 4) | `LocationProviding` (`AnyPublisher`) | Una sola cadena: ubicación → lugares → estado |
| Modelo | `CampusPlace` (UUID, `categoryID`, coordenadas) | `Place` (`String`, `address`, coordenadas) | Modelo de presentación de la lista |
| Método | `load()` | `loadPlaces()` | Nombre del contrato comprobable |

No se reemplaza el código anterior: se agregan **dos adaptadores** (A.2 y A.3) que implementan el contrato del laboratorio 5 sobre los componentes que ya existen.

### A.2 Adaptador de ubicación sobre `CampusLocationKit`

Crea `CampusExplorer/Places/Adapters/CampusLocationProvider.swift`:

```swift
import Combine
import CoreLocation
import CampusLocationKit

enum LocationProviderError: LocalizedError {
    case notAuthorized

    var errorDescription: String? {
        "No fue posible obtener la ubicación."
    }
}

@MainActor
final class CampusLocationProvider: @preconcurrency LocationProviding {
    private let locationManager: any LocationManaging
    private var pendingRequest: PassthroughSubject<CLLocationCoordinate2D, Error>?

    init(locationManager: any LocationManaging) {
        self.locationManager = locationManager
        locationManager.delegate = self
    }

    func requestCurrentLocation() -> AnyPublisher<CLLocationCoordinate2D, Error> {
        if let coordinate = locationManager.lastKnownCoordinate {
            return Just(coordinate.clLocationCoordinate2D)
                .setFailureType(to: Error.self)
                .eraseToAnyPublisher()
        }

        switch locationManager.authorizationStatus {
        case .denied, .restricted:
            return Fail(error: LocationProviderError.notAuthorized)
                .eraseToAnyPublisher()

        case .notDetermined:
            locationManager.requestWhenInUseAuthorization()

        case .authorizedWhenInUse:
            locationManager.startUpdatingLocation()
        }

        let request = PassthroughSubject<CLLocationCoordinate2D, Error>()
        pendingRequest = request
        return request.first().eraseToAnyPublisher()
    }

    private func finish(with result: Result<CLLocationCoordinate2D, Error>) {
        guard let request = pendingRequest else { return }
        pendingRequest = nil
        locationManager.stopUpdatingLocation()

        switch result {
        case .success(let coordinate):
            request.send(coordinate)
            request.send(completion: .finished)
        case .failure(let error):
            request.send(completion: .failure(error))
        }
    }
}

extension CampusLocationProvider: CampusLocationManagerDelegate {
    func campusLocationManager(
        _ manager: any LocationManaging,
        didChange authorizationStatus: LocationAuthorizationStatus
    ) {
        switch authorizationStatus {
        case .authorizedWhenInUse:
            manager.startUpdatingLocation()
        case .denied, .restricted:
            finish(with: .failure(LocationProviderError.notAuthorized))
        case .notDetermined:
            break
        }
    }

    func campusLocationManager(
        _ manager: any LocationManaging,
        didUpdate coordinate: LocationCoordinate
    ) {
        finish(with: .success(coordinate.clLocationCoordinate2D))
    }

    func campusLocationManager(
        _ manager: any LocationManaging,
        didFailWith error: Error
    ) {
        finish(with: .failure(error))
    }
}
```

Notas:

- `@preconcurrency LocationProviding` permite que una clase `@MainActor` cumpla un protocolo no aislado en Swift 6.1. Sin este atributo el compilador muestra un error de aislamiento.
- `LocationManaging` admite **un solo delegado**. El mapa del laboratorio 4 ya usa su propio `LocationManager`, así que este adaptador recibe **otra instancia** (ver A.4).

### A.3 Adaptador del repositorio sobre el laboratorio 3

Crea `CampusExplorer/Places/Adapters/CampusPlacesRepository.swift`:

```swift
import Combine
import CoreLocation
import CampusDomain

extension Place {
    init(campusPlace: CampusPlace) {
        self.init(
            id: campusPlace.id.uuidString,
            name: campusPlace.name,
            address: "",
            latitude: campusPlace.latitude,
            longitude: campusPlace.longitude
        )
    }
}

extension Array where Element == Place {
    func sorted(byDistanceTo coordinate: CLLocationCoordinate2D) -> [Place] {
        let origin = CLLocation(
            latitude: coordinate.latitude,
            longitude: coordinate.longitude
        )

        return sorted {
            CLLocation(latitude: $0.latitude, longitude: $0.longitude).distance(from: origin) <
            CLLocation(latitude: $1.latitude, longitude: $1.longitude).distance(from: origin)
        }
    }
}

final class CampusPlacesRepository: PlacesRepositoryProtocol {
    private let repository: any CampusExplorerRepository

    init(repository: any CampusExplorerRepository) {
        self.repository = repository
    }

    func fetchPlaces(
        near coordinate: CLLocationCoordinate2D
    ) -> AnyPublisher<[Place], Error> {
        let repository = self.repository
        let latitude = coordinate.latitude
        let longitude = coordinate.longitude

        return Deferred {
            Future<[Place], Error> { promise in
                nonisolated(unsafe) let promise = promise

                Task {
                    do {
                        let origin = CLLocationCoordinate2D(
                            latitude: latitude,
                            longitude: longitude
                        )
                        let places = try await repository.fetchPlaces()
                            .map(Place.init(campusPlace:))
                            .sorted(byDistanceTo: origin)
                        promise(.success(places))
                    } catch {
                        promise(.failure(error))
                    }
                }
            }
        }
        .eraseToAnyPublisher()
    }
}
```

Notas:

- `Deferred` retrasa la solicitud hasta que alguien se suscribe, igual que `dataTaskPublisher`. Sin `Deferred`, el `Future` empezaría a trabajar al crearse.
- `nonisolated(unsafe)` es necesario porque el `promise` de `Future` no es `Sendable`. Es seguro aquí porque se invoca una sola vez.
- `CampusPlace` no tiene dirección, por eso `address` queda vacío. Si tu API la devuelve, agrégala al modelo del laboratorio 3.

### A.4 Conectar los adaptadores en el contenedor

En `AppDependencyContainer`:

```swift
import CampusLocationKit

@MainActor
final class AppDependencyContainer {
    private lazy var campusExplorerRepository: any CampusExplorerRepository =
        CampusAPIRepository(baseURL: URL(string: "https://api.campus.edu")!)

    func makePlacesViewModel() -> PlacesViewModel {
        PlacesViewModel(
            repository: CampusPlacesRepository(
                repository: campusExplorerRepository
            ),
            locationProvider: CampusLocationProvider(
                locationManager: LocationManager()
            )
        )
    }
}
```

- Sustituye `https://api.campus.edu` por el `baseURL` que usaste en el laboratorio 3.
- El contenedor es `@MainActor` porque `LocationManager` y `CampusLocationProvider` lo son.
- Las pruebas **no** usan el contenedor ni los adaptadores; siguen usando `PlacesRepositoryStub` y `LocationProviderStub` del paso 4.

### A.5 Notas sobre snapshots

- **`.image(on: .iPhone13)`** define el **tamaño y la escala de la imagen** (390 × 844 pt, @3x), no el simulador. Las pruebas se ejecutan en iPhone 16 y es correcto. Lo importante es que la Mac y CI usen **el mismo simulador y el mismo runtime** (iPhone 16, iOS 18.5); si no, cambian las fuentes y el suavizado y la comparación falla.
- **`isRecording`** funciona en swift-snapshot-testing 1.17.0, pero puede mostrar un aviso de obsolescencia. La forma recomendada desde la versión 1.17 es:

  ```swift
  withSnapshotTesting(record: .all) {
      assertSnapshot(of: screen, as: .image(on: .iPhone13))
  }
  ```

  Para esta práctica se mantiene `isRecording = true` / `false` (paso 8). Nunca confirmes el código con `isRecording = true`.
- Graba las referencias en el mismo simulador que usará CI y súbelas en el mismo commit que las pruebas.

### A.6 Verificación antes de publicar

Confirma estos puntos compilando una vez el proyecto completo:

| Verificar | Origen | Si no coincide |
|---|---|---|
| El contenedor se llama `AppDependencyContainer` | Laboratorio 2, paso 7 | Usa el nombre de tu proyecto (`DependencyContainer` en el laboratorio 1) |
| `CampusAPIRepository(session:baseURL:)` existe | Laboratorio 3, paso 3 | Si conservaste la versión con `CampusAPIService`, crea el repositorio con ese servicio |
| `LocationManaging.lastKnownCoordinate` existe | Laboratorio 4, paso 2 | Elimina el atajo inicial de `requestCurrentLocation()`; el resto funciona igual |
| `xcodebuild test` termina en `** TEST SUCCEEDED **` con 5 pruebas unitarias y 4 snapshots | Pasos 5 a 9 | Revisa los puntos 4 y 5 del paso 7 |
| El job de GitHub Actions termina en verde en el pull request | Pasos 10 y 11 | Revisa el problema 2 |

### A.7 Resumen de los ajustes respecto a la versión anterior de la guía

| Ajuste | Dónde |
|---|---|
| Nota de continuidad con los laboratorios 3 y 4 | Paso 2, punto 8 |
| Adaptadores hacia los laboratorios 3 y 4 | A.1 a A.4 |
| La fábrica `makeScreen` devuelve el ViewModel (antes las pruebas accedían a una propiedad privada y no compilaban) | Paso 7, puntos 4, 7 y 8 |
| Dobles de prueba compartidos entre targets | Paso 7, punto 5 |
| Instalación de `xcpretty` y uso de `set -o pipefail` | Paso 10 |
| Notas sobre `.iPhone13` e `isRecording` | A.5 |