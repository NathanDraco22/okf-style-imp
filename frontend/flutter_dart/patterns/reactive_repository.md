---
type: Pattern
title: ReactiveRepository Mixin
description: Mixin que añade un StreamController broadcast al repositorio para notificar a los ReadCubits cuando se crean, actualizan o eliminan items. El ReadCubit se sincroniza sin refetch y sin cache en el repositorio.
tags: [flutter, repository, reactivo, stream, patron]
---

# ReactiveRepository Mixin

> **CLI:** `onion dart {entity}`, `onion dart-cubit {entity}` y `onion flutter-module {entity}` preguntan si se genera con ReactiveRepository (default: **No**). Con `--reactive` (o respondiendo Sí al prompt) el CLI genera el mixin en `lib/src/tools/reactive_repo/reactive_repository.dart`, el repository con `with ReactiveRepository<T>` + `notifyXxx` sin cache, y los cubits con suscripción automática al `eventStream`. Ver [cli_commands_flutter.md](cli_commands_flutter.md).

Permite que los WriteCubits notifiquen automáticamente a los ReadCubits cuando los datos cambian, sin refetch manual. El **ReadCubit es la única fuente de verdad** de la lista: el repositorio solo llama al datasource y emite eventos, nunca guarda estado.

## Eventos y mixin

```dart
import 'dart:async';

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

mixin ReactiveRepository<T> {
  final _controller = StreamController<RepoEvent<T>>.broadcast();

  Stream<RepoEvent<T>> get eventStream => _controller.stream;

  void notifyItemCreated(T item) {
    if (!_controller.isClosed) _controller.sink.add(RepoItemCreated(item));
  }

  void notifyItemUpdated(T item) {
    if (!_controller.isClosed) _controller.sink.add(RepoItemUpdated(item));
  }

  void notifyItemDeleted(T item) {
    if (!_controller.isClosed) _controller.sink.add(RepoItemDeleted(item));
  }

  void disposeReactiveRepo() => _controller.close();
}
```

## Uso en Repository (sin cache)

El repositorio **no mantiene listas en memoria**: parsea la respuesta, emite el evento y devuelve el item. La lista vive en el estado del ReadCubit.

```dart
class ItemRepository with ReactiveRepository<Item> {
  final ItemDataSource dataSource;

  Future<Item> createItem(CreateItem create) async {
    final result = await dataSource.createItem(create.toJson());
    final newItem = Item.fromJson(result);
    notifyItemCreated(newItem);  // ← Notifica al ReadCubit
    return newItem;
  }

  Future<Item> updateItem(String id, UpdateItem update) async {
    final result = await dataSource.updateItemById(id, update.toJson());
    final updatedItem = Item.fromJson(result);
    notifyItemUpdated(updatedItem);
    return updatedItem;
  }

  Future<Item> deleteItem(String id) async {
    final result = await dataSource.deleteItemById(id);
    final deletedItem = Item.fromJson(result);
    notifyItemDeleted(deletedItem);
    return deletedItem;
  }
}
```

## Uso en ReadCubit

La suscripción se crea en el **constructor**; cada evento actualiza el estado sin refetch:

```dart
class ReadItemCubit extends Cubit<ReadItemState> {
  final ItemRepository itemRepository;
  StreamSubscription? _subscription;

  ReadItemCubit({required this.itemRepository}) : super(ReadItemInitial()) {
    _subscription = itemRepository.eventStream.listen(_handleRepoEvent);
  }

  void _handleRepoEvent(RepoEvent<Item> event) {
    switch (event) {
      case RepoItemCreated(:final item):
        markItemCreated(item);
      case RepoItemUpdated(:final item):
        markItemUpdated(item);
      case RepoItemDeleted(:final item):
        markItemDeleted(item);
    }
  }

  void markItemCreated(Item item) {
    final current = state;
    if (current is! ReadItemSuccess) return;
    emit(current.copyWith(
      items: [item, ...current.items.where((i) => i.id != item.id)],
      newItems: [...current.newItems, item],
    ));
  }

  void markItemUpdated(Item item) {
    final current = state;
    if (current is! ReadItemSuccess) return;
    emit(current.copyWith(
      items: current.items.map((i) => i.id == item.id ? item : i).toList(),
      updatedItems: [...current.updatedItems, item],
    ));
  }

  void markItemDeleted(Item item) {
    final current = state;
    if (current is! ReadItemSuccess) return;
    emit(current.copyWith(
      items: current.items.where((i) => i.id != item.id).toList(),
      deletedItems: [...current.deletedItems, item],
    ));
  }

  @override
  Future<void> close() async {
    _subscription?.cancel();
    await super.close();
  }
}
```

Los campos `newItems`/`updatedItems`/`deletedItems` del estado forman un **changelog** para la UI (animaciones, banners "se agregó/actualizó X") — ver [Estados de Lectura](cubit_states_read.md).

**Variante:** algunos módulos prefieren que `markItemDeleted` **conserve el item en `items`** y solo lo registre en `deletedItems`, para que la UI muestre la eliminación en contexto antes de esconderlo. Decide por módulo.

## Flujo completo

```
WriteCubit.create(dto)
  → Repository.create()
    → DataSource.create() → API
    → Repository.notifyItemCreated(item)
  → ReadCubit._handleRepoEvent(event)
    → ReadCubit emite nuevo estado con item agregado (sin refetch)
  → BlocBuilder rebuild
```

## Reglas

- **Broadcast StreamController** — múltiples ReadCubits pueden suscribirse al mismo repositorio
- **Sin cache en memoria en el repositorio** — la lista vive en el estado del ReadCubit; un cache en el repo duplica estado y se desincroniza con la fuente real
- Los eventos se emiten **después** de la operación exitosa en API
- El ReadCubit **no necesita refetch** al recibir el evento; solo actualiza su estado
- **Guard `isClosed`** en los notifiers — evita `add()` sobre un stream cerrado
- La suscripción se crea en el **constructor del ReadCubit** y se cancela en su `close()`
- `disposeReactiveRepo()` se llama al cerrar el repositorio (fin de ciclo de vida en DI)

## Cuándo usar reactive vs BlocListener

| Situación | Recomendado | Por qué |
|-----------|-------------|---------|
| WriteCubits efímeros (uno por diálogo/edición) | **Reactive** | El WriteCubit puede instanciarse y destruirse sin romper la comunicación: la notificación viaja por el stream del repository, no por el árbol de providers |
| Varios ReadCubits comparten un repository | **Reactive** | Broadcast stream: todos se enteran de un solo `notify` |
| Snacks/feedback de éxito (toast "creado") | **Ambos** | Reactive no elimina la necesidad de escuchar el estado del WriteCubit para feedback local; el stream actualiza la lista, el BlocListener (o listener del WriteCubit) muestra el mensaje |
| Módulo sin escrituras (solo lectura) | **No reactive** | No hay eventos que escuchar |
| Un solo WriteCubit de ciclo de vida global acoplado al ReadCubit en el mismo provider | **BlocListener** | El acoplamiento por provider es suficiente; reactive agrega complejidad sin beneficio |

> **Ventaja clave de reactive:** el ReadCubit queda desacoplado del WriteCubit. Crear un WriteCubit nuevo dentro de un diálogo (`WriteXxxCubit(repository)`) y cerrarlo al terminar no desincroniza la lista: el ReadCubit ya estaba suscrito al repository.

## Variante fallback: BlocListener + markXxx

Sin reactive (modo clásico generado por defecto), la vista conecta la escritura con la lectura usando `markXxx` manualmente desde un `BlocListener`:

```dart
BlocListener<WriteItemCubit, WriteItemState>(
  listener: (context, state) {
    final readCubit = context.read<ReadItemCubit>();
    switch (state) {
      case ItemCreated(:final item):
        readCubit.markItemCreated(item);
      case ItemUpdated(:final item):
        readCubit.markItemUpdated(item);
      case ItemDeleted(:final item):
        readCubit.markItemDeleted(item);
      default:
        break;
    }
  },
  child: ...,
)
```

Este patrón exige que ambos cubits existan en el mismo árbol de providers y que la vista conozca los dos. Funciona para flujos acoplados, pero no permite crear WriteCubits efímeros sin perder la sincronización.

## Relacionados

- [Read / Write Cubits](read_write_cubits.md) — los dos canales que se conectan por eventos
- [Estados de Lectura](cubit_states_read.md) — el changelog `newItems`/`updatedItems`/`deletedItems`
- [ViewController Pattern](view_controller_pattern.md) — el estado efímero de pantalla no participa de los eventos
