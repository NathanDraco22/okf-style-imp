# Patrones — Flutter / Dart

Patrones de diseño y técnicas usados en Flutter y Dart.

> **Generado por Onion CLI:** El CLI `onion` genera el scaffolding completo de cada patrón (archivos, clases, CRUD básico, barrel files). El desarrollador completa la lógica de negocio específica (modelos con campos reales, reglas de negocio en repositorios, endpoints adicionales). Ver [cli_commands_flutter.md](cli_commands_flutter.md) para la referencia de comandos.
>
> **Regla #1 — Prioridad CLI:** toda estructura nueva se genera con Onion CLI (`onion ...`), **nunca a mano**. El CLI ya hace buen trabajo: estructura, clases y barrels consistentes. Crear archivos manualmente rompe el patrón.

## Flujo y arquitectura

- [Flujo Onion en Flutter](flutter_onion_flow.md) — Screen → Cubit → Repository → DataSource → API
- [Estructura de Pantalla](screen_structure.md) — MultiBlocProvider → BlocListener → BlocBuilder
- [Segmentación de Vistas UI](view_segmentation.md) — Orquestador, unidades, contexto, fuente de verdad única
- [ViewController Pattern](view_controller_pattern.md) — Estado en memoria de la pantalla: acciones, mapeo a DTOs, ciclo de vida

## Modelos y datos

- [Patrón de Modelos Flutter](flutter_models.md) — Create/Update/InDb con fromJson/toJson
- [ListResponse Flutter](list_response_flutter.md) — Response genérico de listas con count
- [Query Params Tipados](query_params.md) — Filtros con objetos toMap()

## State Management (Cubits)

- [Read / Write Cubits](read_write_cubits.md) — Separación estricta de lectura y escritura
- [Estados de Lectura](cubit_states_read.md) — Initial → Loading → Success → Refreshing → Error
- [Estados de Escritura](cubit_states_write.md) — Initial → Writing → Created/Updated/Deleted → Error
- [Paginación Infinita y Búsqueda](pagination_infinite_scroll.md) — Scroll infinito + Searching que conserva datos

## Capa de datos

- [DataSource Pattern](data_source_pattern.md) — Singleton + HttpService mixin
- [HttpService Mixin](http_service_mixin.md) — HTTP methods + status code processing
- [ReactiveRepository](reactive_repository.md) — StreamController broadcast para reactividad

## Infraestructura

- [Inyección de Dependencias](di_flutter.md) — RepositoryProvider + BlocProvider + GetIt
- [Mediator Pattern](mediator_pattern.md) — InheritedWidget + ChangeNotifier para forms
- [Manejo de Excepciones](exception_handling_flutter.md) — Jerarquía sellada de errores HTTP
- [GoRouter](go_router_pattern.md) — StatefulShellRoute + auth redirect
- [Funciones de Flujo](action_flow_functions.md) — Acciones complejas como funciones top-level
- [Patrón Dialog/Modal](dialog_show_functions.md) — Show functions fuera del router: viewers, editors, searchers, bottom modals

## CLI

- [Comandos Onion CLI para Flutter/Dart](cli_commands_flutter.md) — Referencia de comandos onion para Flutter

## Responsive

- [Diseño Responsive](responsive_design.md) — mobile/ vs web/ con part files
