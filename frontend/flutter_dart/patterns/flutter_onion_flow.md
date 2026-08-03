---
type: Pattern
title: Flujo Onion en Flutter
description: Arquitectura en capas Screen → Cubit → Repository → DataSource → API, con inyección de dependencias vertical.
tags: [flutter, dart, onion, arquitectura, flujo]
---

# Flujo Onion en Flutter

> **CLI:** El comando `onion dart {entity}` genera el scaffolding completo (DataSource, Repository, Models) y barrel files. El comando `onion flutter-module {entity}` agrega la capa de presentación (cubits, screen, widgets). Los comandos `onion dart-cubit`, `onion dart-model`, `onion dart-view` generan piezas individuales.

La arquitectura Onion organiza el código en capas con dependencia unidireccional hacia adentro:

```
Screen (UI)
  → Cubit (Estado)
    → Repository (Lógica de negocio)
      → DataSource (HttpService mixin)
        → API (HTTP)
```

## Capas

| Capa | Ubicación | Responsabilidad |
|------|-----------|-----------------|
| **Screen** | `lib/src/modules/{feature}/` | Widgets UI, BlocBuilder, BlocListener |
| **Cubit** | `lib/cubits/{entity}/` (globales), `lib/src/cubits/{entity}/` (compartidos) o `modules/{feature}/cubit/` (solo si lo amerita) | Estado de la UI, llama al Repository |
| **Repository** | `packages/*/domain/repositories/` | Lógica de negocio, convierte modelos |
| **DataSource** | `packages/*/data/` | Cliente HTTP con `HttpService` mixin, singletons |
| **Service** | `packages/*/services/` | HTTP raw, Hive, manejo de excepciones |

## Flujo de datos

```
Usuario interactúa
  → Screen (setState, BlocBuilder)
    → Cubit (método: create/get/update/delete)
      → Repository.create(item)
        → DataSource.create(item.toJson())
          → HttpService.postQuery(uri, body, headers)
            → http.post() → Map<String, dynamic>
        → Item.fromJson(result)
      → emit(writeSuccess(item))
    → BlocListener captura → SnackBar éxito
    → BlocBuilder rebuild → UI actualizada
```

## Reglas

- **Screen** nunca importa DataSource o Service directamente
- **Cubit** solo conoce Repository (inyectado por constructor)
- **Repository** orquesta DataSource y aplica lógica de negocio
- **DataSource** es una capa delgada que solo hace HTTP
- **Flujo estrictamente unidireccional** — nunca al revés
