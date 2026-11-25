# Bitácora de cambios

## 2026-11-25 — Patrones de Frontend: Responsive y Alcance de Cubits

- **Reescrito `responsive_design.md`:**
  - Tres variantes `mobile/`, `tablet/` y `desktop/` con cortes `600`/`840` en el paquete de negocio.
  - Extensiones `isMobile/isTablet/isDesktop` con `MediaQuery.sizeOf`.
  - Entry point con `part` files, dispatch en `_Body` y providers arriba del dispatch.
  - Recetas: sidebar 280 + `maxWidth` 1100, grid con `maxCrossAxisExtent`, cards mobile, tablas a cards y hover.
  - Checklist de 375/600/839/840/1280 y regla de no recargar al redimensionar.
- **Nuevo `cubit_scopes.md`:**
  - Globales en `lib/cubits/` → GetIt desde `AppDependencies.create()`; sin `GlobalCubitProvider`, sin doble registro.
  - Compartidos en `lib/src/cubits/` y de pantalla en `modules/{feature}/cubit/` con providers.
  - Diálogos con `BlocProvider.value` o creación en `showXxx`.
  - Consumo global con `bloc: getIt<T>()`; `getIt.reset()` en tests.
  - Anti-patrones y guía de migración desde el wrapper.
- **Actualizados:** `di_flutter.md` (tabla de alcances y composition root), `project_structure_flutter.md` (`app_dependencies`, tablet/desktop, constantes), `screen_structure.md` (providers y responsive), `cli_commands_flutter.md` (split manual y limitaciones), `naming_conventions_flutter.md`, índices y README (25 → 26 patrones).
- **Implementación de referencia:** `kardex_app_front/lib/src/modules/home/view` para el responsive; `lib/cubits/` y `provider_container.dart` para los alcances. La convención nueva elimina `GlobalCubitProvider` y unifica los globales en GetIt.

## 2026-11-25 — Patrón Counter Manager (correlativos atómicos)

- **Nuevo patrón (`counter_manager.md`):**
  - Documenta la generación de correlativos secuenciales por tipo de documento (`invoiceNumber`, `orderNumber`, etc.) con `find_one_and_update` + `$inc` + `upsert` + `ReturnDocument.AFTER`.
  - Cadena completa sin Router/Controller: `CounterManager → CountersRepository → CountersDataSource → CountersCollection → MongoDB`.
  - Variantes de scope: global, por tenant y por tenant + periodo; el reinicio periódico es implícito al cambiar la clave.
  - `session` opcional: operación atómica de un solo documento sin transacción, o incremento transaccional junto con la inserción del documento.
  - Concurrencia: índice único obligatorio, retry de `DuplicateKeyError` en el upsert y alternativa de pre-crear contadores.
  - Formato centralizado en el manager (prefijo + padding), `set_counter` para importaciones y decisión consciente sobre gaps.
- **Implementación de referencia:** `kardex-server/.../core/counter_manager/`; el documento corrige los puntos a adaptar (índice no único, scope multi-tenant, sesión requerida, excepción de migración, singleton con `__new__` y formato sin prefijo).
- **Índices actualizados:** enlace agregado en `backend/python_fastapi/patterns/INDEX.md`, árbol y conteo del `README.md` (18 → 19 patrones) y bullet en Convenciones Críticas de Persistencia.

## 2026-11-22 — Gate Único con `GateFilter` Componibles

- **Nuevo patrón (`gate_filter_pattern.md`):**
  - Documenta la estrategia de consolidar controles de visibilidad y permisos en un único widget `Gate` (`StatefulWidget`) que recibe `List<GateFilter>` (análogo a `inputFormatters` en `TextField`), evitando árboles anidados y banderas booleanas dispersas.
  - **Contrato `GateFilter`:** `isAllowed(context)`, `watchStreams(context)` (con deduplicación de streams en `Gate`) y `buildFallback(context)` para fallbacks diferenciados por motivo de bloqueo.
  - **Combinadores y modos:** `GateMatchMode.all` / `any`, `AnyGateFilter`, `NotGateFilter` y `PredicateGateFilter`.
  - **Unificación UI + lógica síncrona:** reemplaza flags booleanos en descriptores de menú (`NavSubmoduleItem.gates`) para alimentar tanto a `Gate` como a resolvers síncronos de rutas/atajos.
  - Incluye compatibilidad con `Expanded`/`Flexible`, reglas de implementación, testing con `pumpAndSettle` y checklist de adopción.
- **Índices actualizados:** enlace agregado en `frontend/flutter_dart/patterns/INDEX.md`; `README.md` actualizado a 25 patrones con sección resumen y árbol ampliado.

## 2026-11-13 — Fechas previas a 1970 (epoch negativo)

- **Bug reportado:** al crear un paciente de 57 años (nacido en 1969), la API rechazó el registro porque `birthDate` llegaba como `-20_131_200_000`.
- **Causa raíz:** epoch ms es un entero con signo; la fecha se validó con una regla de auditoría (`gt=0`) que asume que todo timestamp es positivo. El resultado fue un `422 Input should be greater than 0`.
- **Fix documentado:** `timestamps.md` ahora distingue timestamps de auditoría de fechas de dominio y valida estas últimas por rango plausible (`[-2_208_988_800_000, now]`), sin `abs()` ni casts unsigned.
- **Docs actualizados:** `backend/python_fastapi/patterns/timestamps.md` (sección nueva + checklist), `frontend/flutter_dart/patterns/query_params.md` (filtros negativos), `frontend/flutter_dart/patterns/flutter_models.md` (date pickers y `fromMillisecondsSinceEpoch`).

## 2026-11-10 — Pipeline-Steps para Procesos Complejos

- **Nuevo patrón (`pipeline_steps.md`)**: documenta la orquestación de procesos multi-paso con un contexto compartido, pasos independientes y un resultado final tipado.
- **Contratos base:** `PipelineStep` (`name`, `should_execute`, `async execute`), `Pipeline` (`run` secuencial) y `PipelineResult` (`context`, `executed_steps`, `skipped_steps`).
- **Separación de responsabilidades:** el orquestador conserva la transacción, el mutex y el retry; los steps no abren transacciones, no hacen commit/abort y reciben sus dependencias por constructor.
- **Contexto:** solo datos (entradas, estado intermedio y resultado); las colecciones/repositorios no se guardan en el contexto.
- **Ramas opcionales:** `should_execute` permite omitir pasos sin llenar el pipeline de condicionales.
- **Ejemplo genérico:** `OrderApproval` con validación, carga de cliente, registro de pago, persistencia de factura, aprobación y construcción del resultado.
- **Implementación de referencia:** `kardex-server/.../core/adjustment_requests/` y `test/test_adjustment_request_pipeline.py`; el documento mantiene el ejemplo agnóstico al dominio.
- **Índices actualizados:** `backend/python_fastapi/patterns/INDEX.md`, árbol y conteo del `README.md` (17 → 18 patrones), y cross-link desde `mongo_transactions.md`.

## 2026-11-08 — Formatter Global con Resolver Inyectado

- **Nuevo patrón (`global_formatter_resolver.md`)**:
  - Documenta la estrategia de un formatter puro que lee configuración global desde un Cubit mediante un callback registrado una sola vez.
  - Incluye la clase `NumberFormatter` con `CurrencyResolver`, el enum `AppCurrency` y el registro/limpieza del resolver en `AppSettingsCubit`.
  - Explica por qué no se necesita un setter al cargar o guardar: el formatter consulta el estado vivo del Cubit en cada formateo.
  - Cubre ciclo de vida, fallback, testing, isolates, múltiples Cubits y el checklist para reutilizarlo en otros proyectos.
  - Aplicable a moneda, locale, formatos de fecha, sistema de unidades o cualquier configuración global de formato.
- **Índices actualizados**: enlace agregado en `frontend/flutter_dart/patterns/INDEX.md`; README actualizado a 24 patrones y árbol de archivos ampliado.

## 2026-11-03 — Redacción Integral de README.md y Patrones OKF (GoRouter + Cross-Cubit Sync)

- **README.md General**:
  - Redacción completa del documento institucional del repositorio en formato UTF-8 markdown.
  - Explica la filosofía del estándar OKF v0.2, el agnosticismo de dominio, la prioridad CLI y el control de ciclo de vida de memoria y streams.
  - Detalla la arquitectura Frontend (Flujo Onion, Read/Write Cubits, ReactiveRepository, Cross-Cubit Event Sync, GoRouter reactivo, Show functions, ViewController) y Backend (Flujo Onion de 5 capas, soft delete, timestamps Unix ms, transacciones MongoDB).
  - Incluye guía de navegación para desarrolladores y agentes de IA, y catálogo de comandos Onion CLI.
- **GoRouter Reactivo (`go_router_pattern.md`)**:
  - Incorporada máquina de estados de sesión multi-fase (`Bootstrapping/Splash` $\to$ `Onboarding` $\to$ `Inactive/Suspended` $\to$ `Login` $\to$ `Dashboard`).
  - Patrón de preservación pasiva de ruta (`_lastKnownLocation`) para evitar reinicios forzados al Home ante mutaciones de contexto global (cambio de organización/sucursal, refresco de token en segundo plano).
  - Subrutas operativas a pantalla completa (`parentNavigatorKey: rootNavigatorKey`) que ocultan el menú Shell conservando URLs navegables e historial nativo.
  - Gestión y desecho estricto de suscripciones en `GoRouterRefreshStream`.
- **Cross-Cubit Event Sync (`cross_cubit_event_sync.md`)**:
  - Nuevo patrón de arquitectura para sincronización entre Cubits independientes.
  - Sustituye los anti-patrones de acoplamiento directo (Cubit-to-Cubit) y coordinación manual frágil en UI por un **Event Broker tipado** mediante `ReactiveRepository.eventStream`.
  - Control de ciclo de vida con guarda `isClosed` y cancelación obligatoria de `StreamSubscription` en el método `close()` del Cubit.
  - Documento registrado e indexado en `frontend/flutter_dart/patterns/INDEX.md`.

## 2026-08-02 — Cierre: CLI v0.6.0 commiteado y OKF al día

- El repo `onion-cli` quedó en **v0.6.0** con todo commiteado (reactive repository, fastapi-init con clon del template, fix circular import en `base_mongo_collection_template.py`).
- El tool global reinstalado (`uv tool install --force --no-cache`) → `onion v0.6.0` en PATH.
- Docs del bundle actualizados con nota de **versión mínima onion ≥ 0.6.0** en `cli_commands.md` y `cli_commands_flutter.md`.
- Cobertura OKF confirmada: reactive (frontend), fastapi-init clone + pymongo async (backend), fix circular import (log), estado del template DB-agnóstico (cli_commands).

## 2026-08-02 — `fastapi-init` clona el template de GitHub + pymongo async

- **CLI**: `onion project fastapi-init` ahora **clona https://github.com/NathanDraco22/fastapi-onion-template** en vez de copiar el `project_base` local (que estaba incompleto: solo `app/` + `test/`). El clon trae todo (pyproject.toml, Dockerfile, uv.lock, .vscode). Tras clonar: borra `.git` (proyecto limpio), renombra `examples` → nombre del proyecto, y renombra `name`/`description` en pyproject.toml. Si git/red falla → error claro (sin fallback). `fastapi-app` se mantiene igual.
- **Fix Windows**: `shutil.rmtree` falla al borrar `.git` clonado (archivos pack read-only) → handler `force_remove_tree` con `os.chmod(S_IWRITE)`.
- **pymongo async (motor deprecado)**: el CLI ya generaba `AsyncMongoClient`/`AsyncCollection` (pymongo ≥ 4.17 async nativo). El experimento usó motor por tener el CLI viejo. Corregido bug de compilación en `base_mongo_collection_template.py`: `from services import MongoService` → `from .mongo_service import MongoService` (circular import que rompía todo proyecto generado).
- **Decisión de diseño**: el template de GitHub **no incluye pymongo a propósito** — es agnóstico de base de datos. El flujo documentado hace `uv add pymongo` solo cuando se usa Mongo.
- **Docs OKF**: `cli_commands.md` (flujo canónico con clone + nota DB agnóstica), `singleton.md` (AsyncIOMotorClient → AsyncMongoClient), `onion_flow.md` (motor → pymongo async, `fastapi-init` clona el template).
- **Verificado end-to-end**: clon → uv sync → uv add pymongo → crud-mongo → ruff limpio → pytest CRUD contra MongoDB (Docker) pasa.

## 2026-08-02 — ReactiveRepository en Onion CLI

- **CLI (repo local `C:\Users\sytru\Developer\Python\onion-cli`)**: `onion dart`, `onion dart-cubit` y `onion flutter-module` ahora preguntan **"¿Generar con ReactiveRepository? [y/N]"** (default **No**) con flag `--reactive/--no-reactive` para saltarla en CI.
- **Nuevo template** `templates/dart/reactive_repository_template.py`: mixin `ReactiveRepository<T>` + eventos `RepoEvent` sellados, generado en `lib/src/tools/reactive_repo/reactive_repository.dart` (idempotente, busca el `pubspec.yaml` para ubicar la raíz del paquete).
- **`repository_template.py`**: variante reactive con `with ReactiveRepository<XInDb>`, sin cache, `notifyXxx` post-operación.
- **`read_cubit_template.py` / `write_cubit_template.py`**: variante reactive con repository inyectado, suscripción `eventStream` + `cancel()` en `close()`, write con métodos reales y reset a `Initial`. Estados ahora usan `{Entity}InDb`.
- **Bugs de compilación corregidos de paso en templates**: imports rotos (`package:{entity}_model.dart`), falta de imports de modelo/datasource/ListResponse/HttpService en repo y datasource, y `a.name` hardcodeado en el sort/searchLocal del modo clásico (eliminado).
- **Docs OKF actualizados**: `cli_commands_flutter.md` (sección Reactive Repository), `reactive_repository.md` (generación CLI + tabla decisión reactive vs BlocListener + variante fallback), `read_write_cubits.md` (nota de generación).
- **Verificado** en proyecto scratch: `--reactive` y `--no-reactive` compilan (flutter analyze sin errores), prompt respeta default No y Sí, mixin se genera una sola vez en la raíz.

## 2026-08-02 — Experimento "Biblioteca" (full-stack con Onion CLI)

App demo en `experiment_app/`: FastAPI + MongoDB (Docker) en `server/`, Flutter web en `frontend/`. Entidades: autores y libros con relación, búsqueda con acentos, soft delete, timestamps Unix ms.

### Bugs del CLI (versión instalada y git f2d19bc)
- **`onion project fastapi-init`/`fastapi-app` roto**: `base_path = Path("onion/project_base/fast_api_app")` es ruta relativa al CWD. Arreglado resolviendo desde `Path(__file__)`. Afecta a `copy_fastapi_full_project.py` y `copy_fastapi_project.py` del paquete instalado.
- **Circular import en `services/__init__.py`**: importa `base_mongo_collection` antes que `mongo_service`, y la clase base hace `from services import MongoService`. Fix raíz: import relativo `from .mongo_service import MongoService` en la clase base (el orden alfabético de ruff rompe el fix por orden).
- **`app/tools/__init__.py` roto**: usa `from time_tools import TimeTools` (absoluto). Fix: import relativo.
- **`fastapi-init` incompleto**: la plantilla instalada no incluye `pyproject.toml`, `Dockerfile`, `uv.lock`, `.env.example` (solo `app/` y `test/`). Hubo que copiarlos del repo canónico `onion_template`. El `test_demo.py` generado referencia un módulo inexistente.
- **Dependencias**: `onion` solo existe en git — requiere `[tool.uv.sources]` con el repo.

### Divergencias CLI vs patrones documentados (a pulir)
- El CLI genera `created_at`/`updated_at` como `str` (snake_case); los docs exigen `createdAt`/`updatedAt` como `int` (Unix ms), `isDeleted` y soft delete. El CLI genera **hard delete** (`find_one_and_delete`) y sin filtros `isDeleted` en lecturas. Proyectos reales (farmacia) también hacen hard delete — los docs van adelante del CLI.
- `UuidTool` y `SearchTools` están en el árbol de estructura documentado pero el CLI no los genera.
- Dart: `ListResponse` generado espera `total/page/page_size`; el contrato backend/bundle es `count/data`. El DataSource de búsqueda genera `/search/{keyword}` en vez de `?search=` (query param).
- `dart-cubit`/`flutter-module` generan imports inválidos (`package:author_model.dart`, tipos `Author`/`Book` inexistentes) — requieren trabajo manual completo.
- `dart-service`/`dart-res` escriben en el CWD en vez de `lib/src/`.

### Patrones validados en la práctica
- Flujo Onion completo backend: Router → Controller → Repository → DataSource → Collection → MongoDB, con DI por constructor + `get_instance()` singleton.
- Patrón de modelos Base/Create/Update/InDb con camelCase, timestamps e `isDeleted`; `model_dump(exclude_unset=True)` para PATCH.
- `searchPrefixes` + índice `$text` (spanish, pesos name>prefixes) + aggregation con textScore: búsqueda `marquez` → "Gabriel García Márquez" (con acentos).
- Soft delete con 404 tras borrar; ListResponse `count/data` en ambos lados.
- Frontend: Read/Write cubits con estados sellados (Initial → Loading/Refreshing → Success/Error), go_router StatefulShellRoute con 2 ramas, show functions para diálogos, MultiRepositoryProvider + CORS para web.
- Tests: `pytest` con `asyncio_mode=auto` y loop de sesión (singletons de Motor requieren un solo loop); `flutter analyze` limpio + widget test.

## 2026-07-26
- **Contenido Flutter**: Poblado completo de patrones Flutter/Dart (18 conceptos) y estilo (2 conceptos).
- **Contenido Backend**: Poblado completo de patrones backend (17 conceptos) y estilo (2 conceptos) para Python/FastAPI.
- **Reestructuración**: Separación en `frontend/` y `backend/` con subcarpetas por tecnología (`flutter_dart`, `python_fastapi`), cada una con `style/` y `patterns/`.
- **Inicialización**: Estructura base del bundle OKF con categorías `style/` y `patterns/`.
