# Bitácora de cambios

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
