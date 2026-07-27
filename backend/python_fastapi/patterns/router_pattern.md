---
type: Pattern
title: Patrón Router
description: Cada entidad tiene un APIRouter con 5 endpoints CRUD estándar que delegan al Controller singleton.
tags: [fastapi, router, patron, endpoints]
---

# Router

Cada entidad expone 5 endpoints REST estándar en `api/v1/{entity}/{entity}_router.py`.

> **CLI:** Generado por `onion crud {entity} --version 1` o `onion router {entity} --version 1`. El CLI también actualiza `api/v1/router.py` automáticamente.

## Estructura

```python
from fastapi import APIRouter
from repos.v1.products import CreateProduct, UpdateProduct, ProductInDb
from responses.v1 import ListResponse
from .products_controller import products_controller

products_router = APIRouter(tags=["productsV1"])

@products_router.post("")
async def create_product_endpoint(body: CreateProduct) -> ProductInDb:
    return await products_controller.create_product(body)

@products_router.get("")
async def get_all_products_endpoint() -> ListResponse[ProductInDb]:
    return await products_controller.get_all_products()

@products_router.get("/{product_id}")
async def get_product_by_id_endpoint(product_id: str) -> ProductInDb:
    return await products_controller.get_product_by_id(product_id)

@products_router.patch("/{product_id}")
async def update_product_by_id_endpoint(
    product_id: str, body: UpdateProduct
) -> ProductInDb:
    return await products_controller.update_product_by_id(product_id, body)

@products_router.delete("/{product_id}")
async def delete_product_by_id_endpoint(product_id: str) -> ProductInDb:
    return await products_controller.delete_product_by_id(product_id)
```

## Registro

Los routers se registran en `api/v1/router.py`:

```python
from fastapi import APIRouter
from .products.products_router import products_router
# ... otros imports

router_v1 = APIRouter(tags=["apiV1"])
router_v1.include_router(products_router, prefix="/products")
# ... otros include_router
```

Y en `main.py`:

```python
app.include_router(router_v1, prefix="/api/v1")
```

## Convenciones

| Endpoint | Método | Ruta | Body | Respuesta |
|----------|--------|------|------|-----------|
| Crear | POST | `{prefix}` | `Create{Entity}` | `{Entity}InDb` |
| Listar | GET | `{prefix}` | — | `ListResponse[{Entity}InDb]` |
| Obtener | GET | `{prefix}/{id}` | — | `{Entity}InDb` |
| Actualizar | PATCH | `{prefix}/{id}` | `Update{Entity}` | `{Entity}InDb` |
| Eliminar | DELETE | `{prefix}/{id}` | — | `{Entity}InDb` |
