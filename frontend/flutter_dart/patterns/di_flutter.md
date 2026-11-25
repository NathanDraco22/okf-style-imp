---
type: Pattern
title: Inyección de Dependencias en Flutter
description: Repositorios con RepositoryProvider, cubits locales con BlocProvider, cubits globales y servicios con GetIt desde un composition root; sin doble registro.
tags: [flutter, di, dependencias, provider, get_it, composition-root, patron]
---

# Inyección de Dependencias

Hay tres mecanismos y cada uno tiene un alcance distinto:

| Dependencia | Mecanismo | Dónde se crea | Vida |
|---|---|---|---|
| DataSource | Constructor directo | Composition root | Toda la app |
| Repositorio | `RepositoryProvider` | Composition root o módulo | La app o el módulo |
| Cubit global | GetIt | `AppDependencies.create()` | Toda la app |
| Cubit de pantalla | `BlocProvider` | Entry de la pantalla | La ruta |
| Cubit de diálogo | `BlocProvider.value` o `BlocProvider` | `showXxx` | El diálogo |
| Servicio | GetIt | `initServices()` | Toda la app |

La regla corta: **GetIt para globales, providers para locales**. Si un cubit
está en GetIt, no se provee en el árbol; si está en el árbol, no se busca con
GetIt.

## 1. Composition root

Los data sources, repositorios y cubits globales se construyen una sola vez en
`AppDependencies`, antes de `runApp`:

```dart
// lib/app_dependencies.dart
class AppDependencies {
  AppDependencies._({
    required this.appSessionCubit,
    required this.authRepository,
    required this.itemsRepository,
  });

  factory AppDependencies.create() {
    final authDataSource = AuthDataSource();
    final itemsDataSource = ItemsDataSource();

    final authRepository = AuthRepository(authDataSource);
    final itemsRepository = ItemsRepository(itemsDataSource);

    final appSessionCubit = AppSessionCubit(
      authRepository: authRepository,
    );

    getIt.registerSingleton<AppSessionCubit>(appSessionCubit);

    return AppDependencies._(
      appSessionCubit: appSessionCubit,
      authRepository: authRepository,
      itemsRepository: itemsRepository,
    );
  }

  final AppSessionCubit appSessionCubit;
  final AuthRepository authRepository;
  final ItemsRepository itemsRepository;
}
```

```dart
// lib/main.dart
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  await Hive.initFlutter();
  initServices(); // GetIt: servicios

  final dependencies = AppDependencies.create();

  runApp(MyApp(dependencies: dependencies));
}
```

## 2. Repositorios en el árbol

El contenedor raíz expone las **mismas** instancias que recibieron los cubits
globales. No se crean repositorios nuevos:

```dart
// lib/provider_container.dart
class ProviderContainer extends StatelessWidget {
  const ProviderContainer({
    super.key,
    required this.dependencies,
    required this.child,
  });

  final AppDependencies dependencies;
  final Widget child;

  @override
  Widget build(BuildContext context) {
    return MultiRepositoryProvider(
      providers: [
        RepositoryProvider.value(value: dependencies.authRepository),
        RepositoryProvider.value(value: dependencies.itemsRepository),
      ],
      child: child,
    );
  }
}
```

Si un repositorio se crea dentro del árbol, usar `create` y dejar que el
provider gestione su ciclo de vida:

```dart
RepositoryProvider(
  create: (_) => ItemsRepository(ItemsDataSource()),
)
```

Nunca crear dos repositorios para el mismo data source: el segundo pierde los
eventos del `ReactiveRepository` y duplica estado.

## 3. Cubits locales

### Pantalla

```dart
class ItemsScreen extends StatelessWidget {
  const ItemsScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return MultiBlocProvider(
      providers: [
        BlocProvider(
          create: (ctx) => ReadItemCubit(
            RepositoryProvider.of<ItemsRepository>(ctx),
          )..getAll(),
        ),
        BlocProvider(
          create: (ctx) => WriteItemCubit(
            RepositoryProvider.of<ItemsRepository>(ctx),
          ),
        ),
      ],
      child: const _RootScaffold(),
    );
  }
}
```

- El `ReadCubit` carga en `create` con `..getAll()`.
- El `WriteCubit` espera la acción del usuario.
- Los providers van arriba del dispatch responsive. Ver
  [Diseño Responsive](responsive_design.md) y
  [Alcance y Ciclo de Vida de Cubits](cubit_scopes.md).

### Diálogo

```dart
Future<void> showItemEditor(BuildContext context) {
  final writeCubit = WriteItemCubit(
    context.read<ItemsRepository>(),
  );

  return showDialog(
    context: context,
    builder: (_) => BlocProvider.value(
      value: writeCubit,
      child: const ItemEditorDialog(),
    ),
  ).whenComplete(writeCubit.close);
}
```

Si la pantalla ya tiene el `WriteCubit`, pasarlo con `BlocProvider.value` en
lugar de crear otro.

## 4. Cubits globales con GetIt

Se registran en `AppDependencies` y se consumen con `bloc:`:

```dart
BlocBuilder<AppSessionCubit, AppSessionState>(
  bloc: getIt<AppSessionCubit>(),
  builder: (context, state) => Text(state.userName),
)
```

```dart
final branchId = getIt<AppSessionCubit>().currentBranchId;
```

- No se proveen en el árbol.
- No se leen con `context.watch/read`.
- No se registran dos veces.
- En tests: `getIt.reset()`.

Detalle completo en [Alcance y Ciclo de Vida de Cubits](cubit_scopes.md).

## 5. Servicios

```dart
// lib/config/get_it_config.dart
final getIt = GetIt.instance;

void initServices() {
  if (!getIt.isRegistered<ThermalPrinterService>()) {
    getIt.registerLazySingleton<ThermalPrinterService>(
      () => ThermalPrinterService(),
    );
  }
}
```

Los servicios también pueden registrarse en `AppDependencies` si dependen de
configuración de la app. Lo que no se hace es registrar cubits de pantalla en
GetIt.

## 6. Reglas

- **Una instancia, un dueño.** Si `AppDependencies` crea un repositorio, el
  árbol lo recibe con `.value`; no se vuelve a construir.
- **GetIt solo para globales y servicios.** Nunca para pantalla o diálogo.
- **Providers para todo lo demás.** El alcance más chico posible.
- **Sin doble registro.** GetIt y `BlocProvider.value` son caminos
  excluyentes para una misma instancia.
- **Inyección por constructor.** Los cubits reciben repositorios; no los
  buscan con `getIt` por dentro.
- **Tests.** `getIt.reset()` y proveer repositorios/cubits locales en el
  widget bajo prueba.
- **Ciclo de vida.** `BlocProvider` cierra sus cubits; los globales de GetIt
  viven toda la app y se limpian explícitamente en tests.

## Referencias

- [Alcance y Ciclo de Vida de Cubits](cubit_scopes.md)
- [Estructura de Pantalla](screen_structure.md)
- [Diseño Responsive](responsive_design.md)
- [Read / Write Cubits](read_write_cubits.md)
- [ReactiveRepository](reactive_repository.md)
- [Estructura del Proyecto Flutter](../style/project_structure_flutter.md)
