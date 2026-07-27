---
type: Pattern
title: Patrón de Modelos (Base/Create/Update/InDb)
description: Cuatro clases Pydantic por entidad: Base con campos compartidos, Create para POST, Update para PATCH, InDb para respuestas.
tags: [fastapi, pydantic, modelos, crud]
---

# Patrón de Modelos

Cada entidad define 4 clases Pydantic en `repos/v1/{entity}/models/{entity}_model.py`.

> **CLI:** El CLI genera clases vacías (campos a completar). El desarrollador agrega los campos reales con sus tipos y validaciones (`Field`, `String`, etc.).

## Estructura

```python
from pydantic import BaseModel

class BaseProduct(BaseModel):
    name: str
    description: str | None = None
    price: float = Field(..., gt=0)

class CreateProduct(BaseProduct):
    """Hereda todos los campos de Base, requeridos para creación."""
    pass

class UpdateProduct(BaseModel):
    """Todos los campos opcionales para PATCH."""
    name: str | None = None
    description: str | None = None
    price: float | None = Field(None, gt=0)

class ProductInDb(BaseProduct):
    """Hereda Base + campos de infraestructura."""
    id: str
    createdAt: int
    updatedAt: int | None = None
    isDeleted: bool = False
```

## Convenciones

| Clase | Propósito | Uso |
|-------|-----------|-----|
| `Base{Entity}` | Campos compartidos entre creación y lectura | Base para Create e InDb |
| `Create{Entity}` | Campos requeridos para crear | Request body del POST |
| `Update{Entity}` | Todos opcionales | Request body del PATCH |
| `{Entity}InDb` | Modelo completo incluyendo id/timestamps | Response de todos los endpoints |

## Reglas

- `Create{Entity}` hereda de `Base{Entity}` — usa `pass` si no hay lógica extra
- `Update{Entity}` **no** hereda de Base — todos los campos son opcionales (`None`) para merge parcial
- `{Entity}InDb` usa `model_validate()` desde dicts de MongoDB
- `Update{Entity}` usa `model_dump(exclude_unset=True)` para solo enviar campos modificados
