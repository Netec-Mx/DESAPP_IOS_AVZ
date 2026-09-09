# 4 Práctica incremental

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 252 minutos |
| Complejidad | Media |
| Nivel de Bloom | Aplicar |

## Descripción general

En esta práctica se sustituirá la fuente estática de lugares creada en el laboratorio anterior por una fuente remota reactiva basada en `URLSession.DataTaskPublisher` y Combine. Se ampliará el paquete `CampusNetworking` para validar respuestas HTTP, decodificar datos `Codable` y representar errores mediante `CampusNetworkError`.

También se actualizará `PlacesViewModel` para exponer estados explícitos de pantalla y realizar búsquedas reactivas con `debounce`, `removeDuplicates`, `map`, `switchToLatest`, `receive(on:)` y `catch`. La navegación mediante coordinators, el contenedor de dependencias y la estructura modular del laboratorio anterior se conservan.

## Objetivos de aprendizaje

Al finalizar la práctica, podrá:

- [ ] Consumir una API HTTP mediante `URLSession.DataTaskPublisher`.
- [ ] Mapear errores de transporte, HTTP y decodificación a `CampusNetworkError`.
- [ ] Modelar los estados `idle`, `loading`, `loaded`, `empty` y `error` en un `ViewModel`.
- [ ] Implementar una búsqueda reactiva con espera, deduplicación y cancelación de solicitudes obsoletas.
- [ ] Validar el comportamiento mediante pruebas unitarias y la ejecución en el simulador.

## Prerrequisitos

### Conocimientos requeridos

- Haber completado el laboratorio `01-00-01` y disponer de la etiqueta Git `lab-01-complete`.
- Comprender protocolos, `Codable`, closures y la inyección de dependencias.
- Conocer los fundamentos de Combine: `Publisher`, `Subscriber`, `AnyCancellable`, `@Published` y `PassthroughSubject`.
- Conocer el patrón MVVM y la arquitectura modular configurada en el laboratorio anterior.

### Acceso requerido

- Acceso al repositorio local `~/Developer/CampusExplorer`.
- Conexión a Internet para consultar la API pública Open Brewery DB.
- Permiso para ejecutar Xcode, Simulator y comandos `xcodebuild`.
- Xcode 16.4 con runtime iOS 18.5 instalado.

## Entorno del laboratorio

| Componente | Versión o valor requerido |
|---|---|
| Sistema operativo | macOS Sequoia 15.5 |
| IDE | Xcode 16.4 |
| Lenguaje | Swift 6.1 |
| SDK de compilación | iOS 18.5 |
| Deployment target | iOS 18.0 |
| Simulador | iPhone 16, iOS 18.5 |
| Proyecto | `CampusExplorer.xcodeproj` |
| Esquema compartido | `CampusExplorer` |
| Rama principal | `main` |
| Directorio de trabajo | `~/Developer/CampusExplorer` |

Abra Terminal y prepare el repositorio:

```bash
cd ~/Developer/CampusExplorer
git switch main
git status
git tag --list "lab-01-complete"
```

La salida esperada debe incluir la etiqueta `lab-01-complete` y no debe indicar cambios no controlados que no desee conservar.

Verifique que la estructura de paquetes locales existe:

```bash
find Packages -maxdepth 2 -name Package.swift -print
```

Debe aparecer una salida equivalente a:

```text
Packages/CampusDomain/Package.swift
Packages/CampusNetworking/Package.swift
Packages/CampusDesignSystem/Package.swift
```

---

## Desarrollo paso a paso

### Paso 1. Confirmar el punto de partida y crear una copia de seguridad lógica

**Objetivo:** verificar que el proyecto del laboratorio anterior compila antes de modificar la fuente de datos.

**Instrucciones:**

1. Abra el proyecto:

   ```bash
   open CampusExplorer.xcodeproj
   ```

2. En Xcode, seleccione el esquema compartido `CampusExplorer`.

3. Seleccione el destino:

   ```text
   iPhone 16 — iOS 18.5
   ```

4. Ejecute la aplicación con <kbd>Cmd</kbd> + <kbd>R</kbd>.

5. Compruebe que la pantalla de lugares creada en el laboratorio anterior puede abrirse desde su coordinator.

6. Cree un commit de referencia si todavía no existe un estado limpio:

   ```bash
   git add .
   git commit -m "chore: baseline before reactive places networking"
   ```

   Si Git indica que no hay cambios para confirmar, continúe con el siguiente paso.

**Resultado esperado:**

La aplicación compila y muestra la pantalla de lugares con la fuente de datos estática del laboratorio anterior.

**Verificación:**

Ejecute la compilación desde Terminal:

```bash
xcodebuild \
  -project CampusExplorer.xcodeproj \
  -scheme CampusExplorer \
  -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5' \
  build
```

La última línea relevante debe contener:

```text
** BUILD SUCCEEDED **
```

---

### Paso 2. Definir el modelo de dominio de un lugar

**Objetivo:** disponer de un modelo independiente de la API remota y reutilizable por las capas de presentación, navegación y datos.

**Instrucciones:**

1. En el paquete `CampusDomain`, cree o actualice el archivo:

   ```text
   Packages/CampusDomain/Sources/CampusDomain/Place.swift
   ```

2. Defina el modelo `Place`:

   ```swift
   import Foundation

   public struct Place: Identifiable, Equatable, Sendable {
       public let id: String
       public let name: String
       public let category: String
       public let address: String
       public let city: String
       public let region: String

       public init(
           id: String,
           name: String,
           category: String,
           address: String,
           city: String,
           region: String
       ) {
           self.id = id
           self.name = name
           self.category = category
           self.address = address
           self.city = city
           self.region = region
       }

       public var subtitle: String {
           let location = [city, region]
               .filter { !$0.isEmpty }
               .joined(separator: ", ")

           return [category, location]
               .filter { !$0.isEmpty }
               .joined(separator: " · ")
       }
   }
   ```

3. Elimine o deje de usar el arreglo estático de lugares como origen de datos de la pantalla. Puede conservarlo temporalmente para pruebas visuales, pero no debe inyectarse en producción al finalizar la práctica.

4. Compruebe que el manifiesto de `CampusDomain` expone el producto de biblioteca esperado. Debe contener un producto similar a este:

   ```swift
   .library(
       name: "CampusDomain",
       targets: ["CampusDomain"]
   )
   ```

**Resultado esperado:**

El paquete `CampusDomain` contiene un modelo de presentación neutral respecto a la API y no depende de `UIKit`, `Combine` ni `CampusNetworking`.

**Verificación:**

Desde la raíz del repositorio, ejecute:

```bash
swift package --package-path Packages/CampusDomain describe
```

La salida debe incluir el producto `CampusDomain` y el target del mismo nombre.

---

### Paso 3. Crear los tipos de networking y el contrato de búsqueda

**Objetivo:** encapsular el contrato de red, los errores y el protocolo de consulta dentro de `CampusNetworking`.

**Instrucciones:**

1. Verifique que `CampusNetworking` depende de `CampusDomain`. En:

   ```text
   Packages/CampusNetworking/Package.swift
   ```

   asegúrese de que exista una dependencia local equivalente:

   ```swift
   // swift-tools-version: 6.1
   import PackageDescription

   let package = Package(
       name: "CampusNetworking",
       platforms: [
           .iOS(.v18)
       ],
       products: [
           .library(
               name: "CampusNetworking",
               targets: ["CampusNetworking"]
           )
       ],
       dependencies: [
           .package(path: "../CampusDomain")
       ],
       targets: [
           .target(
               name: "CampusNetworking",
               dependencies: [
                   .product(name: "CampusDomain", package: "CampusDomain")
               ]
           ),
           .testTarget(
               name: "CampusNetworkingTests",
               dependencies: ["CampusNetworking"]
           )
       ]
   )
   ```

2. Cree el archivo:

   ```text
   Packages/CampusNetworking/Sources/CampusNetworking/CampusNetworkError.swift
   ```

3. Añada el tipo de error:

   ```swift
   import Foundation

   public enum CampusNetworkError: Error, Equatable, LocalizedError, Sendable {
       case invalidURL
       case transport(String)
       case invalidResponse
       case httpStatus(Int)
       case decoding(String)

       public var errorDescription: String? {
           switch self {
           case .invalidURL:
               return "No se pudo construir la URL de la solicitud."
           case .transport(let description):
               return "No fue posible conectar con el servicio: \(description)"
           case .invalidResponse:
               return "El servicio devolvió una respuesta inválida."
           case .httpStatus(let code):
               return "El servicio respondió con el código HTTP \(code)."
           case .decoding:
               return "No fue posible interpretar los datos recibidos."
           }
       }

       public var userMessage: String {
           switch self {
           case .transport:
               return "Compruebe su conexión a Internet e inténtelo de nuevo."
           case .httpStatus(let code) where code == 429:
               return "Se alcanzó el límite temporal de consultas. Inténtelo más tarde."
           default:
               return errorDescription ?? "Ocurrió un error inesperado."
           }
       }
   }
   ```

4. Cree el archivo:

   ```text
   Packages/CampusNetworking/Sources/CampusNetworking/PlacesSearching.swift
   ```

5. Defina el protocolo que consumirá el `ViewModel`:

   ```swift
   import Combine
   import CampusDomain

   public protocol PlacesSearching {
       func searchPlaces(query: String) -> AnyPublisher<[Place], CampusNetworkError>
   }
   ```

**Resultado esperado:**

El `ViewModel` podrá depender de `PlacesSearching` en lugar de depender de `URLSession` o de una URL concreta.

**Verificación:**

Ejecute:

```bash
swift build --package-path Packages/CampusNetworking
```

Debe finalizar con:

```text
Build complete!
```

---

### Paso 4. Implementar el servicio HTTP reactivo

**Objetivo:** utilizar `URLSession.DataTaskPublisher` para solicitar datos remotos, validar HTTP, decodificar JSON y mapearlos a entidades de dominio.

**Instrucciones:**

1. Cree el archivo:

   ```text
   Packages/CampusNetworking/Sources/CampusNetworking/OpenBreweryPlacesService.swift
   ```

2. Implemente el servicio. La API pública Open Brewery DB se utilizará como catálogo remoto de lugares. Aunque sus resultados sean cervecerías, el dominio los tratará como lugares consultables:

   ```swift
   import Combine
   import Foundation
   import CampusDomain

   public final class OpenBreweryPlacesService: PlacesSearching {
       private let session: URLSession
       private let baseURL = URL(string: "https://api.openbrewerydb.org/v1/breweries/search")!

       public init(session: URLSession = .shared) {
           self.session = session
       }

       public func searchPlaces(query: String) -> AnyPublisher<[Place], CampusNetworkError> {
           guard var components = URLComponents(
               url: baseURL,
               resolvingAgainstBaseURL: false
           ) else {
               return Fail(error: CampusNetworkError.invalidURL)
                   .eraseToAnyPublisher()
           }

           components.queryItems = [
               URLQueryItem(name: "query", value: query),
               URLQueryItem(name: "per_page", value: "20")
           ]

           guard let url = components.url else {
               return Fail(error: CampusNetworkError.invalidURL)
                   .eraseToAnyPublisher()
           }

           var request = URLRequest(url: url)
           request.httpMethod = "GET"
           request.setValue("application/json", forHTTPHeaderField: "Accept")
           request.timeoutInterval = 15

           return session.dataTaskPublisher(for: request)
               .mapError { error in
                   CampusNetworkError.transport(error.localizedDescription)
               }
               .tryMap { data, response -> Data in
                   guard let httpResponse = response as? HTTPURLResponse else {
                       throw CampusNetworkError.invalidResponse
                   }

                   guard (200...299).contains(httpResponse.statusCode) else {
                       throw CampusNetworkError.httpStatus(httpResponse.statusCode)
                   }

                   return data
               }
               .decode(type: [BreweryDTO].self, decoder: JSONDecoder())
               .map { breweries in
                   breweries.map { brewery in
                       Place(
                           id: brewery.id,
                           name: brewery.name,
                           category: brewery.breweryType.replacingOccurrences(
                               of: "_",
                               with: " "
                           ).capitalized,
                           address: brewery.address ?? "",
                           city: brewery.city ?? "",
                           region: brewery.stateProvince ?? ""
                       )
                   }
               }
               .mapError { error in
                   if let networkError = error as? CampusNetworkError {
                       return networkError
                   }

                   if let urlError = error as? URLError {
                       return .transport(urlError.localizedDescription)
                   }

                   return .decoding(error.localizedDescription)
               }
               .eraseToAnyPublisher()
       }
   }

   private struct BreweryDTO: Decodable {
       let id: String
       let name: String
       let breweryType: String
       let address: String?
       let city: String?
       let stateProvince: String?

       enum CodingKeys: String, CodingKey {
           case id
           case name
           case breweryType = "brewery_type"
           case address = "address_1"
           case city
           case stateProvince = "state_province"
       }
   }
   ```

3. Analice la cadena de Combine implementada:

   - `dataTaskPublisher(for:)` inicia la solicitud HTTP.
   - `mapError` transforma errores de transporte en `CampusNetworkError.transport`.
   - `tryMap` valida que la respuesta sea HTTP y que el código esté en el rango `200...299`.
   - `decode` convierte el JSON a `[BreweryDTO]`.
   - `map` convierte los DTO de networking en `[Place]`.
   - El segundo `mapError` unifica errores HTTP, de decodificación y de transporte.

4. No actualice ninguna vista desde este servicio. Su única responsabilidad es publicar entidades de dominio o un error de networking.

**Resultado esperado:**

`OpenBreweryPlacesService` publica un arreglo de `Place` o finaliza con un `CampusNetworkError`.

**Verificación:**

Compile el paquete:

```bash
swift build --package-path Packages/CampusNetworking
```

En Xcode, compruebe que no aparecen errores de importación para `CampusDomain` o `Combine`.

---

### Paso 5. Modelar el estado explícito de la pantalla

**Objetivo:** representar todos los estados visibles de la pantalla de lugares mediante un único tipo reproducible.

**Instrucciones:**

1. En el target principal de la aplicación, cree o actualice:

   ```text
   CampusExplorer/Features/Places/Presentation/PlacesViewState.swift
   ```

2. Implemente el estado:

   ```swift
   import CampusDomain

   enum PlacesViewState: Equatable {
       case idle
       case loading
       case loaded([Place])
       case empty
       case error(message: String)
   }
   ```

3. Mantenga los siguientes significados funcionales:

   | Estado | Significado en interfaz |
   |---|---|
   | `.idle` | No hay texto de búsqueda o la pantalla está preparada para una nueva consulta. |
   | `.loading` | Existe una consulta válida y hay una solicitud activa. |
   | `.loaded` | La solicitud terminó y contiene uno o más lugares. |
   | `.empty` | La solicitud terminó correctamente, pero no contiene resultados. |
   | `.error` | La solicitud falló y debe mostrarse una acción de reintento. |

4. No use una lista vacía para representar todos los casos. Una lista vacía no permite distinguir entre “sin consulta”, “sin resultados” y “fallo de red”.

**Resultado esperado:**

La pantalla cuenta con una representación explícita y comprobable de cada condición de carga.

**Verificación:**

Revise que `Place` sea `Equatable`. Esto permite que `PlacesViewState` también sea `Equatable`, lo que simplifica las pruebas unitarias y de snapshots.

---

### Paso 6. Implementar el ViewModel reactivo

**Objetivo:** conectar la entrada de búsqueda con el repositorio remoto y cancelar solicitudes que dejan de ser relevantes.

**Instrucciones:**

1. Cree o actualice el archivo:

   ```text
   CampusExplorer/Features/Places/Presentation/PlacesViewModel.swift
   ```

2. Implemente el `ViewModel`:

   ```swift
   import Combine
   import Foundation
   import CampusNetworking

   final class PlacesViewModel {
       @Published var searchText: String = ""
       @Published private(set) var state: PlacesViewState = .idle

       private let repository: any PlacesSearching
       private let retrySubject = PassthroughSubject<Void, Never>()
       private var cancellables = Set<AnyCancellable>()

       init(repository: any PlacesSearching) {
           self.repository = repository
           configureSearchPipeline()
       }

       func retry() {
           retrySubject.send(())
       }

       private func configureSearchPipeline() {
           let typedQueries = $searchText
               .map { $0.trimmingCharacters(in: .whitespacesAndNewlines) }
               .debounce(for: .milliseconds(350), scheduler: RunLoop.main)
               .removeDuplicates()
               .eraseToAnyPublisher()

           let retryQueries = retrySubject
               .map { [weak self] _ in
                   self?.searchText.trimmingCharacters(in: .whitespacesAndNewlines) ?? ""
               }
               .eraseToAnyPublisher()

           Publishers.Merge(typedQueries, retryQueries)
               .map { [repository] query -> AnyPublisher<PlacesViewState, Never> in
                   guard !query.isEmpty else {
                       return Just(.idle)
                           .eraseToAnyPublisher()
                   }

                   return repository.searchPlaces(query: query)
                       .map { places -> PlacesViewState in
                           places.isEmpty ? .empty : .loaded(places)
                       }
                       .catch { error in
                           Just(.error(message: error.userMessage))
                       }
                       .prepend(.loading)
                       .eraseToAnyPublisher()
               }
               .switchToLatest()
               .receive(on: DispatchQueue.main)
               .sink { [weak self] newState in
                   self?.state = newState
               }
               .store(in: &cancellables)
       }
   }
   ```

3. Identifique el comportamiento de la tubería:

   ```text
   searchText
       → normalización
       → debounce de 350 ms
       → removeDuplicates
       → publisher de red
       → switchToLatest
       → actualización de estado en hilo principal
   ```

4. Observe que `switchToLatest()` es esencial. Cuando el usuario escribe una nueva consulta, cancela la suscripción interna anterior. De esta forma, una respuesta antigua no puede sustituir a los resultados de la consulta más reciente.

5. Observe que `retry()` vuelve a emitir el texto actual sin estar bloqueado por `removeDuplicates()`. Esto permite reintentar la misma búsqueda después de un error.

**Resultado esperado:**

El `ViewModel` mantiene sus suscripciones en `Set<AnyCancellable>`, expone una entrada observable (`searchText`) y una salida observable (`state`).

**Verificación:**

Busque en el archivo estas operaciones:

```swift
debounce(for: .milliseconds(350), scheduler: RunLoop.main)
removeDuplicates()
switchToLatest()
receive(on: DispatchQueue.main)
catch
```

Todas deben existir en la cadena de búsqueda.

---

### Paso 7. Inyectar el servicio remoto desde el contenedor de dependencias

**Objetivo:** conservar el desacoplamiento introducido por el coordinator y sustituir únicamente la implementación de datos inyectada.

**Instrucciones:**

1. Localice el contenedor de dependencias creado en el laboratorio anterior. Puede tener un nombre similar a:

   ```text
   CampusExplorer/App/AppDependencyContainer.swift
   ```

2. Añada la dependencia de `CampusNetworking`:

   ```swift
   import CampusNetworking
   ```

3. Cree una factoría para el `ViewModel`:

   ```swift
   final class AppDependencyContainer {
       private lazy var placesRepository: any PlacesSearching = {
           OpenBreweryPlacesService()
       }()

       func makePlacesViewModel() -> PlacesViewModel {
           PlacesViewModel(repository: placesRepository)
       }
   }
   ```

4. Actualice el coordinator de lugares para solicitar el `ViewModel` al contenedor:

   ```swift
   func showPlaces() {
       let viewModel = dependencyContainer.makePlacesViewModel()
       let viewController = PlacesViewController(viewModel: viewModel)
       navigationController.pushViewController(viewController, animated: true)
   }
   ```

5. No cree `URLSession`, `OpenBreweryPlacesService` ni URLs dentro de `PlacesViewController`.

**Resultado esperado:**

La navegación sigue siendo responsabilidad del coordinator y la creación de dependencias queda centralizada en el contenedor.

**Verificación:**

Compruebe que `PlacesViewController` recibe un `PlacesViewModel` ya construido:

```swift
init(viewModel: PlacesViewModel)
```

No debe recibir una URL ni crear directamente una instancia de `OpenBreweryPlacesService`.

---

### Paso 8. Conectar el estado con la interfaz UIKit

**Objetivo:** reflejar de forma consistente los estados de carga, resultados, vacío y error, incluyendo una acción de reintento.

**Instrucciones:**

1. En `PlacesViewController`, asegúrese de disponer de estos elementos visuales, creados por código o en storyboard:

   - `UISearchBar`
   - `UITableView`
   - Vista o etiqueta de estado vacío
   - `UIActivityIndicatorView`
   - Botón `Reintentar`

2. Añada una suscripción al estado del `ViewModel`:

   ```swift
   import Combine
   import UIKit
   import CampusDomain

   final class PlacesViewController: UIViewController {
       private let viewModel: PlacesViewModel
       private var cancellables = Set<AnyCancellable>()
       private var places: [Place] = []

       // Conecte estas propiedades con sus vistas reales.
       private let searchBar = UISearchBar()
       private let tableView = UITableView()
       private let activityIndicator = UIActivityIndicatorView(style: .large)
       private let messageLabel = UILabel()
       private let retryButton = UIButton(type: .system)

       init(viewModel: PlacesViewModel) {
           self.viewModel = viewModel
           super.init(nibName: nil, bundle: nil)
       }

       required init?(coder: NSCoder) {
           fatalError("init(coder:) has not been implemented")
       }

       override func viewDidLoad() {
           super.viewDidLoad()
           configureBindings()
       }

       private func configureBindings() {
           searchBar.delegate = self

           retryButton.addTarget(
               self,
               action: #selector(didTapRetry),
               for: .touchUpInside
           )

           viewModel.$state
               .receive(on: DispatchQueue.main)
               .sink { [weak self] state in
                   self?.render(state)
               }
               .store(in: &cancellables)
       }

       private func render(_ state: PlacesViewState) {
           switch state {
           case .idle:
               places = []
               activityIndicator.stopAnimating()
               messageLabel.text = "Escriba un lugar para comenzar la búsqueda."
               messageLabel.isHidden = false
               retryButton.isHidden = true
               tableView.reloadData()

           case .loading:
               activityIndicator.startAnimating()
               messageLabel.text = "Buscando lugares…"
               messageLabel.isHidden = false
               retryButton.isHidden = true

           case .loaded(let places):
               self.places = places
               activityIndicator.stopAnimating()
               messageLabel.isHidden = true
               retryButton.isHidden = true
               tableView.reloadData()

           case .empty:
               places = []
               activityIndicator.stopAnimating()
               messageLabel.text = "No se encontraron lugares para esta búsqueda."
               messageLabel.isHidden = false
               retryButton.isHidden = true
               tableView.reloadData()

           case .error(let message):
               places = []
               activityIndicator.stopAnimating()
               messageLabel.text = message
               messageLabel.isHidden = false
               retryButton.isHidden = false
               tableView.reloadData()
           }
       }

       @objc private func didTapRetry() {
           viewModel.retry()
       }
   }

   extension PlacesViewController: UISearchBarDelegate {
       func searchBar(_ searchBar: UISearchBar, textDidChange searchText: String) {
           viewModel.searchText = searchText
       }
   }
   ```

3. Conserve la implementación existente de `UITableViewDataSource`, pero asegúrese de que utilice la propiedad `places` actualizada por `render(_:)`.

4. Si el laboratorio anterior ya usa una vista de estado reutilizable del paquete `CampusDesignSystem`, utilícela en lugar de crear controles duplicados. La lógica de decisión de estado debe permanecer en el `ViewController`; los componentes visuales reutilizables deben permanecer en `CampusDesignSystem`.

**Resultado esperado:**

La interfaz presenta un indicador durante la carga, una lista al recibir resultados, un mensaje cuando no hay resultados y un botón de reintento si ocurre un error.

**Verificación:**

Ejecute la aplicación y realice esta secuencia:

1. Deje el campo vacío: debe mostrarse el estado inicial.
2. Escriba `san diego`.
3. Espere aproximadamente 350 ms.
4. Debe aparecer `Buscando lugares…`.
5. Debe mostrarse una lista de resultados remotos.
6. Borre el texto: debe volver al estado inicial.

---

### Paso 9. Añadir pruebas unitarias del ViewModel

**Objetivo:** comprobar que los estados de presentación se producen de forma determinista sin realizar solicitudes HTTP reales.

**Instrucciones:**

1. En el target de pruebas de la aplicación, cree:

   ```text
   CampusExplorerTests/PlacesViewModelTests.swift
   ```

2. Cree un repositorio simulado:

   ```swift
   import Combine
   import XCTest
   @testable import CampusExplorer
   import CampusDomain
   import CampusNetworking

   final class PlacesRepositorySpy: PlacesSearching {
       var result: Result<[Place], CampusNetworkError> = .success([])
       private(set) var receivedQueries: [String] = []

       func searchPlaces(query: String) -> AnyPublisher<[Place], CampusNetworkError> {
           receivedQueries.append(query)

           return result.publisher
               .eraseToAnyPublisher()
       }
   }
   ```

3. Añada una prueba para el estado cargado:

   ```swift
   final class PlacesViewModelTests: XCTestCase {
       private var cancellables = Set<AnyCancellable>()

       func testSearchWhenRepositoryReturnsPlacesPublishesLoadedState() {
           let repository = PlacesRepositorySpy()
           repository.result = .success([
               Place(
                   id: "1",
                   name: "Campus Coffee",
                   category: "Cafe",
                   address: "1 University Avenue",
                   city: "San Diego",
                   region: "CA"
               )
           ])

           let viewModel = PlacesViewModel(repository: repository)
           let expectation = expectation(description: "Publishes loaded state")

           viewModel.$state
               .dropFirst()
               .sink { state in
                   guard case .loaded(let places) = state else { return }
                   XCTAssertEqual(places.count, 1)
                   XCTAssertEqual(places.first?.name, "Campus Coffee")
                   expectation.fulfill()
               }
               .store(in: &cancellables)

           viewModel.searchText = "campus"

           wait(for: [expectation], timeout: 2)
           XCTAssertEqual(repository.receivedQueries, ["campus"])
       }
   }
   ```

4. Añada una prueba para el estado vacío:

   ```swift
   func testSearchWhenRepositoryReturnsNoPlacesPublishesEmptyState() {
       let repository = PlacesRepositorySpy()
       repository.result = .success([])

       let viewModel = PlacesViewModel(repository: repository)
       let expectation = expectation(description: "Publishes empty state")

       viewModel.$state
           .dropFirst()
           .sink { state in
               guard state == .empty else { return }
               expectation.fulfill()
           }
           .store(in: &cancellables)

       viewModel.searchText = "unmatched-place"

       wait(for: [expectation], timeout: 2)
   }
   ```

5. Añada una prueba para el error:

   ```swift
   func testSearchWhenRepositoryFailsPublishesErrorState() {
       let repository = PlacesRepositorySpy()
       repository.result = .failure(.transport("Offline"))

       let viewModel = PlacesViewModel(repository: repository)
       let expectation = expectation(description: "Publishes error state")

       viewModel.$state
           .dropFirst()
           .sink { state in
               guard case .error = state else { return }
               expectation.fulfill()
           }
           .store(in: &cancellables)

       viewModel.searchText = "campus"

       wait(for: [expectation], timeout: 2)
   }
   ```

6. Mantenga el tiempo de espera de las pruebas por debajo de dos segundos. El `debounce` de 350 ms forma parte intencional del comportamiento que se está validando.

**Resultado esperado:**

Las pruebas validan los estados `.loaded`, `.empty` y `.error` sin depender de Internet ni de la API pública.

**Verificación:**

Ejecute las pruebas desde Xcode con <kbd>Cmd</kbd> + <kbd>U</kbd> o mediante Terminal:

```bash
xcodebuild \
  -project CampusExplorer.xcodeproj \
  -scheme CampusExplorer \
  -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5' \
  test
```

La salida debe finalizar con:

```text
** TEST SUCCEEDED **
```

---

### Paso 10. Validar cancelación, deduplicación y estados en ejecución

**Objetivo:** comprobar visualmente que la búsqueda reactiva evita solicitudes innecesarias y que la interfaz no queda bloqueada.

**Instrucciones:**

1. Ejecute la aplicación en `iPhone 16 — iOS 18.5`.

2. Abra la pantalla de lugares.

3. Escriba rápidamente esta secuencia sin pausas prolongadas:

   ```text
   s → sa → san → san  → san d → san diego
   ```

4. Espere más de 350 ms después de escribir el último valor.

5. Abra el panel de red de Xcode o añada temporalmente un punto de interrupción en:

   ```swift
   func searchPlaces(query: String)
   ```

6. Compruebe que no se lanza una solicitud por cada carácter. Debe predominar la consulta final `san diego`.

7. Escriba el mismo valor dos veces consecutivas. Por ejemplo, establezca `campus`, borre y vuelva a establecer exactamente el mismo valor si su flujo no ha emitido otro valor intermedio. Compruebe que `removeDuplicates()` evita trabajo redundante cuando las emisiones consecutivas son iguales.

8. Simule un fallo de red desactivando temporalmente la conexión del Mac o usando un `URLProtocol` de prueba que responda con error. Confirme que aparece el mensaje y el botón **Reintentar**.

9. Restaure la conectividad y pulse **Reintentar**.

**Resultado esperado:**

- La interfaz permanece interactiva mientras se realiza la solicitud.
- El indicador de carga se muestra durante la consulta.
- Las solicitudes obsoletas se cancelan mediante `switchToLatest()`.
- Un error ofrece una recuperación explícita mediante reintento.

**Verificación:**

Use la siguiente lista de comprobación:

- [ ] La consulta vacía muestra `.idle`.
- [ ] Una consulta activa muestra `.loading`.
- [ ] Una respuesta con elementos muestra `.loaded`.
- [ ] Una respuesta correcta sin elementos muestra `.empty`.
- [ ] Un fallo de red o HTTP muestra `.error`.
- [ ] El botón de reintento vuelve a iniciar la consulta actual.
- [ ] No se actualiza UIKit fuera del hilo principal.

---

## Validación y pruebas

Ejecute la validación completa antes de confirmar los cambios:

```bash
cd ~/Developer/CampusExplorer

swift build --package-path Packages/CampusDomain
swift build --package-path Packages/CampusNetworking

xcodebuild \
  -project CampusExplorer.xcodeproj \
  -scheme CampusExplorer \
  -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5' \
  clean test
```

Revise también que el esquema esté compartido:

```bash
find CampusExplorer.xcodeproj/xcshareddata/xcschemes -name "CampusExplorer.xcscheme"
```

La salida debe incluir:

```text
CampusExplorer.xcodeproj/xcshareddata/xcschemes/CampusExplorer.xcscheme
```

Finalmente, revise los cambios y cree el commit del laboratorio:

```bash
git status
git add CampusExplorer Packages
git commit -m "feat: add reactive places search with Combine"
git tag lab-02-complete
git log --oneline -3
```

La rama activa debe seguir siendo `main`:

```bash
git branch --show-current
```

Salida esperada:

```text
main
```

## Resolución de problemas

### Problema 1: la aplicación muestra siempre el error de conexión o no devuelve resultados

**Síntoma:** al buscar `san diego`, la pantalla muestra un mensaje de error de red, un código HTTP o nunca llega a `.loaded`.

**Causa:** no hay conexión a Internet, la API pública no está disponible temporalmente, la URL se construyó manualmente sin codificación correcta o se eliminó `URLQueryItem`.

**Solución:**

1. Compruebe conectividad:

   ```bash
   curl -I "https://api.openbrewerydb.org/v1/breweries/search?query=san%20diego&per_page=20"
   ```

2. Verifique que la URL se construye con `URLComponents` y `URLQueryItem`, no por concatenación de texto.
3. Confirme que `tryMap` acepta códigos entre `200...299`.
4. Pruebe de nuevo desde una red diferente si recibe un error HTTP temporal.
5. Mantenga las pruebas unitarias usando el repositorio simulado; no deben depender de la disponibilidad de esta API.

### Problema 2: aparecen resultados antiguos después de escribir una consulta nueva

**Síntoma:** el usuario escribe una búsqueda, cambia rápidamente el texto y la tabla termina mostrando resultados correspondientes a una consulta anterior.

**Causa:** la cadena de Combine usa `flatMap` sin limitar publishers, no incluye `switchToLatest()`, o se crean suscripciones nuevas en cada pulsación sin cancelar las anteriores.

**Solución:**

1. Confirme que existe una única suscripción creada en `configureSearchPipeline()`.
2. Verifique que la transformación devuelve un publisher interno:

   ```swift
   .map { query -> AnyPublisher<PlacesViewState, Never> in
       // ...
   }
   ```

3. Verifique que inmediatamente después se usa:

   ```swift
   .switchToLatest()
   ```

4. No sustituya `switchToLatest()` por `flatMap` para este caso de uso.
5. Confirme que los `AnyCancellable` se almacenan en el conjunto `cancellables` del `ViewModel`.

## Limpieza

1. Detenga la aplicación en Xcode con <kbd>Cmd</kbd> + <kbd>.</kbd>.

2. Elimine únicamente artefactos temporales de compilación de paquetes si necesita recuperar espacio:

   ```bash
   rm -rf Packages/CampusDomain/.build
   rm -rf Packages/CampusNetworking/.build
   ```

3. No elimine los directorios obligatorios `Packages/CampusDomain`, `Packages/CampusNetworking` ni `Packages/CampusDesignSystem`.

4. Si necesita limpiar datos derivados del proyecto desde Xcode:

   ```text
   Product > Clean Build Folder
   ```

5. Verifique que el repositorio conserva los cambios confirmados:

   ```bash
   git status
   git tag --list "lab-02-complete"
   ```

## Resumen

En esta práctica se reemplazó una fuente estática de lugares por un servicio remoto basado en `URLSession.DataTaskPublisher`. El paquete `CampusNetworking` ahora valida respuestas HTTP, decodifica DTOs y transforma fallos técnicos en `CampusNetworkError`.

El `PlacesViewModel` implementa una tubería reactiva con `debounce`, `removeDuplicates`, `map`, `switchToLatest`, `catch` y `receive(on:)`. La interfaz representa explícitamente los estados de carga, éxito, vacío y error, mientras que el coordinator y el contenedor de dependencias mantienen desacopladas la navegación, la presentación y el acceso a datos.

### Recursos opcionales

- [Documentación de Combine de Apple](https://developer.apple.com/documentation/combine)
- [URLSession.DataTaskPublisher](https://developer.apple.com/documentation/foundation/urlsession/datataskpublisher)
- [Open Brewery DB API](https://www.openbrewerydb.org/documentation)
- [Swift Package Manager](https://www.swift.org/documentation/package-manager/)
