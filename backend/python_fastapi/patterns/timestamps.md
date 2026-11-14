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

## Fechas anteriores a 1970 (epoch negativo)

El timestamp Unix en milisegundos es un **entero con signo**. Toda fecha anterior a `1970-01-01T00:00:00Z` produce un valor negativo, y ese valor es **válido**: no es un error de datos ni un overflow.

Ejemplo real: un paciente de 57 años (nacido el 13 de mayo de 1969) se envía como `birthDate = -20_131_200_000`. Si el modelo valida esa fecha con `gt=0`, la API responde `422 Input should be greater than 0` y el registro no se puede crear.

| Fecha (UTC) | Milisegundos Unix |
|---|---|
| 1900-01-01 | `-2_208_988_800_000` |
| 1969-01-01 | `-31_536_000_000` |
| 1969-05-13 | `-20_131_200_000` |
| 1970-01-01 | `0` |
| 2026-11-13 | `1_794_528_000_000` |

### Auditoría vs. fecha de dominio

| Campo | Origen | ¿Puede ser negativo? |
|---|---|---|
| `createdAt` / `updatedAt` | `TimeTools.get_now_in_milliseconds()` | No: siempre es "ahora" |
| `birthDate`, `startDate`, `endDate`, fechas de documentos | Dato ingresado por el usuario | **Sí**, si es anterior a 1970 |

La regla `> 0` solo tiene sentido para marcas de tiempo generadas por el sistema. Las fechas de dominio se validan por **rango plausible**, no por signo.

### Validación correcta en Pydantic

```python
from pydantic import BaseModel, Field, field_validator

EPOCH_MIN_MS = -2_208_988_800_000  # 1900-01-01T00:00:00Z

class BasePatient(BaseModel):
    # ❌ Rechaza a cualquier nacido antes de 1970
    # birthDate: int = Field(..., gt=0)

    # ✅ Cota inferior real: nada anterior a 1900
    birthDate: int = Field(..., ge=EPOCH_MIN_MS)

    @field_validator("birthDate")
    @classmethod
    def validate_not_future(cls, value: int) -> int:
        # ✅ Cota superior dinámica: se evalúa en cada request
        if value > TimeTools.get_now_in_milliseconds():
            raise ValueError("birthDate no puede estar en el futuro")
        return value
```

### Reglas

- **No** aplicar `gt=0` / `ge=0` a fechas de dominio; usar un rango como `[EPOCH_MIN_MS, now]`.
- **No** normalizar con `abs()`: un nacimiento de 1969 se convertiría en una fecha futura.
- **No** hacer cast a unsigned (`uint64`, `unsigned int`): el negativo se transforma en un valor enorme y positivo.
- **No** usar `0` como centinela de "sin fecha"; usar `None` (`birthDate: int | None = None`).
- Los filtros de rango (`$gte` / `$lte`) funcionan con negativos; evitar condiciones implícitas como `{"birthDate": {"$gt": 0}}`.
- MongoDB almacena el valor como `int64` y ordena correctamente los negativos.
- En tests, incluir una fecha previa a 1970 (p. ej. `-20_131_200_000`) para cubrir la regresión.

### Checklist de revisión

1. ¿El campo es fecha de auditoría o fecha de dominio?
2. ¿Hay `gt=0`, `ge=0` o `$gt: 0` sobre una fecha de dominio?
3. ¿Se usa `abs()` o un cast unsigned en algún punto del flujo?
4. ¿El date picker del cliente permite seleccionar fechas anteriores a 1970?
