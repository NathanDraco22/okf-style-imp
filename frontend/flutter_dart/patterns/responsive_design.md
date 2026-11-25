---
type: Pattern
title: Diseño Responsive (mobile / tablet / desktop)
description: Separación de la UI en variantes mobile/, tablet/ y desktop/ con breakpoints únicos en el paquete de negocio, extensiones de contexto y part files; incluye recetas de layout, tablas a cards y la regla de providers arriba del dispatch.
tags: [flutter, responsive, mobile, tablet, desktop, breakpoints, part-files, patron]
---

# Diseño Responsive

La UI se organiza en **tres variantes**: `mobile`, `tablet` y `desktop`. No es
"mobile vs web": una app web debe funcionar en un viewport de teléfono, en una
tablet y en un monitor, y una app móvil debe aprovechar la tablet.

La regla central es simple: **el layout cambia, el estado no**. Los cubits,
repositorios y controladores viven por encima del cambio de variante.

## 1. Breakpoints únicos

Los cortes viven en el paquete de negocio, no en la app:

```dart
// packages/{app}_business/lib/src/tools/constant.dart
const maxPhoneScreenWidth = 600;
const maxTabletScreenWidth = 840;
```

| Variante | Ancho | Layout típico |
|---|---|---|
| `mobile` | `< 600` | Una columna, cards, bottom sheets, CTA fijo |
| `tablet` | `600 – 839` | Grid de 2–3 columnas, rail/drawer, más densidad |
| `desktop` | `>= 840` | Sidebar fija, grid amplio, tablas, hover |

Reglas:

- **Nunca** hardcodear `600` o `840` en un widget. Siempre importar las constantes.
- Las constantes viven en el paquete de negocio para que las compartan la app,
  los tests y los widgets reutilizables.
- Si un componente necesita reaccionar a **su propio** ancho y no al de la
  ventana, usar `LayoutBuilder`, no las extensiones de contexto.

## 2. Extensiones de contexto

```dart
// lib/src/tools/extensiones.dart
import 'package:{app}_business/{app}_business.dart';

extension BuildContextExtension on BuildContext {
  bool isMobile() => MediaQuery.sizeOf(this).width < maxPhoneScreenWidth;

  bool isTablet() {
    final width = MediaQuery.sizeOf(this).width;
    return width >= maxPhoneScreenWidth && width < maxTabletScreenWidth;
  }

  bool isDesktop() =>
      MediaQuery.sizeOf(this).width >= maxTabletScreenWidth;

  bool isMobileOrTablet() =>
      MediaQuery.sizeOf(this).width < maxTabletScreenWidth;

  bool isTabletOrDesktop() =>
      MediaQuery.sizeOf(this).width >= maxPhoneScreenWidth;
}
```

- Usar `MediaQuery.sizeOf(context)`, no `MediaQuery.of(context)`: solo se
  suscribe al tamaño, no a todo el `MediaQueryData`.
- `isMobileOrTablet()` sirve para decisiones de interacción (por ejemplo,
  bottom sheet vs diálogo), no para elegir el layout completo.

## 3. Entry point y dispatch

La pantalla pública crea las dependencias y **después** delega el layout:

```dart
// view/items_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

part 'mobile/items_screen_mobile.dart';
part 'tablet/items_screen_tablet.dart';
part 'desktop/items_screen_desktop.dart';

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

class _RootScaffold extends StatelessWidget {
  const _RootScaffold();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Items')),
      body: const _Body(),
    );
  }
}

class _Body extends StatelessWidget {
  const _Body();

  @override
  Widget build(BuildContext context) {
    if (context.isMobile()) return const _ItemsMobileBody();
    if (context.isTablet()) return const _ItemsTabletBody();
    return const _ItemsDesktopBody();
  }
}
```

> **Regla crítica:** los `MultiBlocProvider` van **arriba** de `_Body`. Si se
> colocan dentro de una variante, redimensionar la ventana destruye y recrea
> los cubits, se pierde el estado y se repiten las llamadas HTTP.

Los cubits globales no se declaran aquí: viven en GetIt. Ver
[Alcance y Ciclo de Vida de Cubits](cubit_scopes.md).

## 4. Estructura de carpetas

```text
view/
├── items_screen.dart                 # Entry point + part declarations + dispatch
├── mobile/
│   └── items_screen_mobile.dart      # part of '../items_screen.dart';
├── tablet/
│   └── items_screen_tablet.dart
├── desktop/
│   ├── items_screen_desktop.dart
│   ├── sections/                     # part files de secciones desktop
│   └── widgets/                      # widgets exclusivos de desktop
└── widgets/                          # compartidos entre las tres variantes
    ├── item_card.dart
    └── item_empty_state.dart
```

- Las variantes son `part` del entry point para compartir imports y clases
  privadas. Cada archivo de variante empieza con
  `part of '../items_screen.dart';`. Las secciones de `desktop/sections/` son
  `part` del archivo desktop: `part of '../items_screen_desktop.dart';`.
- Los widgets compartidos van en `view/widgets/`; los específicos, en la
  carpeta de su variante.
- La lógica de negocio no se duplica: vive en cubits, repositorios o en el
  `view_controller` compartido. Las variantes solo renderizan y delegan.

## 5. Recetas de layout

### Desktop (`>= 840`)

Shell clásico de sidebar + contenido:

```dart
Row(
  crossAxisAlignment: CrossAxisAlignment.stretch,
  children: [
    const SizedBox(
      width: 280,
      child: DesktopSidebar(),
    ),
    Expanded(
      child: SingleChildScrollView(
        padding: const EdgeInsets.all(32),
        child: Center(
          child: ConstrainedBox(
            constraints: const BoxConstraints(maxWidth: 1100),
            child: const DesktopContent(),
          ),
        ),
      ),
    ),
  ],
)
```

- Sidebar fija de **280 px**; el contenido usa `Expanded`.
- `ConstrainedBox(maxWidth: 1100–1200)` evita líneas y grids demasiado anchos
  en monitores grandes.
- Los elementos interactivos usan `MouseRegion` + `AnimatedContainer` para
  hover; el hover **no** es la única forma de descubrir una acción.

### Tablet (`600 – 839`)

Una sola columna scrolleable con grids que se acomodan solos:

```dart
GridView.builder(
  shrinkWrap: true,
  physics: const NeverScrollableScrollPhysics(),
  itemCount: items.length,
  gridDelegate: const SliverGridDelegateWithMaxCrossAxisExtent(
    maxCrossAxisExtent: 380,
    mainAxisExtent: 104,
    crossAxisSpacing: 14,
    mainAxisSpacing: 14,
  ),
  itemBuilder: (context, index) => ItemCard(item: items[index]),
)
```

- `SliverGridDelegateWithMaxCrossAxisExtent` calcula las columnas según el
  ancho disponible; no hay que hardcodear `crossAxisCount`.
- La navegación puede ser un rail colapsado o un drawer, no una sidebar fija.

### Mobile (`< 600`)

- `ListView` vertical con cards compactas, `maxLines` y `overflow: ellipsis`.
- Filtros y selectores en `showModalBottomSheet`; acciones destructivas en
  diálogos.
- CTA principal fijo al fondo cuando el flujo lo requiera (checkout, carrito).
- Paddings horizontales de 16 px; tipografías y alturas táctiles de al menos
  48 px.

### Tablas

| Desktop | Mobile / tablet |
|---|---|
| `DataTable` o tabla custom con columnas | Lista de cards con los mismos campos |
| Acciones inline | Menú de acciones o bottom sheet |
| Hover y selección múltiple | Selección por tap largo |

Ambas variantes consumen el **mismo** cubit y la misma lista. No crear un
cubit distinto para la tabla y para las cards.

## 6. Interacción y presentación

- **Hover:** solo desktop. Usar `MouseRegion`; nunca obligar a pasar el mouse
  para completar una acción.
- **Diálogos:** en desktop `showDialog`; en mobile `showModalBottomSheet`.
  La función `showXxx` decide según `context.isMobile()` y devuelve el mismo
  tipo de resultado. Ver [Funciones de Diálogo](dialog_show_functions.md).
- **Atajos de teclado:** solo desktop/web; no bloquear el flujo táctil.
- **Scroll:** en desktop puede haber scroll anidado (sidebar + contenido); en
  mobile una sola dirección vertical.

## 7. Estado y ciclo de vida

- Providers arriba del dispatch (ver sección 3).
- No crear cubits, `TextEditingController` ni `ScrollController` dentro de una
  variante; se crean en el entry point o en el `view_controller` compartido.
- Un cambio de breakpoint no debe disparar un `getAll()` de nuevo.
- Los cubits globales de GetIt no se reinician al cambiar de layout. Ver
  [Alcance y Ciclo de Vida de Cubits](cubit_scopes.md).

## 8. Checklist

Antes de cerrar una pantalla responsive:

- [ ] Probada a 375, 600, 839, 840 y 1280 px.
- [ ] Sin overflow, texto cortado ni botones fuera de pantalla.
- [ ] Redimensionar no recarga datos ni pierde el estado.
- [ ] Las tres variantes consumen el mismo cubit/controller.
- [ ] No hay acciones que dependan solo de hover.
- [ ] Los widgets compartidos están en `view/widgets/`.
- [ ] Las constantes `600`/`840` no están hardcodeadas.
- [ ] `flutter analyze` pasa.

## Cuándo no usar tres variantes

Si la tablet no tiene diferencias reales con desktop, `tablet` puede delegar
en el layout desktop:

```dart
if (context.isMobile()) return const _ItemsMobileBody();
return const _ItemsDesktopBody();
```

Lo que **no** se hace es duplicar una variante solo por cumplir la estructura.
La decisión se documenta en el módulo.

## Referencias

- [Estructura del Proyecto Flutter](../style/project_structure_flutter.md)
- [Estructura de Pantalla](screen_structure.md)
- [Alcance y Ciclo de Vida de Cubits](cubit_scopes.md)
- [Funciones de Diálogo](dialog_show_functions.md)
- [Comandos Onion CLI](cli_commands_flutter.md)
