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

## Clases

| Elemento | Convención | Ejemplo |
|----------|-----------|---------|
| Screen widget | `{Feature}Screen` | `PatientsScreen` |
| ViewController | `{Feature}ViewController` | `ClientInvoiceViewController` |
| Mediator | `{Feature}Mediator` | `ClientInvoiceWebMediator` |
| Read Cubit | `Read{Entity}Cubit` | `ReadPatientCubit` |
| Read State | `Read{Entity}State` | `ReadPatientState` |
| Write Cubit | `Write{Entity}Cubit` | `WritePatientCubit` |
| Write State | `Write{Entity}State` | `WritePatientState` |
| Search Cubit | `Search{Entity}Cubit` | `SearchPatientCubit` |
| Search State | `Search{Entity}State` | `SearchPatientState` |
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
| `cubits/{entity}/` | Read + Write cubits de una entidad |
| `modules/{feature}/view/` | Screens de la feature |
| `modules/{feature}/view/mobile/` | Versión mobile |
| `modules/{feature}/view/web/` | Versión web |
| `modules/{feature}/view/web/sections/` | Secciones part de web |
| `modules/{feature}/widgets/` | Widgets del módulo |
| `modules/{feature}/cubit/` | Cubits del módulo |
