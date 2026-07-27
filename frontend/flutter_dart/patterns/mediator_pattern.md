---
type: Pattern
title: Patrón Mediator (InheritedWidget + ChangeNotifier)
description: Para formularios complejos con múltiples campos, se usa un ChangeNotifier como controlador de vista expuesto via InheritedWidget.
tags: [flutter, mediator, inheritedwidget, changenotifier, patron]
---

# Patrón Mediator

Para pantallas con formularios complejos (facturas, inventario) donde los cubits no son suficientes, se usa un "mediator" que combina `InheritedWidget` + `ChangeNotifier`.

## ViewController (ChangeNotifier)

```dart
class ItemFormController extends ChangeNotifier {
  Item? _selectedClient;
  DateTime _selectedDate = DateTime.now();
  String _paymentType = 'cash';
  final List<ItemLine> _items = [];

  // Getters
  Item? get selectedClient => _selectedClient;
  List<ItemLine> get items => List.unmodifiable(_items);
  int get total => items.fold(0, (sum, item) => sum + item.total);

  // Mutaciones con notificación
  void selectClient(Item client) {
    _selectedClient = client;
    notifyListeners();
  }

  void addItem(ItemLine item) {
    _items.add(item);
    notifyListeners();
  }

  void removeItem(int index) {
    _items.removeAt(index);
    notifyListeners();
  }

  CreateItem generateCreateItem() {
    return CreateItem(
      clientId: _selectedClient!.id,
      date: _selectedDate,
      items: _items.map((e) => e.toItemLine()).toList(),
    );
  }

  void clear() {
    _selectedClient = null;
    _items.clear();
    notifyListeners();
  }
}
```

## Mediator (InheritedWidget / InheritedNotifier)

```dart
class ItemFormMediator extends InheritedNotifier<ItemFormController> {
  const ItemFormMediator({
    super.key,
    required ItemFormController notifier,
    required super.child,
  });

  static ItemFormController of(BuildContext context) {
    return context.dependOnInheritedWidgetOfExactType<ItemFormMediator>()!.notifier!;
  }

  static ItemFormController? read(BuildContext context) {
    return context.getInheritedWidgetOfExactType<ItemFormMediator>()?.notifier;
  }
}
```

## Uso en Screen

```dart
class ClientInvoicesScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final viewController = ItemFormController();
    return MultiBlocProvider(
      providers: [
        BlocProvider(create: (_) => WriteItemCubit(repo: RepoProvider.of(context))),
      ],
      child: ItemFormMediator(
        notifier: viewController,
        child: const _RootScaffold(),
      ),
    );
  }
}
```

## Uso en Widgets

```dart
// Para leer y escuchar cambios (rebuild):
final controller = ItemFormMediator.of(context);  // se inscribe
final total = controller.total;

// Para leer sin escuchar (cambio no afecta UI):
final controller = ItemFormMediator.read(context);
final invoice = controller.generateCreateItem();
```

## ¿Cuándo usar Mediator vs Cubit?

| Caso | Usar |
|------|------|
| Datos del servidor (listas, CRUD) | Cubits (Read/Write) |
| Estado de formulario (input fields, selecciones) | ViewController + Mediator |
| Estado que mezcla UI + lógica compleja | Mediator que usa cubits internamente |

## Reglas

- ViewController es `ChangeNotifier` — llama a `notifyListeners()` en cada mutación
- Mediator usa `InheritedNotifier<ViewController>` para propagar cambios eficientemente
- `of(context)` escucha cambios; `read(context)` solo lee sin escuchar
- Los formularios complejos se separan en secciones con `part` files para mantenerlos manejables
