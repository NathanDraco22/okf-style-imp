---
type: StyleGuide
title: Convenciones de Nomenclatura
description: Reglas de nomenclatura para archivos, clases, variables y directorios en proyectos FastAPI Onion.
tags: [estilo, naming, convenciones, python]
---

# Convenciones de Nomenclatura

## Archivos y directorios

| Elemento | Convención | Ejemplo |
|----------|-----------|---------|
| Archivo de entidad | `{plural}_` + rol | `products_router.py`, `products_controller.py` |
| Archivo de modelo | `{singular}_model.py` | `product_model.py` |
| Archivo de carpeta data | `{plural}_datasource.py` | `products_datasource.py` |
| Archivo de collection | `{plural}_collection.py` | `products_collection.py` |
| Directorio de entidad | `{plural}/` | `products/` |
| Directorio de versión | `v{numero}/` | `v1/`, `v2/` |
| Init file | `__init__.py` | Re-exporta símbolos públicos |

## Clases

| Elemento | Convención | Ejemplo |
|----------|-----------|---------|
| Router | `{Entity}Router` | `ProductsRouter` (variable: `products_router`) |
| Controller | `{Entity}Controller` | `ProductsController` |
| Repository | `{Entity}Repository` | `ProductsRepository` |
| DataSource | `{Entity}DataSource` | `ProductsDataSource` |
| Collection | `{Entity}Collection` | `ProductsCollection` |
| Model (create) | `Create{Entity}` | `CreateProduct` |
| Model (update) | `Update{Entity}` | `UpdateProduct` |
| Model (in db) | `{Entity}InDb` | `ProductInDb` |
| Response | `ListResponse` | Genérico con Generic[T] |

## Variables y naming

| Elemento | Convención |
|----------|-----------|
| Módulo entity en router/controller | `{entity}_router`, `{entity}_controller` |
| Controller singleton | `products_controller` |
| Endpoint handler | `{action}_{entity}_endpoint` |

## Prefijos MongoDB

- Los IDs usan el campo `id` (string UUID), no `_id` de MongoDB
- Timestamps: `createdAt`, `updatedAt` (camelCase)
- Soft delete: `isDeleted`
