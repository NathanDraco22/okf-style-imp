# Patrones — Python / FastAPI

Patrones de diseño y técnicas usados en proyectos FastAPI con arquitectura Onion.

> **Generado por Onion CLI:** El CLI `onion` genera el scaffolding completo de cada patrón: estructura de archivos, clases del CRUD, barrel files, y registro automático en el router maestro. El desarrollador completa los campos de los modelos, la lógica de negocio en repositorios y los endpoints custom. Ver [cli_commands.md](cli_commands.md) para la referencia de comandos.
>
> **Regla #1 — Prioridad CLI:** toda estructura nueva se genera con Onion CLI (`onion ...`), **nunca a mano**. El CLI ya hace buen trabajo: estructura, capas y registro en routers consistentes. Crear archivos manualmente rompe el patrón.

## Flujo y arquitectura

- [Flujo Onion](onion_flow.md) — Router → Controller → Repository → DataSource → Collection → MongoDB
- [Versionado](versioning.md) — Organización v1/v2 en api, repos y collections

## Capas

- [Patrón Router](router_pattern.md) — 5 endpoints CRUD por entidad
- [Patrón Controller](controller_pattern.md) — DI del Repository y manejo de 404
- [CRUD en Repository](repository_crud.md) — Create/Read/Update/Delete estándar
- [Capa DataSource](datasource_layer.md) — Capa delgada dict-to-dict
- [CRUD en Collection](collection_crud.md) — MongoDB CRUD con soft delete

## Modelos y respuestas

- [Patrón de Modelos](model_pattern.md) — Base/Create/Update/InDb
- [ListResponse](list_response.md) — Respuesta genérica de listas con count

## Persistencia

- [Soft Delete](soft_delete.md) — isDeleted en todas las entidades
- [Timestamps](timestamps.md) — createdAt/updatedAt como Unix ms
- [Búsqueda por Texto](search_pattern.md) — searchPrefixes + $text index
- [Transacciones MongoDB](mongo_transactions.md) — Atomicidad con session.start_transaction()

## Infraestructura

- [Singleton](singleton.md) — Singleton en DataSources y Collections
- [Cableado de Dependencias](di_wiring.md) — DI manual por constructor
- [Manejo de Excepciones](exception_handling.md) — Jerarquía de excepciones + handler global

## CLI

- [Comandos Onion CLI para Python/FastAPI](cli_commands.md) — Referencia de comandos onion para backend
- [Comandos Onion CLI para Flutter/Dart](../../../frontend/flutter_dart/patterns/cli_commands_flutter.md) — Referencia de comandos onion para frontend
