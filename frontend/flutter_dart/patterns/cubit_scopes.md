---
type: Pattern
title: Alcance y Ciclo de Vida de Cubits
description: Regla de ubicación y creación de cubits: los globales viven en GetIt desde un composition root, los compartidos y de pantalla usan providers, y los diálogos reciben su cubit con BlocProvider.value o lo crean en el show function.
tags: [flutter, cubit, get_it, provider, di, ciclo-de-vida, composition-root, patron]
---

# Alcance y Ciclo de Vida de Cubits

El alcance define el mecanismo, no al revés:

- **Global** → GetIt, creado una sola vez en el composition root.
- **Compartido entre módulos** → provider en el punto común más alto.
- **Pantalla** → `MultiBlocProvider` en el entry point.
- **Diálogo** → `BlocProvider.value` o creación dentro del `showXxx`.

GetIt existe **solo** para estado global usado en muchos lugares o accesible
sin `BuildContext`. Nunca para estado de pantalla o de diálogo.

## Tabla de alcances

| Alcance | Ubicación | Creación | Consumo | Vida |
|---|---|---|---|---|
| Global | `lib/cubits/{entity}/` | `AppDependencies.create()` en `main()` + `getIt.registerSingleton` | `BlocBuilder(bloc: getIt<T>())`, `BlocListener(bloc: getIt<T>())`, `getIt<T>()` | Toda la app |
| Compartido | `lib/src/cubits/{entity}/` | Provider en el shell o módulo común | `context.watch/read` | La ruta/módulo que lo provee |
| Pantalla | `modules/{feature}/cubit/` | `MultiBlocProvider` en el entry | `context.watch/read` | La pantalla |
| Diálogo | `modules/{feature}/dialogs/` o `widgets/dialogs/` | `BlocProvider.value` o creación en `showXxx` | `context.watch/read` | El diálogo |

## 1. Global: GetIt

### Cuándo

- El cubit se usa desde código sin `BuildContext`: redirects del router,
  guards, data sources, uploaders, formatters, access managers.
- El estado debe sobrevivir a la navegación y lo consultan módulos que no
  comparten un ancestro en el árbol.
- Se usa en muchos lugares y una sola instancia es la fuente de verdad.

Si el cubit solo lo usa una pantalla, **no** va a GetIt.

### Composition root

Los globales se crean en `AppDependencies`, antes de `runApp`:

```dart
// lib/app_dependencies.dart
import 'package:{app}/config/get_it_config.dart';
import 'package:{app}/cubits/app_session/app_session_cubit.dart';
import 'package:{app}/cubits/app_settings/app_settings_cubit.dart';
import 'package:{app}_business/{app}_business.dart';

class AppDependencies {
  AppDependencies._({
    required this.appSessionCubit,
    required this.appSettingsCubit,
    required this.authRepository,
    required this.itemsRepository,
  });

  factory AppDependencies.create() {
    final authDataSource = AuthDataSource();
    final itemsDataSource = ItemsDataSource();
    final settingsDataSource = AppSettingsDataSource();

    final authRepository = AuthRepository(authDataSource);
    final itemsRepository = ItemsRepository(itemsDataSource);
    final settingsRepository = AppSettingsRepository(settingsDataSource);

    final appSettingsCubit = AppSettingsCubit(settingsRepository);
    final appSessionCubit = AppSessionCubit(
      authRepository: authRepository,
    );

    getIt
      ..registerSingleton<AppSettingsCubit>(appSettingsCubit)
      ..registerSingleton<AppSessionCubit>(appSessionCubit);

    return AppDependencies._(
      appSessionCubit: appSessionCubit,
      appSettingsCubit: appSettingsCubit,
      authRepository: authRepository,
      itemsRepository: itemsRepository,
    );
  }

  final AppSessionCubit appSessionCubit;
  final AppSettingsCubit appSettingsCubit;
  final AuthRepository authRepository;
  final ItemsRepository itemsRepository;
}
```

```dart
// lib/main.dart
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  await Hive.initFlutter();
  initServices();

  final dependencies = AppDependencies.create();

  runApp(MyApp(dependencies: dependencies));
}
```

`AppDependencies` también expone los repositorios para que el árbol los
provea. No se registran repositorios en GetIt.

### ProviderContainer

El contenedor raíz solo provee repositorios:

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

No hay `MultiBlocProvider` para globales ni un `GlobalCubitProvider`. Si un
cubit es global, se registra en GetIt y se consume con `bloc:`.

### Consumo

```dart
BlocBuilder<AppSessionCubit, AppSessionState>(
  bloc: getIt<AppSessionCubit>(),
  builder: (context, state) {
    return switch (state) {
      AppSessionAuthenticated(:final user) => Text(user.name),
      _ => const SizedBox.shrink(),
    };
  },
)
```

```dart
BlocListener<ToastCubit, ToastState>(
  bloc: getIt<ToastCubit>(),
  listener: (context, state) {
    // side effect
  },
  child: child,
)
```

Para una lectura puntual, sin reconstruir:

```dart
final branchId = getIt<AppSessionCubit>().currentBranchId;
```

Reglas de consumo:

- El cubit global **no** se provee en el árbol.
- No se lee con `context.watch` ni `context.read`.
- No se registra dos veces (GetIt + `BlocProvider.value`).
- `getIt.isRegistered<T>()` solo para dependencias opcionales o de arranque
  temprano; no es el camino normal.

### Ciclo de vida y tests

- Los globales viven toda la vida de la app. No hay un widget que los cierre.
- Si abren un `StreamSubscription`, este acompaña a la app; es aceptable para
  estado global. Si el cubit puede reemplazarse, exponer `close()` y llamarlo
  explícitamente.
- En tests:

```dart
setUp(() async {
  await getIt.reset();
});

tearDown(() async {
  await getIt.reset();
});
```

`getIt.reset()` desregistra las instancias; si el test necesita cerrar streams,
hacerlo antes del reset.

### Servicios vs cubits

| Registro | Dónde | Qué contiene |
|---|---|---|
| Servicios | `config/get_it_config.dart` → `initServices()` | Printer, storage, analytics, clients |
| Cubits globales | `app_dependencies.dart` → `AppDependencies.create()` | Sesión, settings, tema, carrito, etc. |

`initServices()` no registra cubits y `AppDependencies` no registra servicios.

## 2. Compartido entre módulos

Cuando dos o más módulos necesitan el mismo estado pero **no** es global de
toda la app, el cubit va en `lib/src/cubits/{entity}/` y se provee en el
ancestro común:

```dart
MultiBlocProvider(
  providers: [
    BlocProvider(
      create: (ctx) => ReadOrderCubit(
        RepositoryProvider.of<OrdersRepository>(ctx),
      )..getAll(),
    ),
  ],
  child: const OrdersShell(),
)
```

- No va a GetIt.
- Si el ancestro común es toda la app, se vuelve global y se reconsidera el
  alcance; no se resuelve duplicando la instancia.
- `context.watch/read` es el acceso correcto.

## 3. Pantalla

El entry point crea los cubits y los provee **arriba** del dispatch responsive:

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

- Se crean una vez por entrada a la ruta.
- `ReadCubit` dispara su carga en `create` con `..getAll()`.
- `WriteCubit` no carga solo; espera la acción del usuario.
- Los providers van arriba del `_Body` que elige mobile/tablet/desktop. Ver
  [Diseño Responsive](responsive_design.md).

## 4. Diálogo

Si el diálogo **reutiliza** un cubit de la pantalla, se pasa con
`BlocProvider.value`:

```dart
Future<void> showItemEditor(
  BuildContext context, {
  required WriteItemCubit writeCubit,
}) {
  return showDialog(
    context: context,
    builder: (_) => BlocProvider.value(
      value: writeCubit,
      child: const ItemEditorDialog(),
    ),
  );
}
```

Si el diálogo es autónomo, el cubit se crea dentro del `showXxx` y se cierra
con el diálogo:

```dart
Future<void> showCreateItem(BuildContext context) {
  final repository = context.read<ItemsRepository>();
  final writeCubit = WriteItemCubit(repository);

  return showDialog(
    context: context,
    builder: (_) => BlocProvider.value(
      value: writeCubit,
      child: const CreateItemDialog(),
    ),
  ).whenComplete(writeCubit.close);
}
```

Reglas:

- No crear el cubit dentro del `build` del diálogo.
- Si la pantalla ya tiene el `WriteCubit`, pasarlo; no crear un segundo.
- El diálogo no busca cubits de pantalla con `getIt`.

## 5. Árbol de decisión

```text
¿Se usa sin BuildContext o desde muchos módulos no emparentados?
  └── Sí → Global: lib/cubits/ + AppDependencies + getIt
  └── No → ¿Lo comparten varios módulos con un ancestro común?
        └── Sí → Compartido: lib/src/cubits/ + provider en el ancestro
        └── No → ¿Es estado de una pantalla?
              └── Sí → Pantalla: MultiBlocProvider en el entry
              └── No → ¿Es una mutación efímera?
                    └── Sí → Diálogo: BlocProvider.value o crear en showXxx
```

## 6. Anti-patrones

- **`GlobalCubitProvider` o wrapper similar:** un widget que crea globales y
  las registra en GetIt. Reemplazado por `AppDependencies`.
- **Doble registro:** `getIt.registerSingleton` + `BlocProvider.value` de la
  misma instancia. Elegir un solo camino.
- **Mezclar accesos:** `context.watch<GlobalCubit>()` para un cubit que solo
  está en GetIt, o `getIt<ScreenCubit>()` para uno de pantalla.
- **Crear cubits en `build()`** o dentro de un `BlocBuilder`.
- **Crear el cubit de pantalla dentro de la variante responsive:** al cambiar
  el ancho se recrea el estado.
- **Usar GetIt para estado de pantalla o diálogo.**
- **Olvidar `getIt.reset()` en tests:** una instancia global contamina el
  siguiente test.

## 7. Migración desde `GlobalCubitProvider`

Si el proyecto ya tiene un wrapper que crea globales:

1. Crear `AppDependencies` con los data sources, repositorios y globales.
2. Mover la creación de `initState` a `AppDependencies.create()`.
3. Registrar los globales en GetIt.
4. Eliminar `getIt.unregister` y `close()` del `dispose` del wrapper.
5. Eliminar los `BlocProvider.value` de esos globales.
6. Reemplazar `context.watch/read<GlobalCubit>` por
   `bloc: getIt<GlobalCubit>()`.
7. Borrar el wrapper y el `MultiBlocProvider` de globales.
8. Ejecutar `flutter analyze` y los tests con `getIt.reset()`.

## 8. Checklist

- [ ] El cubit global está en `lib/cubits/{entity}/`.
- [ ] Se crea en `AppDependencies.create()`, no en un widget.
- [ ] Está registrado en GetIt y no se provee en el árbol.
- [ ] Se consume con `bloc: getIt<T>()` o `getIt<T>()`.
- [ ] No hay doble registro.
- [ ] Los cubits de pantalla están en `MultiBlocProvider` arriba del dispatch.
- [ ] Los diálogos usan `BlocProvider.value` o crean su cubit en `showXxx`.
- [ ] Los tests hacen `getIt.reset()`.

## Referencias

- [Inyección de Dependencias](di_flutter.md)
- [Estructura de Pantalla](screen_structure.md)
- [Diseño Responsive](responsive_design.md)
- [Read / Write Cubits](read_write_cubits.md)
- [Funciones de Diálogo](dialog_show_functions.md)
- [Estructura del Proyecto Flutter](../style/project_structure_flutter.md)
