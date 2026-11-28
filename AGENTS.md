# AGENTS.md — OKF TEC

Este bundle sigue el formato **Open Knowledge Format (OKF) v0.2**.

## Regla #1: Usar ONION CLI

**Siempre** usar `onion` CLI para generar cualquier estructura nueva. No crear archivos manualmente.

- **Backend:** `onion crud`, `onion crud-mongo`, `onion repo`, `onion repo-mongo`, `onion router`, `onion project`
- **Frontend:** `onion dart`, `onion dart-model`, `onion dart-cubit`, `onion flutter-module`, `onion dart-view`, `onion dart-service`, `onion dart-res`, `onion barrel`

Ver [Comandos CLI Backend](backend/python_fastapi/patterns/cli_commands.md) y [Comandos CLI Flutter](frontend/flutter_dart/patterns/cli_commands_flutter.md).

> **Notas CLI (onion ≥ 0.6.0):** `onion project fastapi-init` clona el template de GitHub; en comandos Flutter (`onion dart`, `onion dart-cubit`, `onion flutter-module`) hay flag `--reactive/--no-reactive` para el ReactiveRepository.

### Verificar instalación

Antes de usar cualquier comando `onion`, verificar que está instalado ejecutando:

```bash
onion --help
```

Si el comando falla, instalar ONION CLI antes de continuar.

## Regla #2: Decisiones de Negocio en Policy Objects

**Siempre** encapsular decisiones de negocio densas o repetidas, combinaciones de banderas, cálculo de beneficios y validaciones puras en una clase **Policy** inmutable dentro de `core/policies/<dominio>/` — nunca dispersar bloques de `if` en pantallas, cubits, routers, controllers o repositories.

- **Ubicación obligatoria (frontend y backend):**
  - Backend: `core/policies/<dominio>/<nombre>_policy.py` dentro del paquete de negocio.
  - Frontend: `lib/src/core/policies/<dominio>/<nombre>_policy.dart` dentro del paquete de negocio.
  - No se crean policies fuera de `core/policies/`; cada dominio tiene su subcarpeta y las decisiones compartidas van en `decisions/`.
- **Pureza estricta:** una Policy **no hace I/O**, no consulta MongoDB, no conoce Flutter ni FastAPI y no muta estado.
- **Taxonomía de retornos:** `bool` (binario), `enum`/`Enum` (excluyente), `T?`/`T | None` (opcional), objeto compuesto inmutable (múltiples atributos) y `PolicyDecision` (validación pura con `code` y `reason`).
- **Composición:** las policies pequeñas se componen entre sí; el Cubit (frontend) o el Repository/PipelineStep (backend) obtiene los datos y delega la decisión a la Policy.

Ver [Policy Object Backend](backend/python_fastapi/patterns/policy_object.md) y [Policy Object Frontend](frontend/flutter_dart/patterns/policy_object.md).

## Workflow para el agente

### Fase 1 — Descubrimiento (al iniciar)
Leer **todos los INDEX.md** para entender la estructura completa del bundle:

| Orden | Índice | Propósito |
|---|---|---|
| 1 | [Raíz](INDEX.md) | Visión general del bundle (frontend + backend) |
| 2 | [Frontend](frontend/INDEX.md) | Tecnologías frontend |
| 3 | [Backend](backend/INDEX.md) | Tecnologías backend |
| 4 | [Flutter/Dart](frontend/flutter_dart/INDEX.md) | Técnicas y patrones Flutter |
| 5 | [Python/FastAPI](backend/python_fastapi/INDEX.md) | Técnicas y patrones FastAPI |

### Fase 2 — Ejecución (al planificar o codificar)
Según la tecnología involucrada, leer **obligatoriamente** la sección correspondiente:

#### Backend (Python/FastAPI)
1. [Estilo FastAPI](backend/python_fastapi/style/INDEX.md) — convenciones de código
2. [Patrones FastAPI](backend/python_fastapi/patterns/INDEX.md) — patrones de diseño
3. [Policy Object Backend](backend/python_fastapi/patterns/policy_object.md) — decisiones de negocio y validaciones puras
4. [CLI Backend](backend/python_fastapi/patterns/cli_commands.md) — comandos `onion` para backend

#### Frontend (Flutter/Dart)
1. [Estilo Flutter](frontend/flutter_dart/style/INDEX.md) — convenciones de código
2. [Patrones Flutter](frontend/flutter_dart/patterns/INDEX.md) — patrones de diseño
3. [Policy Object Frontend](frontend/flutter_dart/patterns/policy_object.md) — decisiones de negocio y validaciones puras
4. [CLI Flutter](frontend/flutter_dart/patterns/cli_commands_flutter.md) — comandos `onion` para frontend

## Convenciones del bundle

- Todos los archivos `INDEX.md` están en **mayúscula**.
- Cada concepto tiene **frontmatter YAML** con `type`, `title`, `description`, `tags`.
- Los tags permiten búsqueda transversal entre frontend y backend.
