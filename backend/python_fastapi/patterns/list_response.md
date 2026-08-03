---
type: Pattern
title: ListResponse Genérico
description: Modelo Pydantic genérico para respuestas de listas con count y data. (No implementa paginación.)
tags: [fastapi, pydantic, response, patron]
---

# ListResponse

Respuesta estándar para endpoints que devuelven listas. Definido en `responses/v1/list_response.py`.

> **Nota:** `ListResponse` agrupa una lista completa con su `count`. Si el módulo necesita paginación real (`limit`/`offset`), se agrega como parámetros de query en el router y `skip`/`limit` en la Collection — el contrato de respuesta no cambia.

## Implementación

```python
from typing import TypeVar, Generic
from pydantic import BaseModel

T = TypeVar("T", bound=BaseModel)

class ListResponse(BaseModel, Generic[T]):
    count: int
    data: list[T]
```

## Uso en Controller

```python
async def get_all_products(self) -> ListResponse[ProductInDb]:
    products = await self.products_repo.get_all_products()
    return ListResponse(count=len(products), data=products)
```

## Uso en Router

```python
@products_router.get("")
async def get_all_products_endpoint() -> ListResponse[ProductInDb]:
    return await products_controller.get_all_products()
```
