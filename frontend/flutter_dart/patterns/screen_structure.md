---
type: Pattern
title: Estructura de Pantalla
description: Patrón de organización de cada pantalla: MultiBlocProvider envuelve un Scaffold con BlocListener para side effects y BlocBuilder para UI.
tags: [flutter, screen, estructura, patron, ui]
---

# Estructura de Pantalla

Cada pantalla sigue la misma estructura de 3 partes y vive en un archivo
`{feature}_screen.dart` dentro de `modules/{feature}/view/`.

> Esta estructura aplica a cubits **no globales** (`lib/src/cubits/{feature}/`).
> Los cubits **globales** no se crean aquí: viven en `lib/cubits/`, se
> registran en GetIt y se consumen con `bloc: getIt<T>()`. Ver
> [Alcance y Ciclo de Vida de Cubits](cubit_scopes.md).

## 1. Screen pública (entry point)

Crea los cubits con sus dependencias y los provee al árbol.

```dart
// modules/items/view/items_screen.dart
class ItemsScreen extends StatelessWidget {
  const ItemsScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return MultiBlocProvider(
      providers: [
        BlocProvider(
          create: (ctx) => ReadItemCubit(
            RepositoryProvider.of<ItemRepository>(ctx),
          )..getAll(),
        ),
        BlocProvider(
          create: (ctx) => WriteItemCubit(
            RepositoryProvider.of<ItemRepository>(ctx),
          ),
        ),
      ],
      child: const _RootScaffold(),
    );
  }
}
```

## 2. Root Scaffold (privado, con BlocListener para side effects)

```dart
class _RootScaffold extends StatelessWidget {
  const _RootScaffold();

  @override
  Widget build(BuildContext context) {
    return BlocListener<WriteItemCubit, WriteItemState>(
      listener: (context, state) {
        switch (state) {
          case WritingItem():
            LoadingDialogManager.showLoadingDialog(context);
          case ItemCreated(:final item):
            ScaffoldMessenger.of(context).showSnackBar(
              SnackBar(content: Text('${item.name} creado')),
            );
          case ItemUpdated(:final item):
            ScaffoldMessenger.of(context).showSnackBar(
              SnackBar(content: Text('${item.name} actualizado')),
            );
          case WriteItemError(:final message):
            ScaffoldMessenger.of(context).showSnackBar(
              SnackBar(content: Text('Error: $message')),
            );
          default:
        }
      },
      child: Scaffold(
        appBar: AppBar(title: const Text('Pacientes')),
        body: const _Body(),
      ),
    );
  }
}
```

## 3. Body (UI reactiva con BlocBuilder)

```dart
class _Body extends StatelessWidget {
  const _Body();

  @override
  Widget build(BuildContext context) {
    return BlocBuilder<ReadItemCubit, ReadItemState>(
      builder: (context, state) {
        return switch (state) {
          ReadItemLoading() => const Center(child: CircularProgressIndicator()),
          ReadItemSuccess(:final items) => _ItemsList(items: items),
          ReadItemError(:final message) => Center(child: Text('Error: $message')),
          _ => const SizedBox(),
        };
      },
    );
  }
}
```

## 4. Providers y responsive

La screen pública provee los cubits y **después** el body elige la variante:

```text
ItemsScreen (MultiBlocProvider)
  └── _RootScaffold
        └── _Body  ← context.isMobile() / isTablet() / isDesktop()
              ├── _ItemsMobileBody
              ├── _ItemsTabletBody
              └── _ItemsDesktopBody
```

Reglas:

- El `MultiBlocProvider` va **arriba** de `_Body`, nunca dentro de una
  variante. Si se coloca abajo, redimensionar la ventana recrea los cubits y
  se pierde el estado.
- La screen es el **único** lugar que instancia los cubits no globales. Las
  variantes y los widgets hijos los consumen por `context`.
- Las variantes no crean cubits ni controllers; reciben datos y callbacks.
- Los cubits globales de GetIt no se proveen aquí.

Si la pantalla necesita estado efímero (formulario, filtros, selección), la
screen es `StatefulWidget` y crea su **ViewController** en `initState`; lo
comparte con las secciones vía Mediator. El cubit solo recibe DTOs. Ver
[ViewController Pattern](view_controller_pattern.md).

Ver [Diseño Responsive](responsive_design.md).

## Reglas

| Capa | Rol | Widget base |
|------|-----|-------------|
| **Screen pública** | Crea cubits locales con DI, provee al árbol | `MultiBlocProvider` |
| **Root Scaffold** | Side effects (loading, SnackBars, navegación) | `BlocListener` + `Scaffold` |
| **Body** | UI reactiva que cambia con el estado | `BlocBuilder` / `BlocSelector` |

- La Screen pública es `const StatelessWidget`
- El Scaffold es privado (prefijo `_`)
- El Body es privado y puede ser `StatefulWidget` si necesita controladores
- Los cubits globales se consumen con `bloc: getIt<T>()`; no se proveen en la
  pantalla. Ver [Alcance y Ciclo de Vida de Cubits](cubit_scopes.md).
