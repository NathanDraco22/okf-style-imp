---
type: Pattern
title: Separación Read / Write Cubits
description: Cada entidad tiene dos cubits separados: ReadCubit para cargar datos y WriteCubit para mutaciones (crear, actualizar, eliminar).
tags: [flutter, cubit, estado, separacion, patron]
---

# Read / Write Cubits

Se separan estrictamente los cubits de lectura y escritura para mantener estados simples y predecibles.

> **CLI:** El comando `onion dart-cubit {entity}` genera ambos cubits (read + write) con sus estados y barrel files. El desarrollador completa los métodos con la lógica de negocio específica.

## Read Cubit

Responsable de cargar datos. Nunca muta.

> Los cambios del WriteCubit llegan al ReadCubit **sin refetch manual**: el repositorio emite eventos que el ReadCubit escucha (ver [ReactiveRepository Mixin](reactive_repository.md)).

```dart
class ReadItemCubit extends Cubit<ReadItemState> {
  final ItemRepository itemRepository;
  StreamSubscription? _subscription;

  ReadItemCubit({required this.itemRepository})
    : super(ReadItemInitial()) {
    // Escucha eventos del ReactiveRepository para actualizarse automáticamente
    _subscription = itemRepository.eventStream.listen(_handleRepoEvent);
  }

  Future<void> getAll() async {
    emit(ReadItemLoading());
    try {
      final items = await ItemRepository.getAllItems();
      emit(ReadItemSuccess(items));
    } catch (e) {
      emit(ReadItemError(e.toString()));
    }
  }

  void _handleRepoEvent(RepoEvent event) {
    if (state is! ReadItemSuccess) return;
    final current = state as ReadItemSuccess;
    switch (event) {
      case RepoItemCreated(:final item):
        emit(current.copyWith(
          items: [item, ...current.items],
          newItems: [item, ...current.newItems],
        ));
      case RepoItemUpdated(:final item):
        emit(current.copyWith(
          items: current.items.map((i) => i.id == item.id ? item : i).toList(),
          updatedItems: [item, ...current.updatedItems],
        ));
      case RepoItemDeleted(:final item):
        emit(current.copyWith(
          items: current.items.where((i) => i.id != item.id).toList(),
          deletedItems: [item, ...current.deletedItems],
        ));
    }
  }

  @override
  Future<void> close() {
    _subscription?.cancel();
    return super.close();
  }
}
```

## Write Cubit

Responsable de mutaciones. Cada operación emite su propio estado.

```dart
class WriteItemCubit extends Cubit<WriteItemState> {
  final ItemRepository itemRepository;

  WriteItemCubit({required this.itemRepository})
    : super(WriteItemInitial());

  Future<void> create(CreateItem create) async {
    emit(WritingItem());
    try {
      final item = await itemRepository.createItem(create);
      emit(ItemCreated(item));
      emit(WriteItemInitial());  // Reset para reutilizar el cubit
    } catch (e) {
      emit(WriteItemError(e.toString()));
    }
  }

  Future<void> update(String id, UpdateItem update) async {
    emit(WritingItem());
    try {
      final item = await itemRepository.updateItemById(id, update);
      emit(ItemUpdated(item));
      emit(WriteItemInitial());
    } catch (e) {
      emit(WriteItemError(e.toString()));
    }
  }

  Future<void> delete(String id) async {
    emit(WritingItem());
    try {
      final item = await itemRepository.deleteItemById(id);
      emit(ItemDeleted(item));
      emit(WriteItemInitial());
    } catch (e) {
      emit(WriteItemError(e.toString()));
    }
  }
}
```

## Reglas

- **ReadCubit** solo expone métodos de carga (`getAll`, `getById`, `refresh`)
- **WriteCubit** expone `create`, `update`, `delete`
- **Nunca** un mismo cubit hace lectura y escritura
- WriteCubit **resetea a Initial** después de cada éxito para permitir operaciones consecutivas
- ReadCubit se suscribe al `eventStream` del Repository para reflejar cambios automáticamente
