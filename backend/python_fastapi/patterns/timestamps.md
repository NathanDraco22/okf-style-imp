---
type: Pattern
title: Timestamps en Unix Milliseconds
description: Todos los timestamps se almacenan como enteros de milisegundos Unix (epoch) usando TimeTools.
tags: [fastapi, timestamps, unix, patron]
---

# Timestamps

Las fechas se almacenan como enteros de milisegundos desde Unix epoch (1970-01-01). No se usan objetos datetime ni strings ISO en la base de datos.

## TimeTools

```python
from datetime import datetime, timezone

class TimeTools:
    @staticmethod
    def get_now_in_milliseconds() -> int:
        return int(datetime.now(timezone.utc).timestamp() * 1000)
```

## Uso en Repository

```python
# Create
new_product = ProductInDb(
    id=UuidTool.generate_uuid(),
    createdAt=TimeTools.get_now_in_milliseconds(),
    updatedAt=None,  # explícitamente None en creación
    **create_product.model_dump(),
)

# Update
updated_data = product.model_dump(exclude_unset=True)
updated_data["updatedAt"] = TimeTools.get_now_in_milliseconds()
```

## Convenciones

| Campo | Tipo | Cuándo se asigna |
|-------|------|------------------|
| `createdAt` | `int` | Solo en creación (una vez) |
| `updatedAt` | `int \| None` | `None` en create, siempre se inyecta en update |
