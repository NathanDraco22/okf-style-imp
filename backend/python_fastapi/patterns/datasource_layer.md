---
type: Pattern
title: Capa DataSource
description: Capa delgada entre Repository y Collection. Recibe dicts, envía dicts, sin lógica de negocio ni validación.
tags: [fastapi, datasource, capa, patron]
---

# DataSource

La capa DataSource es intencionalmente delgada. Su única responsabilidad es conectar el Repository con la Collection.

> **CLI:** Generado por `onion crud` o `onion repo`. El CLI crea el DataSource con métodos placeholder. El desarrollador completa la inyección de la Collection y las llamadas a sus métodos.

## Estructura

```python
from typing import Any
from services.mongo_collections.v1 import ProductsCollection

class ProductsDataSource:
    async def create_product(self, product: dict[str, Any]) -> dict[str, Any]:
        collection = ProductsCollection.get_instance()
        await collection.create_product(product)
        return product

    async def get_all_products(self) -> list[dict[str, Any]]:
        collection = ProductsCollection.get_instance()
        return await collection.fetch_all_products()

    async def get_product_by_id(self, product_id: str) -> dict[str, Any] | None:
        collection = ProductsCollection.get_instance()
        return await collection.fetch_product_by_id(product_id)

    async def update_product_by_id(
        self, product_id: str, product: dict[str, Any]
    ) -> dict[str, Any] | None:
        collection = ProductsCollection.get_instance()
        return await collection.update_product_by_id(product_id, product)

    async def delete_product_by_id(self, product_id: str) -> dict[str, Any] | None:
        collection = ProductsCollection.get_instance()
        return await collection.delete_product_by_id(product_id)
```

## Reglas

- **Solo dicts** — nunca recibe ni devuelve objetos Pydantic
- **Sin lógica de negocio** — no genera IDs, no calcula timestamps, no valida
- **Respeta la firma** — cada método del DataSource coincide 1:1 con la Collection
- **Singleton** — `get_instance()` o `__new__` según el proyecto (ver [singleton](./singleton.md)); con MongoDB se usa `Collection.get_instance()` (ver [Cableado de Dependencias](di_wiring.md))
