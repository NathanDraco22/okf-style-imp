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

## Reglas

- Los nombres de entidad deben ser **singular** (el CLI rechaza plurales en inglés)
- `--output-dir` opcional para cambiar la carpeta de salida (default: `./lib/src`)
- Los barrel files se generan automáticamente con cada comando
- Las entidades se pluralizan automáticamente en inglés para nombres de archivo y carpetas
