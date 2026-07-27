---
type: Reference
title: Comandos Onion CLI para Flutter/Dart
description: Referencia de comandos del CLI onion para generar código Flutter/Dart en la arquitectura Onion.
tags: [cli, onion, codegen, flutter, dart, comandos]
---

# Comandos Onion CLI para Flutter/Dart

## Capa completa (DataSource + Repository + Models)

```bash
# Crea datasource, repository y modelos para la entidad
onion dart product

# Con --output-dir para especificar carpeta
onion dart product --output-dir lib/src/features
```

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
