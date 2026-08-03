---
type: Pattern
title: Patrón Dialog/Modal (Show Functions)
description: Dialogs, modals y pantallas desechables se exponen como funciones top-level showXxx(context) → Future<T?> fuera del router. Categorías: viewers, inspectors, editors, searchers/selectors, bottom modals y managers como excepción.
tags: [flutter, dialog, modal, bottomsheet, show, patron, ui, reutilizable]
---

# Patrón Dialog/Modal (Show Functions)

Los diálogos, modales y pantallas desechables (ver/editar/buscar un recurso) **no son rutas de go_router**: se exponen como **funciones top-level** `showXxx(BuildContext context, ...)` que devuelven `Future<T?>`.

## Por qué show functions y no rutas

- **Reutilizables** — se invocan desde cualquier widget o [action flow](action_flow_functions.md) que tenga `context`, sin registrar rutas ni pasar parámetros por la URL
- **Contrato de resultado tipado** — `Future<T?>` con `null` = cancelar: el llamador decide sin callbacks ni estados
- **Componibles** — un buscador puede abrir un editor al seleccionar y devolver el resultado en cadena
- **El router queda limpio** — solo navegación real entre pantallas de la app (ver [GoRouter](go_router_pattern.md))

## Mecanismo según el caso

| Mecanismo | Cuándo | Ejemplo de firma |
|---|---|---|
| `showDialog<T>()` | Diálogos reales: editores con formulario, confirmaciones | `Future<UnitResult?> showUnitEditor(context, {initial})` |
| `showModalBottomSheet<T>()` | Bottom sheets: acciones rápidas, pickers, confirmaciones contextuales | `Future<bool?> showConfirmSendBottomModal(context, {...})` |
| `Navigator.push(MaterialPageRoute)` | Pantallas completas imperativas: viewers, buscadores, inspectors | `Future<void> showInvoiceViewer(context, invoice)` |

## Contrato de retorno

- `null` → cancelar/descartar. Los botones de cancelación hacen `context.pop(null)` o simplemente cierran
- Valor tipado → el resultado confirmado. Los modelos de resultado viven en `models/` con sufijo `{Entity}Result`:

```dart
class UnitResult {
  final List<Unit> units;
  final int defaultIndex;
  UnitResult({required this.units, required this.defaultIndex});
}
```

## Las 5 categorías (+ la excepción)

| Categoría | Propósito | Mecanismo | Carpeta | Firma |
|---|---|---|---|---|
| **Viewer** | Ver un recurso (solo lectura) | `push` | `dialogs/viewers/` | `Future<void> show{Entity}Viewer(context, entity)` |
| **Inspector** | Análisis detallado de un recurso | `push` | `dialogs/inspectors/` | `Future<void> show{Entity}Inspector(context, entity)` |
| **Editor** | Crear o editar (formulario) | `showDialog` | `dialogs/editors/` | `Future<{Entity}Result?> show{Entity}Editor(context, {initial...})` |
| **Searcher / Selector** | Buscar o elegir una entidad | `push` o `showDialog` | `dialogs/search_products/`, `dialogs/selectors/` | `Future<{Selected}?> show{Entity}Search(context, {...})` |
| **Bottom modal** | Acción contextual rápida | `showModalBottomSheet` | `modals/` (o `modals/` del módulo) | `Future<T?> show{Action}BottomModal(context, {...})` |
| **Manager** (excepción) | Feedback global: info, error, confirmar, loading | `showDialog` | `dialogs/dialog_manager.dart` | `DialogManager.confirmAction(context, msg)` |

La **única excepción** al patrón son los managers globales (`DialogManager`, `LoadingDialogManager`): clases estáticas de utilidad para confirmaciones, errores, info y loading. No son show functions por entidad, pero viven en `widgets/dialogs/` como el resto.

## Estructura de carpetas

```
lib/widgets/
├── dialogs/                        # Dialogs, modals y pantallas con función show
│   ├── dialog_manager.dart         # Excepción: managers globales (info/error/confirm)
│   ├── loading_dialog_manager.dart # Excepción: loading global
│   ├── viewers/                    # show{Entity}Viewer — solo lectura (push)
│   ├── inspectors/                 # show{Entity}Inspector — análisis detallado (push)
│   ├── editors/                    # show{Entity}Editor — crear/editar (showDialog)
│   ├── search_products/            # show{Entity}Search — buscadores (push)
│   ├── selectors/                  # show{Entity}Selector — selección de entidad
│   └── models/                     # {entity}_result.dart — modelos de resultado
└── modals/                         # Bottom sheets genéricos
    └── {action}_bottom_modal.dart  # show{Action}BottomModal
```

Los específicos de una feature viven **dentro del módulo** (`modules/{feature}/modals/` o `widgets/` del módulo), siguiendo [Reutilizar lo genérico, copiar lo que diverge](view_segmentation.md).

## Estructura del archivo

Función `show` + widget privado `_Xxx` en el mismo archivo:

```dart
// viewers/invoice_viewer.dart
Future<void> showInvoiceViewer(BuildContext context, InvoiceInDb invoice) async {
  final branch = getIt<AuthService>().currentBranch;
  if (!context.mounted) return;
  await Navigator.push(
    context,
    MaterialPageRoute(
      builder: (context) => _InvoiceViewer(invoice: invoice, branch: branch),
    ),
  );
}

class _InvoiceViewer extends StatelessWidget {
  const _InvoiceViewer({required this.invoice, required this.branch});
  final InvoiceInDb invoice;
  final BranchInDb branch;
  // ... UI de solo lectura
}
```

```dart
// editors/unit_editor_dialog.dart
Future<UnitResult?> showUnitEditor(
  BuildContext context, {
  List<Unit> initialUnits = const [],
  int initialDefaultIndex = 0,
}) async {
  return showDialog<UnitResult?>(
    context: context,
    builder: (context) => _UnitEditorDialog(
      initialUnits: initialUnits,
      initialDefaultIndex: initialDefaultIndex,
    ),
  );
}

class _UnitEditorDialog extends StatefulWidget { ... }
```

```dart
// modals/confirm_send_bottom_modal.dart
Future<bool?> showConfirmSendBottomModal(
  BuildContext context, {
  required String title,
  String? subtitle,
}) async {
  return showModalBottomSheet<bool?>(
    context: context,
    builder: (context) => _ConfirmSendModal(title: title, subtitle: subtitle),
  );
}
```

## Composición

Las show functions se encadenan: el buscador abre un editor al seleccionar y devuelve el resultado al llamador original:

```dart
// Searcher: al tocar un producto abre el modal de cantidades
final result = await showProductSearch(context, restrictStock: true);
if (result == null) return;          // canceló
await viewController.addItem(result); // usó el resultado
```

Uso desde un [action flow](action_flow_functions.md):

```dart
final paymentResult = await showPaymentConfirmationDialog(context, total);
if (paymentResult == null) return;
if (!context.mounted) return;
await context.read<WriteDocumentCubit>().create(...);
```

## Reglas

- Toda interacción desechable es una **función top-level** `showXxx(context, ...) → Future<T?>` — nunca una ruta
- **`null` = cancelar** — el llamador decide qué hacer con el resultado, sin callbacks
- Resultados tipados en `models/{entity}_result.dart`, nunca `Map`/`dynamic`
- Función `show` + widget privado `_Xxx` en el **mismo archivo**; solo la función se exporta
- `context.mounted` después de cada `await` antes de usar el context
- **Genéricos reutilizables** en `widgets/dialogs/` y `widgets/modals/`; **específicos** dentro del módulo
- Elegir mecanismo por caso: `showDialog` (diálogos), `showModalBottomSheet` (acciones rápidas), `push` (pantallas completas)
- Los managers globales (`DialogManager`, `LoadingDialogManager`) son la única excepción: estáticos, en `widgets/dialogs/`, **nunca en `src/tools/`**
- La misma show function se comparte entre `web/` y `mobile/` (solo el widget interno puede variar por plataforma)

## Relacionados

- [Funciones de Flujo](action_flow_functions.md) — los flujos invocan las show functions
- [ViewController Pattern](view_controller_pattern.md) — el resultado alimenta el estado local de la pantalla
- [GoRouter](go_router_pattern.md) — el router solo maneja navegación real, no dialogs
- [Segmentación de Vistas UI](view_segmentation.md) — reutilizar lo genérico, copiar lo que diverge
- [Convenciones de Nomenclatura](../style/naming_conventions_flutter.md) — naming de funciones y archivos
