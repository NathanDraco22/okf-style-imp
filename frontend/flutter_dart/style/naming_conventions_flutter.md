---
type: StyleGuide
title: Convenciones de Nomenclatura (Flutter/Dart)
description: Reglas de nomenclatura para archivos, clases, widgets, cubits, modelos y repositorios en proyectos Flutter Onion.
tags: [flutter, dart, estilo, naming, convenciones]
---

# Convenciones de Nomenclatura (Flutter/Dart)

## Archivos

| Elemento | Convención | Ejemplo |
|----------|-----------|---------|
| Screen | `{feature}_screen.dart` | `patient_screen.dart` |
| Cubit | `{read/write}_{entity}_cubit.dart` | `read_patients_cubit.dart` |
| State | `{read/write}_{entity}_state.dart` | `read_patients_state.dart` |
| Model | `{entity}_model.dart` | `patient_model.dart` |
| DataSource | `{entity}_data_source.dart` | `patients_data_source.dart` |
| Repository | `{entity}_repository.dart` | `patients_repository.dart` |
| Response | `list_response.dart` | Genérico |
| Barrel | `{name}.dart` | `data_sources.dart`, `repositories.dart` |
| Widget | `{widget_name}.dart` | `client_selector.dart` |
| Dialog | `{entity}_form_dialog.dart` | `patient_form_dialog.dart` |
| Viewer | `viewers/{entity}_viewer.dart` | `invoice_viewer.dart` |
| Editor | `editors/{entity}_editor_dialog.dart` | `unit_editor_dialog.dart` |
| Inspector | `inspectors/{entity}_inspector.dart` | `invoice_inspector.dart` |
| Bottom modal | `modals/{action}_bottom_modal.dart` | `confirm_send_bottom_modal.dart` |
| Result model | `models/{entity}_result.dart` | `unit_result.dart` |

## Funciones show

| Elemento | Convención | Ejemplo |
|----------|-----------|---------|
| Mostrar viewer | `show{Entity}Viewer(context, entity)` | `showInvoiceViewer(context, invoice)` |
| Mostrar inspector | `show{Entity}Inspector(context, entity)` | `showInvoiceInspector(context, invoice)` |
| Editar / crear | `show{Entity}Editor(context, {initial...})` | `showUnitEditor(context, initialUnits: units)` |
| Buscar / seleccionar | `show{Entity}Search(context, {...})` | `showProductSearch(context, restrictStock: true)` |
| Bottom modal | `show{Action}BottomModal(context, {...})` | `showConfirmSendBottomModal(context, title: 'Enviar')` |

Todas devuelven `Future<T?>` (ver [Patrón Dialog/Modal](dialog_show_functions.md)).

## Clases

| Elemento | Convención | Ejemplo |
|----------|-----------|---------|
| Screen widget | `{Feature}Screen` | `PatientsScreen` |
| ViewController | `{Feature}ViewController` | `ClientInvoiceViewController` |
| Mediator | `{Feature}Mediator` | `ClientInvoiceMediator` |
| Read Cubit | `Read{Entity}Cubit` | `ReadPatientCubit` |
| Read State | `Read{Entity}State` | `ReadPatientState` |
| Write Cubit | `Write{Entity}Cubit` | `WritePatientCubit` |
| Write State | `Write{Entity}State` | `WritePatientState` |
| DataSource | `{Entity}DataSource` | `PatientsDataSource` |
| Repository | `{Entity}Repository` | `PatientsRepository` |
| Create model | `Create{Entity}` | `CreatePatient` |
| Update model | `Update{Entity}` | `UpdatePatient` |
| InDb model | `{Entity}InDb` | `PatientInDb` |
| Filter Params | `{Entity}FilterParams` | `TransactionFilterParams` |
| Auth State | `Authenticated`, `Unauthenticated` | (no prefijo) |

## Widgets internos

```dart
// Widgets privados dentro de una screen llevan prefijo _
class _RootScaffold extends StatelessWidget { ... }
class _Body extends StatelessWidget { ... }
class _PatientsList extends StatelessWidget { ... }

// Widgets reutilizables van en widgets/ y no llevan _
class ClientSelectorCard extends StatelessWidget { ... }
```

## Directorios

| Carpeta | Contenido |
|---------|-----------|
| `lib/cubits/{entity}/` | Cubits **globales** de la app (auth, sesión, modo) |
| `lib/src/cubits/{entity}/` | Cubits **compartidos** entre módulos (entidades multi-feature) |
| `modules/{feature}/{feature}_screen.dart` | Entry point del módulo (crea el ViewController, MultiBlocProvider) |
| `modules/{feature}/{feature}_mediator.dart` | InheritedWidget mudo compartido web/mobile |
| `modules/{feature}/view_controller.dart` | ChangeNotifier (estado local de la pantalla) |
| `modules/{feature}/cubit/` | Cubits del módulo — **solo cuando lo amerita** |
| `modules/{feature}/mobile/` | Versión mobile |
| `modules/{feature}/web/` | Versión web |
| `modules/{feature}/web/sections/` | Secciones part de web |
| `modules/{feature}/widgets/` | Widgets compartidos del módulo |
