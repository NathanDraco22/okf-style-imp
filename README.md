# OKF TEC — Open Knowledge Format (v0.2)

> **Base canónica de conocimiento, patrones arquitectónicos y estilo de código para desarrollo full-stack moderno en Flutter/Dart y Python/FastAPI.**

Este repositorio organiza la sabiduría técnica acumulada en proyectos de producción siguiendo el estándar **Open Knowledge Format (OKF v0.2)**. Todo su contenido está diseñado para ser **100% agnóstico al dominio de negocio** (usando entidades universales como `Item`, `Tenant`, `Workspace`, `User`), permitiendo que cualquier desarrollador o agente de IA traslade estos patrones directamente a aplicaciones de cualquier escala (ERP, CRM, SaaS, e-commerce, etc.).

---

## 🏛️ Principios Rectores de la Arquitectura

1. **Agnosticismo de Dominio:** Los patrones resuelven problemas universales de software, nunca casos acoplados a una lógica comercial específica.
2. **Arquitectura Onion (Clean Architecture):** Separación unidireccional estricta de responsabilidades tanto en el frontend como en el backend. Las capas externas dependen de las internas, nunca al revés.
3. **Prioridad CLI (`onion`):** Cero scaffolding manual. Toda estructura, clase o módulo nuevo se genera con `onion` CLI para garantizar consistencia y barrels actualizados.
4. **Reactividad Segura y Gestión de Ciclo de Vida:** Control riguroso de streams y memoria. Toda suscripción reactiva cuenta con cancelación asíncrona garantizada (`close()`), guardas de emisión (`isClosed`) y liberación de recursos en repositorios (`disposeReactiveRepo()`).

---

## 📂 Mapa General del Repositorio

```
okf-style-imp/
├── INDEX.md                          # Índice raíz del bundle (metadatos OKF v0.2)
├── AGENTS.md                         # Guía de operación para Agentes de IA (Fase Descubrimiento / Ejecución)
├── README.md                         # Este documento: visión global y explicativa del bundle
├── log.md                            # Bitácora histórica de cambios, experimentos y versiones
│
├── frontend/                         # Conocimientos del Stack Frontend
│   ├── INDEX.md                      # Índice general de frontend
│   └── flutter_dart/
│       ├── INDEX.md                  # Índice de Flutter/Dart
│       ├── style/                    # Estilo y convenciones de código
│       │   ├── INDEX.md
│       │   ├── naming_conventions_flutter.md  # Naming camelCase, PascalCase, archivos
│       │   └── project_structure_flutter.md   # Árbol estándar de carpetas lib/src/
│       └── patterns/                 # Catálogo de 24 patrones de diseño
│           ├── INDEX.md              # Índice temático de patrones Flutter
│           ├── flutter_onion_flow.md # Flujo Screen → Cubit → Repo → DataSource → API
│           ├── read_write_cubits.md  # Separación estricta ReadCubit vs WriteCubit
│           ├── cross_cubit_event_sync.md # Sincronización desacoplada vía Repository Stream
│           ├── global_formatter_resolver.md # Formatter puro + callback del Cubit global
│           ├── reactive_repository.md# Mixin broadcast para reactividad sin cache
│           ├── go_router_pattern.md  # Router reactivo a streams + lastKnownLocation
│           ├── dialog_show_functions.md # Show functions tipadas fuera del router
│           ├── view_controller_pattern.md # Estado efímero de vistas desacoplado
│           ├── view_segmentation.md  # Orquestador, unidades y fuente de verdad
│           ├── screen_structure.md   # MultiBlocProvider → Listener → Builder
│           ├── cubit_states_read.md  # Estados sellados de lectura (Initial..Success..Error)
│           ├── cubit_states_write.md # Estados sellados de mutación (Initial..Writing..Created)
│           ├── pagination_infinite_scroll.md # Scroll infinito con búsqueda en caliente
│           ├── data_source_pattern.md# Singletons y HttpService mixin
│           ├── http_service_mixin.md # Cliente HTTP unificado con tipado de errores
│           ├── di_flutter.md         # Inyección de dependencias (RepositoryProvider / BlocProvider)
│           ├── mediator_pattern.md   # Formularios complejos con InheritedWidget
│           ├── exception_handling_flutter.md # Excepciones selladas de red y negocio
│           ├── action_flow_functions.md # Acciones compuestas como top-level functions
│           ├── flutter_models.md     # Base / Create / Update / InDb con JSON
│           ├── list_response_flutter.md # DTO de listas con conteo
│           ├── query_params.md       # Filtros tipados toMap()
│           ├── responsive_design.md  # Adaptabilidad Desktop / Web / Mobile
│           └── cli_commands_flutter.md # Referencia de comandos `onion` para Flutter
│
└── backend/                          # Conocimientos del Stack Backend
    ├── INDEX.md                      # Índice general de backend
    └── python_fastapi/
        ├── INDEX.md                  # Índice de Python / FastAPI
        ├── style/                    # Estilo y convenciones de código
        │   ├── INDEX.md
        │   ├── naming_conventions.md # snake_case, PascalCase, convenciones REST
        │   └── project_structure.md  # Árbol de carpetas app/ (routers, controllers, etc.)
        └── patterns/                 # Catálogo de 18 patrones de diseño
            ├── INDEX.md              # Índice temático de patrones FastAPI
            ├── onion_flow.md         # Flujo Router → Controller → Repo → DataSource → Collection → DB
            ├── pipeline_steps.md     # Procesos multi-paso con contexto, steps y transacción
            ├── router_pattern.md     # 5 endpoints CRUD estándar por entidad
            ├── controller_pattern.md # Inyección de repo, 404 handler y DTOs
            ├── repository_crud.md    # Reglas de negocio y orquestación
            ├── datasource_layer.md   # Conversión dict-to-dict
            ├── collection_crud.md    # CRUD sobre MongoDB con AsyncMongoClient
            ├── model_pattern.md      # Pydantic Base / Create / Update / InDb
            ├── list_response.md      # Response genérico count / data
            ├── soft_delete.md        # isDeleted universal y filtros de lectura
            ├── timestamps.md         # createdAt / updatedAt en Unix milisegundos
            ├── search_pattern.md     # Búsqueda de texto: searchPrefixes + índice $text
            ├── mongo_transactions.md # Atomicidad multivariable con session
            ├── singleton.md          # get_instance() en DataSources y Collections
            ├── di_wiring.md          # DI manual por constructor
            ├── exception_handling.md # Jerarquía sellada de excepciones y handler global
            ├── versioning.md         # Versionado v1/v2 estructurado
            └── cli_commands.md       # Referencia de comandos `onion` para backend
```

---

## 📱 Frontend — Flutter / Dart

El stack de frontend implementa una variante cliente de la **Arquitectura Onion** potenciada por `flutter_bloc` (Cubits) y Streams reactivos.

```
┌────────────────────────────────────────────────────────┐
│                        Screen                          │  UI / Widgets
└──────────────────────────┬─────────────────────────────┘
                           ▼
┌────────────────────────────────────────────────────────┐
│               ReadCubit / WriteCubit                   │  Gestión de Estado
└──────────────────────────┬─────────────────────────────┘
                           ▼
┌────────────────────────────────────────────────────────┐
│           Repository (with ReactiveRepository)         │  Reglas de Negocio / Event Broker
└──────────────────────────┬─────────────────────────────┘
                           ▼
┌────────────────────────────────────────────────────────┐
│                      DataSource                        │  HttpService / Serialization
└──────────────────────────┬─────────────────────────────┘
                           ▼
┌────────────────────────────────────────────────────────┐
│                     Backend API                        │  Red / REST
└────────────────────────────────────────────────────────┘
```

### 1. State Management: Separación Read / Write
- **`ReadCubit`:** Responsable exclusivamente de cargar datos (`getAll`, `getById`, `refresh`). Expone estados sellados: `Initial`, `Loading`, `Success` (con changelog `newItems`, `updatedItems`, `deletedItems`) y `Error`.
- **`WriteCubit`:** Responsable de mutaciones (`create`, `update`, `delete`). Emite estados transitorios (`Writing`, `ItemCreated`, `ItemUpdated`, `ItemDeleted`, `Error`) y resetea automáticamente a `Initial` tras el éxito para permitir operaciones consecutivas sin reconstruir el cubit.

### 2. Capa de Datos Reactiva (`ReactiveRepository<T>`)
- El repositorio **no mantiene colecciones ni caché en memoria** (la única fuente de verdad de la lista es el `ReadCubit`).
- Utiliza un `StreamController<RepoEvent<T>>.broadcast()` para emitir eventos atómicos (`RepoItemCreated`, `RepoItemUpdated`, `RepoItemDeleted`) únicamente después de que la mutación fue confirmada por el servidor.
- Cuenta con guardas `if (!_controller.isClosed)` y método `disposeReactiveRepo()` para su finalización.

### 3. Sincronización Desacoplada (`Cross-Cubit Event Sync`)
- Resuelve la comunicación entre Cubits independientes sin acoplarlos entre sí (evitando inyecciones directas Cubit-a-Cubit o listeners dispersos en widgets).
- El repositorio actúa como **Event Broker tipado**: cualquier Cubit global (como `AppSessionCubit` o `WorkspaceCubit`) puede escuchar `repository.eventStream` en su constructor, reaccionar a cambios en entidades compartidas y cancelar su suscripción de forma estricta en su método `close()`.

### 4. Navegación Avanzada con `GoRouter`
- **Reacción Atómica por Streams (`GoRouterRefreshStream`):** El router escucha el stream de sesión global y redirige reactivamente ante cambios de ciclo de vida (`Splash` $\to$ `Onboarding` $\to$ `Inactive` $\to$ `Login` $\to$ `Dashboard`).
- **Preservación Pasiva de Ruta (`lastKnownLocation`):** Cuando ocurre un cambio de contexto global (ej. conmutar de organización/sucursal o refrescar tokens), GoRouter preserva la pantalla de trabajo del usuario en lugar de redirigir forzosamente al Home.
- **Rutas Overlay a Pantalla Completa:** Subrutas profundas (`/items/new`) usan `parentNavigatorKey: rootNavigatorKey` para ocupar el 100% de la ventana por encima del `StatefulShellRoute`.

### 5. Diálogos y Modales fuera del Router (`Show Functions`)
- Los diálogos, modales y visores son funciones top-level (`showItemEditor(context)`, `showItemViewer(context)`) que devuelven `Future<T?>`.
- Garantizan contratos fuertemente tipados (`null` = cancelado), componibilidad y mantienen limpio el árbol de rutas de GoRouter.

### 6. Formatter Global con Resolver Inyectado
- Las utilidades de formato viven en paquetes puros y no conocen el Cubit ni `GetIt`.
- El Cubit global registra un callback una sola vez (`NumberFormatter.configureCurrencyResolver(() => currency)`); el formatter lo ejecuta en cada operación para leer el estado actual.
- No hay setters repartidos: `emit(AppSettingsLoaded(...))` cambia la respuesta del Cubit y el siguiente formateo usa el nuevo valor.
- El resolver se limpia en `close()` para evitar referencias al Cubit destruido; se aplica a moneda, locale, fechas o unidades. Ver [global_formatter_resolver.md](frontend/flutter_dart/patterns/global_formatter_resolver.md).

---

## ⚡ Backend — Python / FastAPI

El backend está construido sobre FastAPI y MongoDB, estructurado en **5 capas concéntricas unidireccionales**:

```
Request ──> Router ──> Controller ──> Repository ──> DataSource ──> Collection ──> MongoDB
```

| Capa | Responsabilidad | Entrada / Salida |
|---|---|---|
| **Router** | Endpoints HTTP, validación de schemas con Pydantic, status codes REST | Pydantic Models $\leftrightarrow$ JSON HTTP |
| **Controller** | Manejo de 404, orquestación de DTOs, inyección manual del Repository | Pydantic Models $\leftrightarrow$ Pydantic Models |
| **Repository** | Lógica y reglas de negocio, validaciones complejas, cálculo de estados | Objetos de dominio $\leftrightarrow$ DTOs |
| **DataSource** | Conversión entre modelos de negocio tipados y diccionarios primitivos | Dicts / Primitivos $\leftrightarrow$ Pydantic Models |
| **Collection** | Driver asíncrono (`AsyncMongoClient`), queries MongoDB, soft delete y timestamps | Dicts BSON $\leftrightarrow$ Base de Datos |

### Procesos Complejos

Cuando un caso de uso combina validación, lecturas, varios documentos y movimientos transaccionales, se orquesta con **Pipeline-Steps**: un contexto compartido, pasos independientes y un resultado final tipado. El orquestador conserva la transacción; el mutex y el retry quedan en el Manager que lo invoca; los steps no hacen commit ni abort. Ver [Pipeline-Steps](backend/python_fastapi/patterns/pipeline_steps.md).

### Convenciones Críticas de Persistencia
- **Soft Delete Universal:** Todas las entidades implementan el campo booleano `isDeleted: bool = False`. El borrado lógico se realiza con `update_one({"$set": {"isDeleted": True}})`. Las lecturas excluyen automáticamente registros eliminados (`{"isDeleted": False}`).
- **Timestamps en Milisegundos Unix:** Campos `createdAt: int` y `updatedAt: int` expresados en enteros UTC Unix ms (generados mediante `time_tools`).
- **Búsqueda por Texto Óptima:** Combinación de prefijos generados (`searchPrefixes`) con índices `$text` de MongoDB ponderados por relevancia (`textScore`).
- **Transacciones Atómicas:** Operaciones multivariable aseguradas mediante `async with await client.start_session() as session: async with session.start_transaction():`.

---

## 🛠️ Herramientas de Automatización: Onion CLI

El estándar establece como **Regla Fundamental #1 la Prioridad CLI**: toda estructura nueva se genera con el CLI `onion` para evitar discrepancias humanas.

### Comandos Clave de Frontend (`onion >= 0.6.0`)
```bash
# Generar entidad completa (Modelos, DataSource, Repository reactivo y Cubits)
onion dart item --reactive

# Generar solo la pareja Read/Write Cubit con suscripción reactiva
onion dart-cubit item --reactive

# Generar módulo visual completo (Screen, Vista, Controller, etc.)
onion flutter-module item

# Generar contratos de modelos Base / Create / Update / InDb
onion dart-model item
```

### Comandos Clave de Backend
```bash
# Inicializar nuevo microservicio/API desde el template canónico
onion project fastapi-init

# Generar CRUD completo en MongoDB (Router, Controller, Repo, DataSource, Collection)
onion crud-mongo item

# Generar Router individual con 5 endpoints REST
onion router item
```

---

## 🤖 Guía de Consulta para Desarrolladores y Agentes de IA

Cuando trabajes en un proyecto que siga este estándar:

1. **Fase de Descubrimiento (al inicio):**
   - Comienza siempre en [`INDEX.md`](INDEX.md).
   - Lee el índice de la tecnología objetivo: [`frontend/flutter_dart/INDEX.md`](frontend/flutter_dart/INDEX.md) o [`backend/python_fastapi/INDEX.md`](backend/python_fastapi/INDEX.md).
   - Revisa las guías de estilo correspondientes (`style/INDEX.md`) antes de proponer cambios de sintaxis o nomenclatura.
2. **Fase de Ejecución (al planificar o codificar):**
   - Consulta el catálogo de patrones en `patterns/INDEX.md`.
   - Si vas a crear código nuevo, prioriza los comandos de `onion` documentados en `cli_commands.md` o `cli_commands_flutter.md`.
   - Consulta [`log.md`](log.md) para conocer las decisiones de diseño previas, advertencias y experimentos realizados.

---

## 📜 Licencia y Mantenimiento

Este compendio es de uso abierto para la comunidad bajo la filosofía **Open Knowledge Format (OKF)**. Sus patrones son permanentemente enriquecidos a partir de aprendizajes extraídos de proyectos reales de software.
