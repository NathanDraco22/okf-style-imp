---
type: Pattern
title: Alcance y Ciclo de Vida de Cubits
description: Regla de ubicación y creación de cubits: globales en lib/cubits con GetIt, no globales en lib/src/cubits/{feature} con providers, diálogos con BlocProvider.value y sincronización por ReactiveRepository; el ViewController queda para estado efímero.
tags: [flutter, cubit, get_it, provider, di, ciclo-de-vida, composition-root, viewcontroller, patron]
---

# Alcance y Ciclo de Vida de Cubits

La ubicación no es opcional:

| Alcance | Ubicación | Mecanismo | Consumo |
|---|---|---|---|
| **Global** | `lib/cubits/{feature}/` | GetIt desde `AppDependencies.create()` | `bloc: getIt<T>()`, `getIt<T>()` |
| **No global** | `lib/src/cubits/{feature}/` | `MultiBlocProvider` en la screen o ancestro común | `context.watch/read` |
| **Diálogo** | `lib/src/cubits/{feature}/` | `BlocProvider.value` o creación en `showXxx` | `context.watch/read` |

- `lib/cubits/` es **solo** para estado global.
- `lib/src/cubits/` es para **todo** cubit que no sea global, separado por
  feature en plural (`products/`, `orders/`, `categories/`).
- No existe `modules/{feature}/cubit/`.
- El **ViewController** no es un cubit: vive en el módulo y maneja estado
  efímero de la pantalla.

## 1. Global: `lib/cubits/`

### Cuándo

- El cubit se usa desde código sin `BuildContext`: redirects del router,
  guards, data sources, uploaders, formatters, access managers.
- El estado debe sobrevivir a la navegación y lo consultan módulos que no
  comparten un ancestro en el árbol.
- Se usa en muchos lugares y una sola instancia es la fuente de verdad.

Si el cubit solo lo usa una pantalla, **no** va a `lib/cubits/`.

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

### Servicios vs cubits

| Registro | Dónde | Qué contiene |
|---|---|---|
| Servicios | `config/get_it_config.dart` → `initServices()` | Printer, storage, analytics, clients |
| Cubits globales | `app_dependencies.dart` → `AppDependencies.create()` | Sesión, settings, tema, carrito global, etc. |

`initServices()` no registra cubits y `AppDependencies` no registra servicios.

## 2. No globales: `lib/src/cubits/{feature}/`

Todo cubit que no sea global vive aquí, con una carpeta por feature en plural:

```text
lib/src/cubits/
├── products/
│   ├── read_products_cubit.dart
│   ├── read_products_state.dart
│   ├── write_products_cubit.dart
│   └── write_products_state.dart
├── orders/
│   ├── read_orders_cubit.dart
│   ├── read_orders_state.dart
│   ├── write_orders_cubit.dart
│   └── write_orders_state.dart
└── categories/
    └── ...
```

Reglas:

- **Read y Write separados.** Nunca un cubit que haga lectura y escritura.
- `Read{Entity}Cubit` carga (`getAll`, `getById`, `refresh`) y escucha el
  `eventStream` del repositorio.
- `Write{Entity}Cubit` muta (`create`, `update`, `delete`) y resetea a
  `Initial` tras cada éxito.
- La carpeta usa el **plural** del feature, igual que los módulos del backend
  (`app/api/v1/products`, `app/api/v1/orders`).
- El nombre del feature debe ser estable: si el backend lo llama `products`,
  la carpeta es `products/`.

El alcance del provider decide la vida, no la carpeta:

- **Pantalla:** `MultiBlocProvider` en el entry; muere con la ruta.
- **Compartido:** provider en el ancestro común; muere con ese subárbol.
- Si dos módulos no comparten ancestro y ambos necesitan el estado, el cubit
  es global y se mueve a `lib/cubits/`.

## 3. Pantalla: `view/{feature}_screen.dart`

Cada vista tiene un archivo `screen` que es el **único** que instancia los
cubits no globales:

```dart
// modules/items/view/items_screen.dart
class ItemsScreen extends StatelessWidget {
  const ItemsScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return MultiBlocProvider(
      providers: [
        BlocProvider(
          create: (ctx) => ReadProductCubit(
            RepositoryProvider.of<ProductsRepository>(ctx),
          )..getAll(),
        ),
        BlocProvider(
          create: (ctx) => WriteProductCubit(
            RepositoryProvider.of<ProductsRepository>(ctx),
          ),
        ),
      ],
      child: const _RootScaffold(),
    );
  }
}
```

- El archivo se llama `{feature}_screen.dart` y vive en
  `modules/{feature}/view/`.
- La screen es el único lugar con `MultiBlocProvider` para esos cubits.
- Las variantes `mobile/`, `tablet/` y `desktop/` y los widgets hijos **no**
  crean cubits; reciben el estado por `context`.
- Los providers van arriba del dispatch responsive. Ver
  [Diseño Responsive](responsive_design.md).
- Si la screen necesita estado efímero, es `StatefulWidget` y crea su
  ViewController en `initState` (ver sección 5).

## 4. Comunicación: ReactiveRepository primero

El paso de cubits por valor (`BlocProvider.value`) es útil para **compartir
una instancia** o inyectar una dependencia en un diálogo, pero no es el
mecanismo para comunicar cubits entre sí.

La sincronización Read/Write va por el repositorio:

```text
WriteCubit.create(dto)
  → Repository.create(dto)
    → DataSource / API
    → notifyItemCreated(item)
      → repository.eventStream
        → ReadCubit actualiza su lista (sin refetch)
```

Reglas:

- `ReadCubit` se suscribe al `eventStream` del `ReactiveRepository` en su
  constructor y cancela en `close()`.
- `WriteCubit` no conoce al `ReadCubit`; solo llama al repositorio.
- El diálogo que crea su propio `WriteCubit` debe usar **la misma instancia**
  del repositorio para que el evento llegue al `ReadCubit`.
- No se pasa un `ReadCubit` a un diálogo solo para refrescar la lista; el
  evento del repositorio ya lo hace.
- `BlocProvider.value` se usa para que el diálogo comparta el `WriteCubit` de
  la pantalla, no para sincronizar datos.

Ver [ReactiveRepository](reactive_repository.md) y
[Read / Write Cubits](read_write_cubits.md).

### Diálogo

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

Si el diálogo es autónomo, el cubit se crea dentro del `showXxx` con el
repositorio del árbol y se cierra con el diálogo:

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

## 5. ViewController: estado efímero

El **ViewController** es un `ChangeNotifier` para el estado en memoria de la
pantalla o de un componente reusable complejo. No es un cubit y no va en
`lib/cubits/` ni en `lib/src/cubits/`.

| Estado | Dónde vive |
|---|---|
| Datos del servidor (listas, CRUD) | Read/Write Cubit en `lib/src/cubits/{feature}/` |
| Formularios, filtros, selecciones, wizard, carrito de una pantalla | ViewController en `modules/{feature}/view_controller.dart` |
| Estado efímero interno de un componente reusable complejo | ViewController propio del componente |
| Estado compartido entre secciones de la pantalla | ViewController + Mediator |

Reglas:

- Se crea en `initState` de la screen (o del componente) y muere con ella.
- Se comparte con las secciones vía **Mediator** (`InheritedNotifier`), sin
  prop-drilling.
- El cubit recibe **DTOs**; nunca se le inyecta el ViewController.
- Si un componente complejo necesita estado propio, tiene su VC; si ese estado
  también lo necesita la pantalla, se eleva al VC de la screen.
- Después de persistir, el action flow llama `viewController.clear()`; la
  lista se sincroniza por el evento del repositorio, no por refetch.

Ver [ViewController Pattern](view_controller_pattern.md) y
[Mediator Pattern](mediator_pattern.md).

## 6. Árbol de decisión

```text
¿Se usa sin BuildContext o desde muchos módulos no emparentados?
  └── Sí → Global: lib/cubits/{feature}/ + AppDependencies + GetIt
  └── No → ¿Es estado del servidor (listas, CRUD)?
        └── Sí → No global: lib/src/cubits/{feature}/ + provider
        └── No → ¿Es estado efímero de la pantalla/componente?
              └── Sí → ViewController + Mediator
              └── No → Revisar el diseño: probablemente sea un cubit
```

## 7. Anti-patrones

- **`GlobalCubitProvider` o wrapper similar:** un widget que crea globales y
  las registra en GetIt. Reemplazado por `AppDependencies`.
- **`modules/{feature}/cubit/`:** los no globales viven en
  `lib/src/cubits/{feature}/`.
- **Mezclar `lib/cubits` y `lib/src/cubits`:** si es global, GetIt; si no,
  provider.
- **Doble registro:** `getIt.registerSingleton` + `BlocProvider.value` de la
  misma instancia.
- **Mezclar accesos:** `context.watch<GlobalCubit>()` para un cubit que solo
  está en GetIt, o `getIt<ScreenCubit>()` para uno no global.
- **Crear cubits en `build()`** o dentro de un `BlocBuilder`.
- **Crear el cubit de pantalla dentro de la variante responsive:** al cambiar
  el ancho se recrea el estado.
- **Pasar un ReadCubit a un diálogo para refrescar:** usar el evento del
  repositorio.
- **Meter estado efímero en un cubit:** usar ViewController.
- **Olvidar `getIt.reset()` en tests:** una instancia global contamina el
  siguiente test.

## 8. Migración

Si el proyecto tiene cubits en `modules/{feature}/cubit/`:

1. Mover `read_*` y `write_*` a `lib/src/cubits/{feature}/` (plural).
2. Actualizar imports a `package:{app}/src/cubits/{feature}/...`.
3. Mantener el `MultiBlocProvider` en la screen; no moverlo a la variante.
4. Mover el estado efímero de los cubits al ViewController del módulo.
5. Revisar que la comunicación Read/Write pase por `ReactiveRepository`.
6. Si un cubit se usa en muchos módulos sin ancestro común, moverlo a
   `lib/cubits/` y registrarlo en GetIt.
7. Ejecutar `flutter analyze` y los tests con `getIt.reset()`.

Si el proyecto tiene un `GlobalCubitProvider`:

1. Crear `AppDependencies` con data sources, repositorios y globales.
2. Mover la creación de `initState` a `AppDependencies.create()`.
3. Registrar los globales en GetIt.
4. Eliminar `getIt.unregister` y `close()` del `dispose` del wrapper.
5. Eliminar los `BlocProvider.value` de esos globales.
6. Reemplazar `context.watch/read<GlobalCubit>` por
   `bloc: getIt<GlobalCubit>()`.
7. Borrar el wrapper y el `MultiBlocProvider` de globales.
8. Ejecutar `flutter analyze` y los tests con `getIt.reset()`.

## 9. Checklist

- [ ] El cubit global está en `lib/cubits/{feature}/`.
- [ ] El cubit no global está en `lib/src/cubits/{feature}/` (plural).
- [ ] Hay `read_*` y `write_*` separados.
- [ ] Los globales se crean en `AppDependencies.create()`, no en un widget.
- [ ] Los globales se consumen con `bloc: getIt<T>()` o `getIt<T>()`.
- [ ] Los no globales se proveen en la screen o ancestro común.
- [ ] La screen es el único lugar que instancia cubits.
- [ ] El estado efímero está en el ViewController, no en un cubit.
- [ ] La sincronización Read/Write usa `ReactiveRepository`.
- [ ] No hay doble registro ni mezcla de accesos.
- [ ] Los tests hacen `getIt.reset()`.

## Referencias

- [Inyección de Dependencias](di_flutter.md)
- [Estructura de Pantalla](screen_structure.md)
- [Diseño Responsive](responsive_design.md)
- [Read / Write Cubits](read_write_cubits.md)
- [ReactiveRepository](reactive_repository.md)
- [ViewController Pattern](view_controller_pattern.md)
- [Funciones de Diálogo](dialog_show_functions.md)
- [Estructura del Proyecto Flutter](../style/project_structure_flutter.md)
