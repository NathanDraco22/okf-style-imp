---
type: StyleGuide
title: Estructura del Proyecto
description: Organización de carpetas y propósito de cada directorio en un proyecto FastAPI Onion.
tags: [estilo, estructura, proyecto, python]
---

# Estructura del Proyecto

## Árbol estándar

```
app/
├── main.py                           # Entry point: FastAPI(), include_router, CORS
├── app_lifespan.py                   # Startup: init_services(), scheduler
├── regis_exception_handlers.py       # Mapeo excepciones → HTTP
├── .env / .env.example               # Variables de entorno
│
├── api/                              # Capa HTTP
│   └── v{version}/
│       ├── router.py                 # Router maestro de la versión
│       └── {entity}/
│           ├── {entity}_router.py    # Endpoints
│           ├── {entity}_controller.py # Orquestación
│           └── {entity}_schemas.py   # Schemas extra (request bodies custom)
│
├── config/
│   ├── onion-config.toml             # Manifest de collections (auto-manejado)
│   └── service_public_key.pem        # Llaves, configs externas
│
├── core/                             # Lógica de negocio pura
│   ├── services_initializer.py       # init_services()
│   ├── exceptions/exceptions.py      # Jerarquía de excepciones de dominio
│   └── {domain}/                     # Módulos de negocio (security, inventory, etc.)
│
├── repos/                            # Capa de acceso a datos
│   └── v{version}/
│       ├── common/                   # Modelos y filtros compartidos entre entidades
│       │   ├── models/
│       │   └── filter_params/
│       └── {entity}/
│           ├── __init__.py           # Re-exporta símbolos públicos
│           ├── {entity}_repository.py
│           ├── data/
│           │   └── {entity}_datasource.py
│           └── models/
│               └── {entity}_model.py
│
├── responses/                        # Schemas de respuesta globales
│   └── v{version}/
│       └── list_response.py
│
├── services/                         # Infraestructura
│   ├── mongo_service.py              # Cliente MongoDB singleton
│   ├── base_mongo_collection.py      # Clase base para collections
│   └── mongo_collections/
│       └── v{version}/
│           └── {entity}_collection.py
│
├── middleware/                        # Middleware custom
│   └── depends/
│       ├── auth_middleware.py
│       └── multi-tenant.py
│
└── tools/                            # Utilidades transversales
    ├── time_tools.py                 # Timestamps Unix ms
    ├── uuid_tool.py                  # Generación de UUIDs
    └── search_tools.py               # generate_prefixes() para búsqueda
```

## Paquete de negocio separado (opcional)

Para proyectos grandes, la lógica de negocio vive en un paquete aparte:

```
packages/
└── kardex-business/
    ├── pyproject.toml
    └── src/kardex/
        ├── repos/v{version}/       # Data access
        ├── services/                # MongoDB + collections
        ├── core/                    # Lógica de negocio
        └── tools/                   # Utilidades
```
