---
type: Pattern
title: Estructura de Pantalla
description: Patrón de organización de cada pantalla: MultiBlocProvider envuelve un Scaffold con BlocListener para side effects y BlocBuilder para UI.
tags: [flutter, screen, estructura, patron, ui]
---

# Estructura de Pantalla

Cada pantalla sigue la misma estructura de 3 partes.

> Esta estructura aplica a cubits de **pantalla** y **compartidos**. Los cubits
> **globales** no se crean aquí: viven en GetIt y se consumen con
> `bloc: getIt<T>()`. Ver [Alcance y Ciclo de Vida de Cubits](cubit_scopes.md).

## 1. Screen pública (entry point)

Crea los cubits con sus dependencias y los provee al árbol.

```dart
class ItemScreen extends StatelessWidget {
  const ItemScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return MultiBlocProvider(
      providers: [
        BlocProvider(
          create: (ctx) => ReadItemCubit(
            itemRepository: RepositoryProvider.of<ItemRepository>(ctx),
          )..getAll(),
        ),
        BlocProvider(
          create: (ctx) => WriteItemCubit(
            itemRepository: RepositoryProvider.of<ItemRepository>(ctx),
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
- Las variantes no crean cubits ni controllers; reciben datos y callbacks.
- Los cubits globales de GetIt no se proveen aquí.

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
