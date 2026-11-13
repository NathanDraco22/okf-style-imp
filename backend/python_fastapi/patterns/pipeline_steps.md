---
type: Pattern
title: Pipeline-Steps para Procesos Complejos
description: Orquestación de procesos multi-paso con un contexto compartido, pasos independientes y un resultado final tipado.
tags: [fastapi, python, pipeline, steps, context, orquestacion, transacciones, patron]
---

# Pipeline-Steps para Procesos Complejos

Cuando un caso de uso combina validación, lecturas, varios documentos y movimientos transaccionales, una sola función de proceso termina mezclando demasiadas responsabilidades:

```text
validar → cargar datos → preparar documentos → ejecutar inventario →
persistir documentos → actualizar estado → construir respuesta
```

El patrón **Pipeline-Steps** separa ese proceso en pasos pequeños e independientes que comparten un **contexto**. El pipeline define el orden; el orquestador define la transacción y el resultado final.

> **CLI:** el CRUD base se genera con Onion CLI (`onion crud-mongo {entity} --version 1`). El pipeline es lógica de orquestación y se implementa siguiendo este patrón; el CLI no genera los steps.

## Cuándo usarlo

- Procesos con **3 o más fases** claramente diferenciadas.
- Varias escrituras que deben ser atómicas dentro de una misma transacción.
- Flujos con ramas opcionales (ej. entrada/salida, contado/crédito).
- Procesos que necesitan trazabilidad de pasos ejecutados u omitidos.
- Casos donde se quiere testear cada paso sin levantar toda la infraestructura.

## Cuándo no usarlo

- CRUD simple de una sola entidad.
- Operaciones de un solo documento que no requieren varios pasos.
- Procesos con efectos externos irreversibles dentro de la transacción (correos, HTTP, colas). Esos efectos van **después del commit**.

## Estructura

```text
core/{process}/
├── {process}.py                    # Orquestador + transacción
└── pipeline/
    ├── base.py                     # Pipeline, PipelineStep, PipelineResult
    ├── context.py                  # Contexto + resultado de dominio
    ├── build_pipeline.py           # Orden y cableado de dependencias
    └── steps/
        ├── validation_steps.py     # Validaciones y carga inicial
        ├── {domain}_steps.py       # Pasos propios del proceso
        └── finalize_steps.py       # Persistencia final y resultado
```

> Si dos o más procesos adoptan el patrón, extraer `base.py` a `core/pipeline/base.py` y reutilizarlo.

## Contratos base

```python
from abc import ABC, abstractmethod
from collections.abc import Sequence
from dataclasses import dataclass
from typing import Generic, TypeVar

TContext = TypeVar("TContext")


class PipelineStep(ABC, Generic[TContext]):
    @property
    def name(self) -> str:
        return type(self).__name__

    def should_execute(self, context: TContext) -> bool:
        """Permite omitir el paso cuando el contexto no tiene trabajo para él."""
        return True

    @abstractmethod
    async def execute(self, context: TContext) -> None:
        raise NotImplementedError


@dataclass
class PipelineResult(Generic[TContext]):
    context: TContext
    executed_steps: list[str]
    skipped_steps: list[str]


class Pipeline(Generic[TContext]):
    def __init__(self, steps: Sequence[PipelineStep[TContext]]):
        self.steps = list(steps)

    async def run(self, context: TContext) -> PipelineResult[TContext]:
        executed_steps: list[str] = []
        skipped_steps: list[str] = []

        for step in self.steps:
            if not step.should_execute(context):
                skipped_steps.append(step.name)
                continue

            await step.execute(context)
            executed_steps.append(step.name)

        return PipelineResult(
            context=context,
            executed_steps=executed_steps,
            skipped_steps=skipped_steps,
        )
```

- `PipelineResult` es el resultado de **ejecución**: contexto + pasos ejecutados/omitidos.
- El resultado de **dominio** vive en `context.result` y lo construye el último paso.

## Contexto

El contexto es un `dataclass` que transporta datos, no dependencias:

```python
# Imports de dominio omitidos por brevedad.
from dataclasses import dataclass, field
from typing import Any


@dataclass
class OrderApprovalResult:
    order: OrderInDb
    payment_id: str | None = None
    invoice_id: str | None = None


@dataclass
class OrderApprovalContext:
    # Entradas
    order: OrderInDb
    approved_by: UserInfo
    session: Any

    # Datos intermedios
    now: int = field(default_factory=TimeTools.get_now_in_milliseconds)
    customer: CustomerInDb | None = None
    payment_id: str | None = None
    invoice_id: str | None = None

    # Resultado
    updated_order: OrderInDb | None = None
    result: OrderApprovalResult | None = None
```

Reglas del contexto:

- Contiene entradas, estado intermedio, el `session` de la transacción y el resultado.
- **No** contiene colecciones, repositorios ni servicios globales.
- Las dependencias se inyectan en cada step por constructor.
- El `session` es la única pieza de infraestructura que viaja en el contexto porque conecta todos los steps con la misma transacción.
- Cada paso escribe solo los campos que le corresponden.

## Steps

Un step tiene una sola responsabilidad y no conoce a los demás pasos:

```python
# Imports de dominio omitidos por brevedad.
class ValidateOrderStep(PipelineStep[OrderApprovalContext]):
    async def execute(self, context: OrderApprovalContext) -> None:
        if context.order.status != "pending":
            raise BusinessRuleException("ORDER_NOT_PENDING")

        if context.approved_by.id == context.order.createdBy.id:
            raise BusinessRuleException("SELF_APPROVAL_NOT_ALLOWED")


class LoadCustomerStep(PipelineStep[OrderApprovalContext]):
    def __init__(self, customers_repo: CustomersRepository):
        self.customers_repo = customers_repo

    async def execute(self, context: OrderApprovalContext) -> None:
        customer = await self.customers_repo.get_customer_by_id(
            context.order.customerId
        )
        if customer is None:
            raise NotFoundException("CUSTOMER_NOT_FOUND")

        context.customer = customer


class CreatePaymentRecordStep(PipelineStep[OrderApprovalContext]):
    def __init__(self, payments_col: PaymentsCollection):
        self.payments_col = payments_col

    def should_execute(self, context: OrderApprovalContext) -> bool:
        return context.order.paymentType == "credit"

    async def execute(self, context: OrderApprovalContext) -> None:
        context.payment_id = await self.payments_col.create_payment_with_session(
            context.session,
            order_id=context.order.id,
            amount=context.order.total,
        )


class BuildResultStep(PipelineStep[OrderApprovalContext]):
    async def execute(self, context: OrderApprovalContext) -> None:
        if context.updated_order is None:
            raise BusinessRuleException("PIPELINE_RESULT_MISSING_ORDER")

        context.result = OrderApprovalResult(
            order=context.updated_order,
            payment_id=context.payment_id,
            invoice_id=context.invoice_id,
        )
```

Convenciones de los steps:

- `async execute(context)`.
- Sin `commit`, `abort` ni apertura de sesiones.
- Sin llamadas directas a otros steps.
- Dependencias por constructor.
- `should_execute` cuando el paso es opcional.
- Errores de dominio tipados; se propagan para abortar la transacción.

## Orquestador

El orquestador abre la transacción, construye el contexto, ejecuta el pipeline y devuelve el resultado de dominio:

```python
# Imports de dominio omitidos por brevedad.
async def approve_order(order: OrderInDb, approved_by: UserInfo) -> OrderInDb:
    async with MongoService().client.start_session() as session:
        async with await session.start_transaction():
            context = OrderApprovalContext(
                order=order,
                approved_by=approved_by,
                session=session,
            )

            pipeline_result = await build_order_approval_pipeline().run(context)
            result = pipeline_result.context.result

            if result is None:
                raise BusinessRuleException("PIPELINE_RESULT_MISSING")

            return result.order
```

El mutex, el retry por write conflict y la captura de errores quedan **fuera** del pipeline, normalmente en un `Manager`:

```python
async with adjust_mutex:
    for attempt in range(max_retries):
        try:
            return await approve_order(order, approved_by)
        except OperationFailure as error:
            if error.code != 112:  # WriteConflict
                raise
            await asyncio.sleep(0.3 * (attempt + 1))
```

## Builder del pipeline

El builder concentra el orden y el cableado de dependencias:

```python
# Imports de dominio omitidos por brevedad.
def build_order_approval_pipeline() -> Pipeline[OrderApprovalContext]:
    return Pipeline(
        [
            ValidateOrderStep(),
            LoadCustomerStep(CustomersRepository(CustomersDataSource())),
            CreatePaymentRecordStep(PaymentsCollection.get_instance()),
            PersistInvoiceStep(InvoicesCollection.get_instance()),
            ApproveOrderStep(OrdersCollection.get_instance()),
            BuildResultStep(),
        ]
    )
```

El orden es explícito y testeable. Los pasos opcionales se omiten con `should_execute` sin llenar el pipeline de `if`.

`PersistInvoiceStep` y `ApproveOrderStep` siguen el mismo contrato: reciben su colección por constructor y escriben usando `context.session`.

## Resultado y observabilidad

- `PipelineResult.executed_steps` y `skipped_steps` sirven para logs, métricas y tests.
- `context.result` contiene el resultado de dominio del proceso.
- El orquestador puede devolver solo lo que necesita la API (`result.order`) o el resultado completo si el llamador necesita más datos.

## Reglas

- La transacción vive en el orquestador, nunca en un step.
- Los steps no hacen commit ni abort.
- El contexto solo contiene datos; las dependencias van por constructor.
- Un step no llama directamente a otro step.
- Los pasos opcionales usan `should_execute`.
- Los efectos externos (notificaciones, emails, HTTP) se ejecutan **después** del commit.
- Los errores de dominio se propagan; no se silencian dentro del pipeline.
- Mantener cada step pequeño y con una sola responsabilidad.
- Los nombres de los steps deben describir la acción: `Validate...`, `Load...`, `Prepare...`, `Execute...`, `Persist...`, `Build...`.

## Anti-patrones

| Anti-patrón | Por qué evitarlo |
|---|---|
| Step que abre su propia transacción | Rompe la atomicidad del proceso completo |
| Step que llama a otro step | Vuelve a acoplar el flujo y dificulta el testing |
| Contexto con colecciones o repositorios | Convierte el contexto en un service locator |
| Lógica de negocio en el orquestador | El orquestador debe orquestar, no decidir reglas |
| Email/HTTP dentro de la transacción | Un fallo externo revierte operaciones ya válidas o deja efectos inconsistentes |
| Silenciar excepciones en un step | El pipeline continúa con datos incompletos |

## Testing

**Pipeline con pasos falsos**

```python
import pytest

# Imports de dominio omitidos por brevedad.
class RecordingStep(PipelineStep[list[str]]):
    def __init__(self, label: str, run: bool = True):
        self.label = label
        self.run = run

    @property
    def name(self) -> str:
        return self.label

    def should_execute(self, context: list[str]) -> bool:
        return self.run

    async def execute(self, context: list[str]) -> None:
        context.append(self.label)


@pytest.mark.asyncio
async def test_pipeline_order_and_skip():
    context: list[str] = []
    pipeline = Pipeline(
        [RecordingStep("validate"), RecordingStep("payment", run=False)]
    )

    result = await pipeline.run(context)

    assert context == ["validate"]
    assert result.executed_steps == ["validate"]
    assert result.skipped_steps == ["payment"]
```

**Steps individuales**

- Construir un contexto con datos falsos.
- Usar repositorios/colecciones fake inyectados por constructor.
- Verificar campos del contexto y excepciones de dominio.

**Integración**

- Probar el orquestador contra MongoDB con replica set.
- Verificar que un fallo en un step intermedio no persiste documentos ni movimientos.
- Verificar el resultado final y los pasos ejecutados.

## Relación con otros patrones

- [Transacciones MongoDB](mongo_transactions.md) — límite transaccional del orquestador.
- [CRUD en Repository](repository_crud.md) — los steps reutilizan repositorios.
- [Manejo de Excepciones](exception_handling.md) — errores de dominio que abortan el pipeline.
- [Flujo Onion](onion_flow.md) — el pipeline vive en `core/`, entre el controller y los repositorios.

## Implementación de referencia

El proyecto **Kardex App** implementa este patrón en el proceso de aprobación de solicitudes de ajuste de inventario:

```text
kardex-server/packages/kardex-business/src/kardex/core/adjustment_requests/
├── process_adjustment_request.py       # ~34 líneas; solo transacción + pipeline
└── pipeline/
    ├── base.py
    ├── context.py
    ├── build_pipeline.py
    └── steps/
        ├── request_steps.py
        ├── exit_steps.py
        ├── entry_steps.py
        └── finalize_steps.py
```

Pasos reales:

```text
ValidateRequest → SplitItems → LoadBranch → LoadProducts → LoadAccounts
→ PrepareExit → ExecuteExit → PersistExitDocument
→ PrepareEntry → ExecuteEntry → PersistEntryDocument
→ ApproveRequest → BuildResult
```

Tests: `kardex-server/test/test_adjustment_request_pipeline.py`.

La implementación real usa el patrón para un dominio específico; este documento mantiene el ejemplo genérico (`OrderApproval`) para que el patrón sea reutilizable en cualquier proyecto.

## Checklist

- [ ] El proceso tiene 3+ fases o varias escrituras atómicas.
- [ ] El contexto solo contiene datos y el resultado de dominio.
- [ ] Cada step tiene una responsabilidad y dependencias inyectadas.
- [ ] Los pasos opcionales implementan `should_execute`.
- [ ] La transacción está en el orquestador.
- [ ] El retry/mutex está fuera del pipeline.
- [ ] Los efectos externos van después del commit.
- [ ] Hay tests de orden/skip y de los pasos críticos.
