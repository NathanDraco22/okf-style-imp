---
type: Pattern
title: Flujo Onion en FastAPI
description: Arquitectura en capas con flujo unidireccional Router → Controller → Repository → DataSource → Collection → MongoDB.
tags: [fastapi, onion, arquitectura, flujo]
---

# Flujo Onion

La arquitectura Onion organiza el código en capas concéntricas donde la dependencia apunta hacia adentro:

> **CLI:** El comando `onion project fastapi-init` clona el template canónico de GitHub ([fastapi-onion-template](https://github.com/NathanDraco22/fastapi-onion-template)) con todo el proyecto (pyproject.toml, Dockerfile, uv.lock, .vscode). `onion crud {entity} --version 1` genera todas las capas para una entidad nueva. El CLI asegura que la estructura y dependencias entre capas sean consistentes.

```
Router → Controller → Repository → DataSource → Collection → MongoDB
```

## Capas

| Capa | Carpeta | Responsabilidad |
|------|---------|-----------------|
| **Router** | `api/v1/{entity}/` | Define endpoints HTTP, path/query params, tipos de request/response |
| **Controller** | `api/v1/{entity}/` | Orquesta la petición, llama al Repository, levanta HTTPException si es necesario |
| **Repository** | `repos/v1/{entity}/` | Lógica de negocio: genera IDs, timestamps, transforma modelos Pydantic ↔ dicts |
| **DataSource** | `repos/v1/{entity}/data/` | Capa delgada: pasa dicts entre Repository y Collection |
| **Collection** | `services/mongo_collections/v1/` | Operaciones crudas contra MongoDB (insert_one, find, etc.) |
| **MongoService** | `services/` | Cliente MongoDB singleton (pymongo async: `AsyncMongoClient`) |

## Regla de dependencia

Una capa **nunca** importa de una capa superior. El flujo es estrictamente unidireccional:

- Router → Controller
- Controller → Repository
- Repository → DataSource
- DataSource → Collection
- Collection → MongoService

## Ejemplo visual

```
POST /api/v1/products
  → products_router.py
    → products_controller.py (ProductsController.create_product)
      → products_repository.py (ProductsRepository.create_product)
        → products_datasource.py (ProductsDataSource.create_product)
          → products_collection.py (ProductsCollection.create_product)
            → MongoService.get_collection("Products").insert_one(...)
```
