---
type: Pattern
title: Counter Manager
description: Generación atómica de correlativos secuenciales por tipo de documento mediante find_one_and_update con $inc y upsert sobre una colección de contadores.
tags: [mongodb, contadores, correlativos, atomicidad, patron, pymongo]
---

# Counter Manager

El `CounterManager` centraliza la generación de números secuenciales por tipo de documento (`invoiceNumber`, `orderNumber`, `receiptNumber`, etc.) de forma atómica, sin coordinación en memoria y sin depender de transacciones multi-documento.

> **No es una entidad de negocio.** Los contadores son infraestructura: no tienen Router, Controller ni endpoints propios, no llevan `isDeleted` y no se exponen directamente.
>
> **CLI:** el `CounterManager` no lo genera `onion`, pero la base de la collection y el repositorio sí: `onion repo-mongo counter --version 1` crea las capas y registra `Counters` en `onion-config.toml`; luego se recortan al contrato de este patrón (sin CRUD genérico ni soft delete).

## Cuándo usarlo

- Se necesita un número legible y secuencial para un documento (factura, orden, recibo, solicitud).
- Varias requests concurrentes pueden generar el mismo tipo de documento al mismo tiempo.
- El número debe ser único aunque no exista una transacción que abarque la inserción del documento.

No usarlo para IDs internos: esos se generan con UUID y no necesitan secuencia.

## Estructura

```
app/
├── core/counter_manager/
│   └── counter_manager.py          # Scope, formato y API pública
├── repos/v1/counters/
│   ├── __init__.py
│   ├── counters_repository.py
│   ├── data/
│   │   └── counters_datasource.py
│   └── models/
│       └── counter_model.py
└── services/mongo_collections/v1/
    └── counters_collection.py
```

La cadena es la habitual de Onion, pero sin Router ni Controller:

```
CounterManager → CountersRepository → CountersDataSource → CountersCollection → MongoDB
```

## Modelo

```python
from pydantic import BaseModel


class BaseCounter(BaseModel):
    name: str   # clave lógica: tipo de contador + scope
    value: int  # último valor entregado


class CounterInDb(BaseCounter): ...
```

- `name` es la clave. Si el proyecto es multi-tenant o reinicia por periodo, esa dimensión va incluida en el nombre (ver [Clave y scope](#clave-y-scope)).
- No se agrega `isDeleted`: el contador no se borra.
- `updatedAt` es opcional; si se agrega, sirve para auditoría y no interviene en la generación.

## Manager

El manager concentra dos responsabilidades: **componer la clave** y **formatear el número**. El repositorio se mantiene genérico.

```python
from typing import Any
from repos.v1.counters import CountersRepository


class CounterManager:
    _instance: "CounterManager | None" = None

    def __init__(self, counters_repository: CountersRepository) -> None:
        self.counters_repository = counters_repository

    @classmethod
    def get_instance(cls) -> "CounterManager":
        if cls._instance is None:
            cls._instance = cls(CountersRepository.get_instance())
        return cls._instance

    async def next_number(
        self,
        counter_name: str,
        *,
        prefix: str = "",
        width: int = 4,
        tenant_id: str | None = None,
        period: str | None = None,
        session: Any = None,
    ) -> str:
        key = self._build_key(counter_name, tenant_id, period)
        counter = await self.counters_repository.increment_counter(key, session)
        return f"{prefix}{counter.value:0{width}d}"

    @staticmethod
    def _build_key(
        counter_name: str,
        tenant_id: str | None,
        period: str | None,
    ) -> str:
        parts = [part for part in (tenant_id, counter_name, period) if part]
        return ":".join(parts)
```

Uso desde un repositorio o manager de negocio:

```python
order_number = await CounterManager.get_instance().next_number(
    "orderNumber",
    prefix="ORD-",
    width=6,
    tenant_id=tenant.id,
    period="2026",
)
# → "ORD-000124"
```

> **Singleton:** usar `get_instance()` (Variante 2 de [Singleton](singleton.md)). No implementarlo con `__new__` si el `__init__` recibe dependencias: `__new__` devuelve la instancia existente, pero `__init__` se vuelve a ejecutar en cada llamada y recrea el repositorio.

## Repository y DataSource

Son capas delgadas de paso; la lógica vive en el manager y en la collection.

```python
class CountersRepository:
    _instance: "CountersRepository | None" = None

    def __init__(self, counters_ds: CountersDataSource) -> None:
        self.counters_ds = counters_ds

    @classmethod
    def get_instance(cls) -> "CountersRepository":
        if cls._instance is None:
            cls._instance = cls(CountersDataSource())
        return cls._instance

    async def increment_counter(
        self, counter_name: str, session: Any = None
    ) -> CounterInDb:
        raw = await self.counters_ds.increment_counter(counter_name, session)
        return CounterInDb.model_validate(raw)


class CountersDataSource:
    async def increment_counter(
        self, counter_name: str, session: Any = None
    ) -> dict[str, Any]:
        return await CountersCollection.get_instance().increment_counter(
            counter_name, session
        )
```

## Collection

La operación clave es una sola: `find_one_and_update` con `$inc`, `upsert=True` y `ReturnDocument.AFTER`. No se lee primero para luego escribir.

```python
from typing import Any

from pymongo import ReturnDocument
from pymongo.errors import DuplicateKeyError


class CountersCollection(BaseMongoCollection):
    collection_name = "Counters"

    async def increment_counter(
        self, counter_name: str, session: Any = None
    ) -> dict[str, Any]:
        try:
            return await self._increment_once(counter_name, session)
        except DuplicateKeyError:
            # Otra request insertó el contador entre el match y el upsert.
            # El segundo intento lo encuentra y solo incrementa.
            return await self._increment_once(counter_name, session)

    async def _increment_once(
        self, counter_name: str, session: Any = None
    ) -> dict[str, Any]:
        result = await self._collection.find_one_and_update(
            {"name": counter_name},
            {"$inc": {"value": 1}},
            upsert=True,
            session=session,
            return_document=ReturnDocument.AFTER,
        )
        if result is None:
            raise CounterException(
                f"No se pudo incrementar el contador '{counter_name}'"
            )
        return result

    async def set_counter(
        self, counter_name: str, value: int, session: Any = None
    ) -> dict[str, Any]:
        result = await self._collection.find_one_and_update(
            {"name": counter_name},
            {"$set": {"value": value}},
            upsert=True,
            session=session,
            return_document=ReturnDocument.AFTER,
        )
        if result is None:
            raise CounterException(
                f"No se pudo fijar el contador '{counter_name}'"
            )
        return result

    async def get_counter(self, counter_name: str) -> dict[str, Any] | None:
        return await self._collection.find_one({"name": counter_name})

    async def create_indexes(self) -> None:
        await self._collection.create_index([("name", 1)], unique=True)
```

Notas:

- `CounterException` debe ser una excepción de dominio propia, definida junto a la jerarquía de excepciones del proyecto (ver [Manejo de Excepciones](exception_handling.md)). No usar `MongoMigrationException` ni excepciones de infraestructura de migraciones para un fallo en runtime.
- El acceso a la colección (`self._collection` o `self.__collection`) depende de la `BaseMongoCollection` del proyecto. Ver [CRUD en Collection](collection_crud.md).

## Clave y scope

Toda dimensión que deba ser independiente va incluida en `name`:

| Variante | `name` | Reinicio |
|---|---|---|
| Global (single-tenant) | `orderNumber` | Nunca |
| Por tenant | `acme:orderNumber` | Nunca |
| Por tenant y periodo | `acme:orderNumber:2026` | Cada periodo |

- En una base multi-tenant **el scope por tenant es obligatorio**; sin él, todos los negocios comparten la misma secuencia.
- Si el negocio espera reiniciar la numeración cada año, usar la variante con periodo. Así el reinicio es implícito: el 1 de enero cambia la clave y arranca en 1, sin un job de reseteo.
- No mezclar contadores globales y por tenant con el mismo nombre.

## Sesiones y atomicidad

El parámetro `session` es **opcional**. `find_one_and_update` con `$inc` es atómico a nivel de documento, aunque no haya transacción.

```python
# Sin transacción: operación atómica de un solo documento.
order_number = await CounterManager.get_instance().next_number(
    "orderNumber", prefix="ORD-", width=6, tenant_id=tenant_id
)

# Con transacción: el incremento se confirma o revierte junto con el resto.
async with session.start_transaction():
    invoice_number = await CounterManager.get_instance().next_number(
        "invoiceNumber", prefix="INV-", width=6, session=session
    )
    await invoices_repository.create(
        {**invoice, "docNumber": invoice_number}, session=session
    )
```

Ver [Transacciones MongoDB](mongo_transactions.md) para el manejo de transacciones y retry de `WriteConflict`.

## Concurrencia

- **El índice debe ser único.** Un `create_index([("name", 1)])` sin `unique=True` permite que dos upserts concurrentes inserten dos documentos con el mismo `name` y la secuencia se parta en dos. En proyectos que siempre escriben dentro de una transacción el riesgo queda oculto; sin transacciones es real.
- **El upsert no está libre de carreras.** Si dos requests no encuentran el contador, ambas intentan insertarlo; una gana y la otra recibe `DuplicateKeyError`. El retry del ejemplo lo resuelve.
- **Alternativa sin retry:** pre-crear los contadores al inicializar el proyecto o al dar de alta un tenant (`value = 0`). Luego el `$inc` siempre encuentra el documento.
- **No usar un mutex en memoria** para "proteger" el contador: no funciona con múltiples instancias del backend.

## Formato

El manager compone el número final: `prefijo` + `value` con padding (`zfill` o formato `{value:0{width}d}`).

- El valor entero se guarda en `Counters`; el número formateado se persiste en el documento de negocio (`orderNumber`, `docNumber`).
- Si el proyecto ordena o filtra por número, guardar también el entero crudo (`numberValue`) evita ordenar lexicográficamente un string con padding.
- El padding debe ser fijo por tipo de contador. Cambiarlo a mitad de año produce códigos de distinta longitud.
- El prefijo puede ser constante (`ORD-`, `CIT-`) o venir de configuración del tenant; en ambos casos lo aplica el manager, no la collection.

## set_counter y reseteos

`set_counter` existe para **importaciones y migraciones**: cuando se importan documentos históricos y hay que continuar la secuencia desde un número conocido.

- No usarlo como mecanismo de reseteo en runtime; para reinicios periódicos, usar el scope por periodo.
- Fijar un valor menor al último entregado puede generar duplicados. Validar contra el máximo existente antes de migrar.
- Al importar documentos con números propios de otro sistema, no llamar `next_number` por cada uno: insertarlos con su número original y, al final, `set_counter` con el máximo importado. Si en cambio se genera un número nuevo por registro, el contador ya queda al día y no hace falta `set_counter`.

## Huecos (gaps)

El contador se incrementa antes de insertar el documento de negocio. Si la inserción falla, el número se consume y queda un hueco en la secuencia.

Es una decisión consciente:

- Un hueco es preferible a un duplicado o a bloquear la generación con un lock.
- Si el negocio exige secuencias sin huecos, hay que usar transacción (cuando la inserción y el incremento ocurren juntos) o un proceso de reconciliación; ambos agregan complejidad.
- Los huecos no afectan la unicidad ni el orden relativo de los documentos existentes.

## Reglas

- **Una sola operación atómica:** `find_one_and_update` + `$inc` + `upsert` + `ReturnDocument.AFTER`. Nunca leer, calcular en Python y escribir.
- **Clave única:** índice `unique=True` sobre `name` o `_id` como clave. Sin unicidad no hay garantía.
- **Scope explícito:** en multi-tenant, `tenantId` siempre en la clave; si aplica reinicio, el periodo también.
- **Formato en el manager:** prefijo y padding centralizados; la collection solo incrementa.
- **`session` opcional:** pasar la sesión solo cuando el incremento deba ser atómico con otros documentos; en caso contrario, omitirla.
- **Excepción de dominio:** fallo de incremento → excepción propia, nunca una de migración.
- **Singleton con `get_instance()`** y dependencias inyectadas por constructor.
- **Probar concurrencia:** test con `asyncio.gather` sobre N incrementos y verificar que los valores sean únicos y consecutivos (salvo gaps por fallos simulados).
- **No exponer el contador por API** salvo como lectura administrativa; la API pública devuelve el número del documento.

## Implementación de referencia

`kardex-server/packages/kardex-business/src/kardex/core/counter_manager/`

Cadena completa: `CounterManager → CountersRepository → CountersDataSource → CountersCollection`, con `find_one_and_update({"name": counter_name}, {"$inc": {"value": 1}}, upsert=True, session=session, return_document=ReturnDocument.AFTER)`.

Al adaptarla a otro proyecto, revisar estos puntos:

| Punto | Implementación kardex | Ajuste recomendado |
|---|---|---|
| Scope | Global, sin `tenantId` | Incluir tenant (y periodo si aplica) en la clave |
| Índice | `create_index([("name", 1)])` sin `unique=True` | Índice único + retry de `DuplicateKeyError` |
| Sesión | Requerida en casi todos los métodos | Opcional (`session=None`), salvo transacciones |
| Formato | Devuelve el entero como string | Prefijo + padding en el manager |
| Excepción | `MongoMigrationException` | Excepción de dominio |
| Singleton | `__new__` + `__init__` con repositorio | `get_instance()` con DI por constructor |
| `set_counter` | Disponible para importaciones | Mantener, con validación contra el máximo |
