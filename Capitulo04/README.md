# 4 Práctica incremental

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 252 minutos |
| Complejidad | Alta |
| Nivel de Bloom | Aplicar |

## Descripción general

En esta práctica ampliarás la aplicación `CampusExplorer` para mostrar lugares del campus en un mapa interactivo basado en `MapKit`. Crearás un módulo reutilizable llamado `CampusLocationKit` que encapsulará la autorización de ubicación, la obtención de coordenadas y el cálculo de distancias mediante `Core Location`.

La pantalla de mapa será accesible desde el `PlacesCoordinator` existente, mostrará anotaciones para los lugares cargados desde el repositorio y usará una región predefinida cuando el usuario no haya autorizado el acceso a su ubicación.

## Objetivos de aprendizaje

Al finalizar la práctica, podrás:

- [ ] Crear una pantalla UIKit con `MKMapView`, anotaciones seleccionables y una región inicial del campus.
- [ ] Gestionar los estados de permiso `notDetermined`, `authorizedWhenInUse`, `denied` y `restricted` mediante `CLLocationManagerDelegate`.
- [ ] Calcular y presentar la distancia aproximada entre la ubicación actual y un lugar seleccionado.
- [ ] Encapsular Core Location en el framework reutilizable `CampusLocationKit`.
- [ ] Integrar el framework mediante inyección de dependencias desde el `PlacesCoordinator`.

## Prerrequisitos

### Conocimientos necesarios

- Haber completado la práctica `03-00-01` y disponer de la etiqueta Git `lab-03-complete`.
- Comprender protocolos, delegados, `async/await`, inyección de dependencias y el patrón Coordinator.
- Conocer la estructura MVVM utilizada en el proyecto anterior.
- Tener nociones básicas de `MapKit`, `CLLocationCoordinate2D` y regiones cartográficas.

### Acceso necesario

- Acceso al repositorio local `~/Developer/CampusExplorer`.
- Xcode 16.4 con el runtime de iOS Simulator 18.5 instalado.
- Simulador `iPhone 16` disponible.
- Conexión a Internet para resolver dependencias de Swift Package Manager.
- Opcionalmente, un iPhone físico con iOS 18.5 para validar permisos reales.

## Entorno del laboratorio

### Software requerido

| Componente | Versión esperada |
|---|---|
| macOS | Sequoia 15.5 o compatible |
| Xcode | 16.4 |
| Swift | 6.1 |
| SDK de compilación | iOS 18.5 |
| Deployment target | iOS 18.0 |
| Simulador | iPhone 16, iOS 18.5 |
| Git | 2.46.0 o compatible |

### Estructura esperada del repositorio

```text
~/Developer/CampusExplorer
├── CampusExplorer.xcodeproj
├── Packages
│   ├── CampusDomain
│   ├── CampusNetworking
│   ├── CampusDesignSystem
│   └── CampusLocationKit       ← se creará en esta práctica
└── ...
```

### Preparación inicial

Abre Terminal y ejecuta los siguientes comandos:

```bash
cd ~/Developer/CampusExplorer
git status
git branch --show-current
git tag --list "lab-03-complete"
```

Confirma que estás en la rama `main` y crea una rama de trabajo:

```bash
git checkout main
git pull --ff-only
git checkout -b feature/lab-04-location-map
```

Comprueba que el esquema compartido existe:

```bash
xcodebuild -list -project CampusExplorer.xcodeproj
```

Debe aparecer el esquema `CampusExplorer`.

## Procedimiento paso a paso

### Paso 1. Restaurar y verificar el estado base

**Objetivo:** confirmar que el proyecto de la práctica anterior compila antes de introducir cambios de localización y mapas.

**Instrucciones:**

1. Abre el proyecto en Xcode:

   ```bash
   open CampusExplorer.xcodeproj
   ```

2. En Xcode, selecciona el esquema compartido `CampusExplorer`.

3. Selecciona como destino:

   ```text
   iPhone 16 (iOS 18.5)
   ```

4. Ejecuta las pruebas existentes desde Xcode con `⌘U`.

5. Ejecuta también la compilación desde Terminal:

   ```bash
   xcodebuild \
     -project CampusExplorer.xcodeproj \
     -scheme CampusExplorer \
     -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5' \
     clean build
   ```

6. Si la etiqueta de la práctica anterior existe, conserva una referencia verificable:

   ```bash
   git show --stat lab-03-complete
   ```

**Resultado esperado:**

- El proyecto compila correctamente.
- Las pruebas existentes finalizan sin errores.
- La aplicación anterior permite navegar a la lista de lugares.

**Verificación:**

Ejecuta la aplicación en el simulador. Debes poder abrir la pantalla de lugares gestionada por el `PlacesCoordinator` antes de continuar.

---

### Paso 2. Crear el framework local `CampusLocationKit`

**Objetivo:** crear un módulo reutilizable que oculte los detalles de `CLLocationManager` y exponga una interfaz pública independiente de la aplicación principal.

**Instrucciones:**

1. Crea la estructura inicial del paquete:

   ```bash
   cd ~/Developer/CampusExplorer

   mkdir -p Packages/CampusLocationKit/Sources/CampusLocationKit
   mkdir -p Packages/CampusLocationKit/Tests/CampusLocationKitTests
   ```

2. Crea el archivo `Packages/CampusLocationKit/Package.swift`:

   ```swift
   // swift-tools-version: 6.1

   import PackageDescription

   let package = Package(
       name: "CampusLocationKit",
       platforms: [
           .iOS(.v18),
           .macOS(.v14)
       ],
       products: [
           .library(
               name: "CampusLocationKit",
               targets: ["CampusLocationKit"]
           )
       ],
       targets: [
           .target(
               name: "CampusLocationKit"
           ),
           .testTarget(
               name: "CampusLocationKitTests",
               dependencies: ["CampusLocationKit"]
           )
       ]
   )
   ```

3. Crea el archivo `Packages/CampusLocationKit/Sources/CampusLocationKit/LocationCoordinate.swift`:

   ```swift
   import CoreLocation

   public struct LocationCoordinate: Sendable, Equatable, Codable {
       public let latitude: Double
       public let longitude: Double

       public init(latitude: Double, longitude: Double) {
           self.latitude = latitude
           self.longitude = longitude
       }

       public var clLocationCoordinate2D: CLLocationCoordinate2D {
           CLLocationCoordinate2D(
               latitude: latitude,
               longitude: longitude
           )
       }

       public var clLocation: CLLocation {
           CLLocation(
               latitude: latitude,
               longitude: longitude
           )
       }
   }
   ```

4. Crea el archivo `Packages/CampusLocationKit/Sources/CampusLocationKit/LocationManaging.swift`:

   ```swift
   import Foundation

   public enum LocationAuthorizationStatus: Sendable, Equatable {
       case notDetermined
       case authorizedWhenInUse
       case denied
       case restricted
   }

   @MainActor
   public protocol CampusLocationManagerDelegate: AnyObject {
       func campusLocationManager(
           _ manager: any LocationManaging,
           didChange authorizationStatus: LocationAuthorizationStatus
       )

       func campusLocationManager(
           _ manager: any LocationManaging,
           didUpdate coordinate: LocationCoordinate
       )

       func campusLocationManager(
           _ manager: any LocationManaging,
           didFailWith error: Error
       )
   }

   @MainActor
   public protocol LocationManaging: AnyObject {
       var delegate: (any CampusLocationManagerDelegate)? { get set }
       var authorizationStatus: LocationAuthorizationStatus { get }
       var lastKnownCoordinate: LocationCoordinate? { get }

       func requestWhenInUseAuthorization()
       func startUpdatingLocation()
       func stopUpdatingLocation()

       func distance(
           from origin: LocationCoordinate,
           to destination: LocationCoordinate
       ) -> CLLocationDistance
   }
   ```

5. Observa que `CLLocationDistance` pertenece a Core Location. Añade la importación correspondiente al inicio del archivo anterior:

   ```swift
   import CoreLocation
   import Foundation
   ```

6. Crea el archivo `Packages/CampusLocationKit/Sources/CampusLocationKit/LocationManager.swift`:

   ```swift
   @preconcurrency import CoreLocation
   import Foundation

   @MainActor
   public final class LocationManager: NSObject, LocationManaging {
       public weak var delegate: (any CampusLocationManagerDelegate)?

       public private(set) var lastKnownCoordinate: LocationCoordinate?

       private let locationManager: CLLocationManager

       public override init() {
           locationManager = CLLocationManager()
           super.init()

           locationManager.delegate = self
           locationManager.desiredAccuracy = kCLLocationAccuracyNearestTenMeters
           locationManager.distanceFilter = 10
       }

       public var authorizationStatus: LocationAuthorizationStatus {
           Self.map(locationManager.authorizationStatus)
       }

       public func requestWhenInUseAuthorization() {
           locationManager.requestWhenInUseAuthorization()
       }

       public func startUpdatingLocation() {
           guard authorizationStatus == .authorizedWhenInUse else {
               return
           }

           locationManager.startUpdatingLocation()
       }

       public func stopUpdatingLocation() {
           locationManager.stopUpdatingLocation()
       }

       public func distance(
           from origin: LocationCoordinate,
           to destination: LocationCoordinate
       ) -> CLLocationDistance {
           origin.clLocation.distance(from: destination.clLocation)
       }

       private static func map(
           _ status: CLAuthorizationStatus
       ) -> LocationAuthorizationStatus {
           switch status {
           case .notDetermined:
               return .notDetermined

           case .authorizedWhenInUse, .authorizedAlways:
               return .authorizedWhenInUse

           case .denied:
               return .denied

           case .restricted:
               return .restricted

           @unknown default:
               return .denied
           }
       }
   }

   extension LocationManager: CLLocationManagerDelegate {
       public func locationManagerDidChangeAuthorization(
           _ manager: CLLocationManager
       ) {
           let status = Self.map(manager.authorizationStatus)

           delegate?.campusLocationManager(
               self,
               didChange: status
           )
       }

       public func locationManager(
           _ manager: CLLocationManager,
           didUpdateLocations locations: [CLLocation]
       ) {
           guard let location = locations.last else {
               return
           }

           let coordinate = LocationCoordinate(
               latitude: location.coordinate.latitude,
               longitude: location.coordinate.longitude
           )

           lastKnownCoordinate = coordinate

           delegate?.campusLocationManager(
               self,
               didUpdate: coordinate
           )
       }

       public func locationManager(
           _ manager: CLLocationManager,
           didFailWithError error: Error
       ) {
           delegate?.campusLocationManager(
               self,
               didFailWith: error
           )
       }
   }
   ```

7. Ejecuta las pruebas del paquete, aunque todavía no existan pruebas funcionales:

   ```bash
   cd ~/Developer/CampusExplorer/Packages/CampusLocationKit
   swift test
   ```

**Resultado esperado:**

- El paquete `CampusLocationKit` compila como una librería Swift local.
- El código de Core Location queda contenido en el paquete.
- La aplicación principal no necesita crear directamente un `CLLocationManager`.

**Verificación:**

El comando debe finalizar con un resultado equivalente a:

```text
Build complete!
Test Suite 'All tests' passed
```

---

### Paso 3. Añadir pruebas unitarias al framework

**Objetivo:** verificar que el cálculo de distancia funciona sin depender de permisos, simulador ni interfaz gráfica.

**Instrucciones:**

1. Crea el archivo `Packages/CampusLocationKit/Tests/CampusLocationKitTests/LocationCoordinateTests.swift`:

   ```swift
   import CoreLocation
   import XCTest
   @testable import CampusLocationKit

   final class LocationCoordinateTests: XCTestCase {
       func testDistanceBetweenEqualCoordinatesIsZero() {
           let manager = LocationManager()

           let coordinate = LocationCoordinate(
               latitude: 40.4168,
               longitude: -3.7038
           )

           let distance = manager.distance(
               from: coordinate,
               to: coordinate
           )

           XCTAssertEqual(distance, 0, accuracy: 0.001)
       }

       func testDistanceBetweenTwoCoordinatesIsPositive() {
           let manager = LocationManager()

           let origin = LocationCoordinate(
               latitude: 40.4168,
               longitude: -3.7038
           )

           let destination = LocationCoordinate(
               latitude: 40.4203,
               longitude: -3.6881
           )

           let distance = manager.distance(
               from: origin,
               to: destination
           )

           XCTAssertGreaterThan(distance, 1_000)
           XCTAssertLessThan(distance, 2_000)
       }
   }
   ```

2. Ejecuta de nuevo las pruebas del paquete:

   ```bash
   cd ~/Developer/CampusExplorer/Packages/CampusLocationKit
   swift test
   ```

3. Vuelve al directorio raíz del proyecto:

   ```bash
   cd ~/Developer/CampusExplorer
   ```

**Resultado esperado:**

- Las dos pruebas pasan.
- La prueba de distancia igual produce aproximadamente `0` metros.
- La segunda prueba confirma una distancia aproximada positiva entre dos coordenadas de Madrid.

**Verificación:**

Debes observar una salida similar a:

```text
Test Case 'LocationCoordinateTests.testDistanceBetweenEqualCoordinatesIsZero' passed
Test Case 'LocationCoordinateTests.testDistanceBetweenTwoCoordinatesIsPositive' passed
```

---

### Paso 4. Integrar el paquete y configurar el permiso de ubicación

**Objetivo:** enlazar `CampusLocationKit` con la aplicación y añadir el mensaje de privacidad obligatorio para el permiso de ubicación.

**Instrucciones:**

1. En Xcode, selecciona el proyecto `CampusExplorer`.

2. Selecciona **File > Add Package Dependencies…**.

3. Pulsa **Add Local…**.

4. Selecciona esta carpeta:

   ```text
   ~/Developer/CampusExplorer/Packages/CampusLocationKit
   ```

5. Añade el producto `CampusLocationKit` al target principal de la aplicación `CampusExplorer`.

6. Verifica que el paquete aparece en la sección **Package Dependencies** del proyecto.

7. Selecciona el target de aplicación `CampusExplorer`.

8. Abre la pestaña **Info**.

9. En **Custom iOS Target Properties**, agrega la siguiente clave:

   | Clave | Tipo | Valor |
   |---|---|---|
   | `Privacy - Location When In Use Usage Description` | String | `Usamos tu ubicación para mostrar la distancia a los lugares del campus seleccionados.` |

10. Si el proyecto utiliza un archivo `Info.plist` físico, el contenido equivalente debe ser:

   ```xml
   <key>NSLocationWhenInUseUsageDescription</key>
   <string>Usamos tu ubicación para mostrar la distancia a los lugares del campus seleccionados.</string>
   ```

11. Confirma que el deployment target de la aplicación sigue siendo iOS 18.0.

12. En un archivo temporal de la aplicación, o al crear la pantalla en el paso siguiente, verifica que Xcode resuelve esta importación:

   ```swift
   import CampusLocationKit
   ```

**Resultado esperado:**

- El target principal reconoce `CampusLocationKit`.
- La aplicación dispone de una descripción explícita de privacidad.
- Xcode no muestra errores de módulo no encontrado.

**Verificación:**

Compila el proyecto:

```bash
xcodebuild \
  -project CampusExplorer.xcodeproj \
  -scheme CampusExplorer \
  -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5' \
  build
```

La compilación debe finalizar con:

```text
** BUILD SUCCEEDED **
```

---

### Paso 5. Crear el modelo de presentación y el ViewModel del mapa

**Objetivo:** implementar la lógica de presentación del mapa siguiendo MVVM e inyectando la abstracción `LocationManaging`.

**Instrucciones:**

1. Crea un grupo o carpeta `Places/Map` dentro del target principal de la aplicación.

2. Crea `CampusMapPlace.swift`. Este modelo adapta el modelo de dominio existente al mapa sin modificar el framework:

   ```swift
   import CampusLocationKit
   import Foundation

   struct CampusMapPlace: Identifiable, Equatable {
       let id: String
       let name: String
       let subtitle: String
       let coordinate: LocationCoordinate
   }
   ```

3. Crea `PlacesMapViewModel.swift`:

   ```swift
   import CampusLocationKit
   import CoreLocation
   import Foundation
   import MapKit

   @MainActor
   final class PlacesMapViewModel: NSObject {
       var onRegionChange: ((MKCoordinateRegion) -> Void)?
       var onLocationStateChange: ((String) -> Void)?
       var onDistanceChange: ((String?) -> Void)?
       var onCenterUserLocation: ((LocationCoordinate) -> Void)?

       let places: [CampusMapPlace]

       private let locationManager: any LocationManaging
       private(set) var selectedPlace: CampusMapPlace?
       private var currentCoordinate: LocationCoordinate?

       let campusRegion = MKCoordinateRegion(
           center: CLLocationCoordinate2D(
               latitude: 40.4168,
               longitude: -3.7038
           ),
           span: MKCoordinateSpan(
               latitudeDelta: 0.025,
               longitudeDelta: 0.025
           )
       )

       init(
           places: [CampusMapPlace],
           locationManager: any LocationManaging
       ) {
           self.places = places
           self.locationManager = locationManager
           super.init()

           self.locationManager.delegate = self
       }

       func start() {
           onRegionChange?(campusRegion)
           updateLocationMessage(
               for: locationManager.authorizationStatus
           )
       }

       func requestUserLocation() {
           switch locationManager.authorizationStatus {
           case .notDetermined:
               locationManager.requestWhenInUseAuthorization()

           case .authorizedWhenInUse:
               locationManager.startUpdatingLocation()

           case .denied:
               onLocationStateChange?(
                   "La ubicación está desactivada. Actívala en Ajustes si deseas calcular distancias."
               )

           case .restricted:
               onLocationStateChange?(
                   "La ubicación está restringida en este dispositivo."
               )
           }
       }

       func selectPlace(withID id: String) {
           guard let place = places.first(where: { $0.id == id }) else {
               return
           }

           selectedPlace = place

           let region = MKCoordinateRegion(
               center: place.coordinate.clLocationCoordinate2D,
               span: MKCoordinateSpan(
                   latitudeDelta: 0.008,
                   longitudeDelta: 0.008
               )
           )

           onRegionChange?(region)
           updateDistanceIfPossible()
       }

       private func updateLocationMessage(
           for status: LocationAuthorizationStatus
       ) {
           switch status {
           case .notDetermined:
               onLocationStateChange?(
                   "Selecciona «Usar mi ubicación» para calcular distancias."
               )

           case .authorizedWhenInUse:
               onLocationStateChange?(
                   "Ubicación autorizada. Esperando una posición válida."
               )

           case .denied:
               onLocationStateChange?(
                   "No se autorizó la ubicación. Se muestra la región predeterminada del campus."
               )

           case .restricted:
               onLocationStateChange?(
                   "La ubicación está restringida. Se muestra la región predeterminada del campus."
               )
           }
       }

       private func updateDistanceIfPossible() {
           guard
               let currentCoordinate,
               let selectedPlace
           else {
               onDistanceChange?(nil)
               return
           }

           let meters = locationManager.distance(
               from: currentCoordinate,
               to: selectedPlace.coordinate
           )

           let formatter = MeasurementFormatter()
           formatter.unitStyle = .medium
           formatter.unitOptions = .providedUnit

           let text = formatter.string(
               from: Measurement(
                   value: meters,
                   unit: UnitLength.meters
               )
           )

           onDistanceChange?("Distancia aproximada: \(text)")
       }
   }

   extension PlacesMapViewModel: CampusLocationManagerDelegate {
       func campusLocationManager(
           _ manager: any LocationManaging,
           didChange authorizationStatus: LocationAuthorizationStatus
       ) {
           updateLocationMessage(for: authorizationStatus)

           if authorizationStatus == .authorizedWhenInUse {
               manager.startUpdatingLocation()
           }
       }

       func campusLocationManager(
           _ manager: any LocationManaging,
           didUpdate coordinate: LocationCoordinate
       ) {
           currentCoordinate = coordinate
           onCenterUserLocation?(coordinate)
           updateDistanceIfPossible()
       }

       func campusLocationManager(
           _ manager: any LocationManaging,
           didFailWith error: Error
       ) {
           onLocationStateChange?(
               "No fue posible obtener la ubicación actual."
           )
       }
   }
   ```

4. Adapta los lugares provenientes del repositorio existente. Realiza esta conversión en la capa de presentación o en el `PlacesCoordinator`, no dentro de `CampusLocationKit`.

   Ejemplo; ajusta los nombres de propiedades a tu modelo real de la práctica `03-00-01`:

   ```swift
   let mapPlaces = places.map { place in
       CampusMapPlace(
           id: place.id.uuidString,
           name: place.name,
           subtitle: place.category,
           coordinate: LocationCoordinate(
               latitude: place.latitude,
               longitude: place.longitude
           )
       )
   }
   ```

5. Si el modelo de dominio guarda coordenadas en otra estructura, usa dicha estructura para construir `LocationCoordinate`. El framework no debe importar `CampusDomain`.

**Resultado esperado:**

- El ViewModel no instancia `CLLocationManager`.
- El estado de autorización y las actualizaciones de ubicación se reciben mediante el protocolo público del framework.
- Las coordenadas del dominio se adaptan en la aplicación principal.

**Verificación:**

Comprueba visualmente que el archivo `PlacesMapViewModel.swift` depende de:

```swift
import CampusLocationKit
```

y no contiene:

```swift
CLLocationManager()
```

---

### Paso 6. Crear la pantalla `MKMapView` con anotaciones seleccionables

**Objetivo:** mostrar los lugares del campus sobre un mapa UIKit, permitir su selección y presentar la distancia al lugar seleccionado.

**Instrucciones:**

1. Crea `PlaceAnnotation.swift`:

   ```swift
   import MapKit

   final class PlaceAnnotation: MKPointAnnotation {
       let placeID: String

       init(place: CampusMapPlace) {
           self.placeID = place.id
           super.init()

           title = place.name
           subtitle = place.subtitle
           coordinate = place.coordinate.clLocationCoordinate2D
       }
   }
   ```

2. Crea `PlacesMapViewController.swift`:

   ```swift
   import CampusLocationKit
   import MapKit
   import UIKit

   @MainActor
   final class PlacesMapViewController: UIViewController {
       private let viewModel: PlacesMapViewModel

       private let mapView = MKMapView()
       private let locationButton = UIButton(type: .system)
       private let locationStateLabel = UILabel()
       private let distanceLabel = UILabel()

       init(viewModel: PlacesMapViewModel) {
           self.viewModel = viewModel
           super.init(nibName: nil, bundle: nil)
       }

       required init?(coder: NSCoder) {
           fatalError("init(coder:) has not been implemented")
       }

       override func viewDidLoad() {
           super.viewDidLoad()

           title = "Mapa del campus"
           view.backgroundColor = .systemBackground

           configureViews()
           bindViewModel()
           addPlaceAnnotations()

           viewModel.start()
       }

       private func configureViews() {
           mapView.translatesAutoresizingMaskIntoConstraints = false
           mapView.delegate = self
           mapView.showsUserLocation = true

           locationButton.translatesAutoresizingMaskIntoConstraints = false
           locationButton.configuration = .filled()
           locationButton.configuration?.title = "Usar mi ubicación"
           locationButton.addTarget(
               self,
               action: #selector(didTapLocationButton),
               for: .touchUpInside
           )

           locationStateLabel.translatesAutoresizingMaskIntoConstraints = false
           locationStateLabel.numberOfLines = 0
           locationStateLabel.font = .preferredFont(forTextStyle: .footnote)
           locationStateLabel.textColor = .secondaryLabel

           distanceLabel.translatesAutoresizingMaskIntoConstraints = false
           distanceLabel.numberOfLines = 0
           distanceLabel.font = .preferredFont(forTextStyle: .headline)
           distanceLabel.textColor = .label

           let informationStack = UIStackView(
               arrangedSubviews: [
                   locationButton,
                   locationStateLabel,
                   distanceLabel
               ]
           )
           informationStack.translatesAutoresizingMaskIntoConstraints = false
           informationStack.axis = .vertical
           informationStack.spacing = 8
           informationStack.alignment = .fill
           informationStack.isLayoutMarginsRelativeArrangement = true
           informationStack.layoutMargins = UIEdgeInsets(
               top: 12,
               left: 16,
               bottom: 12,
               right: 16
           )
           informationStack.backgroundColor = .secondarySystemBackground
           informationStack.layer.cornerRadius = 12

           view.addSubview(mapView)
           view.addSubview(informationStack)

           NSLayoutConstraint.activate([
               mapView.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor),
               mapView.leadingAnchor.constraint(equalTo: view.leadingAnchor),
               mapView.trailingAnchor.constraint(equalTo: view.trailingAnchor),
               mapView.bottomAnchor.constraint(equalTo: view.bottomAnchor),

               informationStack.leadingAnchor.constraint(
                   equalTo: view.layoutMarginsGuide.leadingAnchor
               ),
               informationStack.trailingAnchor.constraint(
                   equalTo: view.layoutMarginsGuide.trailingAnchor
               ),
               informationStack.bottomAnchor.constraint(
                   equalTo: view.safeAreaLayoutGuide.bottomAnchor,
                   constant: -12
               )
           ])
       }

       private func bindViewModel() {
           viewModel.onRegionChange = { [weak self] region in
               self?.mapView.setRegion(region, animated: true)
           }

           viewModel.onLocationStateChange = { [weak self] text in
               self?.locationStateLabel.text = text
           }

           viewModel.onDistanceChange = { [weak self] text in
               self?.distanceLabel.text = text
           }

           viewModel.onCenterUserLocation = { [weak self] coordinate in
               let region = MKCoordinateRegion(
                   center: coordinate.clLocationCoordinate2D,
                   span: MKCoordinateSpan(
                       latitudeDelta: 0.006,
                       longitudeDelta: 0.006
                   )
               )

               self?.mapView.setRegion(region, animated: true)
           }
       }

       private func addPlaceAnnotations() {
           let annotations = viewModel.places.map(PlaceAnnotation.init)
           mapView.addAnnotations(annotations)
       }

       @objc
       private func didTapLocationButton() {
           viewModel.requestUserLocation()
       }
   }

   extension PlacesMapViewController: MKMapViewDelegate {
       func mapView(
           _ mapView: MKMapView,
           viewFor annotation: any MKAnnotation
       ) -> MKAnnotationView? {
           guard annotation is PlaceAnnotation else {
               return nil
           }

           let identifier = "CampusPlaceMarker"

           let marker = mapView.dequeueReusableAnnotationView(
               withIdentifier: identifier
           ) as? MKMarkerAnnotationView
           ?? MKMarkerAnnotationView(
               annotation: annotation,
               reuseIdentifier: identifier
           )

           marker.annotation = annotation
           marker.canShowCallout = true
           marker.markerTintColor = .systemBlue
           marker.glyphImage = UIImage(
               systemName: "mappin.and.ellipse"
           )

           return marker
       }

       func mapView(
           _ mapView: MKMapView,
           didSelect view: MKAnnotationView
       ) {
           guard let annotation = view.annotation as? PlaceAnnotation else {
               return
           }

           viewModel.selectPlace(withID: annotation.placeID)
       }
   }
   ```

3. Revisa el comportamiento previsto:

   - La pantalla se centra inicialmente en la región predefinida.
   - Se agregan anotaciones correspondientes a los lugares entregados por el repositorio.
   - Al seleccionar una anotación, el mapa se centra sobre el lugar.
   - Si ya existe ubicación válida, se calcula y presenta la distancia.
   - El permiso solo se solicita tras pulsar **Usar mi ubicación**.

**Resultado esperado:**

- La pantalla muestra un mapa interactivo.
- Las anotaciones usan `MKMarkerAnnotationView`.
- La selección de un marcador actualiza el lugar seleccionado en el ViewModel.
- No se solicita permiso automáticamente al abrir la pantalla.

**Verificación:**

Compila desde Xcode. Debe desaparecer cualquier error de tipos relacionado con `MKAnnotation`, `LocationCoordinate` o `CampusLocationKit`.

---

### Paso 7. Conectar la pantalla con `PlacesCoordinator` y validar la ubicación simulada

**Objetivo:** incorporar el flujo de navegación desde la funcionalidad de lugares existente y validar permisos y coordenadas en el simulador.

**Instrucciones:**

1. Localiza el `PlacesCoordinator` creado en la práctica anterior.

2. Añade una función de navegación para el mapa. Ajusta el tipo `[Place]` al modelo real que devuelve tu repositorio:

   ```swift
   import CampusLocationKit

   func showMap(places: [Place]) {
       let mapPlaces = places.map { place in
           CampusMapPlace(
               id: place.id.uuidString,
               name: place.name,
               subtitle: place.category,
               coordinate: LocationCoordinate(
                   latitude: place.latitude,
                   longitude: place.longitude
               )
           )
       }

       let locationManager: any LocationManaging = LocationManager()

       let viewModel = PlacesMapViewModel(
           places: mapPlaces,
           locationManager: locationManager
       )

       let viewController = PlacesMapViewController(
           viewModel: viewModel
       )

       navigationController.pushViewController(
           viewController,
           animated: true
       )
   }
   ```

3. Desde la pantalla de lista de lugares, agrega una acción de navegación. Puede ser un botón de barra de navegación llamado **Mapa**.

   Ejemplo orientativo en un `UIViewController`:

   ```swift
   navigationItem.rightBarButtonItem = UIBarButtonItem(
       title: "Mapa",
       style: .plain,
       target: self,
       action: #selector(didTapMap)
   )
   ```

4. En la acción del botón, usa el coordinator existente. El View Controller no debe construir directamente el mapa ni el `LocationManager`:

   ```swift
   @objc
   private func didTapMap() {
       coordinator?.showMap(places: viewModel.places)
   }
   ```

5. Ejecuta la aplicación en el simulador `iPhone 16`.

6. Abre la lista de lugares y pulsa **Mapa**.

7. Confirma que se muestra la región predefinida y las anotaciones del campus.

8. Pulsa **Usar mi ubicación**.

9. Cuando el simulador muestre el diálogo del sistema, selecciona **Permitir al usar la app**.

10. En la barra de menú del simulador, selecciona:

    ```text
    Features > Location > Custom Location…
    ```

11. Introduce una coordenada próxima a los lugares de prueba. Puedes usar, por ejemplo:

    | Campo | Valor |
    |---|---:|
    | Latitude | `40.4168` |
    | Longitude | `-3.7038` |

12. Selecciona un marcador del mapa y comprueba el texto de distancia.

13. Para validar el flujo denegado, restablece los permisos desde el simulador:

    ```text
    Settings > Developer > Reset Location & Privacy
    ```

    También puedes eliminar la aplicación y volver a instalarla.

14. Ejecuta de nuevo el flujo y, esta vez, selecciona **No permitir**. La aplicación debe mantener la región predeterminada y mostrar un mensaje comprensible.

**Resultado esperado:**

- El botón de mapa usa el `PlacesCoordinator`.
- El mapa recibe lugares desde el repositorio previamente cargado.
- La autorización se solicita tras una acción explícita del usuario.
- Con autorización, el mapa se centra cerca de la ubicación simulada.
- Al seleccionar un punto de interés, se presenta una distancia aproximada.
- Sin autorización, la aplicación sigue siendo usable y muestra el campus.

**Verificación:**

Comprueba los siguientes casos funcionales:

| Caso | Comportamiento esperado |
|---|---|
| Primera apertura | El mapa muestra la región predeterminada; no aparece diálogo de permiso. |
| Pulsación de “Usar mi ubicación” | Aparece el diálogo de permiso del sistema. |
| Permiso autorizado | Se obtiene ubicación simulada y el mapa se centra en ella. |
| Marcador seleccionado | Se centra el lugar y se muestra una distancia aproximada. |
| Permiso denegado | Se conserva la región del campus y se explica la limitación. |
| Permiso restringido | Se informa que el sistema restringe la ubicación. |

## Validación y pruebas

Ejecuta las pruebas unitarias del framework:

```bash
cd ~/Developer/CampusExplorer/Packages/CampusLocationKit
swift test
```

Ejecuta las pruebas de la aplicación en el simulador obligatorio:

```bash
cd ~/Developer/CampusExplorer

xcodebuild test \
  -project CampusExplorer.xcodeproj \
  -scheme CampusExplorer \
  -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5'
```

Realiza una compilación limpia final:

```bash
xcodebuild clean build \
  -project CampusExplorer.xcodeproj \
  -scheme CampusExplorer \
  -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.5'
```

Valida además la arquitectura mediante esta lista:

- [ ] `CampusLocationKit` contiene `LocationManager`, protocolos públicos, estados de autorización y coordenadas.
- [ ] El target principal importa `CampusLocationKit`.
- [ ] El target principal no crea instancias directas de `CLLocationManager`.
- [ ] La dependencia se inyecta como `any LocationManaging` en `PlacesMapViewModel`.
- [ ] `PlacesCoordinator` crea el flujo de mapa y realiza el `pushViewController`.
- [ ] La clave `NSLocationWhenInUseUsageDescription` tiene un mensaje concreto y comprensible.
- [ ] La pantalla funciona tanto con permiso concedido como denegado.
- [ ] El mapa no intenta calcular miles de anotaciones ni rutas automáticamente.
- [ ] La distancia se calcula solamente cuando existe una ubicación y un lugar seleccionado.

Guarda el trabajo en Git:

```bash
cd ~/Developer/CampusExplorer

git status
git add \
  CampusExplorer.xcodeproj \
  Packages/CampusLocationKit \
  .

git commit -m "feat: add campus location kit and places map"
git tag lab-04-complete
git status
```

> Revisa cuidadosamente los archivos incluidos antes de ejecutar `git add .` si el repositorio contiene archivos locales, datos derivados o configuraciones personales.

## Resolución de problemas

### 1. La aplicación termina al solicitar ubicación o no aparece el diálogo de permiso

**Síntoma:** al pulsar **Usar mi ubicación**, la aplicación se cierra, aparece un mensaje relacionado con privacidad o no se presenta el diálogo de autorización.

**Causa:** falta la clave `NSLocationWhenInUseUsageDescription`, está vacía, se añadió al target incorrecto o el simulador conserva una decisión de permiso anterior.

**Solución:**

1. Confirma en el target `CampusExplorer`, pestaña **Info**, que existe la clave:
   ```text
   Privacy - Location When In Use Usage Description
   ```
2. Verifica que contiene un texto no vacío y específico.
3. Comprueba que pertenece al target de aplicación, no al target de pruebas ni al paquete.
4. Restablece el estado de privacidad en el simulador desde:
   ```text
   Settings > Developer > Reset Location & Privacy
   ```
5. Elimina e instala de nuevo la aplicación si el diálogo continúa sin aparecer.

### 2. El mapa se muestra, pero nunca calcula distancia ni centra la ubicación simulada

**Síntoma:** las anotaciones aparecen, pero el texto permanece en “Esperando una posición válida” o no se actualiza al elegir una ubicación personalizada.

**Causa:** no se concedió permiso, no se seleccionó una ubicación simulada en el menú del simulador, `LocationManager.delegate` no apunta al ViewModel o no se llamó a `startUpdatingLocation()` después de autorizar.

**Solución:**

1. Confirma que el permiso de la aplicación es **Permitir al usar la app** en los ajustes del simulador.
2. Selecciona una ubicación mediante:
   ```text
   Features > Location > Custom Location…
   ```
3. Verifica que `PlacesMapViewModel` asigna el delegado:
   ```swift
   self.locationManager.delegate = self
   ```
4. Confirma que, al recibir `.authorizedWhenInUse`, se ejecuta:
   ```swift
   manager.startUpdatingLocation()
   ```
5. Añade temporalmente un punto de interrupción en `didUpdateLocations` de `LocationManager` para comprobar que el simulador entrega coordenadas.

## Limpieza

1. Detén la aplicación en Xcode.
2. Cierra el simulador si no lo vas a utilizar:

   ```bash
   xcrun simctl shutdown "iPhone 16"
   ```

3. Conserva el paquete local en:

   ```text
   ~/Developer/CampusExplorer/Packages/CampusLocationKit
   ```

4. No elimines los paquetes obligatorios existentes:

   ```text
   CampusDomain
   CampusNetworking
   CampusDesignSystem
   ```

5. Comprueba que no quedan cambios sin confirmar:

   ```bash
   cd ~/Developer/CampusExplorer
   git status
   ```

6. Si necesitas volver a la rama principal tras entregar la práctica:

   ```bash
   git checkout main
   ```

## Resumen

En esta práctica has integrado un mapa UIKit mediante `MKMapView`, anotaciones seleccionables y una región inicial segura para escenarios sin autorización. También has creado `CampusLocationKit`, un framework local reutilizable que encapsula los detalles de `Core Location`, expone una interfaz basada en protocolos y permite inyección de dependencias.

La aplicación principal conserva la responsabilidad de presentación, adaptación del modelo de dominio y navegación mediante `PlacesCoordinator`, mientras que el framework administra los permisos, actualizaciones de posición y cálculos de distancia. Esta separación reduce el acoplamiento, facilita las pruebas unitarias y prepara el proyecto para futuras funcionalidades, como rutas con `MKDirections`, búsquedas con `MKLocalSearch` o agrupación de anotaciones.

### Recursos opcionales

- [Apple Developer Documentation: MapKit](https://developer.apple.com/documentation/mapkit)
- [Apple Developer Documentation: Core Location](https://developer.apple.com/documentation/corelocation)
- [Apple Developer Documentation: Requesting authorization for location services](https://developer.apple.com/documentation/corelocation/requesting-authorization-for-location-services)
- [Apple Developer Documentation: MKMapView](https://developer.apple.com/documentation/mapkit/mkmapview)
