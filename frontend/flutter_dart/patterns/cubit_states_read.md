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

/// Búsqueda en curso: mantiene los datos visibles mientras llega la respuesta
final class ReadItemSearching extends ReadItemSuccess {
  ReadItemSearching(super.items);
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
              Success → Searching (búsqueda)
```

## Búsqueda que conserva datos

Cuando hay una búsqueda en curso, **nunca se vacía la pantalla**: se emite `ReadItemSearching` con los datos actuales y la UI muestra un indicador discreto (barra de progreso lineal), no un spinner central:

```dart
if (state is ReadItemLoading || state is ReadItemSearching)
  const LinearProgressIndicator(minHeight: 3),
```

Al terminar la búsqueda (o al limpiarla), se vuelve a `ReadItemSuccess` con los resultados — o con la cache previa si el query queda vacío. Ver [Paginación Infinita y Búsqueda](pagination_infinite_scroll.md).

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

- **Success** mantiene `newItems`, `updatedItems`, `deletedItems` para animaciones visuales — se llenan con los eventos del repositorio (ver [ReactiveRepository Mixin](reactive_repository.md))
- **Refreshing** extiende Success para mantener datos visibles mientras recarga
- **Searching** extiende Success: los datos visibles nunca se borran mientras se busca
- **Error** lleva mensaje para mostrar al usuario
- Todos los estados son `final` excepto Success (que Refreshing/Searching extienden)
