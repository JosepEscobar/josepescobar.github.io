---
title: "Navegación Modular en SwiftUI - Un enfoque basado en Enum"
author: "Josep Escobar"
date: 2023-09-20 13:45:00 +0800
categories: [Desarrollo de Software, Programación, Lenguajes de Programación]
tags: [SwiftUI, Programación, Desarrollo de Software, Arquitectura de software]
pin: true
math: true
mermaid: true
---

# Navegación Modular en SwiftUI - Un enfoque basado en Enum
| Utilizando el nuevo NavigationStack con el poder de las enumeraciones de manera modular y expansible.

## Introducción
En iOS 16, SwiftUI finalmente nos permite encapsular adecuadamente la navegación sin verse obligados a pasar una tonelada de vinculaciones o utilizar NavigationLink directamente en la interfaz de usuario.

Sin embargo, una cosa que siempre introduce desafíos adicionales es la modularización. En el mejor de los casos, cada módulo funciona de manera independiente y no tiene dependencias en otros módulos (en la misma capa).

Aunque es posible lograrlo, siempre existe la necesidad de navegar de un módulo a otro. Un enfoque que resuelve elegantemente este problema es el uso de enumeraciones para la navegación interna y externa.

## Concepto
La aplicación (o partes de ella) se agrupan dentro de un único router. El punto de entrada (App/Root View/TabViews) crea y mantiene NavigationPath y lo pasa a un NavigationStack. Dado que NavigationPath es un tipo de referencia, podemos crear nuestro router principal y pasarle ese mismo camino. Ahora, todo lo que el router principal necesita son algunas operaciones para agregar y quitar vistas.

En cuanto a los módulos, cada módulo contiene un router personalizado que maneja la navegación interna del módulo. Estos routers obtienen una referencia al router principal, lo que les permite modificar el NavigationPath inicial llamando a los métodos del router principal.

Para modificar la pila de vistas, cada módulo tiene una enumeración que contiene todas las rutas posibles. Esa enumeración es procesada por el modificador de vista view.navigationDestination().

Cada módulo administra la navegación a través de una enumeración interna que se asigna mediante el modificador de vista view.navigationDestination(). Dado que el router de cada módulo sigue llamando al router principal, el NavigationPath se altera, sin que los routers tengan una referencia real al camino en sí.

La última parte complicada es hacer que los módulos naveguen entre sí. En realidad, nunca hacen eso. En su lugar, llaman a los mismos métodos del router principal como si estuvieran navegando internamente, pero esta vez pasan otro valor de enumeración. Esta enumeración maneja los puntos de salida para el módulo. Técnicamente, podría haber múltiples salidas, todas con diferentes valores asociados, para hacer distinciones y transferencias de datos entre los módulos de manera sencilla.

La aplicación real asigna cada enumeración de salida para cada módulo mediante view.navigationDestination(). Y, dado que la navegación ocurre a nivel de aplicación esta vez, ningún módulo sabe a qué módulo se está navegando, lo que hace que los módulos sean completamente independientes entre sí.

## Implementación
Ahora que se ha esbozado la idea general, pasemos a la implementación real.

### Nivel de Aplicación
Lo primero que se debe hacer es crear un router global con el que todos los módulos interactúen.

#### App Router
```swift
public class AppRouter: ObservableObject {
    // Restricción: Debe ser privado para que el NavigationStack pueda vincularse a él
    @Published var path: NavigationPath

    init(with path: NavigationPath) {
        self.path = path
    }

    func navigate(to destination: any Hashable) {
        path.append(destination)
    }

    func pop() {
        path.removeLast()
    }
}
```

> NOTA: Como se mencionó anteriormente, el router de entrada principal no necesariamente tiene que ser la propia aplicación, podría ser varios routers para cada pestaña en una vista de pestañas u algo similar. El concepto sigue siendo el mismo.

### Nivel de Módulo
Ahora que se ha definido el router principal, podemos pasar a los módulos. Como se mencionó, cada módulo tiene su propio router, pero también tiene sentido tener algún tipo base para el router del módulo.

#### Protocolo del Router del Módulo
```swift
protocol ModuleRouter {
    var appRouter: AppRouter { get }
}
```

#### Implementación de Ejemplo del Router del Módulo
```swift
// Enum para rutas internas
enum DashboardRoute: Hashable {
    case details
}

// Enum para rutas externas (salidas de este módulo)
enum DashboardExit: Hashable {
    case logout
    case settings
}

// El router del módulo
class DashboardRouter: ModuleRouter {
    var appRouter: AppRouter

    init(with appRouter: AppRouter) {
        self.appRouter = appRouter
    }

    func navigate(to target: DashboardRoute) {
        appRouter.navigate(to: target)
    }

    func pop() {
        appRouter.pop()
    }
}
```

Para que nuestra aplicación principal no tome decisiones con respecto a la navegación de nuestros módulos, definimos una extensión de View que asigna la enumeración de rutas del módulo a las vistas específicas. Este modificador puede ser llamado fácilmente desde la aplicación principal, sin conocer ningún detalle sobre la navegación y las vistas reales.

#### Extensión de Vista
```swift
// Modificador de vista personalizado para la navegación de este módulo
public extension View {
    public func withDashboardRoutes() -> some View {
        self.navigationDestination(for: DashboardRoute.self) { destination in
            switch destination {
                // Maneja la lógica de navegación para cada ruta de este módulo
                case .details:
                    DetailView()
                ...
            }
        }
    }
}
```

Ahora que tenemos los módulos configurados, podemos vincular su navegación en nuestra aplicación real.

```swift
import Dashboard
import OtherModule
import EvenAnotherModule
@main
struct Application: App {
    @ObservedObject var router: AppRouter
    
    init() {
        // En una aplicación real, esto debería provenir de un contenedor de inyección de dependencias (DI).
        self.router = AppRouter()
    }
    
    var body: some Scene {
        WindowGroup {
            NavigationStack(path: $router.path) {
                // Nuevamente, en una aplicación real, esto debería provenir de un contenedor de inyección de dependencias (DI).
                DashboardView(router: DashboardRouter(with: router)) 
                    .withDashboardRoutes()
                    .withOtherModuleRoutes()
                    .withEvenAnotherModuleRoutes()
            }
        }
    }
}
```

> ADVERTENCIA: No olvides aplicar tus modificadores de ruta en tu aplicación principal.

## Navegación entre Módulos
Ahora que los módulos pueden navegar internamente y toda la navegación está vinculada a la aplicación, lo único que queda es la navegación entre módulos. Para eso, simplemente agregamos destinos de navegación adicionales para cada
