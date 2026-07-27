---
type: Reference
title: Comandos Onion CLI para Python/FastAPI
description: Referencia de comandos del CLI onion para generar código backend en la arquitectura Onion.
tags: [cli, onion, codegen, fastapi, python, comandos]
---

# Comandos Onion CLI para Python/FastAPI

## CRUD completo con módulo

```bash
# Crea router + controller + repository + datasource + models
onion crud product --version 1
onion crud client --version 1

# Múltiples entidades a la vez
onion crud product client supplier --version 1
```

## CRUD completo + MongoDB collection

```bash
# Incluye collection con create_indexes() y onion-config.toml
onion crud-mongo product --version 1
```

## Solo módulo (repository + datasource + models)

```bash
onion repo product --version 1
onion repo-mongo product --version 1  # + mongo collection
```

## Solo router + controller

```bash
onion router product --version 1
```

## Proyecto completo

```bash
# Sobrescribe app/ con la plantilla FastAPI completa
onion project
```

## Reglas

- Los nombres de entidad deben ser **singular** (el CLI rechaza plurales en inglés)
- `--version` es requerido en todos los comandos backend
- `--output-dir` opcional para cambiar la carpeta de salida
- Las entidades se pluralizan automáticamente en inglés para nombres de archivo y rutas
- `onion-config.toml` se actualiza automáticamente al usar `*-mongo`

> Para comandos Flutter/Dart, ver [cli_commands_flutter.md](../../../frontend/flutter_dart/patterns/cli_commands_flutter.md).
