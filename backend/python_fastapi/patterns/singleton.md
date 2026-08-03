---
type: Pattern
title: Singleton en DataSource y Collection
description: Todas las DataSources y Collections implementan singleton para evitar múltiples conexiones y estados inconsistentes.
tags: [fastapi, singleton, patron, creacional]
---

# Singleton

DataSources, Collections y servicios globales (MongoService, etc.) usan singleton.

## Variante 1: `__new__` (kardex-server)

```python
from typing_extensions import Self

class ProductsDataSource:
    def __new__(cls) -> Self:
        if not hasattr(cls, "instance"):
            cls.instance = super().__new__(cls)
        return cls.instance
```

Uso:

```python
ds = ProductsDataSource()  # siempre la misma instancia
```

## Variante 2: factory classmethod (serum_app_back)

```python
class ProductsDataSource:
    _instance: "ProductsDataSource | None" = None

    @classmethod
    def get_instance(cls) -> "ProductsDataSource":
        if cls._instance is None:
            cls._instance = cls()
        return cls._instance
```

Uso:

```python
ds = ProductsDataSource.get_instance()
```

## Variante 3: MongoService

Usa el driver async nativo de **pymongo ≥ 4.17** (`AsyncMongoClient`), que reemplaza a motor (deprecado):

```python
class MongoService:
    _instance: "MongoService | None" = None
    _client: AsyncMongoClient | None = None

    def __new__(cls) -> Self:
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    async def init_service(self):
        if self._client is None:
            self._client = AsyncMongoClient(self._mongo_url)

    def get_client(self) -> AsyncMongoClient:
        """Cliente completo — usado por transacciones multi-collección."""
        return self._client

    def get_collection(self, name: str):
        return self._client[self._db_name][name]
```

> El CLI (`crud-mongo`) genera `MongoService` con `AsyncMongoClient` + `AsyncCollection` (`from pymongo import AsyncMongoClient`, `from pymongo.asynchronous.collection import AsyncCollection`). Motor ya no se usa.

- `get_collection()` — para operaciones simples de una colección (CRUD estándar)
- `get_client()` + `session.start_transaction()` — para transacciones atómicas multi-colección (ver [Transacciones MongoDB](mongo_transactions.md))

## ¿Dónde se usa singleton?

| Clase | Razón |
|-------|-------|
| `MongoService` | Una sola conexión a MongoDB |
| `DataSource` | Sin estado, no necesita múltiples instancias |
| `Collection` | Sin estado, el cliente Mongo es compartido |
| `Repository` | En algunos proyectos (serum) también es singleton |
