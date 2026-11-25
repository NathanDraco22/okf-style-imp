---
type: StyleGuide
title: Estructura del Proyecto Flutter
description: Organización de carpetas en proyectos Flutter con arquitectura Onion, business package separado, composition root en main, cubits globales en GetIt y módulos con variantes mobile/tablet/desktop.
tags: [flutter, dart, estilo, estructura, proyecto, responsive, get_it]
---

# Estructura del Proyecto Flutter

## Árbol estándar

```text
lib/
├── main.dart                         # Entry point + AppDependencies.create()
├── app_dependencies.dart             # Composition root: datasources, repos, global cubits
├── provider_container.dart           # MultiRepositoryProvider (solo repositorios)
├── config/
│   ├── app_env.dart                  # Variables de entorno (SERVER_URL, etc.)
│   ├── app_router.dart               # GoRouter + StatefulShellRoute
│   ├── app_theme.dart                # Tema Material 3
│   ├── get_it_config.dart            # GetIt: servicios (no cubits)
│   └── scroll_behavior.dart
├── constants/
│   ├── route_names.dart              # Constantes de rutas
│   └── shortcut_registry.dart        # Atajos de teclado
├── cubits/                           # Cubits globales (GetIt)
│   ├── auth/                         # AuthCubit / AppSessionCubit
│   ├── app_settings/                 # Preferencias globales
│   └── toast/                        # Notificaciones globales
│
├── src/
│   ├── core/                         # Managers singleton (sesión, config, acceso)
│   │   ├── session_manager.dart
│   │   ├── app_config_manager.dart
│   │   └── access_manager.dart
│   │
│   ├── cubits/                       # Cubits NO globales (provider)
│   │   ├── products/                 # read_products, write_products
│   │   └── orders/                   # read_orders, write_orders
│   │
│   ├── modules/                      # Módulos de funcionalidad
│   │   └── {feature}/
│   │       ├── {feature}_mediator.dart # InheritedWidget compartido entre variantes
│   │       ├── view_controller.dart   # ChangeNotifier: estado efímero de la pantalla
│   │       ├── view/
│   │       │   ├── {feature}_screen.dart  # Entry point: MultiBlocProvider + dispatch
│   │       │   ├── mobile/
│   │       │   │   └── {feature}_screen_mobile.dart
│   │       │   ├── tablet/
│   │       │   │   └── {feature}_screen_tablet.dart
│   │       │   ├── desktop/
│   │       │   │   ├── {feature}_screen_desktop.dart
│   │       │   │   ├── sections/
│   │       │   │   └── widgets/
│   │       │   └── widgets/           # Compartidos entre variantes
│   │       ├── dialogs/               # Show functions tipadas
│   │       └── widgets/               # Widgets compartidos del módulo
│   │
│   └── tools/                        # Utilidades
│       ├── extensiones.dart          # context.isMobile/isTablet/isDesktop
│       ├── navigation_tool.dart
│       └── exports/
│
└── widgets/                          # Widgets globales reutilizables
    ├── buttons/
    ├── dialogs/                      # Dialogs/modals con función show
    │   ├── dialog_manager.dart
    │   ├── loading_dialog_manager.dart
    │   ├── viewers/
    │   ├── inspectors/
    │   ├── editors/
    │   ├── search_products/
    │   ├── selectors/
    │   └── models/
    ├── modals/
    ├── tables/
    └── gates/                        # RolePermissionGate, ExpirationGate
```

## Paquete de negocio separado

```text
packages/{app}_business/
├── pubspec.yaml                      # Pure Dart (sin Flutter)
└── lib/
    ├── {app}_business.dart           # Barrel file
    ├── {app}_client.dart             # Singleton config holder
    ├── {app}_client_config.dart      # Abstract config interface
    ├── constants/
    │   └── default_values.dart
    └── src/
        ├── data/                     # DataSources (singleton + HttpService)
        │   ├── data_sources.dart     # Barrel
        │   └── item_data_source.dart
        ├── domain/
        │   ├── models/
        │   │   └── item/
        │   │       └── item_model.dart
        │   ├── repositories/
        │   │   ├── repositories.dart # Barrel
        │   │   └── items_repository.dart
        │   ├── responses/            # ListResponse<T>, etc.
        │   └── query_params/         # Filtros tipados
        ├── services/                 # Infraestructura
        │   ├── http_service.dart     # Mixin
        │   ├── hive_service.dart     # Mixin
        │   └── exceptions/
        │       └── http_exceptions.dart
        └── tools/                    # Utilidades
            ├── constant.dart         # maxPhoneScreenWidth = 600, maxTabletScreenWidth = 840
            ├── http_tool.dart        # generateUri, generateAuthHeaders
            ├── token_manager.dart
            └── reactive_repo/
                └── reactive_repository.dart
```

## Dónde vive cada cubit

| Alcance | Ubicación | Mecanismo | Consumo |
|---|---|---|---|
| Global | `lib/cubits/{feature}/` | GetIt, creado en `app_dependencies.dart` | `bloc: getIt<T>()`, `getIt<T>()` |
| No global (pantalla) | `lib/src/cubits/{feature}/` | `MultiBlocProvider` en la screen | `context.watch/read` |
| No global (compartido) | `lib/src/cubits/{feature}/` | Provider en el ancestro común | `context.watch/read` |
| Diálogo | `lib/src/cubits/{feature}/` | `BlocProvider.value` o creación en `showXxx` | `context.watch/read` |
| Estado efímero | `modules/{feature}/view_controller.dart` | `ChangeNotifier` + Mediator | `Mediator.of(context)` |

Detalle en [Alcance y Ciclo de Vida de Cubits](../patterns/cubit_scopes.md).

## Responsive

- Los cortes viven en el paquete de negocio: `maxPhoneScreenWidth = 600`,
  `maxTabletScreenWidth = 840`.
- Extensiones de contexto en `lib/src/tools/extensiones.dart`.
- Cada pantalla que cambia de layout tiene `view/mobile/`, `view/tablet/` y
  `view/desktop/`, declaradas como `part` del entry point.
- Los providers van arriba del dispatch responsive.

Detalle en [Diseño Responsive](../patterns/responsive_design.md).

## Reglas

- **Regla #1 — Onion CLI:** toda estructura nueva se genera con `onion`
  (`onion dart`, `onion dart-cubit`, `onion flutter-module`, `onion barrel`,
  ...), nunca manualmente. Ver
  [Comandos Onion CLI Flutter](../patterns/cli_commands_flutter.md).
- Los DataSources, Repositorios y Models van en el **business package**
  (pure Dart).
- Las Screens, Cubits y Widgets van en `lib/` del app (Flutter).
- **Cubits globales** en `lib/cubits/{feature}/` + GetIt; **todos los no
  globales** en `lib/src/cubits/{feature}/` con provider, en carpetas plurales
  (`products/`, `orders/`). No existe `modules/{feature}/cubit/`.
- La **screen** (`modules/{feature}/view/{feature}_screen.dart`) es el único
  lugar que instancia los cubits no globales.
- El **ViewController** (`modules/{feature}/view_controller.dart`) maneja el
  estado efímero de la pantalla o de un componente reusable complejo; no es un
  cubit y no va en `src/cubits/`.
- `app_dependencies.dart` es el único composition root; `provider_container`
  solo provee repositorios.
- `get_it_config.dart` registra servicios, no cubits.
- El **mediator** es compartido en la raíz del módulo
  (`{feature}_mediator.dart`); solo se duplica por variante cuando una necesita
  exponer más datos que el ViewController.
- Las secciones desktop usan `part files` para acceder a miembros privados.
- Los barrel files (`data_sources.dart`, `repositories.dart`, `models.dart`)
  son generados por el CLI.
- El estado no se duplica entre variantes: mismos cubits, mismo controller,
  mismo repositorio.
