---
type: Pattern
title: CRUD en Repository
description: Implementación estándar de create/get_all/get_by_id/update_by_id/delete_by_id en el Repository.
tags: [fastapi, repository, crud, patron]
---

# CRUD en Repository

> **CLI:** El CLI genera métodos CRUD con `NotImplementedError` como placeholder. El desarrollador reemplaza con `UuidTool.generate_uuid()`, `TimeTools.get_now_in_milliseconds()`, y las transformaciones `model_dump()`/`model_validate()`.

## Create

```python
async def create_product(self, create_product: CreateProduct) -> ProductInDb:
    new_product = ProductInDb(
        id=UuidTool.generate_uuid(),
        createdAt=TimeTools.get_now_in_milliseconds(),
        updatedAt=None,
        **create_product.model_dump(),
    )
    result = await self.products_ds.create_product(new_product.model_dump())
    return ProductInDb.model_validate(result)
```

## Get All

```python
async def get_all_products(self) -> list[ProductInDb]:
    results = await self.products_ds.get_all_products()
    return [ProductInDb.model_validate(r) for r in results]
```

## Get By ID

```python
async def get_product_by_id(self, product_id: str) -> ProductInDb | None:
    result = await self.products_ds.get_product_by_id(product_id)
    if result is None:
        return None
    return ProductInDb.model_validate(result)
```

## Update

```python
async def update_product_by_id(
    self, product_id: str, product: UpdateProduct
) -> ProductInDb | None:
    updated_data = product.model_dump(exclude_unset=True)
    updated_data["updatedAt"] = TimeTools.get_now_in_milliseconds()
    result = await self.products_ds.update_product_by_id(product_id, updated_data)
    if result is None:
        return None
    return ProductInDb.model_validate(result)
```

## Delete (soft)

```python
async def delete_product_by_id(self, product_id: str) -> ProductInDb | None:
    result = await self.products_ds.delete_product_by_id(product_id)
    if result is None:
        return None
    return ProductInDb.model_validate(result)
```

## Reglas clave

| Operación | ID | Timestamps | Transformación |
|-----------|-----|-------------|----------------|
| **create** | `UuidTool.generate_uuid()` | `createdAt` (nunca `updatedAt`) | `model_dump()` → DataSource |
| **update** | — | `updatedAt` siempre se inyecta | `model_dump(exclude_unset=True)` → + `updatedAt` |
| **delete** | — | — | Soft delete via DataSource |
| **get** | — | — | DataSource dict → `model_validate()` |
