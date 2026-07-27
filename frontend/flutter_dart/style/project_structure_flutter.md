---
type: StyleGuide
title: Estructura del Proyecto Flutter
description: Organización de carpetas en proyectos Flutter con arquitectura Onion, business package separado y módulos con mobile/web split.
tags: [flutter, dart, estilo, estructura, proyecto]
---

# Estructura del Proyecto Flutter

## Árbol estándar

```
lib/
├── main.dart                         # Entry point
├── provider_container.dart           # DI: MultiRepositoryProvider + MultiBlocProvider
├── config/
│   ├── app_env.dart                  # Variables de entorno (SERVER_URL, etc.)
│   ├── app_router.dart               # GoRouter + StatefulShellRoute
│   ├── app_theme.dart                # Tema Material 3
│   ├── get_it_config.dart            # GetIt para cubits globales
│   └── scroll_behavior.dart
├── constants/
│   ├── route_names.dart              # Constantes de rutas
│   └── shortcut_registry.dart        # Atajos de teclado
├── cubits/                           # Cubits globales de app
│   ├── auth/                         # AuthCubit (sesión, login, logout)
│   ├── app_mode/                     # Modo normal / práctica
│   └── toast/                        # Notificaciones toast
│
├── src/
│   ├── core/                         # Managers singleton (sesión, config, acceso)
│   │   ├── session_manager.dart
│   │   ├── app_config_manager.dart
│   │   └── access_manager.dart
│   │
│   ├── cubits/                       # Cubits por feature (si están fuera de modules)
│   │   └── transfers/                # read_transfers, write_transfer cubits
│   │
│   ├── modules/                      # Módulos de funcionalidad
│   │   └── {feature}/
│   │       ├── {feature}_screen.dart  # Entry point (MultiBlocProvider)
│   │       ├── view_controller.dart   # ChangeNotifier (form state)
│   │       ├── cubit/
│   │       │   ├── read_{entity}_cubit.dart
│   │       │   ├── read_{entity}_state.dart
│   │       │   ├── write_{entity}_cubit.dart
│   │       │   └── write_{entity}_state.dart
│   │       ├── mobile/
│   │       │   ├── mediator.dart
│   │       │   ├── {feature}_screen_mobile.dart
│   │       │   └── widgets/
│   │       ├── web/
│   │       │   ├── mediator.dart
│   │       │   ├── {feature}_screen_web.dart
│   │       │   ├── sections/
│   │       │   │   ├── header_section.dart
│   │       │   │   └── content_section.dart
│   │       │   └── widgets/
│   │       └── widgets/              # Widgets compartidos mobile/web
│   │
│   └── tools/                        # Utilidades
│       ├── extensiones.dart          # context.isMobile(), etc.
│       ├── navigation_tool.dart
│       ├── loading_dialog.dart
│       └── exports/
│
└── widgets/                          # Widgets globales reutilizables
    ├── buttons/
    ├── dialogs/
    ├── tables/
    └── gates/                        # RolePermissionGate, ExpirationGate
```

## Paquete de negocio separado

```
packages/kardex_business/
├── pubspec.yaml                      # Pure Dart (sin Flutter)
└── lib/
    ├── kardex_business.dart          # Barrel file
    ├── kardex_client.dart            # Singleton config holder
    ├── kardex_client_config.dart     # Abstract config interface
    ├── constants/
    │   └── default_values.dart
    └── src/
        ├── data/                     # DataSources (singleton + HttpService)
        │   ├── data_sources.dart     # Barrel
        │   ├── patient_data_source.dart
        │   └── ...
        ├── domain/
        │   ├── models/               # Modelos por entidad
        │   │   ├── patient/
        │   │   │   └── patient_model.dart
        │   │   └── ...
        │   ├── repositories/         # Repositorios
        │   │   ├── repositories.dart # Barrel
        │   │   └── patient_repository.dart
        │   ├── responses/            # ListResponse<T>, etc.
        │   └── query_params/         # Filtros tipados
        ├── services/                 # Infraestructura
        │   ├── http_service.dart     # Mixin
        │   ├── hive_service.dart     # Mixin
        │   └── exceptions/
        │       └── http_exceptions.dart
        └── tools/                    # Utilidades
            ├── http_tool.dart        # generateUri, generateAuthHeaders
            ├── token_manager.dart
            └── reactive_repo/
                └── reactive_repository.dart
```

## Reglas

- Los DataSources, Repositorios y Models van en el **business package** (pure Dart)
- Las Screens, Cubits y Widgets van en `lib/` del app (Flutter)
- Los módulos con versión mobile/web tienen `mobile/` y `web/` subdirectorios
- Las secciones de web usan `part files` para acceder a miembros privados
- Los barrel files (`data_sources.dart`, `repositories.dart`, `models.dart`) son generados por el CLI
