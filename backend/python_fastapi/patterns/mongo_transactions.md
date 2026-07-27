---
type: Pattern
title: Transacciones MongoDB
description: Uso de session.start_transaction() para operaciones críticas que requieren atomicidad (facturación, pagos, movimientos de inventario).
tags: [mongodb, transacciones, atomicidad, patron]
---

# Transacciones MongoDB

Para operaciones que modifican múltiples colecciones de forma atómica (facturas, pagos, entradas/salidas de inventario).

## Estructura

```python
async def process_invoice(invoice_data: dict) -> InvoiceResult:
    client = MongoService().get_client()
    session = await client.start_session()
    try:
        async with session.start_transaction():
            # 1. Validar estado
            # 2. Crear factura
            # 3. Actualizar inventario
            # 4. Actualizar saldo del cliente
            # 5. Actualizar contadores
            await session.commit_transaction()
            return result
    except Exception:
        await session.abort_transaction()
        raise
    finally:
        session.end_session()
```

## Retry en write conflicts

```python
class PaymentManager:
    async def pay(self, pay_intent: PayIntent) -> PaymentResult:
        max_retries = 5
        for attempt in range(max_retries):
            try:
                return await self._process_payment(pay_intent)
            except WriteConflict as e:
                if attempt == max_retries - 1:
                    raise
                await asyncio.sleep(0.1 * (attempt + 1))
```

## ¿Cuándo se usan?

| Operación | Colecciones involucradas |
|-----------|------------------------|
| Crear factura | Invoices, Products (stock), Clients (saldo), Counters |
| Procesar pago | Orders (status), CashRegisters (balance), CashTransactions |
| Entrada de inventario | EntryDocs, Products (stock), EntryHistories, ExpirationLogs |
| Salida de inventario | ExitDocs, Products (stock), ExitHistories |

## Reglas

- Requiere **MongoDB replica set** (no funciona en standalone)
- Usar `async with session.start_transaction()` para manejo automático
- Implementar **retry con backoff** para `WriteConflict` (error 112)
- Mantener transacciones **cortas** — mínimo de operaciones dentro de la sesión
