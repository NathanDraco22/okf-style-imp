---
type: Pattern
title: Cableado de Dependencias (DI)
description: Las dependencias se inyectan manualmente por constructor. Dos variantes: instanciación directa en controller o singleton classmethod.
tags: [fastapi, di, dependencias, wiring, patron]
---

# Cableado de Dependencias

No se usa un framework de DI. Las dependencias se cablean manualmente.

> **CLI:** El CLI genera el cableado básico. Para entidades con MongoDB (crud-mongo/repo-mongo), se usa `get_instance()` en el DataSource. Para entidades sin MongoDB (crud/repo), se usa instanciación directa en el Controller. Ver [singleton.md](singleton.md).

## Variante 1: Instanciación directa (kardex-server)

Todo el árbol de dependencias se instancia al final del controller:

```python
# api/v1/products/products_controller.py
class ProductsController:
    def __init__(self, products_repo: ProductsRepository) -> None:
        self.products_repo = products_repo

# Module-level singleton con DI explícita
products_controller = ProductsController(
    products_repo=ProductsRepository(
        products_ds=ProductsDataSource(),
    ),
)
```

## Variante 2: Singleton classmethod (serum_app_back)

Cada Repository expone `get_instance()` que crea su DataSource internamente:

```python
# repos/v1/branches/branches_repository.py
class BranchesRepository:
    _instance: "BranchesRepository | None" = None

    def __init__(self, branches_ds: BranchesDataSource):
        self.branches_ds = branches_ds

    @classmethod
    def get_instance(cls) -> "BranchesRepository":
        if cls._instance is None:
            cls._instance = cls(BranchesDataSource())
        return cls._instance
```

El controller usa el singleton directamente:

```python
# api/v1/branches/branches_controller.py
branches_controller = BranchesController(
    branches_repo=BranchesRepository.get_instance(),
)
```

## Árbol de dependencias completo

```
Controller
  └── Repository
        └── DataSource
              └── Collection
                    └── MongoService (singleton global)
```
