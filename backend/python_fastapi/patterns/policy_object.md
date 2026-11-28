---
type: Pattern
title: Policy Object (Decisiones de Negocio Centralizadas)
description: Patrón para encapsular decisiones de negocio y validaciones puras en una clase inmutable sin I/O que recibe un contexto y devuelve bool, Enum, objeto opcional, objeto compuesto o resultado de validación con motivo.
tags: [fastapi, python, policy, decision, enum, dataclass, validation, core, patron]
---

# Policy Object (Decisiones de Negocio Centralizadas)

Patrón para sacar las decisiones de negocio y validaciones puras de routers, controllers, repositories o bloques de `if` dispersos, y concentrarlas en una clase pura que recibe contexto y responde con resultados tipados.

> **CLI:** el CRUD base se genera con Onion CLI (`onion crud-mongo {entity} --version 1`). Las policies son lógica de dominio pura en `core/policies/` y se implementan siguiendo este patrón.

## El problema

Cuando una regla de negocio depende de múltiples campos de una entidad o de configuraciones del sistema, es común que se repita en varios endpoints, repositorios o pasos de un pipeline:

```python
if reservation.room_type == "vip_suite":
    has_free_wine = True
    dinner_discount = 0.20
    can_buy_wine_extension = False
else:
    has_free_wine = False
    dinner_discount = 0.0
    can_buy_wine_extension = reservation.status == "confirmed"
```

Los síntomas típicos:

- condiciones repetidas en controllers, repositories, pipelines y generación de reportes;
- bloques con demasiados `if` anidados difíciles de leer y documentar;
- decisiones que producen varios datos y se recalculan en cada consumidor;
- tests que necesitan levantar MongoDB o mocks de repositorios solo para validar una regla de negocio;
- discrepancias entre lo que decide el backend y lo que muestra el frontend.

## La solución

Un **Policy Object** es una clase pura e inmutable que:

- recibe el contexto mínimo que necesita (entidad, configuración o datos ya cargados);
- agrupa una familia coherente de decisiones de negocio;
- devuelve resultados explícitos y tipados;
- **no hace I/O**, no conoce FastAPI, no consulta MongoDB y no muta estado.

El consumidor deja de saber cómo se combinan los campos y pregunta por la intención de negocio:

```python
policy = ReservationPolicy(reservation)
benefits = policy.benefits()

if benefits.has_free_wine:
    ...
```

Si mañana cambia la regla, se actualiza **en un solo lugar**.

## Regla de retorno

| Situación | Retorno | Ejemplo |
| :--- | :--- | :--- |
| Una decisión binaria | `bool` | `policy.can_enter_party()` |
| Varias decisiones excluyentes | `Enum` | `policy.access_level()` |
| Un resultado que puede existir o no | `T \| None` | `policy.wine_extension_offer()` |
| Un resultado compuesto que siempre existe | `@dataclass(frozen=True)` | `policy.benefits() -> ReservationBenefits` |
| Una validación pura que necesita explicar el rechazo | `PolicyDecision` (`@dataclass(frozen=True)`) | `policy.validate_wine_extension() -> PolicyDecision` |

### 1. Retorno `Enum` con comportamiento

Los `Enum` pueden encapsular propiedades para que el consumidor no vuelva a escribir `if` sobre ellos:

```python
from enum import Enum


class AccessLevel(Enum):
    DENIED = (False, False)
    STANDARD = (True, False)
    VIP = (True, True)

    def __init__(self, can_enter: bool, has_lounge_access: bool) -> None:
        self.can_enter = can_enter
        self.has_lounge_access = has_lounge_access
```

### 2. Objeto compuesto inmutable

Cuando una decisión produce múltiples consecuencias, devuelve un `@dataclass(frozen=True)` con solo los datos resultantes:

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class ReservationBenefits:
    has_free_wine: bool
    dinner_discount_percent: float
    can_buy_wine_extension: bool
```

### 3. Validación pura con motivo (`PolicyDecision`)

Cuando `bool` no es suficiente porque la API o el flujo necesita saber **por qué** se rechazó una acción, la policy devuelve un objeto de decisión en lugar de lanzar excepciones HTTP:

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class PolicyDecision:
    allowed: bool
    code: str | None = None
    reason: str | None = None

    @classmethod
    def allow(cls) -> "PolicyDecision":
        return cls(allowed=True)

    @classmethod
    def deny(cls, code: str, reason: str) -> "PolicyDecision":
        return cls(allowed=False, code=code, reason=reason)
```

> **Ventaja:** la policy sigue siendo una función pura que retorna datos; quien orquesta el flujo (controller, repository o pipeline step) decide si traduce un rechazo a `BusinessRuleException` o lo retorna como parte de una respuesta.

## Ejemplo completo

```python
from dataclasses import dataclass
from typing import Protocol


class ReservationContext(Protocol):
    room_type: str
    status: str
    guest_age: int
    nights: int


@dataclass(frozen=True)
class ReservationBenefits:
    has_free_wine: bool
    dinner_discount_percent: float
    can_buy_wine_extension: bool


@dataclass(frozen=True)
class WineExtensionOffer:
    price_in_cents: int
    bottles_included: int


@dataclass(frozen=True)
class AgePolicy:
    age: int

    def is_adult(self) -> bool:
        return self.age >= 18


@dataclass(frozen=True)
class ReservationPolicy:
    reservation: ReservationContext

    def is_vip(self) -> bool:
        return self.reservation.room_type == "vip_suite"

    def benefits(self) -> ReservationBenefits:
        if self.is_vip():
            return ReservationBenefits(
                has_free_wine=True,
                dinner_discount_percent=20.0,
                can_buy_wine_extension=False,
            )

        return ReservationBenefits(
            has_free_wine=False,
            dinner_discount_percent=0.0,
            can_buy_wine_extension=self.reservation.status == "confirmed",
        )

    def wine_extension_offer(self) -> WineExtensionOffer | None:
        if not self.benefits().can_buy_wine_extension:
            return None
        return WineExtensionOffer(
            price_in_cents=2500,
            bottles_included=max(1, self.reservation.nights),
        )

    def validate_wine_extension_purchase(self) -> PolicyDecision:
        if not AgePolicy(self.reservation.guest_age).is_adult():
            return PolicyDecision.deny(
                code="GUEST_UNDERAGE",
                reason="El huésped debe tener al menos 18 años.",
            )
        if not self.benefits().can_buy_wine_extension:
            return PolicyDecision.deny(
                code="WINE_EXTENSION_NOT_ELIGIBLE",
                reason="La reservación no es elegible para el paquete de vino.",
            )
        return PolicyDecision.allow()
```

## Composición de Policies

Una policy puede delegar reglas transversales en policies más pequeñas (como `AgePolicy` dentro de `ReservationPolicy`). Esto mantiene cada regla atómica, reutilizable y fácil de leer sin duplicar condiciones entre dominios.

Reglas de composición:

- La policy compuesta sigue siendo 100% pura (sin I/O).
- Pasa únicamente los datos mínimos que necesita la sub-policy.
- Evita dependencias circulares entre policies.

## Dónde vive

En la capa de dominio puro `core/policies/`, agrupada por dominio:

```text
core/
└── policies/
    ├── common/
    │   └── policy_decision.py          # PolicyDecision reutilizable
    └── reservations/
        ├── __init__.py
        ├── reservation_policy.py       # Clase Policy principal
        └── reservation_benefits.py     # Objetos de resultado / Enums
```

## Integración con Arquitectura Onion y Pipeline-Steps

La separación de responsabilidades es estricta:

- **Policy (`core/policies/`):** decisión pura en memoria; recibe modelos ya cargados y retorna `bool`, `Enum`, `T | None`, `@dataclass` o `PolicyDecision`.
- **Repository / Controller / PipelineStep:** realiza el I/O (lee de MongoDB, abre transacciones), instancia la Policy con los datos obtenidos y actúa según el resultado.

Ejemplo en un `PipelineStep` o `Repository`:

```python
class ValidateWineExtensionStep(PipelineStep[WinePurchaseContext]):
    async def execute(self, context: WinePurchaseContext) -> None:
        decision = ReservationPolicy(context.reservation).validate_wine_extension_purchase()
        if not decision.allowed:
            raise BusinessRuleException(decision.code or "WINE_EXTENSION_DENIED")
```

## Cuándo usarlo

- La misma decisión o validación se repite en varios endpoints, repositorios o steps.
- Hay demasiados `if` anidados o combinaciones de banderas.
- Una decisión devuelve múltiples datos relacionados, no solo un booleano.
- Se requiere testear la tabla de decisión completa sin levantar FastAPI ni MongoDB.
- La misma regla existe en frontend y backend y conviene compartir exactamente la misma tabla de decisión.

## Cuándo no usarlo

- Un `if` trivial y local que no se repite ni tiene semántica de dominio reutilizable.
- Lógica que necesita consultar base de datos, abrir transacciones o llamar servicios externos: eso pertenece a un **Repository** o a un **Core / Orchestrator** (ver [Pipeline-Steps](pipeline_steps.md)), que luego delega la decisión pura a la Policy.

## Anti-patrones

| Anti-patrón | Por qué evitarlo |
| :--- | :--- |
| Policy que consulta MongoDB o llama repositorios | Rompe la pureza, obliga a usar `async` y mocks en tests unitarios |
| Policy que importa `HTTPException` de FastAPI | Acopla el dominio al transporte HTTP |
| Policy que muta la entidad recibida | Introduce efectos secundarios ocultos y dificulta razonar sobre el flujo |
| Policy singleton con estado global mutable | Pierde determinismo en pruebas y concurrencia |
| Lanzar excepciones de infraestructura dentro de la Policy | Impide reutilizar la decisión en consultas, previews o validaciones múltiples |

## Testing

Al ser pura y determinista, toda la tabla de decisión se prueba con `pytest` y `@pytest.mark.parametrize` sin mocks ni base de datos:

```python
from types import SimpleNamespace
import pytest


@pytest.mark.parametrize(
    "room_type,status,expected_free_wine,expected_discount,expected_extension",
    [
        ("vip_suite", "confirmed", True, 20.0, False),
        ("standard", "confirmed", False, 0.0, True),
        ("standard", "pending", False, 0.0, False),
    ],
)
def test_reservation_benefits_matrix(
    room_type: str,
    status: str,
    expected_free_wine: bool,
    expected_discount: float,
    expected_extension: bool,
) -> None:
    reservation = SimpleNamespace(
        room_type=room_type,
        status=status,
        guest_age=25,
        nights=2,
    )

    benefits = ReservationPolicy(reservation).benefits()

    assert benefits.has_free_wine is expected_free_wine
    assert benefits.dinner_discount_percent == expected_discount
    assert benefits.can_buy_wine_extension is expected_extension
```

## Relación con otros patrones

- [Policy Object en Flutter/Dart](../../../frontend/flutter_dart/patterns/policy_object.md) — contraparte frontend que comparte la misma taxonomía de retornos y tabla de decisión.
- [Pipeline-Steps](pipeline_steps.md) — los steps cargan datos y delegan las decisiones puras a la Policy.
- [Manejo de Excepciones](exception_handling.md) — traducción de `PolicyDecision.deny(...)` a `BusinessRuleException` en la capa de orquestación.
- [Flujo Onion](onion_flow.md) — las policies viven en `core/policies/`, independientes del transporte y la persistencia.

## Checklist de adopción

- [ ] La decisión se repite o combina múltiples condiciones de negocio.
- [ ] La clase Policy es inmutable (`@dataclass(frozen=True)`) y recibe solo el contexto mínimo.
- [ ] Se eligió el tipo de retorno adecuado (`bool`, `Enum`, `T | None`, objeto compuesto o `PolicyDecision`).
- [ ] Cero I/O, cero dependencias de FastAPI y cero mutaciones dentro de la Policy.
- [ ] Vive en `core/policies/<dominio>/`.
- [ ] Los consumidores (controllers, repositories, steps) cargan datos y delegan la decisión a la Policy.
- [ ] Existe test unitario parametrizado cubriendo la tabla de decisión completa.
- [ ] Si la regla también se refleja en UI, la tabla de decisión coincide con la Policy de Flutter.
