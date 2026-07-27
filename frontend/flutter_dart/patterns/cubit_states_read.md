---
type: Pattern
title: Estados de Lectura (Read Cubit States)
description: Jerarquía de estados sellados para cubits de lectura: Initial, Loading, Success, Refreshing y Error.
tags: [flutter, cubit, estados, sealed, patron]
---

# Estados de Lectura

Los Read Cubits usan una jerarquía de estados sellados para representar todas las fases del ciclo de vida de carga de datos.

## Jerarquía

```dart
sealed class ReadItemState {}

final class ReadItemInitial extends ReadItemState {}
final class ReadItemLoading extends ReadItemState {}

class ReadItemSuccess extends ReadItemState {
  final List<Item> items;
  final List<Item> newItems;      // Items creados recientemente
  final List<Item> updatedItems;  // Items actualizados recientemente
  final List<Item> deletedItems;  // Items eliminados recientemente

  ReadItemSuccess(
    this.items, {
    this.newItems = const [],
    this.updatedItems = const [],
    this.deletedItems = const [],
  });

  ReadItemSuccess copyWith({
    List<Item>? items,
    List<Item>? newItems,
    List<Item>? updatedItems,
    List<Item>? deletedItems,
  }) => ReadItemSuccess(
    items ?? this.items,
    newItems: newItems ?? this.newItems,
    updatedItems: updatedItems ?? this.updatedItems,
    deletedItems: deletedItems ?? this.deletedItems,
  );
}

/// Pull-to-refresh: mantiene los datos viejos mientras recarga
final class ReadItemRefreshing extends ReadItemSuccess {
  ReadItemRefreshing(super.items, {super.newItems, super.updatedItems, super.deletedItems});
}

final class ReadItemError extends ReadItemState {
  final String message;
  ReadItemError(this.message);
}
```

## Ciclo de vida

```
Initial → Loading → Success → Refreshing (pull-to-refresh)
                      ↓
                    Error
```

## Uso en UI

```dart
BlocBuilder<ReadItemCubit, ReadItemState>(
  builder: (context, state) {
    return switch (state) {
      ReadItemInitial() => const SizedBox(),
      ReadItemLoading() => const CircularProgressIndicator(),
      ReadItemSuccess() => _ItemsList(items: state.items),
      ReadItemRefreshing() => _ItemsList(items: state.items, isRefreshing: true),
      ReadItemError() => Text('Error: ${state.message}'),
    };
  },
);
```

## Detalles clave

- **Success** mantiene `newItems`, `updatedItems`, `deletedItems` para animaciones visuales
- **Refreshing** extiende Success para mantener datos visibles mientras recarga
- **Error** lleva mensaje para mostrar al usuario
- Todos los estados son `final` excepto Success (que Refreshing extiende)
