# AGENTS.md — OKF TEC

Este bundle sigue el formato **Open Knowledge Format (OKF) v0.2**.

## Regla #1: Usar ONION CLI

**Siempre** usar `onion` CLI para generar cualquier estructura nueva. No crear archivos manualmente.

- **Backend:** `onion crud`, `onion crud-mongo`, `onion repo`, `onion repo-mongo`, `onion router`, `onion project`
- **Frontend:** `onion dart`, `onion dart-model`, `onion dart-cubit`, `onion flutter-module`, `onion dart-view`, `onion dart-service`, `onion dart-res`, `onion barrel`

Ver [Comandos CLI Backend](backend/python_fastapi/patterns/cli_commands.md) y [Comandos CLI Flutter](frontend/flutter_dart/patterns/cli_commands_flutter.md).

### Verificar instalación

Antes de usar cualquier comando `onion`, verificar que está instalado ejecutando:

```bash
onion --help
```

Si el comando falla, instalar ONION CLI antes de continuar.

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
3. [CLI Backend](backend/python_fastapi/patterns/cli_commands.md) — comandos `onion` para backend

#### Frontend (Flutter/Dart)
1. [Estilo Flutter](frontend/flutter_dart/style/INDEX.md) — convenciones de código
2. [Patrones Flutter](frontend/flutter_dart/patterns/INDEX.md) — patrones de diseño
3. [CLI Flutter](frontend/flutter_dart/patterns/cli_commands_flutter.md) — comandos `onion` para frontend

## Convenciones del bundle

- Todos los archivos `INDEX.md` están en **mayúscula**.
- Cada concepto tiene **frontmatter YAML** con `type`, `title`, `description`, `tags`.
- Los tags permiten búsqueda transversal entre frontend y backend.
