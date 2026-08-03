---
type: Pattern
title: Patrón Controller
description: Controlador recibe el Repository por inyección de dependencias, delega la lógica y maneja HTTPException 404.
tags: [fastapi, controller, patron, di]
---

# Controller

El Controller orquesta la petición HTTP. Recibe el Repository por inyección en el constructor y delega toda la lógica.

> **CLI:** Generado por `onion crud` o `onion router`. El CLI crea el Controller con DI placeholder. El desarrollador completa el cableado de dependencias al final del archivo.

## Estructura

```python
from fastapi import HTTPException, status
from repos.v1.products import ProductsRepository, CreateProduct, UpdateProduct, ProductInDb
from responses.v1 import ListResponse

class ProductsController:
    def __init__(self, products_repo: ProductsRepository) -> None:
        self.products_repo = products_repo

    async def create_product(self, body: CreateProduct) -> ProductInDb:
        return await self.products_repo.create_product(body)

    async def get_all_products(self) -> ListResponse[ProductInDb]:
        products = await self.products_repo.get_all_products()
        return ListResponse(count=len(products), data=products)

    async def get_product_by_id(self, product_id: str) -> ProductInDb:
        product = await self.products_repo.get_product_by_id(product_id)
        if product is None:
            raise HTTPException(
                status_code=status.HTTP_404_NOT_FOUND,
                detail="Item not found",
            )
        return product

    async def update_product_by_id(
        self, product_id: str, body: UpdateProduct
    ) -> ProductInDb:
        updated = await self.products_repo.update_product_by_id(product_id, body)
        if updated is None:
            raise HTTPException(status_code=404, detail="Item not found")
        return updated

    async def delete_product_by_id(self, product_id: str) -> ProductInDb:
        deleted = await self.products_repo.delete_product_by_id(product_id)
        if deleted is None:
            raise HTTPException(status_code=404, detail="Item not found")
        return deleted

# Module-level singleton wiring
products_controller = ProductsController(
    products_repo=ProductsRepository(products_ds=ProductsDataSource()),
)
```

> **Criterio de cableado:** con MongoDB → `get_instance()` en DataSource/Collection; sin MongoDB → instanciación directa en el Controller. Ver [Cableado de Dependencias](di_wiring.md).

## Reglas

- **Constructor** recibe el Repository (nunca lo instancia internamente)
- **Maneja 404** — si el Repository devuelve `None`, levanta `HTTPException`
- **No hace lógica de negocio** — solo orquesta y traduce errores HTTP
- **Singleton al final del archivo** — se instancia una vez con todas las dependencias
