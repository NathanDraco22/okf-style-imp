---
type: Pattern
title: Funciones de Flujo (Action Flows)
description: Las acciones complejas de pantalla (cobrar, completar venta, confirmar salida) se extraen a funciones top-level que orquestan diálogos, validaciones y cubits, dejando los widgets libres de lógica.
tags: [flutter, acciones, flujos, dialogos, patron, ui]
---

# Funciones de Flujo (Action Flows)

Cuando una acción de la UI requiere varios pasos (validar → confirmar con diálogo → disparar cubit → manejar resultado), se extrae a una **función top-level** que orquesta el flujo completo. Los widgets solo llaman a la función con `context`.

## Ejemplo: flujo de pago

```dart
Future<void> startTerminalPayment(BuildContext context) async {
  final viewController = SaleTerminalMediator.of(context).viewController;
  if (viewController.items.isEmpty) return;

  if (viewController.hasItemsWithZeroQuantity) {
    await DialogManager.showInfoDialog(
      context,
      "Hay productos sin cantidad. Revisa la venta antes de cobrar.",
    );
    return;
  }

  if (viewController.hasZeroPriceItems) {
    await DialogManager.showErrorDialog(
      context,
      "No se puede cobrar porque hay productos con precio en 0.",
    );
    return;
  }

  final createInvoice = viewController.generateCreateInvoice();
  if (createInvoice == null) return;

  final paymentResult = await showTerminalPaymentConfirmationDialog(
    context,
    createInvoice.total,
    showCredit: hasCredit,
    initialMethod: viewController.selectedPaymentType,
  );
  if (paymentResult == null) return;
  if (!context.mounted) return;

  viewController.description = paymentResult.observations ?? '';
  final invoice = viewController.generateCreateInvoice();
  if (invoice == null) return;

  final finalInvoice = invoice.copyWithPaymentMethod(
    paymentResult.method,
    paymentResult.reference,
  );

  context.read<WriteTerminalInvoiceCubit>().createSaleInvoice(finalInvoice);
}
```

## Flujo de completación (manejo de éxito)

```dart
Future<void> handleTerminalSaleCompletion(BuildContext context, InvoiceInDb invoice) async {
  await showInvoiceViewerDialog(context, invoice);

  if (!context.mounted) return;
  SaleTerminalMediator.of(context).viewController.clear();
}
```

## Guard de salida con cambios sin guardar

```dart
leading: BackButton(
  onPressed: () async {
    if (viewController.items.isNotEmpty) {
      final res = await DialogManager.confirmActionDialog(
        context,
        "Deseas salir sin guardar?",
      );
      if (res != true) return;
    }
    if (!context.mounted) return;
    context.pop();
  },
),
```

## Orquestación en el widget raíz

Las funciones de flujo se disparan desde un `BlocListener` único en la raíz (ver [Estructura de Pantalla](screen_structure.md)) — nunca desde los botones:

```dart
BlocListener<WriteTerminalInvoiceCubit, WriteTerminalInvoiceState>(
  listener: (context, state) async {
    if (state is WriteTerminalInvoiceInProgress) {
      LoadingDialogManager.showLoadingDialog(context);
    }
    if (state is WriteTerminalInvoiceError) {
      LoadingDialogManager.closeLoadingDialog(context);
      DialogManager.showErrorDialog(context, state.message);
    }
    if (state is WriteTerminalInvoiceSuccess) {
      LoadingDialogManager.closeLoadingDialog(context);
      await handleTerminalSaleCompletion(context, state.invoice);
    }
  },
  child: ...,
)
```

## Reglas

- **Una función top-level por flujo** — `startTerminalPayment`, `handleTerminalSaleCompletion`, etc.
- **Vive en el módulo** (archivo propio: `payment_action.dart`, `sale_completion.dart`), no en la sección que la usa
- **El ViewController se lee del Mediator** (`XxxMediator.of(context).viewController`), nunca del cubit — ver [ViewController Pattern](view_controller_pattern.md)
- **Validaciones y diálogos dentro del flujo** — el widget no conoce la lógica, solo llama la función
- **Los diálogos se invocan como show functions** (`showXxx(context)`) — ver [Patrón Dialog/Modal](dialog_show_functions.md)
- **`context.mounted` después de cada `await`** antes de usar el context
- **Los side effects de escritura se orquestan en la raíz** (BlocListener), los flujos solo disparan la mutación
- **Guard de salida** — confirmar antes de abandonar pantallas con datos sin guardar
- Las funciones reciben `BuildContext` y leen dependencias con `context.read`
