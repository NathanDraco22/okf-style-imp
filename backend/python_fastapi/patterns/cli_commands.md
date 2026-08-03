---
type: Reference
title: Comandos Onion CLI para Python/FastAPI
description: Referencia de comandos del CLI onion para generar código backend en la arquitectura Onion.
tags: [cli, onion, codegen, fastapi, python, comandos]
---

# Comandos Onion CLI para Python/FastAPI

> **Versión mínima: onion ≥ 0.6.0** — incluye `fastapi-init` con clon del template de GitHub y generación de pymongo async (`AsyncMongoClient`).

## Proyecto completo (flujo canónico)

`onion project fastapi-init` **clona el template canónico de GitHub** ([fastapi-onion-template](https://github.com/NathanDraco22/fastapi-onion-template)) con todo el proyecto: `pyproject.toml`, `Dockerfile`, `uv.lock`, `.vscode/`, `README.md`, `runtime.txt`. Borra el `.git` del clon (proyecto limpio) y renombra los placeholders `examples` según el nombre del proyecto.

```bash
# 1. Inicializar el proyecto (clona el template completo)
onion project fastapi-init my_app

# 2. Instalar dependencias
cd my_app
uv sync

# 3. Si usas MongoDB: agregar pymongo (el template NO lo incluye a propósito,
#    es agnóstico de base de datos)
uv add pymongo

# 4. Generar entidades
onion crud-mongo product --version 1
```

> **El template no incluye pymongo a propósito**: sirve como base neutral porque mañana podría usarse otra base de datos. Agrega la dependencia solo cuando el proyecto la necesita.
>
> **Paso manual post-clon:** completar `app/core/services_initializer.py` con la inicialización de `MongoService().init_service()`.

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
# Clona el template de GitHub completo (pyproject, Dockerfile, uv.lock, .vscode)
onion project fastapi-init my_app

# Solo sobrescribe app/ con el scaffolding base (sin configs de proyecto)
onion project fastapi-app
```

## Reglas

- Los nombres de entidad deben ser **singular** (el CLI rechaza plurales en inglés)
- `--version` es requerido en todos los comandos backend
- `--output-dir` opcional para cambiar la carpeta de salida
- Las entidades se pluralizan automáticamente en inglés para nombres de archivo y rutas
- `onion-config.toml` se actualiza automáticamente al usar `*-mongo`

> Para comandos Flutter/Dart, ver [cli_commands_flutter.md](../../../frontend/flutter_dart/patterns/cli_commands_flutter.md).
