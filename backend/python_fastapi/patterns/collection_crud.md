---
type: Pattern
title: CRUD en Collection (MongoDB)
description: Operaciones directas contra MongoDB usando collection_name, insert_one, find, find_one_and_update, con soft delete y ReturnDocument.AFTER.
tags: [mongodb, collection, crud, pymongo]
---

# Collection CRUD

Cada entidad tiene su propia clase Collection con operaciones crudas contra MongoDB.

> **CLI:** El comando `onion crud-mongo` o `onion repo-mongo` genera la Collection con CRUD básico y registra la collection en `onion-config.toml`. Para solo CRUD sin collection usar `onion crud` o `onion repo`.

## Estructura base

```python
from typing import Any
from pymongo import ReturnDocument
from services.mongo_service import MongoService

class ProductsCollection:
    collection_name = "Products"

    def __init__(self):
        mongo_service = MongoService()
        self.__collection = mongo_service.get_collection(self.collection_name)
```

## Operaciones estándar

```python
async def create_product(self, product: dict) -> None:
    await self.__collection.insert_one(product)

async def fetch_all_products(self) -> list[dict[str, Any]]:
    cursor = self.__collection.find({"isDeleted": False})
    result = await cursor.to_list(length=None)
    await cursor.close()
    return result

async def fetch_product_by_id(self, product_id: str) -> dict[str, Any] | None:
    return await self.__collection.find_one({"id": product_id, "isDeleted": False})

async def update_product_by_id(
    self, product_id: str, product: dict
) -> dict[str, Any] | None:
    return await self.__collection.find_one_and_update(
        {"id": product_id},
        {"$set": product},
        return_document=ReturnDocument.AFTER,
    )

async def delete_product_by_id(self, product_id: str) -> dict[str, Any] | None:
    return await self.__collection.find_one_and_update(
        {"id": product_id},
        {"$set": {"isDeleted": True}},
        return_document=ReturnDocument.AFTER,
    )
```

## Convenciones

| Operación | Método | Detalle |
|-----------|--------|---------|
| Create | `insert_one` | Recibe dict completo con id generado |
| Read (all) | `find()` + `to_list()` | Siempre cerrar cursor; filtrar `isDeleted: False` (ver [Soft Delete](soft_delete.md)) |
| Read (one) | `find_one({"id": id})` | Busca por campo `id`, no `_id`; filtrar `isDeleted: False` |
| Update | `find_one_and_update` + `$set` | `ReturnDocument.AFTER` para devolver el doc actualizado |
| Delete | `find_one_and_update` + `$set: isDeleted` | Soft delete, no elimina físicamente |
