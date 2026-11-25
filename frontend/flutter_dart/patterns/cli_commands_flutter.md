---
type: Reference
title: Comandos Onion CLI para Flutter/Dart
description: Referencia de comandos del CLI onion para generar código Flutter/Dart en la arquitectura Onion.
tags: [cli, onion, codegen, flutter, dart, comandos]
---

# Comandos Onion CLI para Flutter/Dart

> **Versión mínima: onion ≥ 0.6.0** — incluye el flag `--reactive/--no-reactive` con prompt y la generación del mixin `ReactiveRepository` en `tools/reactive_repo/`.

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
# Genera ReadCubit + WriteCubit con sus estados sellados
onion dart-cubit product

# Solo cubit de lectura
onion dart-cubit product --read-only

# Solo cubit de escritura
onion dart-cubit product --write-only
```

## Módulo Flutter completo

```bash
# Pantalla completa con cubit, diálogos, vistas y widgets
onion flutter-module product
```

> **Responsive:** el CLI genera un solo `view/{plural}_view.dart`. El split
> `mobile/`, `tablet/` y `desktop/` con `part` files se hace después a mano.
> Ver [Diseño Responsive](responsive_design.md). El flag `--responsive` está
> planificado, pero todavía no existe.
>
> **Estructura:** el CLI genera `cubit/` dentro del módulo y
> `view/{plural}_view.dart`. El estándar es `lib/src/cubits/{plural}/` y
> `view/{feature}_screen.dart`; mover y renombrar a mano hasta que el CLI lo
> genere. Ver [Alcance y Ciclo de Vida de Cubits](cubit_scopes.md).

## Vista standalone

```bash
# Crea solo la vista/screen
onion dart-view product
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
| `flutter-module` genera un solo `view/{plural}_view.dart` | Renombrar a `view/{feature}_screen.dart` y crear `mobile/`, `tablet/`, `desktop/` con `part` files. Ver [Diseño Responsive](responsive_design.md). |
| `flutter-module` genera `cubit/` dentro del módulo | Mover `read_*` y `write_*` a `lib/src/cubits/{plural}/`. Ver [Alcance y Ciclo de Vida de Cubits](cubit_scopes.md). |
| `dart-cubit` genera `lib/src/cubits/{name}_cubit/` en singular | Renombrar a `lib/src/cubits/{plural}/`; el estándar usa carpetas plurales. |
| No genera `MultiBlocProvider` | Agregar los providers en la screen, arriba del dispatch. Ver [Alcance y Ciclo de Vida de Cubits](cubit_scopes.md). |
| Los cubits importan `../../domain/...` | Si el business package está separado, cambiar a `package:{business}/{business}.dart`. |
| `dart-view` genera un archivo único | Aplicar el split responsive a mano si la pantalla lo necesita. |
| No hay flag `--responsive` | Planificado; por ahora el split es manual. |

## Reglas

- Los nombres de entidad deben ser **singular** (el CLI rechaza plurales en inglés)
- `--output-dir` opcional para cambiar la carpeta de salida (default: `./lib/src`)
- Los barrel files se generan automáticamente con cada comando
- Las entidades se pluralizan automáticamente en inglés para nombres de archivo y carpetas
