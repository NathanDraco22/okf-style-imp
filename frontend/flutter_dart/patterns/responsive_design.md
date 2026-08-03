---
type: Pattern
title: Diseño Responsive (mobile / web)
description: Separación de UI en subdirectorios mobile/ y web/ con detección por context.isMobile() y uso de part files para secciones.
tags: [flutter, responsive, mobile, web, patron]
---

# Diseño Responsive

Las pantallas que necesitan adaptación mobile/web se organizan con directorios separados y secciones compartidas mediante `part files`.

## Estructura de directorios

```
client_invoices/
├── client_invoices_screen.dart          # Entry point, detecta plataforma
├── client_invoices_mediator.dart        # InheritedWidget mudo compartido web/mobile
├── view_controller.dart                 # ChangeNotifier compartido
├── cubit/                               # Solo si el feature lo amerita
│   ├── write_invoice_cubit.dart
│   └── write_invoice_state.dart
├── mobile/
│   ├── client_invoices_screen_mobile.dart
│   └── widgets/
│       ├── product_selection_card.dart
│       └── invoice_summary_card.dart
├── web/
│   ├── client_invoices_screen_web.dart  # part + sections
│   ├── sections/
│   │   ├── header_section.dart          # part of screen_web.dart
│   │   └── content_section.dart         # part of screen_web.dart
│   └── widgets/
│       ├── invoice_table.dart
│       └── totals_panel.dart
└── widgets/
    ├── client_selector.dart             # Compartido mobile/web
    └── product_search_dialog.dart
```

## Detección de plataforma

```dart
// extensiones.dart
extension BuildContextExtensions on BuildContext {
  bool isMobile() => MediaQuery.sizeOf(this).width < maxPhoneScreenWidth; // 600px
}
```

## Entry point

```dart
// client_invoices_screen.dart
class ClientInvoicesScreen extends StatelessWidget {
  const ClientInvoicesScreen({super.key});

  @override
  Widget build(BuildContext context) {
    final isMobile = context.isMobile();
    return MultiBlocProvider(
      providers: [
        BlocProvider(create: (_) => WriteItemCubit(repo: context.read<ItemRepository>())),
        BlocProvider(create: (_) => ReadItemCubit(repo: context.read<ItemRepository>())),
      ],
      child: isMobile
          ? const _MobileScaffold()
          : const _WebScaffold(),
    );
  }
}
```

## Part files para secciones web

```dart
// web/client_invoices_screen_web.dart
import 'package:flutter_web_plugins/flutter_web_plugins.dart';  // o imports web-only
part 'sections/header_section.dart';
part 'sections/content_section.dart';

class _WebScaffold extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Column(
        children: [
          _HeaderSection(),
          Expanded(child: _ContentSection()),
        ],
      ),
    );
  }
}
```

## Reglas

| Elemento | Mobile (< 600px) | Web (>= 600px) |
|----------|-----------------|----------------|
| Layout | Vertical scroll, cards | Tablas, paneles laterales |
| Inputs | Dialogs modales | Inline sections |
| Navegación | Bottom nav o drawer | Sidebar |
| Directorio | `mobile/` | `web/` |

- Los `part files` permiten que las secciones web accedan a clases privadas del mismo archivo
- Los widgets compartidos van en `widgets/` raíz del módulo
- El `mediator` es **compartido en la raíz del módulo** (`{feature}_mediator.dart`); solo se duplica por plataforma cuando una necesita exponer más datos que el ViewController (ver [Mediator Pattern](mediator_pattern.md))
- El `view_controller.dart` (ChangeNotifier) se comparte entre ambas plataformas
