---
type: Pattern
title: Gate Único con GateFilter Componibles (Composable Gate Pattern)
description: Patrón para consolidar controles de visibilidad y acceso en un único widget Gate que recibe una lista plana de GateFilter componibles (análogo a TextInputFormatter), evitando anidación en el árbol de widgets y duplicación en resolvers imperativos.
tags: [flutter, dart, widgets, permisos, feature-flags, reactividad, composicion, ui, patron]
---

# Gate Único con `GateFilter` Componibles

Patrón para centralizar el renderizado condicional por permisos, roles, feature flags o capacidades del entorno en **un único widget `Gate`** que recibe una lista plana de filtros (`List<GateFilter>`), de forma análoga a `inputFormatters` (`List<TextInputFormatter>`) en `TextField`.

## El problema

A medida que una aplicación crece, aparecen múltiples condiciones de visibilidad: permisos por rol, usuario administrador, feature flags de configuración, capacidades del tenant o estado de sesión. El enfoque ingenuo crea un widget Gate distinto por cada regla:

```text
❌ Árbol profundamente anidado y acoplado
FeatureAGate(
  child: FeatureBGate(
    child: AdminOrOwnerGate(
      child: RolePermissionGate(
        permissions: [ModulePermissions.billing],
        child: MenuOptionCard(...),
      ),
    ),
  ),
)
```

Esto genera cuatro problemas recurrentes:

1. **Explosión de clases y anidación:** por cada nueva bandera o combinación (`FeatureAOrBGate`), se crea otro widget y el árbol crece verticalmente.
2. **Duplicación fuera de la UI:** cuando un catálogo de rutas o un resolver de atajos (`ShortcutResolver`) necesita evaluar las mismas reglas de forma síncrona, termina duplicando `if (item.isFeatureA && ...)` fuera de los widgets.
3. **Banderas booleanas rígidas en modelos de menú:** los ítems de navegación acumulan propiedades (`isFeatureA`, `isFeatureB`, `isAdminOnly`, `permissions`), obligando a modificar el modelo cada vez que nace un nuevo gate.
4. **Múltiples `BlocBuilder` redundantes:** anidar varios Gates suscritos al mismo Cubit multiplica las suscripciones por cada elemento de una lista o menú.

## La solución: `Gate` + `List<GateFilter>`

Separar **el contenedor declarativo (`Gate`)** de **las reglas evaluables (`GateFilter`)**:

```text
✅ Un solo nodo Gate con lista plana de filtros componibles
Gate(
  filters: const [
    FeatureFlagGateFilter(FeatureFlag.orders),
    RolePermissionGateFilter(
      permissions: [ModulePermissions.orders],
    ),
  ],
  child: MenuOptionCard(...),
)
```

La evaluación vive dentro de cada `GateFilter` (`isAllowed(BuildContext)`), lo que permite usar **el mismo filtro**:
- **Declarativamente** dentro de un widget `Gate` (con reactividad automática vía streams).
- **Imperativamente** en resolvers de navegación, buscadores rápidos o guards síncronos (`item.gates.every((f) => f.isAllowed(context))`).

```text
                ┌──────────────────────────────┐
                │   GateFilter (contrato puro) │
                │  • isAllowed(context)        │
                │  • watchStreams(context)     │
                │  • buildFallback(context)    │
                └──────────────┬───────────────┘
                               │
           ┌───────────────────┴───────────────────┐
           ▼                                       ▼
┌────────────────────────┐              ┌─────────────────────────┐
│   Widget Gate (UI)     │              │ Resolver síncrono       │
│ • Suscripción única    │              │ • Nav / Shortcuts       │
│   deduplicada a streams│              │ • gates.every(isAllowed)│
│ • Fallback por filtro  │              └─────────────────────────┘
│ • Sin wrapper extra    │
└────────────────────────┘
```

## Implementación

### 1. Contrato base `GateFilter` y combinadores genéricos

Los combinadores (`AnyGateFilter`, `NotGateFilter`, `PredicateGateFilter`) eliminan la necesidad de crear clases dedicadas para combinaciones booleanas (`A || B`, `!A`, o reglas puntuales):

```dart
import 'package:flutter/widgets.dart';

/// Contrato base para cualquier regla de acceso o visibilidad usada en [Gate].
abstract class GateFilter {
  const GateFilter();

  /// Evalúa de forma síncrona si este filtro permite mostrar el contenido.
  bool isAllowed(BuildContext context);

  /// Streams opcionales a los que [Gate] debe suscribirse para re-evaluar
  /// el filtro reactivamente cuando cambie el estado subyacente.
  Iterable<Stream<Object?>> watchStreams(BuildContext context) => const [];

  /// Fallback opcional cuando este filtro es quien bloquea el acceso.
  Widget? buildFallback(BuildContext context) => null;
}

/// Combinador lógico OR: permite el acceso si al menos uno de [filters] pasa.
class AnyGateFilter extends GateFilter {
  const AnyGateFilter(this.filters, {this.fallback});

  final List<GateFilter> filters;
  final Widget? fallback;

  @override
  bool isAllowed(BuildContext context) {
    return filters.any((filter) => filter.isAllowed(context));
  }

  @override
  Iterable<Stream<Object?>> watchStreams(BuildContext context) {
    return filters.expand((filter) => filter.watchStreams(context));
  }

  @override
  Widget? buildFallback(BuildContext context) => fallback;
}

/// Combinador lógico NOT: invierte el resultado de [filter].
class NotGateFilter extends GateFilter {
  const NotGateFilter(this.filter, {this.fallback});

  final GateFilter filter;
  final Widget? fallback;

  @override
  bool isAllowed(BuildContext context) => !filter.isAllowed(context);

  @override
  Iterable<Stream<Object?>> watchStreams(BuildContext context) {
    return filter.watchStreams(context);
  }

  @override
  Widget? buildFallback(BuildContext context) => fallback;
}

/// Filtro ad-hoc inline mediante predicado y streams opcionales.
class PredicateGateFilter extends GateFilter {
  const PredicateGateFilter(
    this.predicate, {
    this.watch,
    this.fallback,
  });

  final bool Function(BuildContext context) predicate;
  final Iterable<Stream<Object?>> Function(BuildContext context)? watch;
  final Widget? fallback;

  @override
  bool isAllowed(BuildContext context) => predicate(context);

  @override
  Iterable<Stream<Object?>> watchStreams(BuildContext context) {
    return watch?.call(context) ?? const [];
  }

  @override
  Widget? buildFallback(BuildContext context) => fallback;
}
```

### 2. Widget `Gate` con suscripción deduplicada

`Gate` es un `StatefulWidget` que:
1. Recolecta los streams declarados por todos sus `filters` en un `Set<Stream<Object?>>` para **no suscribirse dos veces al mismo Cubit**.
2. Re-evalúa en orden usando `GateMatchMode.all` (AND por defecto) o `GateMatchMode.any` (OR).
3. Prioriza el `fallback` específico del filtro bloqueante sobre el `fallback` general del `Gate`, o `const SizedBox.shrink()`.
4. **No inserta ningún `RenderObjectWidget` extra** cuando `width == null && height == null`, preservando compatibilidad directa con `Expanded` o `Flexible` dentro de `Row` / `Column`.

```dart
import 'dart:async';
import 'package:flutter/widgets.dart';

enum GateMatchMode {
  /// Todos los filtros deben permitir el acceso (AND lógico).
  all,

  /// Basta con que al menos un filtro permita el acceso (OR lógico).
  any,
}

class Gate extends StatefulWidget {
  const Gate({
    super.key,
    required this.filters,
    required this.child,
    this.matchMode = GateMatchMode.all,
    this.fallback,
    this.width,
    this.height,
  });

  final List<GateFilter> filters;
  final Widget child;
  final GateMatchMode matchMode;
  final Widget? fallback;
  final double? width;
  final double? height;

  @override
  State<Gate> createState() => _GateState();
}

class _GateState extends State<Gate> {
  final List<StreamSubscription<Object?>> _subscriptions = [];
  bool _initialized = false;

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    if (!_initialized) {
      _initialized = true;
      _subscribe();
    }
  }

  @override
  void didUpdateWidget(covariant Gate oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (!_filtersMatch(oldWidget.filters, widget.filters)) {
      _unsubscribe();
      _subscribe();
    }
  }

  @override
  void dispose() {
    _unsubscribe();
    super.dispose();
  }

  bool _filtersMatch(List<GateFilter> a, List<GateFilter> b) {
    if (identical(a, b)) return true;
    if (a.length != b.length) return false;
    for (var i = 0; i < a.length; i++) {
      if (a[i] != b[i]) return false;
    }
    return true;
  }

  void _subscribe() {
    final streams = <Stream<Object?>>{};
    for (final filter in widget.filters) {
      streams.addAll(filter.watchStreams(context));
    }

    for (final stream in streams) {
      _subscriptions.add(
        stream.listen((_) {
          if (mounted) setState(() {});
        }),
      );
    }
  }

  void _unsubscribe() {
    for (final sub in _subscriptions) {
      sub.cancel();
    }
    _subscriptions.clear();
  }

  @override
  Widget build(BuildContext context) {
    final (isAllowed, blockingFilter) = _evaluate(context);

    final Widget result;
    if (isAllowed) {
      result = widget.child;
    } else {
      result =
          blockingFilter?.buildFallback(context) ??
          widget.fallback ??
          const SizedBox.shrink();
    }

    if (widget.width != null || widget.height != null) {
      return SizedBox(
        width: widget.width,
        height: widget.height,
        child: result,
      );
    }
    return result;
  }

  (bool, GateFilter?) _evaluate(BuildContext context) {
    if (widget.filters.isEmpty) return (true, null);

    switch (widget.matchMode) {
      case GateMatchMode.all:
        for (final filter in widget.filters) {
          if (!filter.isAllowed(context)) return (false, filter);
        }
        return (true, null);

      case GateMatchMode.any:
        GateFilter? fallbackCandidate;
        for (final filter in widget.filters) {
          if (filter.isAllowed(context)) return (true, null);
          fallbackCandidate ??=
              filter.buildFallback(context) != null ? filter : null;
        }
        return (false, fallbackCandidate);
    }
  }
}
```

### 3. Filtros concretos agnósticos y testables

Cada filtro mantiene constructor `const` e inyección opcional de dependencias (con fallback seguro a `getIt.isRegistered<T>()` o `context.read`) para funcionar tanto en producción como en tests aislados:

```dart
enum PermissionMode { whitelist, blacklist }

class RolePermissionGateFilter extends GateFilter {
  const RolePermissionGateFilter({
    required this.permissions,
    this.mode = PermissionMode.whitelist,
    this.fallback,
    this.authCubit,
  });

  final List<Enum> permissions;
  final PermissionMode mode;
  final Widget? fallback;
  final AuthCubit? authCubit;

  AuthCubit? _resolveAuth() {
    if (authCubit != null) return authCubit;
    return getIt.isRegistered<AuthCubit>() ? getIt<AuthCubit>() : null;
  }

  @override
  bool isAllowed(BuildContext context) {
    final authState = _resolveAuth()?.state;
    if (authState is! Authenticated) return false;

    final user = authState.session.user;
    return switch (mode) {
      PermissionMode.whitelist => permissions.any(
        (p) => AccessManager().hasAccessTo(user, p.name),
      ),
      PermissionMode.blacklist => permissions.every(
        (p) => !AccessManager().hasAccessTo(user, p.name),
      ),
    };
  }

  @override
  Iterable<Stream<Object?>> watchStreams(BuildContext context) {
    final auth = _resolveAuth();
    return auth != null ? [auth.stream] : const [];
  }

  @override
  Widget? buildFallback(BuildContext context) => fallback;
}
```

Ejemplo de filtro combinado que observa dos Cubits globales (`AuthCubit` para capacidades del `Tenant` y `AppSettingsCubit` para preferencias del usuario):

```dart
class ExpirationTrackingGateFilter extends GateFilter {
  const ExpirationTrackingGateFilter({
    this.fallback,
    this.authCubit,
    this.settingsCubit,
  });

  final Widget? fallback;
  final AuthCubit? authCubit;
  final AppSettingsCubit? settingsCubit;

  AuthCubit? _resolveAuth() =>
      authCubit ?? (getIt.isRegistered<AuthCubit>() ? getIt<AuthCubit>() : null);

  AppSettingsCubit? _resolveSettings() =>
      settingsCubit ??
      (getIt.isRegistered<AppSettingsCubit>() ? getIt<AppSettingsCubit>() : null);

  @override
  bool isAllowed(BuildContext context) {
    final authState = _resolveAuth()?.state;
    if (authState is! Authenticated) return false;
    if (!authState.tenant.supportsExpirationTracking) return false;

    return _resolveSettings()?.useExpirationTracking ?? false;
  }

  @override
  Iterable<Stream<Object?>> watchStreams(BuildContext context) {
    final auth = _resolveAuth();
    final settings = _resolveSettings();
    return [
      if (auth != null) auth.stream,
      if (settings != null) settings.stream,
    ];
  }

  @override
  Widget? buildFallback(BuildContext context) => fallback;
}
```

### 4. Uso en UI y en registros de navegación

#### Fallbacks diferenciados en una misma pantalla
```dart
Gate(
  filters: [
    RootUserGateFilter(
      fallback: const AccessDeniedView(message: 'Requiere usuario root'),
    ),
    RolePermissionGateFilter(
      permissions: const [Modules.settings],
      fallback: const AccessDeniedView(message: 'Permiso insuficiente'),
    ),
  ],
  child: const AdminSettingsBody(),
)
```

#### Composición OR + AND sin clases extra
```dart
Gate(
  filters: const [
    AnyGateFilter([
      QuotationsGateFilter(),
      OrdersGateFilter(),
    ]),
    RolePermissionGateFilter(
      permissions: [ModulePermissions.searchDocuments],
    ),
  ],
  child: const DocumentSearchCard(),
)
```

#### Catálogo de menú y resolver síncrono unificados
En lugar de booleanos sueltos en el descriptor del ítem de navegación, basta una propiedad `List<GateFilter> gates`:

```dart
class NavSubmoduleItem {
  const NavSubmoduleItem({
    required this.title,
    required this.routeName,
    this.gates = const [],
  });

  final String title;
  final String routeName;
  final List<GateFilter> gates;
}

// 1. En el menú declarativo:
Widget buildNavCard(NavSubmoduleItem item) {
  Widget card = NavCard(item: item);
  if (item.gates.isNotEmpty) {
    card = Gate(filters: item.gates, child: card);
  }
  return card;
}

// 2. En el resolver síncrono de atajos o búsqueda rápida:
bool canAccessRoute(BuildContext context, NavSubmoduleItem item) {
  return item.gates.every((filter) => filter.isAllowed(context));
}
```

## Reglas de implementación

1. **Seguridad real en el backend:** los `Gate` solo mejoran UX ocultando UI inaccesible; cada endpoint en el servidor debe validar permisos y feature flags de forma independiente.
2. **Constructores `const`:** mantén todos los `GateFilter` inmutables y `const` siempre que sea posible, para que `_filtersMatch` evite re-suscripciones innecesarias en `didUpdateWidget`.
3. **Resolución segura de dependencias:** verifica `getIt.isRegistered<T>()` y acepta inyección opcional por constructor para evitar acoplar tests de widgets a todo el árbol de providers.
4. **No envolver en `SizedBox` por defecto:** solo aplica `SizedBox` cuando `width != null || height != null`, para no romper `Expanded` / `Flexible` hijos dentro de `Row` o `Column`.
5. **Cancelar suscripciones en `dispose()`:** toda suscripción registrada en `_GateState._subscribe` debe cancelarse en `_unsubscribe` al actualizar filtros o desmontar el widget.
6. **Compatibilidad con `TextInputFormatter` mindset:** cada filtro debe tener una única responsabilidad; evita filtros monolíticos cuando puedas componer dos filtros pequeños en `filters: [...]`.

## Testing

Al probar reactividad con Cubits en `flutter_test`, usa `await tester.pumpAndSettle()` después de `cubit.emit(...)` para dar un frame adicional tras la entrega asíncrona del broadcast stream hacia `setState`:

```dart
testWidgets('Gate reacciona a cambios en AppSettingsCubit y AuthCubit', (tester) async {
  final settingsCubit = createTestSettingsCubit();
  getIt.registerSingleton<AppSettingsCubit>(settingsCubit);

  settingsCubit.emit(AppSettingsLoaded(settingsWithOrdersDisabled));

  await tester.pumpWidget(
    const MaterialApp(
      home: Gate(
        filters: [OrdersGateFilter()],
        fallback: Text('Deshabilitado'),
        child: Text('Órdenes'),
      ),
    ),
  );

  expect(find.text('Órdenes'), findsNothing);
  expect(find.text('Deshabilitado'), findsOneWidget);

  settingsCubit.emit(AppSettingsLoaded(settingsWithOrdersEnabled));
  await tester.pumpAndSettle();

  expect(find.text('Órdenes'), findsOneWidget);
  expect(find.text('Deshabilitado'), findsNothing);
});
```

## Ventajas y trade-offs

| Ventaja | Trade-off |
| :--- | :--- |
| Un solo widget `Gate` en el árbol sin importar cuántos filtros tenga | Requiere mantener actualizado `watchStreams` cuando un filtro depende de un nuevo Cubit |
| Misma regla evaluable en UI (`Gate`) y lógica síncrona (`ShortcutResolver`) | En tests que emiten eventos al stream después del primer frame conviene usar `pumpAndSettle()` |
| Combinación libre (`AnyGateFilter`, `NotGateFilter`, `PredicateGateFilter`) | Los filtros con `fallback` dinámico (callbacks) no pueden ser `const` si capturan closures |
| Compatible con `Expanded` / `Flexible` al no crear `RenderObject` intermedio | — |

## Checklist para adoptarlo en otro proyecto

- [ ] Definir `GateFilter` (`isAllowed`, `watchStreams`, `buildFallback`) y los combinadores (`AnyGateFilter`, `NotGateFilter`, `PredicateGateFilter`).
- [ ] Implementar `Gate` (`StatefulWidget`) con `GateMatchMode.all` / `any`, deduplicación de streams y limpieza en `dispose`.
- [ ] Crear filtros concretos por cada eje transversal (permisos de rol, usuario root, feature flags, capacidades del tenant).
- [ ] Reemplazar banderas booleanas de visibilidad en modelos de menú/navegación por `List<GateFilter> gates`.
- [ ] Exportar `Gate` y filtros desde un único barrel (`widgets/gates/gates.dart`).
- [ ] Cubrir evaluación AND/OR, precedencia de `fallback` y reactividad ante cambios de streams en `widget_test`.
