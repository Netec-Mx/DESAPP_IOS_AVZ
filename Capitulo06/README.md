# 5 Práctica Extra

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 180 minutos |
| Complejidad | Alta |
| Nivel de Bloom | Aplicar |

## Descripción general

En esta práctica refactorizarás una funcionalidad de lista y detalle de tareas hacia una arquitectura VIPER con contratos explícitos, inyección de dependencias y navegación centralizada mediante coordinators. Crearás una lista de tareas con estados reutilizables de carga, vacío y error, además de un detalle construido a partir de una entidad seleccionada.

El resultado será una aplicación UIKit navegable en la que la vista no conoce el repositorio ni crea pantallas de detalle directamente. La composición de dependencias quedará concentrada en `AppCoordinator`, `TaskFlowCoordinator` y los constructores de módulo.

## Objetivos de aprendizaje

- [ ] Refactorizar una lista y un detalle hacia módulos VIPER con View, Interactor, Presenter, Entity, Router y Builder.
- [ ] Crear `TaskListStateView`, un componente reutilizable para estados de carga, vacío y error.
- [ ] Aplicar inyección de dependencias mediante `TaskRepositoryProtocol` y un contenedor de composición.
- [ ] Implementar navegación desacoplada con `AppCoordinator` y `TaskFlowCoordinator`.
- [ ] Compilar, ejecutar y validar manualmente el flujo lista → detalle → lista.

## Requisitos previos

### Conocimientos

- Fundamentos de Swift: `struct`, `enum`, clases, protocolos, optionals, closures y gestión de memoria.
- UIKit: `UIViewController`, `UITableView`, `UINavigationController` y Auto Layout.
- Principios de VIPER, Repository Pattern, Dependency Injection, Router Pattern y Coordinator Pattern.
- Uso básico de Git, Xcode y Simulator.
- Conocimiento de tareas `async/await` y del uso de `@MainActor` para actualizaciones de interfaz.

### Acceso y recursos

- Mac con macOS Sequoia compatible.
- Xcode 16.4 o una versión compatible con Swift 6.1 e iOS SDK 18.5.
- Runtime de iOS Simulator 18.5 instalado.
- Simulador **iPhone 16** disponible.
- Acceso de escritura al repositorio local `~/Developer/CampusExplorer`.
- Git configurado con nombre y correo de autor.

## Entorno del laboratorio

### Configuración de software

| Componente | Versión objetivo |
|---|---|
| macOS | Sequoia 15.5 |
| Xcode | 16.4 |
| Swift | 6.1 |
| Deployment Target | iOS 18.0 |
| SDK de compilación | iOS 18.5 |
| Simulador | iPhone 16, iOS 18.5 |
| Git | 2.46.0 o compatible |
| Repositorio | `CampusExplorer` |
| Rama principal | `main` |
| Esquema compartido | `CampusExplorer` |

### Preparación inicial

1. Abre Terminal y accede al directorio obligatorio:

   ```bash
   cd ~/Developer/CampusExplorer
   ```

2. Comprueba el estado del repositorio y la rama actual:

   ```bash
   git status
   git branch --show-current
   ```

3. Si partes de un estado funcional previo, asegúrate de que no existen cambios sin confirmar. Crea la rama solicitada:

   ```bash
   git switch main
   git pull --ff-only
   git switch -c feature/module6-viper-coordinator
   ```

4. Verifica que existen el proyecto y los paquetes locales obligatorios:

   ```bash
   ls CampusExplorer.xcodeproj
   ls Packages
   ```

   La estructura mínima de paquetes debe contener:

   ```text
   Packages/
   ├── CampusDomain/
   ├── CampusNetworking/
   └── CampusDesignSystem/
   ```

5. Si no dispones del proyecto funcional anterior, crea un proyecto UIKit llamado **CampusExplorer** desde Xcode:

   - Selecciona **File > New > Project > iOS > App**.
   - Product Name: `CampusExplorer`.
   - Interface: `Storyboard` desactivado.
   - Life Cycle: `UIKit App Delegate`.
   - Language: `Swift`.
   - Guarda el proyecto dentro de `~/Developer/CampusExplorer`.
   - Configura el deployment target global en **iOS 18.0**.
   - Comparte el esquema `CampusExplorer` mediante **Product > Scheme > Manage Schemes** y marca **Shared**.

> **Nota:** Esta práctica añade código de arquitectura a la aplicación principal. Los paquetes locales obligatorios se conservan como parte del proyecto modular, aunque el repositorio determinista de esta práctica se implementará en `Core` para mantener visible el flujo completo de VIPER.

## Desarrollo paso a paso

### Paso 1. Crear la estructura de módulos y confirmar la rama

**Objetivo:** Organizar el código por responsabilidades y preparar una rama aislada para la refactorización.

**Instrucciones:**

1. En Xcode, dentro del grupo principal del proyecto, crea los siguientes grupos:

   ```text
   App/
   Core/
   Features/
   ├── TaskList/
   └── TaskDetail/
   ```

2. Dentro de `Core`, crea los grupos:

   ```text
   Core/
   ├── Entities/
   ├── Repositories/
   └── Composition/
   ```

3. Dentro de `Features/TaskList`, crea los grupos:

   ```text
   TaskList/
   ├── Contracts/
   ├── View/
   ├── Presenter/
   ├── Interactor/
   ├── Router/
   ├── Builder/
   └── Components/
   ```

4. Dentro de `Features/TaskDetail`, crea una estructura equivalente:

   ```text
   TaskDetail/
   ├── Contracts/
   ├── View/
   ├── Presenter/
   ├── Interactor/
   ├── Router/
   └── Builder/
   ```

5. En Terminal, registra el punto de partida de la práctica:

   ```bash
   git status
   git branch --show-current
   ```

**Resultado esperado:**

La rama actual es `feature/module6-viper-coordinator` y el proyecto contiene grupos para `App`, `Core`, `TaskList` y `TaskDetail`.

**Verificación:**

Ejecuta:

```bash
git branch --show-current
```

La salida debe ser:

```text
feature/module6-viper-coordinator
```

---

### Paso 2. Implementar las entidades y el repositorio determinista

**Objetivo:** Definir el dominio de tareas y desacoplar el origen de datos mediante un protocolo.

**Instrucciones:**

1. Crea el archivo `Core/Entities/Task.swift`:

   ```swift
   import Foundation

   enum TaskStatus: String, CaseIterable, Equatable, Sendable {
       case pending
       case inProgress
       case completed

       var displayName: String {
           switch self {
           case .pending:
               return "Pendiente"
           case .inProgress:
               return "En progreso"
           case .completed:
               return "Completada"
           }
       }
   }

   struct Task: Identifiable, Equatable, Sendable {
       let id: UUID
       let title: String
       let detail: String
       let status: TaskStatus

       init(
           id: UUID = UUID(),
           title: String,
           detail: String,
           status: TaskStatus
       ) {
           self.id = id
           self.title = title
           self.detail = detail
           self.status = status
       }
   }
   ```

2. Crea `Core/Repositories/TaskRepositoryProtocol.swift`:

   ```swift
   import Foundation

   protocol TaskRepositoryProtocol: Sendable {
       func fetchTasks() async throws -> [Task]
   }
   ```

3. Crea `Core/Repositories/InMemoryTaskRepository.swift`:

   ```swift
   import Foundation

   struct InMemoryTaskRepository: TaskRepositoryProtocol {
       enum RepositoryError: LocalizedError {
           case simulatedFailure

           var errorDescription: String? {
               switch self {
               case .simulatedFailure:
                   return "No fue posible recuperar las tareas."
               }
           }
       }

       enum Mode: Sendable {
           case content
           case empty
           case failure
       }

       private let mode: Mode

       init(mode: Mode = .content) {
           self.mode = mode
       }

       func fetchTasks() async throws -> [Task] {
           try await Task.sleep(for: .milliseconds(500))

           switch mode {
           case .content:
               return [
                   Task(
                       id: UUID(uuidString: "00000000-0000-0000-0000-000000000001")!,
                       title: "Revisar arquitectura VIPER",
                       detail: "Comprobar que View, Presenter, Interactor y Router mantienen responsabilidades separadas.",
                       status: .inProgress
                   ),
                   Task(
                       id: UUID(uuidString: "00000000-0000-0000-0000-000000000002")!,
                       title: "Crear pruebas unitarias",
                       detail: "Preparar dobles de prueba para el repositorio y validar la transformación a modelos de vista.",
                       status: .pending
                   ),
                   Task(
                       id: UUID(uuidString: "00000000-0000-0000-0000-000000000003")!,
                       title: "Configurar CI",
                       detail: "Verificar que GitHub Actions puede compilar el esquema compartido CampusExplorer.",
                       status: .completed
                   )
               ]

           case .empty:
               return []

           case .failure:
               throw RepositoryError.simulatedFailure
           }
       }
   }
   ```

4. Compila el proyecto desde Xcode con **Product > Build**.

**Resultado esperado:**

El proyecto reconoce las entidades `Task` y `TaskStatus`, y `InMemoryTaskRepository` implementa el contrato de acceso a datos.

**Verificación:**

Comprueba que las siguientes afirmaciones son correctas:

- `Task` no importa UIKit.
- El protocolo no conoce la implementación concreta del repositorio.
- El repositorio puede devolver contenido, lista vacía o error simulado.
- La latencia de 500 ms permite observar el estado de carga sin bloquear la interfaz.

---

### Paso 3. Crear el componente reutilizable de estado de lista

**Objetivo:** Centralizar la representación visual de carga, vacío y error en un componente UIKit reutilizable.

**Instrucciones:**

1. Crea `Features/TaskList/Components/TaskListStateView.swift`:

   ```swift
   import UIKit

   final class TaskListStateView: UIView {
       enum State {
           case hidden
           case loading
           case empty(message: String)
           case error(message: String)
       }

       var onRetry: (() -> Void)?

       private let activityIndicator = UIActivityIndicatorView(style: .large)
       private let titleLabel = UILabel()
       private let messageLabel = UILabel()
       private let retryButton = UIButton(type: .system)
       private let stackView = UIStackView()

       override init(frame: CGRect) {
           super.init(frame: frame)
           configureView()
       }

       required init?(coder: NSCoder) {
           fatalError("init(coder:) has not been implemented")
       }

       func render(state: State) {
           isHidden = state == .hidden

           activityIndicator.stopAnimating()
           titleLabel.isHidden = true
           messageLabel.isHidden = true
           retryButton.isHidden = true

           switch state {
           case .hidden:
               break

           case .loading:
               activityIndicator.startAnimating()

           case let .empty(message):
               titleLabel.text = "No hay tareas"
               messageLabel.text = message
               titleLabel.isHidden = false
               messageLabel.isHidden = false

           case let .error(message):
               titleLabel.text = "No se pudo cargar"
               messageLabel.text = message
               titleLabel.isHidden = false
               messageLabel.isHidden = false
               retryButton.isHidden = false
           }
       }

       private func configureView() {
           backgroundColor = .systemBackground

           titleLabel.font = .preferredFont(forTextStyle: .headline)
           titleLabel.textAlignment = .center

           messageLabel.font = .preferredFont(forTextStyle: .subheadline)
           messageLabel.textAlignment = .center
           messageLabel.textColor = .secondaryLabel
           messageLabel.numberOfLines = 0

           retryButton.setTitle("Reintentar", for: .normal)
           retryButton.addTarget(self, action: #selector(didTapRetry), for: .touchUpInside)

           stackView.axis = .vertical
           stackView.alignment = .center
           stackView.spacing = 12
           stackView.translatesAutoresizingMaskIntoConstraints = false

           [
               activityIndicator,
               titleLabel,
               messageLabel,
               retryButton
           ].forEach(stackView.addArrangedSubview)

           addSubview(stackView)

           NSLayoutConstraint.activate([
               stackView.centerXAnchor.constraint(equalTo: centerXAnchor),
               stackView.centerYAnchor.constraint(equalTo: centerYAnchor),
               stackView.leadingAnchor.constraint(
                   greaterThanOrEqualTo: leadingAnchor,
                   constant: 24
               ),
               stackView.trailingAnchor.constraint(
                   lessThanOrEqualTo: trailingAnchor,
                   constant: -24
               )
           ])

           render(state: .hidden)
       }

       @objc
       private func didTapRetry() {
           onRetry?()
       }
   }
   ```

2. Observa que el componente recibe un estado mediante `render(state:)` y no conoce al presenter, al interactor ni al repositorio.

3. Compila el proyecto.

**Resultado esperado:**

`TaskListStateView` es una vista autocontenida capaz de mostrar cuatro estados: oculto, carga, vacío y error.

**Verificación:**

Revisa que la única acción externa del componente sea el closure `onRetry`. La vista no debe instanciar controladores, repositorios, routers ni coordinators.

---

### Paso 4. Definir los contratos VIPER de TaskList

**Objetivo:** Establecer colaboraciones explícitas entre las capas de la funcionalidad de lista.

**Instrucciones:**

1. Crea `Features/TaskList/Contracts/TaskListContracts.swift`:

   ```swift
   import Foundation

   struct TaskListItemViewModel: Equatable {
       let title: String
       let subtitle: String
       let statusText: String
   }

   protocol TaskListViewProtocol: AnyObject {
       func showLoading()
       func showTasks(_ items: [TaskListItemViewModel])
       func showEmpty(message: String)
       func showError(message: String)
   }

   protocol TaskListPresenterProtocol: AnyObject {
       func viewDidLoad()
       func didTapRetry()
       func didSelectTask(at index: Int)
   }

   protocol TaskListInteractorProtocol: AnyObject {
       func loadTasks() async throws -> [Task]
   }

   protocol TaskListRouterProtocol: AnyObject {
       func showDetail(for task: Task)
   }

   protocol TaskListRoutingDelegate: AnyObject {
       func showTaskDetail(task: Task)
   }
   ```

2. Revisa la dirección de las responsabilidades:

   | Capa | Conoce |
   |---|---|
   | View | Presenter y modelos de vista |
   | Presenter | View, Interactor, Router y entidades |
   | Interactor | Protocolo de repositorio y entidades |
   | Router | Delegado de navegación y entidad seleccionada |
   | Coordinator | Builders concretos y `UINavigationController` |

3. Compila el proyecto.

**Resultado esperado:**

La funcionalidad dispone de protocolos que permiten cambiar implementaciones y probar cada capa de forma aislada.

**Verificación:**

Confirma que `TaskListViewProtocol` no expone `UITableView`, `UIViewController` ni `TaskRepositoryProtocol`.

---

### Paso 5. Implementar Interactor, Presenter y Router de TaskList

**Objetivo:** Conectar el flujo de datos View → Presenter → Interactor → Repositorio y delegar la navegación al coordinator.

**Instrucciones:**

1. Crea `Features/TaskList/Interactor/TaskListInteractor.swift`:

   ```swift
   import Foundation

   final class TaskListInteractor: TaskListInteractorProtocol {
       private let repository: any TaskRepositoryProtocol

       init(repository: any TaskRepositoryProtocol) {
           self.repository = repository
       }

       func loadTasks() async throws -> [Task] {
           try await repository.fetchTasks()
       }
   }
   ```

2. Crea `Features/TaskList/Router/TaskListRouter.swift`:

   ```swift
   import Foundation

   final class TaskListRouter: TaskListRouterProtocol {
       weak var delegate: TaskListRoutingDelegate?

       init(delegate: TaskListRoutingDelegate) {
           self.delegate = delegate
       }

       func showDetail(for task: Task) {
           delegate?.showTaskDetail(task: task)
       }
   }
   ```

3. Crea `Features/TaskList/Presenter/TaskListPresenter.swift`:

   ```swift
   import Foundation

   @MainActor
   final class TaskListPresenter: TaskListPresenterProtocol {
       private weak var view: TaskListViewProtocol?
       private let interactor: TaskListInteractorProtocol
       private let router: TaskListRouterProtocol

       private var tasks: [Task] = []
       private var loadTask: Task<Void, Never>?

       init(
           view: TaskListViewProtocol,
           interactor: TaskListInteractorProtocol,
           router: TaskListRouterProtocol
       ) {
           self.view = view
           self.interactor = interactor
           self.router = router
       }

       deinit {
           loadTask?.cancel()
       }

       func viewDidLoad() {
           loadTasks()
       }

       func didTapRetry() {
           loadTasks()
       }

       func didSelectTask(at index: Int) {
           guard tasks.indices.contains(index) else {
               return
           }

           router.showDetail(for: tasks[index])
       }

       private func loadTasks() {
           loadTask?.cancel()
           view?.showLoading()

           loadTask = Task { [weak self] in
               guard let self else {
                   return
               }

               do {
                   let tasks = try await interactor.loadTasks()

                   guard !Task.isCancelled else {
                       return
                   }

                   self.tasks = tasks

                   if tasks.isEmpty {
                       view?.showEmpty(
                           message: "Cuando agregues tareas, aparecerán en esta lista."
                       )
                   } else {
                       let viewModels = tasks.map {
                           TaskListItemViewModel(
                               title: $0.title,
                               subtitle: $0.detail,
                               statusText: $0.status.displayName
                           )
                       }

                       view?.showTasks(viewModels)
                   }
               } catch {
                   guard !Task.isCancelled else {
                       return
                   }

                   view?.showError(
                       message: "Comprueba tu conexión o vuelve a intentarlo."
                   )
               }
           }
       }
   }
   ```

4. Analiza la gestión de memoria:

   - `view` es `weak` para evitar un ciclo entre presenter y view.
   - `delegate` del router es `weak` porque será el coordinator.
   - La tarea de carga se cancela antes de iniciar otra carga.
   - El presenter se declara `@MainActor` porque ordena actualizaciones de interfaz.

**Resultado esperado:**

El presenter coordina el caso de uso y convierte entidades `Task` en `TaskListItemViewModel`. El router no presenta controladores directamente: delega la ruta al coordinator.

**Verificación:**

Comprueba que `TaskListInteractor` no importa UIKit y que `TaskListPresenter` no crea instancias de `TaskDetailViewController`.

---

### Paso 6. Implementar la vista y el Builder de TaskList

**Objetivo:** Crear una vista pasiva basada en `UITableView` y ensamblar el módulo con dependencias inyectadas.

**Instrucciones:**

1. Crea `Features/TaskList/View/TaskListViewController.swift`:

   ```swift
   import UIKit

   final class TaskListViewController: UIViewController {
       var presenter: TaskListPresenterProtocol!

       private let tableView = UITableView(frame: .zero, style: .insetGrouped)
       private let stateView = TaskListStateView()
       private var items: [TaskListItemViewModel] = []

       override func viewDidLoad() {
           super.viewDidLoad()
           configureView()
           presenter.viewDidLoad()
       }

       private func configureView() {
           title = "Tareas"
           view.backgroundColor = .systemBackground

           tableView.translatesAutoresizingMaskIntoConstraints = false
           tableView.dataSource = self
           tableView.delegate = self
           tableView.register(
               UITableViewCell.self,
               forCellReuseIdentifier: "TaskCell"
           )

           stateView.translatesAutoresizingMaskIntoConstraints = false
           stateView.onRetry = { [weak self] in
               self?.presenter.didTapRetry()
           }

           view.addSubview(tableView)
           view.addSubview(stateView)

           NSLayoutConstraint.activate([
               tableView.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor),
               tableView.leadingAnchor.constraint(equalTo: view.leadingAnchor),
               tableView.trailingAnchor.constraint(equalTo: view.trailingAnchor),
               tableView.bottomAnchor.constraint(equalTo: view.bottomAnchor),

               stateView.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor),
               stateView.leadingAnchor.constraint(equalTo: view.leadingAnchor),
               stateView.trailingAnchor.constraint(equalTo: view.trailingAnchor),
               stateView.bottomAnchor.constraint(equalTo: view.bottomAnchor)
           ])
       }
   }

   extension TaskListViewController: TaskListViewProtocol {
       func showLoading() {
           stateView.render(state: .loading)
       }

       func showTasks(_ items: [TaskListItemViewModel]) {
           self.items = items
           tableView.reloadData()
           stateView.render(state: .hidden)
       }

       func showEmpty(message: String) {
           items = []
           tableView.reloadData()
           stateView.render(state: .empty(message: message))
       }

       func showError(message: String) {
           items = []
           tableView.reloadData()
           stateView.render(state: .error(message: message))
       }
   }

   extension TaskListViewController: UITableViewDataSource {
       func tableView(
           _ tableView: UITableView,
           numberOfRowsInSection section: Int
       ) -> Int {
           items.count
       }

       func tableView(
           _ tableView: UITableView,
           cellForRowAt indexPath: IndexPath
       ) -> UITableViewCell {
           let cell = tableView.dequeueReusableCell(
               withIdentifier: "TaskCell",
               for: indexPath
           )

           let item = items[indexPath.row]

           var content = cell.defaultContentConfiguration()
           content.text = item.title
           content.secondaryText = "\(item.statusText) · \(item.subtitle)"
           content.secondaryTextProperties.numberOfLines = 2
           cell.contentConfiguration = content
           cell.accessoryType = .disclosureIndicator

           return cell
       }
   }

   extension TaskListViewController: UITableViewDelegate {
       func tableView(
           _ tableView: UITableView,
           didSelectRowAt indexPath: IndexPath
       ) {
           tableView.deselectRow(at: indexPath, animated: true)
           presenter.didSelectTask(at: indexPath.row)
       }
   }
   ```

2. Crea `Features/TaskList/Builder/TaskListModuleBuilder.swift`:

   ```swift
   import UIKit

   enum TaskListModuleBuilder {
       static func build(
           repository: any TaskRepositoryProtocol,
           routingDelegate: TaskListRoutingDelegate
       ) -> UIViewController {
           let viewController = TaskListViewController()
           let interactor = TaskListInteractor(repository: repository)
           let router = TaskListRouter(delegate: routingDelegate)

           let presenter = TaskListPresenter(
               view: viewController,
               interactor: interactor,
               router: router
           )

           viewController.presenter = presenter

           return viewController
       }
   }
   ```

3. Compila el proyecto.

**Resultado esperado:**

El Builder es el único elemento del módulo que conoce las implementaciones concretas de View, Interactor, Presenter y Router.

**Verificación:**

Comprueba el flujo al seleccionar una celda:

```text
UITableViewDelegate
→ TaskListPresenter.didSelectTask(at:)
→ TaskListRouter.showDetail(for:)
→ TaskListRoutingDelegate
→ TaskFlowCoordinator
```

La vista no debe ejecutar `navigationController?.pushViewController(...)`.

---

### Paso 7. Implementar el módulo VIPER mínimo TaskDetail

**Objetivo:** Construir una pantalla de detalle a partir de la entidad seleccionada, sin que la lista cree directamente el controlador de detalle.

**Instrucciones:**

1. Crea `Features/TaskDetail/Contracts/TaskDetailContracts.swift`:

   ```swift
   import Foundation

   struct TaskDetailViewModel: Equatable {
       let title: String
       let detail: String
       let status: String
   }

   protocol TaskDetailViewProtocol: AnyObject {
       func showTask(_ viewModel: TaskDetailViewModel)
   }

   protocol TaskDetailPresenterProtocol: AnyObject {
       func viewDidLoad()
   }

   protocol TaskDetailInteractorProtocol: AnyObject {
       func task() -> Task
   }

   protocol TaskDetailRouterProtocol: AnyObject {
       func close()
   }
   ```

2. Crea `Features/TaskDetail/Interactor/TaskDetailInteractor.swift`:

   ```swift
   import Foundation

   final class TaskDetailInteractor: TaskDetailInteractorProtocol {
       private let selectedTask: Task

       init(task: Task) {
           self.selectedTask = task
       }

       func task() -> Task {
           selectedTask
       }
   }
   ```

3. Crea `Features/TaskDetail/Presenter/TaskDetailPresenter.swift`:

   ```swift
   import Foundation

   @MainActor
   final class TaskDetailPresenter: TaskDetailPresenterProtocol {
       private weak var view: TaskDetailViewProtocol?
       private let interactor: TaskDetailInteractorProtocol

       init(
           view: TaskDetailViewProtocol,
           interactor: TaskDetailInteractorProtocol
       ) {
           self.view = view
           self.interactor = interactor
       }

       func viewDidLoad() {
           let task = interactor.task()

           let viewModel = TaskDetailViewModel(
               title: task.title,
               detail: task.detail,
               status: task.status.displayName
           )

           view?.showTask(viewModel)
       }
   }
   ```

4. Crea `Features/TaskDetail/Router/TaskDetailRouter.swift`:

   ```swift
   import UIKit

   final class TaskDetailRouter: TaskDetailRouterProtocol {
       weak var viewController: UIViewController?

       func close() {
           viewController?.navigationController?.popViewController(animated: true)
       }
   }
   ```

5. Crea `Features/TaskDetail/View/TaskDetailViewController.swift`:

   ```swift
   import UIKit

   final class TaskDetailViewController: UIViewController {
       var presenter: TaskDetailPresenterProtocol!

       private let titleLabel = UILabel()
       private let statusLabel = UILabel()
       private let detailLabel = UILabel()
       private let stackView = UIStackView()

       override func viewDidLoad() {
           super.viewDidLoad()
           configureView()
           presenter.viewDidLoad()
       }

       private func configureView() {
           title = "Detalle"
           view.backgroundColor = .systemBackground

           titleLabel.font = .preferredFont(forTextStyle: .title2)
           titleLabel.numberOfLines = 0

           statusLabel.font = .preferredFont(forTextStyle: .subheadline)
           statusLabel.textColor = .secondaryLabel

           detailLabel.font = .preferredFont(forTextStyle: .body)
           detailLabel.numberOfLines = 0

           stackView.axis = .vertical
           stackView.spacing = 16
           stackView.translatesAutoresizingMaskIntoConstraints = false

           [titleLabel, statusLabel, detailLabel].forEach(stackView.addArrangedSubview)

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
   }

   extension TaskDetailViewController: TaskDetailViewProtocol {
       func showTask(_ viewModel: TaskDetailViewModel) {
           titleLabel.text = viewModel.title
           statusLabel.text = "Estado: \(viewModel.status)"
           detailLabel.text = viewModel.detail
       }
   }
   ```

6. Crea `Features/TaskDetail/Builder/TaskDetailModuleBuilder.swift`:

   ```swift
   import UIKit

   enum TaskDetailModuleBuilder {
       static func build(task: Task) -> UIViewController {
           let viewController = TaskDetailViewController()
           let interactor = TaskDetailInteractor(task: task)
           let router = TaskDetailRouter()

           let presenter = TaskDetailPresenter(
               view: viewController,
               interactor: interactor
           )

           viewController.presenter = presenter
           router.viewController = viewController

           return viewController
       }
   }
   ```

**Resultado esperado:**

El detalle recibe una `Task` como entidad de entrada y muestra un modelo de vista preparado por el presenter.

**Verificación:**

Confirma que `TaskDetailModuleBuilder.build(task:)` es el único punto de construcción del detalle y que `TaskListViewController` no importa ni instancia `TaskDetailViewController`.

---

### Paso 8. Crear el contenedor de composición y los coordinators

**Objetivo:** Centralizar la composición global y sustituir la navegación directa por `AppCoordinator` y `TaskFlowCoordinator`.

**Instrucciones:**

1. Crea `Core/Composition/AppContainer.swift`:

   ```swift
   import Foundation

   final class AppContainer {
       let taskRepository: any TaskRepositoryProtocol

       init(taskRepository: any TaskRepositoryProtocol = InMemoryTaskRepository()) {
           self.taskRepository = taskRepository
       }
   }
   ```

2. Crea `App/TaskFlowCoordinator.swift`:

   ```swift
   import UIKit

   final class TaskFlowCoordinator: TaskListRoutingDelegate {
       private let navigationController: UINavigationController
       private let container: AppContainer

       init(
           navigationController: UINavigationController,
           container: AppContainer
       ) {
           self.navigationController = navigationController
           self.container = container
       }

       func start() {
           let taskListViewController = TaskListModuleBuilder.build(
               repository: container.taskRepository,
               routingDelegate: self
           )

           navigationController.setViewControllers(
               [taskListViewController],
               animated: false
           )
       }

       func showTaskDetail(task: Task) {
           let detailViewController = TaskDetailModuleBuilder.build(task: task)
           navigationController.pushViewController(
               detailViewController,
               animated: true
           )
       }
   }
   ```

3. Crea `App/AppCoordinator.swift`:

   ```swift
   import UIKit

   final class AppCoordinator {
       private let window: UIWindow
       private let container: AppContainer
       private let navigationController: UINavigationController
       private var taskFlowCoordinator: TaskFlowCoordinator?

       init(window: UIWindow, container: AppContainer = AppContainer()) {
           self.window = window
           self.container = container
           self.navigationController = UINavigationController()
       }

       func start() {
           let taskFlowCoordinator = TaskFlowCoordinator(
               navigationController: navigationController,
               container: container
           )

           self.taskFlowCoordinator = taskFlowCoordinator

           window.rootViewController = navigationController
           window.makeKeyAndVisible()

           taskFlowCoordinator.start()
       }
   }
   ```

4. Sustituye el contenido de `App/SceneDelegate.swift` por el siguiente código. Si tu proyecto usa `AppDelegate` sin escenas, adapta esta misma composición al método `application(_:didFinishLaunchingWithOptions:)`.

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
           let coordinator = AppCoordinator(window: window)

           self.window = window
           self.appCoordinator = coordinator

           coordinator.start()
       }
   }
   ```

5. Comprueba que no queda código de inicio anterior que asigne otro `rootViewController`.

**Resultado esperado:**

`AppCoordinator` inicia la aplicación y mantiene el coordinator de flujo. `TaskFlowCoordinator` presenta la lista y empuja el detalle en el `UINavigationController`.

**Verificación:**

La cadena de composición debe ser:

```text
SceneDelegate
→ AppCoordinator
→ AppContainer
→ TaskFlowCoordinator
→ TaskListModuleBuilder
→ TaskListViewController
```

Y la navegación debe ser:

```text
TaskListRouter
→ TaskFlowCoordinator
→ TaskDetailModuleBuilder
→ UINavigationController.pushViewController
```

---

### Paso 9. Documentar las dependencias y compilar desde Terminal

**Objetivo:** Documentar la dirección de dependencias y validar el proyecto desde línea de comandos.

**Instrucciones:**

1. Añade al archivo `README.md` una sección llamada `## Arquitectura VIPER y navegación`.

2. Incluye el siguiente diagrama de dependencias:

   ```markdown
   ## Arquitectura VIPER y navegación

   ```text
   AppCoordinator
      ├── AppContainer
      │    └── TaskRepositoryProtocol
      │         └── InMemoryTaskRepository
      └── TaskFlowCoordinator
           ├── TaskListModuleBuilder
           │    ├── TaskListViewController
           │    ├── TaskListPresenter
           │    ├── TaskListInteractor
           │    └── TaskListRouter
           └── TaskDetailModuleBuilder
                ├── TaskDetailViewController
                ├── TaskDetailPresenter
                ├── TaskDetailInteractor
                └── TaskDetailRouter

   TaskListViewController
      → TaskListPresenter
      → TaskListInteractor
      → TaskRepositoryProtocol

   TaskListPresenter
      → TaskListRouter
      → TaskFlowCoordinator
      → TaskDetailModuleBuilder
   ```

   La View no consulta repositorios ni realiza navegación directa. El Presenter no crea View Controllers. El Router solicita navegación al Coordinator mediante un protocolo.
   ```

3. Lista los esquemas disponibles:

   ```bash
   xcodebuild -list -project CampusExplorer.xcodeproj
   ```

4. Compila usando el destino obligatorio:

   ```bash
   xcodebuild \
     -project CampusExplorer.xcodeproj \
     -scheme CampusExplorer \
     -sdk iphonesimulator \
     -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5' \
     clean build
   ```

**Resultado esperado:**

La salida final de compilación contiene:

```text
** BUILD SUCCEEDED **
```

**Verificación:**

Confirma que el archivo `README.md` documenta el flujo y que el proyecto compila sin errores ni advertencias críticas de concurrencia.

## Validación y pruebas

### Prueba manual del flujo principal

1. Ejecuta el esquema `CampusExplorer` en:

   ```text
   iPhone 16 — iOS 18.5
   ```

2. Comprueba el estado inicial:
   - Se muestra temporalmente el indicador de carga.
   - La barra de navegación muestra el título **Tareas**.

3. Espera la respuesta del repositorio determinista:
   - Deben aparecer tres tareas.
   - Cada celda muestra título, estado y descripción.
   - Cada celda muestra un indicador de navegación.

4. Selecciona **Revisar arquitectura VIPER**:
   - Debe abrirse la pantalla **Detalle**.
   - Deben mostrarse título, estado y descripción completa.
   - La navegación debe usar una transición de `push`.

5. Pulsa el botón Atrás:
   - Debe volver a la lista.
   - La lista debe seguir siendo la pantalla raíz del `UINavigationController`.

### Prueba del estado vacío

1. En `AppContainer.swift`, cambia temporalmente la dependencia por:

   ```swift
   init(
       taskRepository: any TaskRepositoryProtocol =
           InMemoryTaskRepository(mode: .empty)
   ) {
       self.taskRepository = taskRepository
   }
   ```

2. Ejecuta la aplicación.

3. Verifica:
   - No se muestran celdas.
   - Se muestra el título **No hay tareas**.
   - Se muestra el mensaje informativo configurado por el presenter.

4. Restaura el modo `.content` al finalizar la prueba.

### Prueba del estado de error y reintento

1. Cambia temporalmente la dependencia del contenedor:

   ```swift
   init(
       taskRepository: any TaskRepositoryProtocol =
           InMemoryTaskRepository(mode: .failure)
   ) {
       self.taskRepository = taskRepository
   }
   ```

2. Ejecuta la aplicación.

3. Verifica:
   - Se muestra el estado de error.
   - Se muestra el botón **Reintentar**.
   - Al pulsarlo se vuelve a mostrar el indicador de carga y posteriormente el error, porque el repositorio continúa en modo `.failure`.

4. Restaura `.content` antes de confirmar el código final.

### Validación de responsabilidades

Usa esta lista de control antes de entregar:

| Comprobación | Resultado esperado |
|---|---|
| La View carga tareas | No; solo notifica `viewDidLoad()` al Presenter |
| El Presenter consulta el repositorio directamente | No; usa el Interactor |
| El Interactor importa UIKit | No |
| El Presenter crea `TaskDetailViewController` | No |
| La View usa `pushViewController` | No |
| El Router usa un delegate débil | Sí |
| El Coordinator posee el `UINavigationController` | Sí |
| El Builder construye el grafo de dependencias | Sí |
| `TaskListStateView` conoce VIPER | No; solo representa estados visuales |

### Confirmación final en Git

1. Revisa los archivos modificados:

   ```bash
   git status
   git diff --check
   ```

2. Añade los cambios:

   ```bash
   git add .
   ```

3. Crea el commit final con el mensaje definido:

   ```bash
   git commit -m "feat: refactor task flow with VIPER and coordinators"
   ```

4. Comprueba el historial:

   ```bash
   git log --oneline -1
   ```

## Resolución de problemas

### Problema 1: `xcodebuild` no encuentra el simulador iPhone 16 con iOS 18.5

**Síntoma:**

El comando de compilación muestra un error similar a:

```text
Unable to find a destination matching the provided destination specifier
```

**Causa:**

El runtime iOS 18.5 no está instalado, el simulador se llama de forma diferente o Xcode está seleccionando otra instalación de herramientas de desarrollo.

**Solución:**

1. Lista los dispositivos disponibles:

   ```bash
   xcrun simctl list devices available
   ```

2. Instala el runtime iOS 18.5 desde **Xcode > Settings > Components** si no aparece.

3. Verifica la ruta de Xcode activa:

   ```bash
   xcode-select -p
   ```

4. Si el nombre o versión no coincide, usa temporalmente un destino existente y documenta la diferencia. Para la entrega final, vuelve a usar:

   ```bash
   -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5'
   ```

### Problema 2: La lista carga, pero al seleccionar una tarea no se abre el detalle

**Síntoma:**

La celda se selecciona y se deselecciona, pero no se produce navegación hacia `TaskDetailViewController`.

**Causa:**

El `TaskListRouter` no recibió correctamente el `TaskFlowCoordinator` como `routingDelegate`, o el coordinator fue liberado porque no existe una referencia fuerte desde `AppCoordinator`.

**Solución:**

1. Verifica que el Builder recibe `routingDelegate: self` desde `TaskFlowCoordinator`:

   ```swift
   let taskListViewController = TaskListModuleBuilder.build(
       repository: container.taskRepository,
       routingDelegate: self
   )
   ```

2. Confirma que `AppCoordinator` conserva el flujo:

   ```swift
   private var taskFlowCoordinator: TaskFlowCoordinator?
   ```

3. Comprueba que `start()` asigna la referencia antes de mostrar la lista:

   ```swift
   self.taskFlowCoordinator = taskFlowCoordinator
   ```

4. Añade temporalmente un breakpoint en `showTaskDetail(task:)` para comprobar que el router invoca al coordinator.

## Limpieza

1. Restaura el repositorio al modo de contenido:

   ```swift
   InMemoryTaskRepository(mode: .content)
   ```

   También puedes simplificarlo usando el valor por defecto:

   ```swift
   InMemoryTaskRepository()
   ```

2. Detén la aplicación en Simulator.

3. Conserva los archivos fuente, el diagrama de `README.md` y el commit final; forman parte de la entrega.

4. Si generaste archivos temporales fuera del proyecto, elimínalos:

   ```bash
   rm -rf /tmp/CampusExplorerLab
   ```

5. Comprueba el estado final:

   ```bash
   git status
   ```

   El resultado esperado es un árbol de trabajo limpio después del commit.

## Resumen

En esta práctica has transformado una funcionalidad de tareas en una arquitectura VIPER con contratos explícitos. `TaskListPresenter` coordina la presentación, `TaskListInteractor` ejecuta el caso de uso sobre una abstracción de repositorio y `TaskListRouter` solicita navegación al `TaskFlowCoordinator`.

También has creado `TaskListStateView` para reutilizar la interfaz de carga, vacío y error, y has centralizado la composición en `AppContainer` y `AppCoordinator`. Como resultado, la lista y el detalle son módulos construibles, navegables y con responsabilidades separadas, preparados para evolucionar hacia pruebas unitarias, snapshots y automatización continua.

### Recursos opcionales

- [Apple Developer Documentation: UIKit](https://developer.apple.com/documentation/uikit)
- [Apple Developer Documentation: Concurrency](https://developer.apple.com/documentation/swift/concurrency)
- [Swift Package Manager](https://www.swift.org/documentation/package-manager/)
- [WWDC: Modern Swift Concurrency](https://developer.apple.com/videos/)
