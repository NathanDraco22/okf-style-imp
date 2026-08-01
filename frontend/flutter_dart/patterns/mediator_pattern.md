---
type: Pattern
title: Patrón Mediator (InheritedWidget + ChangeNotifier)
description: Para formularios complejos con múltiples campos, se usa un ChangeNotifier como controlador de vista expuesto via InheritedWidget.
tags: [flutter, mediator, inheritedwidget, changenotifier, patron]
---

# Patrón Mediator

Para pantallas con formularios complejos (facturas, inventario) donde los cubits no son suficientes, se usa un "mediator" que combina `InheritedWidget` + `ChangeNotifier`.

## Variantes

| Variante | Cuándo | Cómo se reconstruye |
|---|---|---|
| **InheritedNotifier** | El controlador se crea/estado simple; todos escuchan lo mismo | `dependOnInheritedWidgetOfExactType` propaga cambios |
| **Exposición + ListenableBuilder** | Controlador estable; secciones con intereses distintos (suscripción granular) | `InheritedWidget` con `updateShouldNotify: false` + `ListenableBuilder` por sección |

Para suscripción granular, el mediator es un `InheritedWidget` **mudo** (solo expone, nunca notifica):

```dart
class SaleTerminalWebMediator extends InheritedWidget {
  const SaleTerminalWebMediator({
    super.key,
    required super.child,
    required this.viewController,
  });

  final SaleTerminalViewController viewController;

  static SaleTerminalWebMediator of(BuildContext context) {
    return context.dependOnInheritedWidgetOfExactType<SaleTerminalWebMediator>()!;
  }

  @override
  bool updateShouldNotify(SaleTerminalWebMediator oldWidget) {
    return false;  // No reconstruye nada: cada sección escucha con ListenableBuilder
  }
}
```

Las secciones leen el controlador del mediator y se suscriben de forma granular:

```dart
class _CartSection extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final viewController = SaleTerminalWebMediator.of(context).viewController;
    return ListenableBuilder(
      listenable: viewController,
      builder: (context, _) { /* solo esta sección se reconstruye */ },
    );
  }
}
```

Los widgets internos también pueden suscribirse a **partes** del controlador:

```dart
// En un tile: solo el precio se reconstruye al cambiar el nivel de precio
ListenableBuilder(
  listenable: priceLevelListenable,
  builder: (context, _) => Text(NumberFormatter.convertToMoneyLike(price)),
)
```

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
- Mediator usa `InheritedNotifier<ViewController>` o `InheritedWidget` mudo + `ListenableBuilder` para propagar cambios eficientemente
- Variante **exposición**: `updateShouldNotify: false` + `ListenableBuilder` con `listenable` del controlador
- `of(context)` escucha cambios; `read(context)` solo lee sin escuchar
- Los formularios complejos se separan en secciones con `part` files para mantenerlos manejables
- Los getters derivados (totales, validaciones) viven en el ViewController, nunca en la UI
