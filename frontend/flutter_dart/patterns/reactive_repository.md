---
type: Pattern
title: ReactiveRepository Mixin
description: Mixin que añade un StreamController broadcast al repositorio para notificar a los ReadCubits cuando se crean, actualizan o eliminan items.
tags: [flutter, repository, reactivo, stream, patron]
---

# ReactiveRepository Mixin

Permite que los WriteCubits notifiquen automáticamente a los ReadCubits cuando los datos cambian, sin necesidad de refetch manual.

## Eventos

```dart
sealed class RepoEvent<T> {}

final class RepoItemCreated<T> extends RepoEvent<T> {
  final T item;
  RepoItemCreated(this.item);
}

final class RepoItemUpdated<T> extends RepoEvent<T> {
  final T item;
  RepoItemUpdated(this.item);
}

final class RepoItemDeleted<T> extends RepoEvent<T> {
  final T item;
  RepoItemDeleted(this.item);
}
```

## Mixin

```dart
mixin ReactiveRepository<T> {
  final _controller = StreamController<RepoEvent<T>>.broadcast();
  Stream<RepoEvent<T>> get eventStream => _controller.stream;

  void notifyItemCreated(T item) => _controller.sink.add(RepoItemCreated(item));
  void notifyItemUpdated(T item) => _controller.sink.add(RepoItemUpdated(item));
  void notifyItemDeleted(T item) => _controller.sink.add(RepoItemDeleted(item));

  void disposeReactiveRepo() => _controller.close();
}
```

## Uso en Repository

```dart
class ItemRepository with ReactiveRepository<Item> {
  final ItemDataSource dataSource;
  List<Item> _cache = [];

  Future<Item> createItem(CreateItem create) async {
    final result = await dataSource.createItem(create.toJson());
    final newItem = Item.fromJson(result);
    _cache = [newItem, ..._cache];
    notifyItemCreated(newItem);  // ← Notifica al ReadCubit
    return newItem;
  }
}
```

## Uso en ReadCubit

```dart
class ReadItemCubit extends Cubit<ReadItemState> {
  final ItemRepository itemRepository;
  StreamSubscription? _subscription;

  ReadItemCubit({required this.itemRepository}) : super(ReadItemInitial()) {
    _subscription = itemRepository.eventStream.listen((event) {
      if (state is! ReadItemSuccess) return;
      final current = state as ReadItemSuccess;
      switch (event) {
        case RepoItemCreated(:final item):
          emit(current.copyWith(newItems: [item, ...current.items]));
        case RepoItemUpdated(:final item):
          emit(current.copyWith(updatedItems: [item, ...current.items]));
        case RepoItemDeleted(:final item):
          emit(current.copyWith(
            deletedItems: [item, ...current.deletedItems],
            items: current.items.where((i) => i.id != item.id).toList(),
          ));
      }
    });
  }
}
```

## Flujo completo

```
WriteCubit.create()
  → Repository.create()
    → DataSource.create() → API
    → Repository.notifyItemCreated(item)
  → ReadCubit._handleRepoEvent(event)
    → ReadCubit emite nuevo estado con item agregado
  → BlocBuilder rebuild
```

## Reglas

- **Broadcast StreamController** — múltiples listeners pueden suscribirse
- El ReadCubit **no necesita refetch** cuando recibe el evento
- Los eventos se emiten **después** de la operación exitosa en API
- `disposeReactiveRepo()` se llama en el `dispose` del Repository
