---
type: Pattern
title: Estados de Escritura (Write Cubit States)
description: Jerarquía de estados sellados para cubits de escritura: Initial, Writing, Success (Created/Updated/Deleted) y Error.
tags: [flutter, cubit, estados, sealed, write, patron]
---

# Estados de Escritura

> **CLI:** `onion dart-cubit {entity}` genera la jerarquía de estados sellados (read + write) con sus clases base. El desarrollador completa los campos según la entidad.

Los Write Cubits usan una jerarquía de estados para cada operación de mutación.

## Jerarquía

```dart
sealed class WriteItemState {}

final class WriteItemInitial extends WriteItemState {}

final class WritingItem extends WriteItemState {}

sealed class WriteItemSuccess extends WriteItemState {
  final Item item;
  WriteItemSuccess(this.item);
}

final class ItemCreated extends WriteItemSuccess {
  ItemCreated(super.item);
}

final class ItemUpdated extends WriteItemSuccess {
  ItemUpdated(super.item);
}

final class ItemDeleted extends WriteItemSuccess {
  ItemDeleted(super.item);
}

final class WriteItemError extends WriteItemState {
  final String message;
  WriteItemError(this.message);
}
```

## Ciclo de vida

```
Initial → Writing → ItemCreated → Initial (reset)
          ↓               ↓
        Error        ItemUpdated → Initial (reset)
                          ↓
                    ItemDeleted → Initial (reset)
```

## Uso en UI

```dart
BlocListener<WriteItemCubit, WriteItemState>(
  listener: (context, state) {
    switch (state) {
      case WritingItem():
        LoadingDialogManager.showLoadingDialog(context);
      case ItemCreated(:final item):
        Navigator.pop(context);
        ScaffoldMessenger.showSnackBar(SnackBar(content: Text('${item.name} creado')));
      case ItemUpdated(:final item):
        ScaffoldMessenger.showSnackBar(SnackBar(content: Text('${item.name} actualizado')));
      case ItemDeleted(:final item):
        ScaffoldMessenger.showSnackBar(SnackBar(content: Text('${item.name} eliminado')));
      case WriteItemError(:final message):
        ScaffoldMessenger.showSnackBar(SnackBar(content: Text('Error: $message')));
      default:
    }
  },
  child: ...
)
```

## Detalles clave

- **WriteSuccess** es `sealed` y tiene subtipos específicos (`Created`, `Updated`, `Deleted`)
- El cubit **resetea a Initial** tras cada éxito para permitir operaciones consecutivas
- `WritingState` se usa para mostrar loading/spinner
- `ErrorState` lleva mensaje para feedback al usuario
