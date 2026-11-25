---
type: Reference
title: Comandos Onion CLI para Flutter/Dart
description: Referencia de comandos del CLI onion para generar código Flutter/Dart en la arquitectura Onion.
tags: [cli, onion, codegen, flutter, dart, comandos]
---

# Comandos Onion CLI para Flutter/Dart

> **Versión mínima: onion ≥ 0.8.0** — genera la estructura nueva
> (`src/cubits/{plural}`, `view/{plural}_screen.dart`), el flag `--responsive`
> y los imports de business package con `--business-package`.

## Capa completa (DataSource + Repository + Models)

```bash
# Crea datasource, repository y modelos para la entidad
onion dart product

# Con --output-dir para especificar carpeta
onion dart product --output-dir lib/src/features
```

## Reactive Repository

`onion dart`, `onion dart-cubit` y `onion flutter-module` preguntan **¿Generar con ReactiveRepository? [y/N]** (default: **No**). El flag `--reactive` / `--no-reactive` salta la pregunta (útil en scripts/CI).

```bash
# Pregunta interactiva (default No)
onion dart product

# Reactive explícito: genera repo con mixin + cubits suscritos
onion dart product --reactive
onion dart-cubit product --reactive
onion flutter-module product --reactive

# Sin prompt en CI
onion dart product --no-reactive
```

Con `--reactive` el CLI genera:

- **`lib/src/tools/reactive_repo/reactive_repository.dart`** — mixin `ReactiveRepository<T>` + eventos `RepoEvent` sellados (idempotente: solo si no existe, desde cualquier comando reactivo)
- **Repository** — `with ReactiveRepository<XInDb>`, **sin cache en memoria**, `notifyItemCreated/Updated/Deleted` tras cada operación exitosa
- **ReadCubit** — repository inyectado, suscripción `eventStream` en el constructor, `_handleRepoEvent`, `cancel()` en `close()`
- **WriteCubit** — repository inyectado, `create`/`update`/`delete` reales con reset a `Initial` después de cada éxito

Sin `--reactive` se mantiene el modo clásico (cache en el repository + `markXxx` manual desde la UI/BlocListener).

> **Regla:** usa reactive cuando quieras instanciar WriteCubits efímeros (p. ej. uno por diálogo) sin perder la comunicación con el ReadCubit, o cuando múltiples ReadCubits deban escuchar el mismo repository. Ver [ReactiveRepository Mixin](reactive_repository.md) para los criterios completos.

## Modelos (Create, Update, InDb)

```bash
# Genera Base, Create, Update, InDb con fromJson/toJson
onion dart-model product
```

## Cubits (Read + Write separados)

```bash
# Genera ReadCubit + WriteCubit en lib/src/cubits/products/
onion dart-cubit product

# Business package separado: imports package:{name}/{name}.dart
onion dart-cubit product --business-package catalog_business

# Solo cubit de lectura
onion dart-cubit product --read-only

# Solo cubit de escritura
onion dart-cubit product --write-only

# Cubit global: salida explícita a lib/cubits/
onion dart-cubit app_session --output-dir lib/cubits --business-package catalog_business
```

## Módulo Flutter completo

```bash
# Módulo con screen, providers, cubits y stubs
onion flutter-module product --output-dir lib/src/modules

# Con business package separado y variantes responsive
onion flutter-module product \
  --output-dir lib/src/modules \
  --business-package catalog_business \
  --responsive
```

Genera:

```text
lib/src/cubits/products/            # fuera del módulo
├── read_products_cubit.dart
├── read_products_state.dart
├── write_products_cubit.dart
└── write_products_state.dart

lib/src/modules/products/
├── view/products_screen.dart       # MultiBlocProvider + _Body
├── dialogs/products_dialogs.dart
└── widgets/products_widgets.dart
```

Con `--responsive` agrega `view/mobile/`, `view/tablet/`, `view/desktop/`
como `part` files y genera `lib/src/tools/extensiones.dart` si no existe.

> El CLI lee el nombre del paquete desde `pubspec.yaml`; ejecútalo desde la
> raíz de la app Flutter o una ruta dentro de ella.
> Ver [Diseño Responsive](responsive_design.md) y
> [Alcance y Ciclo de Vida de Cubits](cubit_scopes.md).

## Vista standalone

```bash
# Crea solo la vista/screen
onion dart-view product

# Con variantes responsive
onion dart-view product --output-dir lib/src/modules/products/view --responsive
```

## Servicios

```bash
# Servicio HTTP
onion dart-service http

# Servicio HTTP + Hive (local storage)
onion dart-service http,hive

# Solo Hive
onion dart-service hive
```

## Response genérico

```bash
# Crea ListResponse genérico
onion dart-res
```

## Barrel files

```bash
# Crea export.dart en un directorio
onion barrel lib/src/data
onion barrel lib/src/domain/models
onion barrel lib/src/domain/repositories
```

## Limitaciones actuales

El CLI genera la base, pero hay ajustes manuales que todavía no automatiza:

| Limitación | Ajuste manual |
|---|---|
| No genera `view_controller.dart` ni el mediator | Agregarlos a mano para pantallas con estado efímero. Ver [ViewController Pattern](view_controller_pattern.md). |
| No genera el composition root (`app_dependencies.dart`, `provider_container.dart`) | El CLI genera módulos, no el scaffold de la app; crearlo a mano. Ver [Inyección de Dependencias](di_flutter.md). |
| Los cubits no tienen barrel por feature | Usar `onion barrel lib/src/cubits/{plural}` si se necesita. |
| Sin `--business-package`, los imports asumen `lib/src/domain` | Pasar `--business-package` cuando el business package está separado. |
| `dart-view --responsive` genera bodies placeholder | Completar cada variante con el layout real. |
| No genera tests de widgets | Agregar tests manualmente. |

## Reglas

- Los nombres de entidad deben ser **singular** (el CLI rechaza plurales en inglés)
- `--output-dir` opcional; `onion dart` usa `./lib/src` y `dart-cubit` usa `lib/src/cubits`
- Los barrel files se generan automáticamente con cada comando
- Las entidades se pluralizan automáticamente en inglés para nombres de archivo y carpetas
- Con `--business-package`, el barrel del business package debe exportar modelos,
  repositorios y `ReactiveRepository`/`RepoEvent` si se usa `--reactive`
