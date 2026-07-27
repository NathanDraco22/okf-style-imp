---
type: Pattern
title: Versionado de API
description: Todo el código se organiza por versión (v1, v2...) en api/, repos/, services/mongo_collections/ con un router maestro por versión.
tags: [fastapi, versionado, api, estructura]
---

# Versionado

Todas las capas se organizan por versión usando directorios `v{version}/`.

## Estructura

```
app/
├── api/
│   └── v1/
│       ├── router.py              # Router maestro v1
│       ├── products/              # Entidades de v1
│       └── users/
├── repos/
│   └── v1/
│       ├── products/
│       └── users/
└── services/
    └── mongo_collections/
        └── v1/
            ├── products_collection.py
            └── users_collection.py
```

## Router maestro por versión

```python
# api/v1/router.py
from fastapi import APIRouter
from .products.products_router import products_router
from .users.users_router import users_router

router_v1 = APIRouter(tags=["apiV1"])
router_v1.include_router(products_router, prefix="/products")
router_v1.include_router(users_router, prefix="/users")
```

## Registro en main.py

```python
app.include_router(router_v1, prefix="/api/v1")
# app.include_router(router_v2, prefix="/api/v2")  # cuando exista
```

## Reglas

- Cada versión tiene su propio `api/v{version}/router.py`
- Las rutas se registran con `prefix="/api/v{version}"`
- La migración de v1 a v2 implica duplicar la estructura completa en `v2/`
- El CLI `onion` requiere `--version` para generar código en la versión correcta
