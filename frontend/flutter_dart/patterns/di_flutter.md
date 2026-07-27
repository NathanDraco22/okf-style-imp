---
type: Pattern
title: Inyección de Dependencias en Flutter
description: Las dependencias se inyectan mediante RepositoryProvider + BlocProvider de flutter_bloc, con GetIt como service locator para cubits globales.
tags: [flutter, di, dependencias, provider, patron]
---

# Inyección de Dependencias

Se usan dos mecanismos de DI según el ámbito.

## 1. flutter_bloc (RepositoryProvider + BlocProvider)

Para dependencias por módulo en el árbol de widgets.

```dart
// main.dart - Registro global de repositorios
MultiRepositoryProvider(
  providers: [
    RepositoryProvider<ItemRepository>(
      create: (_) => ItemRepository(ItemDataSource()),
    ),
    RepositoryProvider<ItemRepository>(
      create: (_) => ItemRepository(ItemDataSource()),
    ),
    // ... todos los repositorios
  ],
  child: MultiBlocProvider(
    providers: [
      BlocProvider<ReadItemCubit>(
        create: (_) => ReadItemCubit(itemRepository: ItemRepository(ItemDataSource())),
      ),
    ],
    child: MaterialApp.router(...),
  ),
)
```

## Uso en Screens

```dart
// Obtener repositorio desde el árbol
final repo = RepositoryProvider.of<ItemRepository>(context);

// Pasar al cubit
BlocProvider(
  create: (ctx) => ReadItemCubit(
    itemRepository: RepositoryProvider.of<ItemRepository>(ctx),
  )..getAll(),
),
```

## 2. GetIt (service locator)

Para dependencias globales que necesitan acceso fuera del widget tree.

```dart
// get_it_config.dart
final getIt = GetIt.instance;

void setupGetIt() {
  getIt.registerLazySingleton<SessionCubit>(() => SessionCubit());
}
```

## Uso en DataSources

```dart
class ImageUploader {
  final sessionCubit = getIt<SessionCubit>();  // Acceso desde cualquier lugar
}
```

## ¿Cuándo usar cada uno?

| Mecanismo | Ámbito | Cuándo usarlo |
|-----------|--------|---------------|
| `RepositoryProvider` | Árbol de widgets | Repositorios que solo usan los cubits |
| `BlocProvider` | Árbol de widgets | Cubits que viven en el árbol |
| `GetIt` | Global | Cubits globales (Session, AppState) accesibles desde código no-Widget |

## Reglas

- Preferir `RepositoryProvider` + `BlocProvider` sobre GetIt
- GetIt solo para casos donde no se tiene acceso al `BuildContext` (data sources, servicios)
- La inyección es **manual** — no hay frameworks de DI automáticos
- Los DataSources se crean directamente (son singletons) y se pasan a los Repositorios
