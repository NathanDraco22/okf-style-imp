---
type: Pattern
title: ListResponse Genérico
description: Modelo Pydantic genérico para respuestas paginadas con count y data.
tags: [fastapi, pydantic, response, patron]
---

# ListResponse

Respuesta estándar para endpoints que devuelven listas. Definido en `responses/v1/list_response.py`.

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
