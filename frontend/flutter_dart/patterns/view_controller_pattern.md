---
type: Pattern
title: ViewController Pattern (Estado Local de Pantalla)
description: Un ChangeNotifier por pantalla concentra el estado en memoria (items, selecciones, datos de entrada), expone acciones y getters derivados, y comparte información entre componentes vía Mediator. Nunca controla lo visual y nunca se inyecta al Cubit.
tags: [flutter, viewcontroller, changenotifier, estado, pantalla, patron, arquitectura, replicable]
---

# ViewController Pattern

El **ViewController** es el estado local de la pantalla en memoria. Es un `ChangeNotifier` que:

1. **Concentra el estado efímero** de la screen (items de un carrito, cliente seleccionado, método de pago, fecha, descripción).
2. **Expone acciones** que mutan ese estado y notifican a la UI.
3. **Comparte información entre componentes** — se propaga vía Mediator para que cualquier sección, widget o flujo lea y mute el mismo objeto sin prop-drilling.
4. **Mapea al dominio** — genera los DTOs (`CreateXxx`) que persisten los Cubits.

Vive solo mientras la pantalla existe: se crea en el `State` de la Screen y muere al abandonarla. Es "el estado de la screen" desacoplado de los widgets, para que web y mobile compartan la misma lógica.

## Qué es y qué no es

| ✅ Es | ❌ No es |
|---|---|
| Estado en memoria de la pantalla | Estado del servidor (eso es del Cubit Read) |
| Acciones del formulario/carrito | Persistencia (eso es del Cubit Write) |
| Getters derivados (totales, validaciones) | Control visual (widgets, layouts, estilos) |
| Generación de DTOs hacia el dominio | Navegación, diálogos ni toasts |
| Estado compartido entre plataformas y secciones | Un reemplazo de los Cubits |

## Anatomía del módulo

```
feature/
├── feature_screen.dart          # Crea el ViewController en initState
├── view_controller.dart         # ChangeNotifier: estado + acciones + mapeo
├── cubit/
│   ├── read_feature_cubit.dart  # Carga datos del servidor (canal lectura)
│   └── write_feature_cubit.dart # Persiste DTOs (canal escritura)
├── web/   (screen, mediator, sections/)
├── mobile/ (screen, mediator)
└── actions/ (feature_payment.dart, feature_completion.dart)  # Flujos UI
```

Flujo de datos:

```
Screen (State) crea el VC ──> Mediator lo expone ──> widgets/secciones lo leen
Action flow: VC.generateCreateXxx() ──> DTO ──> WriteCubit.persist(dto) ──> Repository
                                                                              │ emite evento
Sincronización: ReadCubit escucha el evento y emite nuevo estado <─────────────┘
                (sin refetch, ver ReactiveRepository)
```

- **Screen raíz**: ciclo de vida del VC (`initState`/`dispose`) e inyección de dependencias (BlocProviders).
- **Mediator** (`InheritedWidget` mudo): expone el VC sin prop-drilling (ver [Mediator Pattern](mediator_pattern.md)).
- **Cubits**: solo datos y persistencia; nunca conocen el VC (ver [Read / Write Cubits](read_write_cubits.md)).
- **Repository**: sin estado; tras persistir emite un evento para que el ReadCubit sincronice la lista solo (ver [ReactiveRepository Mixin](reactive_repository.md)).
- **Action flows**: orquestan los flujos UI multi-paso leyendo el VC y disparando el cubit (ver [Funciones de Flujo](action_flow_functions.md)).

## Las 3 responsabilidades

### 1. Estado en memoria

```dart
class CartViewController extends ChangeNotifier {
  final List<CartItem> _items = [];
  Client? _selectedClient;
  PaymentMethod _paymentMethod = PaymentMethod.cash;
  String description = '';

  List<CartItem> get items => List.unmodifiable(_items);
  Client? get selectedClient => _selectedClient;
}
```

### 2. Getters derivados y acciones

Los cálculos (totales, validaciones) viven aquí, **nunca en la UI**:

```dart
int get totalItems => _items.length;
int get totalAmount => _items.fold(0, (sum, i) => sum + i.total);
bool get hasZeroPriceItems => _items.any((i) => i.price <= 0);

void addItem(CartItem item) {
  final exists = _items.any((i) => i.productId == item.productId);
  if (exists) return;            // o reemplazar, según la regla del módulo
  _items.add(item);
  notifyListeners();
}

void removeItem(CartItem item) {
  _items.remove(item);
  notifyListeners();
}

void clear() {
  _items.clear();
  _selectedClient = null;
  _paymentMethod = PaymentMethod.cash;
  description = '';
  notifyListeners();
}
```

Setters con **side effects**: cambiar un selector puede recalcular precios o invalidar dependencias. Ese efecto vive en el setter, no en la UI:

```dart
set selectedClient(Client? value) {
  _selectedClient = value;
  _paymentMethod = value?.hasCredit == true
      ? PaymentMethod.credit
      : PaymentMethod.cash;
  if (value != null) _applyPriceLevel(value.defaultPriceLevel);
  notifyListeners();
}
```

### 3. Mapeo al dominio (DTOs)

El VC traduce el estado de UI al objeto que persistirá el cubit. Si requiere servicios globales (auth), se accede vía inyección (GetIt), no con BuildContext de widgets:

```dart
CreateDocument? generateCreateDocument() {
  final items = _items.map((i) => i.toSaleItem()).toList();
  if (items.isEmpty) return null;
  final user = getIt<AuthService>().currentUser;
  return CreateDocument(
    clientId: _selectedClient?.id ?? kAnonymousClientId,
    items: items,
    total: totalAmount,
    createdBy: user,
  );
}
```

## Comportamientos que varían

El patrón es estable; lo que cambia entre módulos son reglas puntuales. Decide por módulo:

| Comportamiento | Variantes |
|---|---|
| `addItem` con duplicado | Ignorar (1 unidad por producto) / reemplazar / quick-add que suma +1 limitado por stock |
| Recálculo de precios | En setter del cliente (nivel de precio) / en setter del nivel de precio global / desactivado con flag por plataforma |
| Cliente | Obligatorio / opcional (venta anónima con id por defecto) |
| `generateCreateXxx()` | Con `BuildContext` o sin él, según el acceso a servicios globales |
| Documento | Factura / orden / cotización — se parametriza en el cubit, no en el VC |

Si una plataforma necesita comportamiento distinto (ej. el móvil no recalcula precios), usa un **flag en el VC** que la screen de cada plataforma configura — nunca copies el VC.

## Ciclo de vida

```dart
class FeatureScreen extends StatefulWidget { ... }

class _FeatureScreenState extends State<FeatureScreen> {
  late final CartViewController viewController;

  @override
  void initState() {
    super.initState();
    viewController = CartViewController();
  }

  @override
  Widget build(BuildContext context) {
    return CartMediator(
      viewController: viewController,
      child: const _RootScaffold(),
    );
  }
}
```

- Se crea en `initState` de la pantalla raíz (único dueño).
- `ChangeNotifier` no requiere `dispose()` manual (sin recursos); solo limpia callbacks ajenos si los configuró.
- Tras persistencia exitosa, el action flow llama `viewController.clear()` para reiniciar la pantalla; la lista de datos se sincroniza sola vía el evento del repositorio (sin refetch, ver [ReactiveRepository Mixin](reactive_repository.md)).

## Regla clave: no inyectar el ViewController al Cubit

El ViewController pertenece a la **pantalla**, no al cubit. El cubit recibe solo DTOs:

```dart
// ✅ Correcto: el cubit no conoce el VC
final dto = CartMediator.of(context).viewController.generateCreateDocument();
if (dto == null) return;
await context.read<WriteDocumentCubit>().create(dto);

// ❌ Anti-patrón: inyectar el VC al cubit para usarlo de puente de acceso
WriteDocumentCubit(repo: repo, viewController: viewController);
// y después: context.read<WriteDocumentCubit>().viewController.clear();
```

El anti-patrón agrega acoplamiento sin beneficio: el cubit guarda el VC pero **nunca lo usa dentro de sus métodos**; solo sirve para que las screens lo alcancen por otro camino. Los widgets y action flows deben leerlo del **Mediator** (`Mediator.of(context).viewController`).

## Cuándo ViewController vs Cubit

| Caso | Usar |
|---|---|
| Datos del servidor (listas, CRUD) | Cubits Read / Write |
| Estado efímero de la pantalla (formularios, carritos, selecciones) | ViewController |
| Persistir lo capturado | Cubit Write — recibe el DTO generado por el VC |
| Flujo UI multi-paso (cobrar, confirmar, guardar) | Funciones de flujo top-level (ver [Funciones de Flujo](action_flow_functions.md)) |

## Reglas

- **Un ViewController por pantalla**, creado en el `State` de la Screen raíz; muere con ella
- Solo **estado en memoria**: nunca widgets, navegación, diálogos ni toasts
- **Sin `BuildContext`** salvo para servicios globales al generar DTOs
- **Toda mutación llama `notifyListeners()`**; toda lectura de estado pasa por getters
- **Getters derivados** (totales, validaciones) en el VC, nunca en la UI
- **Setters con side effects** (recálculo de precios, cambio de método de pago)
- Se comparte entre plataformas y secciones vía **Mediator**, sin prop-drilling
- El **Cubit recibe solo DTOs** — nunca inyectar el VC
- Los flujos UI complejos se extraen a **funciones top-level** para no inflar archivos
- `clear()` tras persistencia exitosa (orquestado en el action flow de completación)
- Comportamiento distinto por plataforma: **flag en el VC**, nunca copiar el VC

## Relacionados

- [Mediator Pattern](mediator_pattern.md) — cómo se propaga el VC a las secciones
- [Funciones de Flujo](action_flow_functions.md) — orquestación de flujos UI que leen el VC
- [Read / Write Cubits](read_write_cubits.md) — los canales de datos que persisten los DTOs
- [ReactiveRepository Mixin](reactive_repository.md) — cómo el ReadCubit sincroniza la lista tras persistir, sin refetch
- [Segmentación de Vistas UI](view_segmentation.md) — fuente de verdad única por pantalla
- [Diseño Responsive](responsive_design.md) — VC compartido entre `web/` y `mobile/`
- [Inyección de Dependencias](di_flutter.md) — acceso a servicios globales desde el VC
