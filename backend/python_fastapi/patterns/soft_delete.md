---
type: Pattern
title: Soft Delete
description: Eliminación lógica mediante campo isDeleted en todas las entidades. Los registros nunca se borran físicamente.
tags: [mongodb, soft-delete, patron, pymongo]
---

# Soft Delete

Todas las entidades usan eliminación lógica (soft delete) en lugar de borrado físico.

## Modelo

```python
class ProductInDb(BaseProduct):
    id: str
    createdAt: int
    updatedAt: int | None = None
    isDeleted: bool = False  # ← campo de soft delete
```

## Collection

```python
async def delete_product_by_id(self, product_id: str) -> dict[str, Any] | None:
    return await self.__collection.find_one_and_update(
        {"id": product_id},
        {"$set": {"isDeleted": True}},
        return_document=ReturnDocument.AFTER,
    )
```

## Filtrado en consultas

Dos variantes según el proyecto:

```python
# Variante 1: filtrar activos explícitamente
cursor = self.__collection.find({"isDeleted": False})

# Variante 2: excluir borrados (tolera ausencia del campo)
cursor = self.__collection.find({"isDeleted": {"$ne": True}})
```

## Reglas

- **Nunca** se usa `delete_one()` — siempre `find_one_and_update` con `$set: isDeleted = True`
- **Endpoints GET** deben filtrar `isDeleted = False` o `$ne: True` para no devolver registros eliminados
- **No hay purge** — los registros soft-deleteados permanecen en DB para auditoría
