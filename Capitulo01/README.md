# 5 Práctica incremental

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 216 minutos |
| Complejidad | Media |
| Nivel de Bloom | Aplicar |
| Proyecto | `CampusExplorer.xcodeproj` |
| Rama principal | `main` |
| Destino global | iOS 18.0 |
| Lenguaje | Swift 6.1 |
| Simulador objetivo | iPhone 16, iOS 18.5 |

## Descripción general

En esta práctica crearás la línea base de **CampusExplorer**, una aplicación UIKit estructurada con MVVM, Coordinator Pattern, Swift Package Manager e inyección de dependencias por inicializador. La aplicación mostrará una lista estática de lugares de interés del campus y permitirá navegar a una pantalla de detalle.

El resultado será un repositorio modular y extensible que se reutilizará en las prácticas posteriores. Las vistas no accederán directamente a servicios ni realizarán navegación; los ViewModels dependerán de protocolos y los Coordinators centralizarán el flujo entre pantallas.

## Objetivos de aprendizaje

Al finalizar la práctica, podrás:

- [ ] Crear el repositorio y proyecto `CampusExplorer` con Swift 6.1 y deployment target iOS 18.0.
- [ ] Aplicar MVVM para separar View Controllers, ViewModels, entidades y repositorios.
- [ ] Implementar `AppCoordinator` y `PlacesCoordinator` para desacoplar la navegación de las pantallas.
- [ ] Crear e integrar los paquetes locales `CampusDomain`, `CampusNetworking` y `CampusDesignSystem`.
- [ ] Aplicar inyección de dependencias mediante protocolos e inicializadores verificables con pruebas unitarias.

## Prerrequisitos

### Conocimientos necesarios

- Sintaxis básica de Swift: estructuras, clases, protocolos, extensiones y enumeraciones.
- Uso básico de Xcode, Simulator y navegación entre archivos.
- Fundamentos de `async/await`.
- Uso elemental de Git.
- Conceptos de MVVM: Vista, ViewModel, Modelo y flujo unidireccional de estado.

### Acceso y herramientas

| Recurso | Requisito |
|---|---|
| Sistema operativo | macOS Sequoia 15.5 o compatible |
| IDE | Xcode 16.4 |
| Lenguaje | Swift 6.1 |
| SDK | iOS 18.5 |
| Simulador | iPhone 16 con runtime iOS 18.5 |
| Control de versiones | Git 2.46.0 o compatible |
| Directorio obligatorio | `~/Developer/CampusExplorer` |
| Internet | Recomendado para verificar herramientas y futuras dependencias |

## Entorno de laboratorio

> **Importante:** utiliza obligatoriamente el directorio `~/Developer/CampusExplorer`. No cambies el nombre del repositorio, del proyecto ni del esquema.

### Preparar el directorio y Git

Abre Terminal y ejecuta:

```bash
mkdir -p ~/Developer/CampusExplorer
cd ~/Developer/CampusExplorer

git init -b main
git config user.name "Tu Nombre"
git config user.email "tu.correo@ejemplo.com"

git status
```

Salida esperada aproximada:

```text
On branch main

No commits yet

nothing to commit
```

Verifica la versión de Xcode disponible:

```bash
xcodebuild -version
swift --version
git --version
```

Debes observar una versión equivalente a Xcode 16.4 y Swift 6.1.

---

## Procedimiento paso a paso

### Paso 1. Crear el proyecto UIKit base

**Objetivo:** crear `CampusExplorer.xcodeproj` con la configuración obligatoria del laboratorio.

**Instrucciones:**

1. Abre Xcode.
2. Selecciona **File > New > Project**.
3. Elige la plantilla **iOS > App**.
4. Configura los valores siguientes:

   | Campo | Valor |
   |---|---|
   | Product Name | `CampusExplorer` |
   | Team | El equipo personal disponible, si aplica |
   | Organization Identifier | `com.campusexplorer` |
   | Interface | `Storyboard` |
   | Language | `Swift` |
   | Testing System | `XCTest` |
   | Storage | `None` |

5. Guarda el proyecto directamente en:

   ```text
   ~/Developer/CampusExplorer
   ```

6. Confirma que Xcode creó:

   ```text
   ~/Developer/CampusExplorer/CampusExplorer.xcodeproj
   ```

7. En el navegador de proyecto, selecciona el proyecto **CampusExplorer** y después el target de la aplicación.
8. En la pestaña **General**, establece **Minimum Deployments** en:

   ```text
   iOS 18.0
   ```

9. En **Build Settings**, busca `Swift Language Version` y selecciona:

   ```text
   Swift 6
   ```

10. Selecciona el esquema `CampusExplorer` y el destino:

    ```text
    iPhone 16 (iOS 18.5)
    ```

11. Ejecuta la aplicación una vez con <kbd>⌘</kbd> + <kbd>R</kbd>. La plantilla inicial debe mostrarse en el simulador.

12. Comparte el esquema para que pueda utilizarse desde línea de comandos y en integración continua:
    - Selecciona **Product > Scheme > Manage Schemes...**
    - Marca la casilla **Shared** para `CampusExplorer`.
    - Cierra la ventana.

**Resultado esperado:**

Existe un proyecto funcional con el nombre exacto `CampusExplorer.xcodeproj`, un esquema compartido `CampusExplorer` y deployment target iOS 18.0.

**Verificación:**

En Terminal, desde el directorio del repositorio:

```bash
cd ~/Developer/CampusExplorer

xcodebuild -list -project CampusExplorer.xcodeproj
```

La salida debe incluir:

```text
Schemes:
    CampusExplorer
```

---

### Paso 2. Crear la estructura de capas y paquetes locales

**Objetivo:** preparar las capas `App`, `Presentation`, `Domain` y `Data`, además de los tres paquetes locales obligatorios.

**Instrucciones:**

1. En Terminal, crea las carpetas de los paquetes locales:

```bash
cd ~/Developer/CampusExplorer

mkdir -p Packages/CampusDomain/Sources/CampusDomain
mkdir -p Packages/CampusNetworking/Sources/CampusNetworking
mkdir -p Packages/CampusDesignSystem/Sources/CampusDesignSystem
```

2. En Xcode, dentro del grupo principal `CampusExplorer`, crea los siguientes grupos:

```text
App
Presentation
Presentation/Places
Presentation/PlaceDetail
Data
```

> Los grupos de Xcode organizan el navegador del proyecto. Los archivos pueden estar físicamente dentro de la carpeta del proyecto principal.

3. Elimina los archivos de plantilla que no utilizarás:
   - `ViewController.swift`
   - El contenido del storyboard principal, si existe.

4. Abre el archivo `Info.plist` del target. Elimina, si existe, la configuración que usa el storyboard como interfaz principal:
   - `Main storyboard file base name`
   - o la clave equivalente `UIMainStoryboardFile`.

5. Conserva `AppDelegate.swift` y `SceneDelegate.swift`, ya que el proyecto utilizará un ciclo de vida UIKit con `UIWindowSceneDelegate`.

6. Crea un archivo `.gitignore` en la raíz del repositorio:

```gitignore
.DS_Store
DerivedData/
build/
xcuserdata/
*.xcuserstate
.swiftpm/xcode/package.xcworkspace/xcuserdata/
```

**Resultado esperado:**

La raíz del repositorio contiene la carpeta `Packages` y los directorios de los tres paquetes locales.

**Verificación:**

```bash
find Packages -maxdepth 3 -type d | sort
```

Debes observar una estructura equivalente a:

```text
Packages
Packages/CampusDesignSystem
Packages/CampusDesignSystem/Sources
Packages/CampusDesignSystem/Sources/CampusDesignSystem
Packages/CampusDomain
Packages/CampusDomain/Sources
Packages/CampusDomain/Sources/CampusDomain
Packages/CampusNetworking
Packages/CampusNetworking/Sources
Packages/CampusNetworking/Sources/CampusNetworking
```

---

### Paso 3. Implementar el paquete CampusDomain

**Objetivo:** extraer la entidad de dominio y las abstracciones de acceso a datos que utilizará la capa de presentación.

**Instrucciones:**

1. Crea el archivo `Packages/CampusDomain/Package.swift`:

```swift
// swift-tools-version: 6.1

import PackageDescription

let package = Package(
    name: "CampusDomain",
    platforms: [
        .iOS(.v18)
    ],
    products: [
        .library(
            name: "CampusDomain",
            targets: ["CampusDomain"]
        )
    ],
    targets: [
        .target(
            name: "CampusDomain"
        )
    ]
)
```

2. Crea `Packages/CampusDomain/Sources/CampusDomain/Place.swift`:

```swift
import Foundation

public struct Place: Identifiable, Equatable, Sendable {
    public let id: String
    public let name: String
    public let category: String
    public let summary: String
    public let building: String

    public init(
        id: String,
        name: String,
        category: String,
        summary: String,
        building: String
    ) {
        self.id = id
        self.name = name
        self.category = category
        self.summary = summary
        self.building = building
    }
}
```

3. Crea `Packages/CampusDomain/Sources/CampusDomain/PlacesRepositoryProtocol.swift`:

```swift
import Foundation

public protocol PlacesRepositoryProtocol {
    func fetchPlaces() async throws -> [Place]
}
```

4. Crea `Packages/CampusDomain/Sources/CampusDomain/PlacesServiceProtocol.swift`:

```swift
import Foundation

public protocol PlacesServiceProtocol {
    func fetchPlaces() async throws -> [Place]
}
```

5. Comprueba que el paquete puede resolverse de forma independiente:

```bash
cd ~/Developer/CampusExplorer/Packages/CampusDomain
swift package describe
```

**Resultado esperado:**

El paquete `CampusDomain` expone:

- La entidad `Place`.
- El protocolo `PlacesRepositoryProtocol`.
- El protocolo `PlacesServiceProtocol`.

**Verificación:**

La salida de `swift package describe` debe identificar el producto y el target `CampusDomain` sin errores de compilación.

> **Decisión arquitectónica:** los protocolos se mantienen en la capa de dominio porque expresan necesidades de la aplicación, no detalles de UIKit, red o almacenamiento. La capa de presentación dependerá de `PlacesRepositoryProtocol`; no dependerá directamente de una URL, de `URLSession` ni de un servicio concreto.

---

### Paso 4. Implementar el paquete CampusNetworking

**Objetivo:** crear una implementación de datos estáticos que cumpla los protocolos del dominio sin acoplar la interfaz a la fuente de datos.

**Instrucciones:**

1. Crea `Packages/CampusNetworking/Package.swift`:

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
                "CampusDomain"
            ]
        )
    ]
)
```

2. Crea `Packages/CampusNetworking/Sources/CampusNetworking/StaticPlacesService.swift`:

```swift
import CampusDomain
import Foundation

public struct StaticPlacesService: PlacesServiceProtocol {
    public init() {}

    public func fetchPlaces() async throws -> [Place] {
        try await Task.sleep(for: .milliseconds(350))

        return [
            Place(
                id: "library",
                name: "Biblioteca Central",
                category: "Estudio",
                summary: "Espacio principal para consulta, lectura y trabajo académico.",
                building: "Edificio A"
            ),
            Place(
                id: "innovation-lab",
                name: "Laboratorio de Innovación",
                category: "Tecnología",
                summary: "Laboratorio para prototipado, desarrollo y proyectos colaborativos.",
                building: "Edificio C"
            ),
            Place(
                id: "sports-center",
                name: "Centro Deportivo",
                category: "Bienestar",
                summary: "Instalaciones para actividad física, eventos y programas de bienestar.",
                building: "Edificio F"
            ),
            Place(
                id: "cafeteria",
                name: "Cafetería Norte",
                category: "Servicios",
                summary: "Punto de encuentro con opciones de alimentos y bebidas para la comunidad.",
                building: "Edificio B"
            )
        ]
    }
}
```

3. Crea `Packages/CampusNetworking/Sources/CampusNetworking/PlacesRepository.swift`:

```swift
import CampusDomain
import Foundation

public struct PlacesRepository: PlacesRepositoryProtocol {
    private let service: any PlacesServiceProtocol

    public init(service: any PlacesServiceProtocol) {
        self.service = service
    }

    public func fetchPlaces() async throws -> [Place] {
        try await service.fetchPlaces()
    }
}
```

4. Valida el paquete:

```bash
cd ~/Developer/CampusExplorer/Packages/CampusNetworking
swift build
```

**Resultado esperado:**

`CampusNetworking` proporciona:

- `StaticPlacesService`, una fuente temporal de lugares.
- `PlacesRepository`, adaptador que cumple `PlacesRepositoryProtocol`.

**Verificación:**

La compilación termina con:

```text
Build complete!
```

> La aplicación podrá reemplazar posteriormente `StaticPlacesService` por una implementación basada en API, persistencia local o caché. El ViewModel no requerirá cambios porque conoce solamente `PlacesRepositoryProtocol`.

---

### Paso 5. Implementar el paquete CampusDesignSystem

**Objetivo:** encapsular estilos visuales reutilizables sin mezclar la configuración de apariencia con ViewModels o Coordinators.

**Instrucciones:**

1. Crea `Packages/CampusDesignSystem/Package.swift`:

```swift
// swift-tools-version: 6.1

import PackageDescription

let package = Package(
    name: "CampusDesignSystem",
    platforms: [
        .iOS(.v18)
    ],
    products: [
        .library(
            name: "CampusDesignSystem",
            targets: ["CampusDesignSystem"]
        )
    ],
    targets: [
        .target(
            name: "CampusDesignSystem"
        )
    ]
)
```

2. Crea `Packages/CampusDesignSystem/Sources/CampusDesignSystem/CampusTheme.swift`:

```swift
import UIKit

public enum CampusTheme {
    public static let primaryColor = UIColor(
        red: 0.08,
        green: 0.26,
        blue: 0.48,
        alpha: 1.0
    )

    public static let accentColor = UIColor(
        red: 0.00,
        green: 0.48,
        blue: 0.72,
        alpha: 1.0
    )

    public static func applyNavigationAppearance() {
        let appearance = UINavigationBarAppearance()
        appearance.configureWithOpaqueBackground()
        appearance.backgroundColor = primaryColor
        appearance.titleTextAttributes = [
            .foregroundColor: UIColor.white
        ]
        appearance.largeTitleTextAttributes = [
            .foregroundColor: UIColor.white
        ]

        UINavigationBar.appearance().standardAppearance = appearance
        UINavigationBar.appearance().scrollEdgeAppearance = appearance
        UINavigationBar.appearance().tintColor = .white
    }
}
```

3. Compila el paquete:

```bash
cd ~/Developer/CampusExplorer/Packages/CampusDesignSystem
swift build
```

**Resultado esperado:**

El paquete `CampusDesignSystem` contiene una configuración visual reutilizable para las pantallas UIKit.

**Verificación:**

La compilación del paquete finaliza correctamente.

---

### Paso 6. Agregar los paquetes locales al proyecto Xcode

**Objetivo:** integrar los tres módulos Swift Package Manager en el target principal de la aplicación.

**Instrucciones:**

1. Regresa a Xcode y abre `CampusExplorer.xcodeproj`.
2. Selecciona **File > Add Package Dependencies...**.
3. Pulsa **Add Local...**.
4. Selecciona:

   ```text
   ~/Developer/CampusExplorer/Packages/CampusDomain
   ```

5. Añade el producto `CampusDomain` al target `CampusExplorer`.
6. Repite el procedimiento para:

   ```text
   ~/Developer/CampusExplorer/Packages/CampusNetworking
   ~/Developer/CampusExplorer/Packages/CampusDesignSystem
   ```

7. Selecciona el proyecto, el target `CampusExplorer` y abre **General > Frameworks, Libraries, and Embedded Content**.
8. Verifica que aparezcan los tres productos:

   ```text
   CampusDomain
   CampusNetworking
   CampusDesignSystem
   ```

9. Ejecuta **File > Packages > Resolve Package Versions** si Xcode muestra dependencias pendientes.

**Resultado esperado:**

El target de aplicación puede importar los módulos locales.

**Verificación:**

Crea temporalmente un archivo Swift en el target principal con estas importaciones:

```swift
import CampusDomain
import CampusNetworking
import CampusDesignSystem
```

Si Xcode no muestra el error `No such module`, la integración es correcta. Elimina el archivo temporal si no será utilizado.

---

### Paso 7. Configurar el punto de composición y la inyección de dependencias

**Objetivo:** crear un `DependencyContainer` que construya las implementaciones concretas y las entregue como abstracciones.

**Instrucciones:**

1. En el grupo `App`, crea `DependencyContainer.swift`.
2. Asegúrate de que el archivo pertenezca únicamente al target `CampusExplorer`.
3. Añade el siguiente código:

```swift
import CampusDomain
import CampusNetworking
import Foundation

final class DependencyContainer {
    func makePlacesRepository() -> any PlacesRepositoryProtocol {
        let service = StaticPlacesService()
        return PlacesRepository(service: service)
    }
}
```

4. Observa la dirección de las dependencias:

   ```text
   Presentation → CampusDomain ← CampusNetworking
                    ↑
                   App
   ```

5. No crees `StaticPlacesService` dentro de un ViewModel ni de un View Controller. Su construcción debe permanecer centralizada en `DependencyContainer`.

**Resultado esperado:**

Existe un punto de composición único que conoce implementaciones concretas de red y repositorio.

**Verificación:**

Comprueba que `DependencyContainer` es el único archivo de la app que contiene:

```swift
StaticPlacesService()
```

Puedes comprobarlo desde Terminal:

```bash
cd ~/Developer/CampusExplorer
grep -R "StaticPlacesService()" --exclude-dir=Packages .
```

La coincidencia relevante debe estar en `DependencyContainer.swift`.

---

### Paso 8. Implementar el flujo Coordinator

**Objetivo:** centralizar la navegación y evitar que los ViewModels conozcan `UINavigationController` o View Controllers concretos.

**Instrucciones:**

1. Crea `App/AppCoordinator.swift`:

```swift
import CampusDesignSystem
import UIKit

final class AppCoordinator {
    private let window: UIWindow
    private let dependencyContainer: DependencyContainer
    private let navigationController: UINavigationController
    private var placesCoordinator: PlacesCoordinator?

    init(
        window: UIWindow,
        dependencyContainer: DependencyContainer
    ) {
        self.window = window
        self.dependencyContainer = dependencyContainer
        self.navigationController = UINavigationController()
    }

    func start() {
        CampusTheme.applyNavigationAppearance()

        let coordinator = PlacesCoordinator(
            navigationController: navigationController,
            dependencyContainer: dependencyContainer
        )

        placesCoordinator = coordinator
        window.rootViewController = navigationController
        window.makeKeyAndVisible()

        coordinator.start()
    }
}
```

2. Crea `App/PlacesCoordinator.swift`:

```swift
import CampusDomain
import UIKit

final class PlacesCoordinator {
    private let navigationController: UINavigationController
    private let dependencyContainer: DependencyContainer

    init(
        navigationController: UINavigationController,
        dependencyContainer: DependencyContainer
    ) {
        self.navigationController = navigationController
        self.dependencyContainer = dependencyContainer
    }

    func start() {
        let viewModel = PlacesListViewModel(
            repository: dependencyContainer.makePlacesRepository()
        )

        viewModel.onPlaceSelected = { [weak self] place in
            self?.showDetail(for: place)
        }

        let viewController = PlacesListViewController(viewModel: viewModel)
        navigationController.setViewControllers(
            [viewController],
            animated: false
        )
    }

    private func showDetail(for place: Place) {
        let viewModel = PlaceDetailViewModel(place: place)
        let viewController = PlaceDetailViewController(viewModel: viewModel)
        navigationController.pushViewController(viewController, animated: true)
    }
}
```

3. Reemplaza el contenido de `SceneDelegate.swift` por lo siguiente:

```swift
import UIKit

class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?
    private var appCoordinator: AppCoordinator?

    func scene(
        _ scene: UIScene,
        willConnectTo session: UISceneSession,
        options connectionOptions: UIScene.ConnectionOptions
    ) {
        guard let windowScene = scene as? UIWindowScene else {
            return
        }

        let window = UIWindow(windowScene: windowScene)
        let dependencyContainer = DependencyContainer()

        let coordinator = AppCoordinator(
            window: window,
            dependencyContainer: dependencyContainer
        )

        self.window = window
        self.appCoordinator = coordinator

        coordinator.start()
    }
}
```

**Resultado esperado:**

El ciclo de vida de la aplicación inicia `AppCoordinator`, que crea `PlacesCoordinator`. Este Coordinator construye la pantalla de lista y controla la navegación al detalle.

**Verificación:**

Revisa que se cumplan estas restricciones:

- `PlacesListViewModel` no importa `UIKit`.
- `PlacesListViewModel` no conoce `UINavigationController`.
- `PlacesListViewController` no instancia `PlaceDetailViewController`.
- Solo el Coordinator hace `pushViewController`.

---

### Paso 9. Implementar el ViewModel y la pantalla de lista

**Objetivo:** aplicar MVVM para representar los estados de carga, contenido vacío, error y lista cargada.

**Instrucciones:**

1. Crea `Presentation/Places/PlacesListViewModel.swift`:

```swift
import CampusDomain
import Foundation

enum PlacesListViewState: Equatable {
    case idle
    case loading
    case loaded([Place])
    case empty
    case error(String)
}

@MainActor
final class PlacesListViewModel {
    private let repository: any PlacesRepositoryProtocol

    private(set) var state: PlacesListViewState = .idle {
        didSet {
            onStateChange?(state)
        }
    }

    var onStateChange: ((PlacesListViewState) -> Void)?
    var onPlaceSelected: ((Place) -> Void)?

    init(repository: any PlacesRepositoryProtocol) {
        self.repository = repository
    }

    func loadPlaces() async {
        state = .loading

        do {
            let places = try await repository.fetchPlaces()

            if places.isEmpty {
                state = .empty
            } else {
                state = .loaded(places)
            }
        } catch {
            state = .error(
                "No fue posible cargar los lugares del campus."
            )
        }
    }

    func selectPlace(at index: Int) {
        guard case .loaded(let places) = state,
              places.indices.contains(index) else {
            return
        }

        onPlaceSelected?(places[index])
    }
}
```

2. Crea `Presentation/Places/PlacesListViewController.swift`:

```swift
import CampusDesignSystem
import UIKit

final class PlacesListViewController: UIViewController {
    private let viewModel: PlacesListViewModel
    private let tableView = UITableView(
        frame: .zero,
        style: .insetGrouped
    )
    private let messageLabel = UILabel()
    private var placesCount = 0

    init(viewModel: PlacesListViewModel) {
        self.viewModel = viewModel
        super.init(nibName: nil, bundle: nil)
    }

    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }

    override func viewDidLoad() {
        super.viewDidLoad()

        title = "Campus Explorer"
        view.backgroundColor = .systemBackground

        configureTableView()
        configureMessageLabel()
        bindViewModel()

        Task {
            await viewModel.loadPlaces()
        }
    }

    private func configureTableView() {
        tableView.translatesAutoresizingMaskIntoConstraints = false
        tableView.dataSource = self
        tableView.delegate = self
        tableView.register(
            UITableViewCell.self,
            forCellReuseIdentifier: "PlaceCell"
        )

        view.addSubview(tableView)

        NSLayoutConstraint.activate([
            tableView.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor),
            tableView.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            tableView.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            tableView.bottomAnchor.constraint(equalTo: view.bottomAnchor)
        ])
    }

    private func configureMessageLabel() {
        messageLabel.translatesAutoresizingMaskIntoConstraints = false
        messageLabel.numberOfLines = 0
        messageLabel.textAlignment = .center
        messageLabel.textColor = .secondaryLabel
        messageLabel.font = .preferredFont(forTextStyle: .body)
        messageLabel.isHidden = true

        view.addSubview(messageLabel)

        NSLayoutConstraint.activate([
            messageLabel.centerXAnchor.constraint(equalTo: view.centerXAnchor),
            messageLabel.centerYAnchor.constraint(equalTo: view.centerYAnchor),
            messageLabel.leadingAnchor.constraint(
                greaterThanOrEqualTo: view.layoutMarginsGuide.leadingAnchor
            ),
            messageLabel.trailingAnchor.constraint(
                lessThanOrEqualTo: view.layoutMarginsGuide.trailingAnchor
            )
        ])
    }

    private func bindViewModel() {
        viewModel.onStateChange = { [weak self] state in
            self?.render(state)
        }
    }

    private func render(_ state: PlacesListViewState) {
        switch state {
        case .idle, .loading:
            title = "Cargando lugares..."
            tableView.isHidden = true
            messageLabel.isHidden = false
            messageLabel.text = "Cargando lugares del campus..."

        case .loaded(let places):
            title = "Campus Explorer"
            placesCount = places.count
            tableView.isHidden = false
            messageLabel.isHidden = true
            tableView.reloadData()

        case .empty:
            title = "Campus Explorer"
            tableView.isHidden = true
            messageLabel.isHidden = false
            messageLabel.text = "No hay lugares disponibles."

        case .error(let message):
            title = "Campus Explorer"
            tableView.isHidden = true
            messageLabel.isHidden = false
            messageLabel.text = message
        }
    }
}

extension PlacesListViewController: UITableViewDataSource {
    func tableView(
        _ tableView: UITableView,
        numberOfRowsInSection section: Int
    ) -> Int {
        placesCount
    }

    func tableView(
        _ tableView: UITableView,
        cellForRowAt indexPath: IndexPath
    ) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(
            withIdentifier: "PlaceCell",
            for: indexPath
        )

        guard case .loaded(let places) = viewModel.state else {
            return cell
        }

        let place = places[indexPath.row]

        var configuration = UIListContentConfiguration.subtitleCell()
        configuration.text = place.name
        configuration.secondaryText = "\(place.category) · \(place.building)"
        configuration.textProperties.color = CampusTheme.primaryColor

        cell.contentConfiguration = configuration
        cell.accessoryType = .disclosureIndicator

        return cell
    }
}

extension PlacesListViewController: UITableViewDelegate {
    func tableView(
        _ tableView: UITableView,
        didSelectRowAt indexPath: IndexPath
    ) {
        tableView.deselectRow(at: indexPath, animated: true)
        viewModel.selectPlace(at: indexPath.row)
    }
}
```

**Resultado esperado:**

La pantalla de lista muestra inicialmente un mensaje de carga y, después, cuatro lugares del campus.

**Verificación:**

Ejecuta la aplicación. Debes observar filas similares a:

```text
Biblioteca Central
Estudio · Edificio A

Laboratorio de Innovación
Tecnología · Edificio C
```

---

### Paso 10. Implementar el ViewModel y la pantalla de detalle

**Objetivo:** mostrar los datos de un lugar seleccionado sin permitir que la pantalla de lista controle directamente la navegación.

**Instrucciones:**

1. Crea `Presentation/PlaceDetail/PlaceDetailViewModel.swift`:

```swift
import CampusDomain
import Foundation

struct PlaceDetailViewState: Equatable {
    let name: String
    let category: String
    let building: String
    let summary: String
}

final class PlaceDetailViewModel {
    let state: PlaceDetailViewState

    init(place: Place) {
        self.state = PlaceDetailViewState(
            name: place.name,
            category: place.category,
            building: place.building,
            summary: place.summary
        )
    }
}
```

2. Crea `Presentation/PlaceDetail/PlaceDetailViewController.swift`:

```swift
import CampusDesignSystem
import UIKit

final class PlaceDetailViewController: UIViewController {
    private let viewModel: PlaceDetailViewModel

    init(viewModel: PlaceDetailViewModel) {
        self.viewModel = viewModel
        super.init(nibName: nil, bundle: nil)
    }

    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }

    override func viewDidLoad() {
        super.viewDidLoad()

        title = viewModel.state.name
        view.backgroundColor = .systemBackground

        configureContent()
    }

    private func configureContent() {
        let categoryLabel = makeLabel(
            text: viewModel.state.category.uppercased(),
            style: .caption1,
            color: CampusTheme.accentColor
        )

        let buildingLabel = makeLabel(
            text: viewModel.state.building,
            style: .title3,
            color: .secondaryLabel
        )

        let summaryLabel = makeLabel(
            text: viewModel.state.summary,
            style: .body,
            color: .label
        )

        let stackView = UIStackView(
            arrangedSubviews: [
                categoryLabel,
                buildingLabel,
                summaryLabel
            ]
        )

        stackView.translatesAutoresizingMaskIntoConstraints = false
        stackView.axis = .vertical
        stackView.spacing = 16
        stackView.alignment = .leading

        view.addSubview(stackView)

        NSLayoutConstraint.activate([
            stackView.topAnchor.constraint(
                equalTo: view.safeAreaLayoutGuide.topAnchor,
                constant: 24
            ),
            stackView.leadingAnchor.constraint(
                equalTo: view.layoutMarginsGuide.leadingAnchor
            ),
            stackView.trailingAnchor.constraint(
                equalTo: view.layoutMarginsGuide.trailingAnchor
            )
        ])
    }

    private func makeLabel(
        text: String,
        style: UIFont.TextStyle,
        color: UIColor
    ) -> UILabel {
        let label = UILabel()
        label.text = text
        label.font = .preferredFont(forTextStyle: style)
        label.textColor = color
        label.numberOfLines = 0
        return label
    }
}
```

3. Ejecuta la aplicación.
4. Selecciona **Biblioteca Central**.
5. Usa el botón de regreso de la barra de navegación.

**Resultado esperado:**

La selección de una fila abre la pantalla de detalle con categoría, edificio y descripción. Al volver, se conserva la lista.

**Verificación:**

Confirma el flujo:

```text
PlacesListViewController
        │
        │ acción del usuario
        ▼
PlacesListViewModel.selectPlace(at:)
        │
        │ closure onPlaceSelected
        ▼
PlacesCoordinator.showDetail(for:)
        │
        ▼
PlaceDetailViewController
```

El ViewModel notifica la intención de selección, pero no realiza `pushViewController` ni importa UIKit.

---

### Paso 11. Crear una prueba unitaria del ViewModel

**Objetivo:** comprobar que la inyección de dependencias permite probar la lógica de presentación sin ejecutar la interfaz ni utilizar datos reales.

**Instrucciones:**

1. En el target de pruebas `CampusExplorerTests`, crea `PlacesListViewModelTests.swift`.
2. Si el archivo de pruebas no puede importar el módulo principal, verifica que el target de pruebas tenga `CampusExplorer` como host o dependencia de prueba.
3. Añade el siguiente código:

```swift
@testable import CampusExplorer
import CampusDomain
import XCTest

struct SuccessfulPlacesRepository: PlacesRepositoryProtocol {
    func fetchPlaces() async throws -> [Place] {
        [
            Place(
                id: "test-library",
                name: "Biblioteca de Prueba",
                category: "Estudio",
                summary: "Lugar utilizado por una prueba unitaria.",
                building: "Edificio Test"
            )
        ]
    }
}

@MainActor
final class PlacesListViewModelTests: XCTestCase {
    func testLoadPlacesPublishesLoadedState() async {
        let viewModel = PlacesListViewModel(
            repository: SuccessfulPlacesRepository()
        )

        await viewModel.loadPlaces()

        XCTAssertEqual(
            viewModel.state,
            .loaded([
                Place(
                    id: "test-library",
                    name: "Biblioteca de Prueba",
                    category: "Estudio",
                    summary: "Lugar utilizado por una prueba unitaria.",
                    building: "Edificio Test"
                )
            ])
        )
    }
}
```

4. Ejecuta las pruebas en Xcode con <kbd>⌘</kbd> + <kbd>U</kbd>.

**Resultado esperado:**

La prueba termina correctamente sin cargar una pantalla, sin iniciar una petición de red y sin usar `StaticPlacesService`.

**Verificación:**

En el navegador de pruebas de Xcode debe aparecer:

```text
PlacesListViewModelTests
    testLoadPlacesPublishesLoadedState ✓
```

> Esta prueba demuestra el beneficio de depender de `PlacesRepositoryProtocol`. La implementación `SuccessfulPlacesRepository` controla el resultado y permite verificar el estado publicado por el ViewModel de forma determinista.

---

### Paso 12. Compilar, ejecutar y versionar la línea base

**Objetivo:** validar el proyecto completo con `xcodebuild` y registrar la línea base arquitectónica en Git.

**Instrucciones:**

1. Desde la raíz del repositorio, lista los simuladores disponibles:

```bash
xcrun simctl list devices available | grep "iPhone 16"
```

2. Ejecuta las pruebas desde Terminal usando el destino obligatorio:

```bash
cd ~/Developer/CampusExplorer

xcodebuild test \
  -project CampusExplorer.xcodeproj \
  -scheme CampusExplorer \
  -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5'
```

3. Revisa los cambios pendientes:

```bash
git status
```

4. Añade los archivos relevantes:

```bash
git add .
```

5. Crea el primer commit de la práctica:

```bash
git commit -m "feat: crear linea base MVVM modular de CampusExplorer"
```

6. Verifica el historial y la rama:

```bash
git branch --show-current
git log --oneline -1
```

**Resultado esperado:**

La aplicación compila, las pruebas pasan y el repositorio contiene un commit en la rama `main`.

**Verificación:**

La salida de `xcodebuild test` debe incluir:

```text
** TEST SUCCEEDED **
```

Y los comandos Git deben mostrar:

```text
main
```

---

## Validación y pruebas

Realiza esta lista final antes de considerar completada la práctica:

| Criterio | Comprobación |
|---|---|
| Proyecto correcto | Existe `CampusExplorer.xcodeproj` en `~/Developer/CampusExplorer`. |
| Esquema compartido | Existe el esquema `CampusExplorer` y está marcado como compartido. |
| Configuración | Deployment target iOS 18.0 y Swift 6.1/Swift 6 configurado. |
| Paquetes locales | Existen `CampusDomain`, `CampusNetworking` y `CampusDesignSystem` dentro de `Packages`. |
| Dominio | `Place`, `PlacesRepositoryProtocol` y `PlacesServiceProtocol` pertenecen a `CampusDomain`. |
| Datos | `StaticPlacesService` y `PlacesRepository` pertenecen a `CampusNetworking`. |
| Diseño | `CampusTheme` pertenece a `CampusDesignSystem`. |
| Inyección de dependencias | `PlacesListViewModel` recibe `any PlacesRepositoryProtocol` por inicializador. |
| MVVM | El ViewModel publica `PlacesListViewState` y no importa UIKit. |
| Navegación | `PlacesCoordinator` realiza la transición al detalle. |
| Pruebas | `PlacesListViewModelTests` finaliza correctamente. |
| Git | La rama activa es `main` y existe un commit inicial. |

Ejecuta como validación final:

```bash
cd ~/Developer/CampusExplorer

xcodebuild test \
  -project CampusExplorer.xcodeproj \
  -scheme CampusExplorer \
  -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5'
```

También valida que el árbol general tenga esta forma:

```text
CampusExplorer/
├── CampusExplorer.xcodeproj
├── CampusExplorer/
│   ├── App/
│   │   ├── AppCoordinator.swift
│   │   ├── DependencyContainer.swift
│   │   └── PlacesCoordinator.swift
│   ├── Presentation/
│   │   ├── Places/
│   │   │   ├── PlacesListViewController.swift
│   │   │   └── PlacesListViewModel.swift
│   │   └── PlaceDetail/
│   │       ├── PlaceDetailViewController.swift
│   │       └── PlaceDetailViewModel.swift
│   ├── AppDelegate.swift
│   └── SceneDelegate.swift
├── CampusExplorerTests/
│   └── PlacesListViewModelTests.swift
├── Packages/
│   ├── CampusDesignSystem/
│   ├── CampusDomain/
│   └── CampusNetworking/
└── .gitignore
```

## Resolución de problemas

### Problema 1: Xcode muestra “No such module 'CampusDomain'” o no resuelve un paquete local

**Síntomas:**

- Al escribir `import CampusDomain`, Xcode muestra un error.
- El paquete aparece en rojo en el navegador del proyecto.
- La compilación falla después de crear los manifiestos `Package.swift`.

**Causa:**

El paquete local no fue agregado al target de la aplicación, la ruta seleccionada no corresponde a la carpeta que contiene `Package.swift`, o Xcode conserva una resolución de paquetes desactualizada.

**Solución:**

1. Confirma que exista el manifiesto:

   ```bash
   ls ~/Developer/CampusExplorer/Packages/CampusDomain/Package.swift
   ```

2. En Xcode, elimina la referencia rota del paquete desde **Package Dependencies** si es necesario.
3. Vuelve a usar **File > Add Package Dependencies... > Add Local...**.
4. Selecciona exactamente la carpeta del paquete, por ejemplo:

   ```text
   ~/Developer/CampusExplorer/Packages/CampusDomain
   ```

5. Asegúrate de añadir el producto al target `CampusExplorer`.
6. Ejecuta **File > Packages > Reset Package Caches** y después **Resolve Package Versions**.
7. Limpia la compilación con <kbd>⇧</kbd> + <kbd>⌘</kbd> + <kbd>K</kbd> y vuelve a compilar.

### Problema 2: La aplicación inicia con pantalla negra o no aparece la lista

**Síntomas:**

- El simulador muestra una ventana negra.
- Se sigue mostrando el controlador inicial del storyboard.
- La lista de lugares no aparece aunque el proyecto compile.

**Causa:**

El proyecto aún tiene un storyboard configurado como interfaz principal, o `SceneDelegate` no conserva/inicia correctamente el `AppCoordinator`.

**Solución:**

1. Abre el `Info.plist` del target y elimina la clave de storyboard principal si existe:
   - `Main storyboard file base name`
   - `UIMainStoryboardFile`
2. Verifica que `SceneDelegate` incluya estas asignaciones:

   ```swift
   self.window = window
   self.appCoordinator = coordinator
   coordinator.start()
   ```

3. Confirma que `AppCoordinator.start()` asigne el controlador raíz:

   ```swift
   window.rootViewController = navigationController
   window.makeKeyAndVisible()
   ```

4. Limpia el proyecto, elimina la aplicación del simulador y vuelve a ejecutar.
5. Si el problema continúa, revisa que `SceneDelegate` esté asociado al ciclo de vida de escenas en la configuración del target.

## Limpieza

Al finalizar, conserva el repositorio porque será la base de las prácticas siguientes. No elimines el proyecto ni los paquetes locales.

Para eliminar artefactos temporales de compilación generados por Swift Package Manager, si fuera necesario:

```bash
cd ~/Developer/CampusExplorer

rm -rf Packages/CampusDomain/.build
rm -rf Packages/CampusNetworking/.build
rm -rf Packages/CampusDesignSystem/.build
```

No elimines:

```text
CampusExplorer.xcodeproj
Packages/
.git/
```

Verifica que el repositorio esté limpio después del commit:

```bash
git status
```

Salida esperada:

```text
On branch main
nothing to commit, working tree clean
```

## Resumen

En esta práctica creaste la base modular de **CampusExplorer**:

- Implementaste MVVM con estados explícitos mediante `PlacesListViewState`.
- Separaste la interfaz UIKit de la lógica de presentación en ViewModels.
- Definiste protocolos de dominio para desacoplar presentación, repositorios y servicios.
- Construiste paquetes locales con Swift Package Manager.
- Centralizaste la construcción de dependencias en `DependencyContainer`.
- Delegaste la navegación en `AppCoordinator` y `PlacesCoordinator`.
- Añadiste una prueba unitaria que valida la lógica del ViewModel de forma aislada.
- Registraste la línea base en el repositorio Git de la rama `main`.

Esta estructura será el punto de partida para incorporar geolocalización, mapas, optimización de rendimiento, pruebas de snapshots, integración continua y módulos VIPER en las siguientes prácticas.
