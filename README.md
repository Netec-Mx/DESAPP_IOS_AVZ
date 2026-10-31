# Desarrollo de aplicaciones móviles para Sistemas Operativos iOS Nivel Avanzado

Este curso avanzado está dirigido a desarrolladores iOS que ya dominan Swift y UIKit, y que desean construir aplicaciones modernas, escalables y mantenibles. A lo largo de seis capítulos, el alumno desarrolla de forma incremental la aplicación **CampusExplorer** con Swift 6.1, Xcode 16.4 e iOS 18.

El curso cubre arquitecturas modulares con MVVM, Coordinator, Swift Package Manager e inyección de dependencias; programación reactiva con Combine y consumo de servicios de red; concurrencia avanzada con `async/await`, actores y análisis de rendimiento con Instruments; mapas y ubicación con MapKit y Core Location, junto con la creación de frameworks propios; pruebas de ViewModels, snapshots e integración continua con GitHub Actions; y modularización con VIPER, componentes, flujo de datos y navegación con coordinators.

Como complemento, algunos capítulos comparan los conceptos de Swift con Kotlin Multiplatform para homologar arquitectura, concurrencia y código compartido entre plataformas.

## Estructura

- `CapituloXX/README.md`: guía de laboratorio por capítulo.
- Cada práctica parte del resultado de la anterior. Al terminar, se crea la etiqueta Git `lab-XX-complete`.

## Entorno común

| Componente | Versión |
|---|---|
| macOS | Sequoia 15.5 o compatible |
| Xcode | 16.4 |
| Swift | 6.1 |
| SDK de compilación | iOS 18.5 |
| Deployment target | iOS 18.0 |
| Simulador | iPhone 16, iOS 18.5 |
| Directorio de trabajo | `~/Developer/CampusExplorer` |

## Lista de laboratorios

### Capítulo 1. Arquitecturas Modulares e Inyección de Dependencias

- [5 Práctica incremental](Capitulo01/README.md#5-práctica-incremental)
  - Descripción: Crear la línea base de CampusExplorer aplicando arquitectura MVVM, Coordinator Pattern, modularización con Swift Package Manager (`CampusDomain`, `CampusNetworking`, `CampusDesignSystem`) e inyección de dependencias.
  - Duración estimada: 216 min

### Capítulo 2. Programación Reactiva con Combine

- [4 Práctica incremental](Capitulo02/README.md#4-práctica-incremental)
  - Descripción: Consumir una API HTTP con `URLSession.DataTaskPublisher` y aplicar operadores de Combine, networking reactivo y gestión de estado en una búsqueda con espera, deduplicación y cancelación.
  - Duración estimada: 252 min

### Capítulo 3. Concurrencia Avanzada y Performance

- [4 Práctica incremental](Capitulo03/README.md#4-práctica-incremental)
  - Descripción: Migrar la carga de datos de Combine a `async/await`, aplicar concurrencia estructurada y el actor `CampusCache`, mover operaciones costosas fuera del hilo principal y documentar la mejora con Instruments.
  - Duración estimada: 288 min

### Capítulo 4. Frameworks: MapKit, Core Location y frameworks propios

- [4 Práctica incremental](Capitulo04/README.md#4-práctica-incremental)
  - Descripción: Mostrar los lugares del campus en un mapa con MapKit, gestionar permisos y distancias con Core Location y encapsular la ubicación en el framework local `CampusLocationKit`.
  - Duración estimada: 252 min

### Capítulo 5. Testing Avanzado, CI y Ambientes de Desarrollo

- [4 Práctica incremental](Capitulo05/README.md#4-práctica-incremental)
  - Descripción: Probar ViewModels con Combine y `XCTestExpectation`, registrar pruebas de snapshot y configurar integración continua con GitHub Actions.
  - Duración estimada: 252 min

### Capítulo 6. Modularización y VIPER

- [5 Práctica Extra](Capitulo06/README.md#5-práctica-extra)
  - Descripción: Refactorizar una lista y un detalle de tareas hacia módulos VIPER con View, Interactor, Presenter, Entity, Router y Builder, aplicando flujo de datos, dependencias, routing y coordinators.
  - Duración estimada: 180 min

## Flujo de colaboración

- Trabajar en `changes_course`.
- Crear Pull Request hacia `main`.
- Merge por `Squash and merge`.