---
type: Pattern
title: Manejo de Excepciones
description: Jerarquía de excepciones de dominio mapeadas a códigos HTTP mediante un register global.
tags: [fastapi, excepciones, errores, handler, patron]
---

# Manejo de Excepciones

Se usa una jerarquía de excepciones de dominio y un `register_exception_handlers` global para mapearlas a HTTP.

## Jerarquía

```python
# core/exceptions/exceptions.py

class AppException(Exception):
    """Base de todas las excepciones de dominio."""
    pass

class NotFoundException(AppException):
    """Recurso no encontrado."""
    pass

class DuplicateException(AppException):
    """Entidad duplicada (código, nombre, etc.)."""
    pass

class BusinessRuleException(AppException):
    """Violación de regla de negocio."""
    pass

class UnauthorizedException(AppException):
    """Autenticación fallida."""
    pass
```

## Mapeo a HTTP

```python
# register_exception_handlers.py

def register_exception_handlers(app: FastAPI):
    @app.exception_handler(NotFoundException)
    async def not_found_handler(request, exc):
        return JSONResponse(status_code=404, content={"detail": str(exc)})

    @app.exception_handler(DuplicateException)
    async def duplicate_handler(request, exc):
        return JSONResponse(status_code=409, content={"detail": str(exc)})

    @app.exception_handler(BusinessRuleException)
    async def business_rule_handler(request, exc):
        return JSONResponse(status_code=400, content={"detail": str(exc)})
```

## Uso en Controllers / Core

```python
# En lugar de HTTPException directamente:
raise NotFoundException("Item not found")
raise DuplicateException("Item code already exists")
raise BusinessRuleException("Cannot delete product with active stock")
```

## Reglas

- Las excepciones de dominio se definen en `core/exceptions/exceptions.py`
- El registro de handlers se hace en `main.py` vía `register_exception_handlers(app)`
- Los Controllers pueden usar `HTTPException` directo o las excepciones de dominio
- El Core (capa de negocio) **solo** usa excepciones de dominio, nunca HTTPException
