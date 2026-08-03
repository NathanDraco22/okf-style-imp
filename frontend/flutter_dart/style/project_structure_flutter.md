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
│   ├── cubits/                       # Cubits compartidos entre módulos
│   │   └── transfers/                # read_transfers, write_transfer cubits
│   │
│   ├── modules/                      # Módulos de funcionalidad
│   │   └── {feature}/
│   │       ├── {feature}_screen.dart  # Entry point (MultiBlocProvider)
│   │       ├── {feature}_mediator.dart # InheritedWidget compartido web/mobile
│   │       ├── view_controller.dart   # ChangeNotifier (estado local de pantalla)
│   │       ├── cubit/                 # Solo si el feature lo amerita
│   │       │   ├── read_{entity}_cubit.dart
│   │       │   ├── read_{entity}_state.dart
│   │       │   ├── write_{entity}_cubit.dart
│   │       │   └── write_{entity}_state.dart
│   │       ├── mobile/
│   │       │   ├── {feature}_screen_mobile.dart
│   │       │   └── widgets/
│   │       ├── web/
│   │       │   ├── {feature}_screen_web.dart
│   │       │   ├── sections/
│   │       │   │   ├── header_section.dart
│   │       │   │   └── content_section.dart
│   │       │   └── widgets/
│   │       ├── tablet/                # (futuro, mismo patrón que mobile/)
│   │       ├── modals/               # Bottom sheets específicos del feature
│   │       └── widgets/              # Widgets compartidos web/mobile
│   │
│   └── tools/                        # Utilidades
│       ├── extensiones.dart          # context.isMobile(), etc.
│       ├── navigation_tool.dart
│       └── exports/
│
└── widgets/                          # Widgets globales reutilizables
    ├── buttons/
    ├── dialogs/                      # Dialogs/modals con función show (ver Patrón Dialog/Modal)
    │   ├── dialog_manager.dart       # Excepción: info/error/confirmar
    │   ├── loading_dialog_manager.dart # Excepción: loading global
    │   ├── viewers/                  # show{Entity}Viewer — solo lectura
    │   ├── inspectors/               # show{Entity}Inspector — análisis detallado
    │   ├── editors/                  # show{Entity}Editor — crear/editar
    │   ├── search_products/          # show{Entity}Search — buscadores
    │   ├── selectors/                # show{Entity}Selector — selección de entidad
    │   └── models/                   # {entity}_result.dart — modelos de resultado
    ├── modals/                       # Bottom sheets genéricos
    │   └── {action}_bottom_modal.dart
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

- **Regla #1 — Onion CLI:** toda estructura nueva se genera con `onion` (`onion dart`, `onion dart-cubit`, `onion flutter-module`, `onion barrel`, ...), nunca manualmente. Ver [Comandos Onion CLI Flutter](../patterns/cli_commands_flutter.md)
- Los DataSources, Repositorios y Models van en el **business package** (pure Dart)
- Las Screens, Cubits y Widgets van en `lib/` del app (Flutter)
- **Cubits globales** en `lib/cubits/` (auth, sesión) — **compartidos** en `lib/src/cubits/` — de feature solo si lo amerita
- El **mediator** es compartido en la raíz del módulo (`{feature}_mediator.dart`)
- Los módulos con versión mobile/web tienen `mobile/` y `web/` subdirectorios
- Las secciones de web usan `part files` para acceder a miembros privados
- Los barrel files (`data_sources.dart`, `repositories.dart`, `models.dart`) son generados por el CLI
