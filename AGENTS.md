# AGENTS.md — OKF TEC

Este bundle sigue el formato **Open Knowledge Format (OKF) v0.2**.

## Regla #1: Usar ONION CLI

**Siempre** usar `onion` CLI para generar cualquier estructura nueva. No crear archivos manualmente.

- **Backend:** `onion crud`, `onion crud-mongo`, `onion repo`, `onion repo-mongo`, `onion router`, `onion project`
- **Frontend:** `onion dart`, `onion dart-model`, `onion dart-cubit`, `onion flutter-module`, `onion dart-view`, `onion dart-service`, `onion dart-res`, `onion barrel`

Ver [Comandos CLI Backend](backend/python_fastapi/patterns/cli_commands.md) y [Comandos CLI Flutter](frontend/flutter_dart/patterns/cli_commands_flutter.md).

## Regla #2: Leer convenciones antes de trabajar

Antes de crear o modificar código en una tecnología, leer **obligatoriamente** sus guías de estilo y patrones:

### Backend (Python/FastAPI)
1. [Estilo FastAPI](backend/python_fastapi/style/INDEX.md) — convenciones de código
2. [Patrones FastAPI](backend/python_fastapi/patterns/INDEX.md) — patrones de diseño

### Frontend (Flutter/Dart)
1. [Estilo Flutter](frontend/flutter_dart/style/INDEX.md) — convenciones de código
2. [Patrones Flutter](frontend/flutter_dart/patterns/INDEX.md) — patrones de diseño

## Cómo usar este bundle

1. **Lee INDEX.md** de cada sección para navegar la estructura.
2. Los conceptos están organizados en `frontend/` y `backend/`, cada uno con `style/` (guías de estilo) y `patterns/` (patrones de diseño).

## Índices principales (obligatorio leer)

- [Raíz](INDEX.md) — Visión general del bundle.
- [Frontend](frontend/INDEX.md) — Tecnologías frontend.
- [Flutter / Dart](frontend/flutter_dart/INDEX.md) — Técnicas y patrones Flutter.
- [Estilo Flutter](frontend/flutter_dart/style/INDEX.md) — Guías de estilo Flutter.
- [Patrones Flutter](frontend/flutter_dart/patterns/INDEX.md) — Patrones de diseño Flutter.
- [Backend](backend/INDEX.md) — Tecnologías backend.
- [Python / FastAPI](backend/python_fastapi/INDEX.md) — Técnicas y patrones FastAPI.
- [Estilo FastAPI](backend/python_fastapi/style/INDEX.md) — Guías de estilo FastAPI.
- [Patrones FastAPI](backend/python_fastapi/patterns/INDEX.md) — Patrones de diseño FastAPI.

## Convenciones del bundle

- Todos los archivos `INDEX.md` están en **mayúscula**.
- Cada concepto tiene **frontmatter YAML** con `type`, `title`, `description`, `tags`.
- Los tags permiten búsqueda transversal entre frontend y backend.
