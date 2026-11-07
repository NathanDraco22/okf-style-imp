---
type: Pattern
title: Cross-Cubit Event Sync (Sincronización Desacoplada vía Repository Stream)
description: Patrón para sincronizar estado entre Cubits independientes utilizando el eventStream de ReactiveRepository como Event Broker tipado, eliminando dependencias directas entre Cubits y garantizando el cierre estricto de suscripciones.
tags: [flutter, cubit, stream, reactivo, desacoplamiento, estado, patron]
---

# Cross-Cubit Event Sync

Patrón arquitectónico para comunicar mutaciones de datos entre Cubits desacoplados sin acoplarlos entre sí ni delegar la sincronización a la vista. Utiliza el `eventStream` del [ReactiveRepository](reactive_repository.md) como un **Event Broker tipado**.

## El Problema: Acoplamiento entre Cubits

En aplicaciones modulares, los cambios realizados en una entidad dentro de un módulo secundario a menudo deben impactar el estado global de la aplicación. Por ejemplo:
- Un `TenantManagementCubit` edita el nombre del tenant activo o lo elimina.
- El `AppSessionCubit` (que gobierna la sesión, el header global y la navegación) necesita reflejar ese cambio de inmediato.

Existen dos anti-patrones comunes que deben evitarse:

```
❌ Anti-patrón 1: Cubit llamando a Cubit (Acoplamiento Fuerte)
   AppSessionCubit ──(depende directamente de)──> TenantManagementCubit
   * Provoca referencias circulares, orden de inicialización frágil y dificulta pruebas unitarias.

❌ Anti-patrón 2: Coordinación Manual en la UI
   BlocListener<TenantManagementCubit>(
     listener: (context, state) => context.read<AppSessionCubit>().syncTenant(state),
   )
   * Frágil: si la mutación ocurre en un diálogo, fondo o deep link sin ese listener montado, la app se desincroniza.
```

## La Solución: El Repositorio como Event Broker

El repositorio reactivo ya conoce cuándo ocurren mutaciones exitosas en la fuente de datos. En lugar de acoplar cubits, el `AppSessionCubit` se suscribe directamente al `eventStream` del repositorio que gestiona esa entidad:

```
                WriteTenantCubit.update(tenant)
                              │
                              ▼
               TenantRepository.updateTenant()
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
               API / DataSource    notifyItemUpdated(tenant)
                                        │
                                        ▼
                                 eventStream (Broadcast)
                                ┌───────┴───────┐
                                ▼               ▼
                        ReadTenantsCubit   AppSessionCubit
                       (refresca lista)   (actualiza tenant activo)
```

## Implementación

### 1. El Repositorio Reactivo

El repositorio utiliza el mixin `ReactiveRepository<T>` y notifica tras mutaciones exitosas:

```dart
class TenantRepository with ReactiveRepository<Tenant> {
  final TenantDataSource dataSource;

  TenantRepository({required this.dataSource});

  Future<Tenant> updateTenant(String id, UpdateTenant dto) async {
    final response = await dataSource.updateTenant(id, dto.toJson());
    final updated = Tenant.fromJson(response);
    notifyItemUpdated(updated); // Emite en eventStream
    return updated;
  }

  Future<void> deleteTenant(String id) async {
    await dataSource.deleteTenant(id);
    notifyItemDeleted(Tenant(id: id)); // Emite en eventStream
  }
}
```

### 2. El Cubit Consumidor (`AppSessionCubit`)

El cubit suscriptor abre la escucha en su constructor y cancela obligatoriamente la suscripción en `close()`:

```dart
class AppSessionCubit extends Cubit<AppSessionState> {
  final TenantRepository tenantRepository;
  StreamSubscription<RepoEvent<Tenant>>? _tenantSubscription;

  AppSessionCubit({
    required this.tenantRepository,
  }) : super(const AppSessionState()) {
    // Suscripción al stream reactivo del repositorio
    _tenantSubscription = tenantRepository.eventStream.listen(_handleTenantEvent);
  }

  void _handleTenantEvent(RepoEvent<Tenant> event) {
    // Guardia de seguridad: evitar emitir si el cubit fue cerrado
    if (isClosed) return;
    if (!state.isAuthenticated) return;

    switch (event) {
      case RepoItemCreated<Tenant>(:final item):
        final currentTenants = state.tenants;
        if (!currentTenants.any((t) => t.id == item.id)) {
          emit(state.copyWith(tenants: [...currentTenants, item]));
        }

      case RepoItemUpdated<Tenant>(:final item):
        final updatedList = state.tenants
            .map((t) => t.id == item.id ? item : t)
            .toList();
        final updatedCurrent =
            state.activeTenant?.id == item.id ? item : state.activeTenant;

        emit(state.copyWith(
          tenants: updatedList,
          activeTenant: updatedCurrent,
        ));

      case RepoItemDeleted<Tenant>(:final item):
        final updatedList = state.tenants
            .where((t) => t.id != item.id)
            .toList();

        emit(state.copyWith(tenants: updatedList));

        // Si se eliminó el tenant activo, conmutar al primero restante o limpiar
        if (state.activeTenant?.id == item.id) {
          final fallbackTenant = updatedList.firstOrNull;
          if (fallbackTenant != null) {
            changeActiveTenant(fallbackTenant.id);
          } else {
            emit(state.copyWith(clearActiveTenant: true));
          }
        }
    }
  }

  Future<void> changeActiveTenant(String tenantId) async {
    final tenant = state.tenants.firstWhere((t) => t.id == tenantId);
    emit(state.copyWith(activeTenant: tenant));
  }

  @override
  Future<void> close() {
    // Cancelación obligatoria para evitar fugas de memoria
    _tenantSubscription?.cancel();
    return super.close();
  }
}
```

## Control del Ciclo de Vida y Reglas de Seguridad

| Regla | Por qué es indispensable |
|---|---|
| **`StreamSubscription.cancel()` en `close()`** | Si el cubit se destruye y la suscripción queda abierta, continuará escuchando eventos en memoria produciendo memory leaks y llamadas ilegales a `emit()`. |
| **Guarda `if (isClosed) return;`** | Protege contra eventos asíncronos en vuelo que puedan procesarse justo después de que el cubit inicie su proceso de desecho. |
| **Broadcast Stream** | `ReactiveRepository` usa `StreamController.broadcast()`, permitiendo que $N$ cubits escuchen el mismo repositorio sin interferir entre ellos. |
| **Desacoplamiento total** | `TenantManagementCubit` y `AppSessionCubit` no se conocen; ambos solo interactúan con `TenantRepository`. |

## Comparativa de Mecanismos

| Enfoque | Acoplamiento | Resiliencia | Ciclo de Vida | Recomendación |
|---|---|---|---|---|
| **Inyectar Cubit A en Cubit B** | Alto (bidireccional) | Frágil (rompe DI) | Complejo | ❌ Prohibido |
| **BlocListener en UI (`markXxx`)** | Medio (acoplado a widget) | Solo funciona si el widget está vivo | Automático | ⚠️ Solo para diálogos locales acoplados |
| **Cross-Cubit Event Sync (Stream)** | **Bajo (solo a Repository)** | **Robusto (global e independiente de UI)** | **Garantizado con cancel()** | **✅ Recomendado para estado global** |

## Relacionados

- [ReactiveRepository Mixin](reactive_repository.md) — infraestructura del `eventStream` y notifiers
- [Read / Write Cubits](read_write_cubits.md) — separación canónica de cubits de lectura y escritura
- [GoRouter](go_router_pattern.md) — router reactivo a los cambios del `AppSessionCubit`
