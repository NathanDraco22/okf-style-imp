---
type: Pattern
title: Segmentación de Vistas UI
description: Reglas generales para estructurar interfaces complejas: la pantalla como orquestador, unidades con una sola responsabilidad, comunicación por contexto, fuente de verdad única, dos canales async, adaptación por capacidad y feedback siempre visible.
tags: [flutter, ui, segmentacion, vistas, mantenible, escalable, arquitectura]
---

# Segmentación de Vistas UI

Reglas generales para estructurar interfaces complejas de forma mantenible y escalable. Agnósticas a lenguaje, framework o librería: aplicables a Flutter, React, Vue, SwiftUI, etc.

## 1. La pantalla es un orquestador, no un monolito

Un componente raíz que **solo arma el layout** y delega cada región visual a un componente independiente. El orquestador no contiene lógica de negocio ni detalles de implementación: conecta regiones y define proporciones, orden y espaciado.

- La raíz debe poder leerse de corrido (10-40 líneas): se entiende la pantalla con una mirada.
- Cada región (header, contenido, panel lateral, barra de acciones...) es un componente propio.

> Ver [Estructura de Pantalla](screen_structure.md): la Screen pública y el `_RootScaffold` son los orquestadores; las regiones se delegan en `_Body` y subwidgets privados.

```dart
class _RootScaffold extends StatelessWidget {
  const _RootScaffold();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Pacientes')),
      body: const Column(
        children: [
          _HeaderSection(),          // Región: filtros y búsqueda
          Expanded(child: _ContentSection()),  // Región: lista
          _FooterSection(),          // Región: totales / acciones
        ],
      ),
    );
  }
}
```

## 2. Una unidad = una responsabilidad

Criterios para extraer una región a su propia unidad:

- Supera un tamaño razonable de líneas (≈200).
- Tiene estado interno o lógica propia (búsqueda, filtros, expansión).
- Se reutiliza en más de un contexto.

Reglas de la unidad extraída:

- Todo lo interno es **privado** a la unidad (clases/componentes `_privados`).
- Solo el orquestador se expone al resto del sistema.
- Una unidad = una razón de cambio. Si un cambio toca dos regiones sin relación, la frontera está mal trazada.

## 3. Comunicación sin prop-drilling

Las secciones necesitan el estado de la pantalla. En lugar de pasar props/callbacks por una cadena de widgets, **exponer el estado por contexto**:

- InheritedWidget / InheritedNotifier, Provider, slots, inyección de dependencias... cualquier mecanismo del stack que permita leer desde cualquier descendiente.
- Las secciones leen el estado directamente (`obtener de contexto`) y notifican intenciones.
- El contrato: la raíz provee, las secciones consumen. No hay intermediarios.

Beneficio: mover, reordenar o extraer secciones no obliga a reescribir la cadena de props.

> En Flutter: [Mediator Pattern](mediator_pattern.md) (`InheritedNotifier.of/read`) para estado de formulario, `RepositoryProvider`/`BlocProvider` ([DI Flutter](di_flutter.md)) para cubits y repositorios.

## 4. Fuente de verdad única por pantalla

Un único objeto de estado mutable (notifier / store / view-model / controller) concentra todo el estado local de la pantalla:

- **Setters con efectos secundarios**: cambiar un selector puede recalcular precios, invalidar dependencias o cambiar el método de pago. Ese efecto vive en el setter, no en la UI.
- **Getters derivados**: totales, validaciones, estados combinados se calculan en el objeto de estado, no en los componentes.
- La UI **solo lee y notifica intención**: nunca muta datos ni esconde reglas de negocio.
- La UI se suscribe de forma **granular**: solo los componentes afectados se reconstruyen (listeners/selectores específicos, no reconstruir la pantalla entera).

> En Flutter: `BlocSelector` / `context.select()` para suscripción granular; `ViewModel` con getters derivados (ver [Mediator Pattern](mediator_pattern.md)).

```dart
// Suscripción granular: solo este widget se reconstruye
BlocSelector<ReadItemCubit, ReadItemState, List<Item>>(
  selector: (state) => state.items,
  builder: (context, items) => _ItemsList(items: items),
)
```

## 5. Estado asíncrono en dos canales

Separar **lectura** (fetch, búsqueda, filtros) de **escritura** (crear, actualizar, eliminar):

- Los filtros y búsquedas **no deben borrar los datos visibles**: usa un estado intermedio de tipo "buscando" que conserva la carga anterior y muestra un indicador discreto (barra de progreso) en lugar de un spinner central que vacía la pantalla.
- Los efectos secundarios de escritura (loading, error, éxito) se **orquestan en un solo lugar** (listener/efecto raíz), no dispersos en cada botón.
- Flujo típico: botón → mutación → estado transitorio "en progreso" → éxito/error → sincronización del canal de lectura.

> En Flutter: ver [Read / Write Cubits](read_write_cubits.md) (canales separados), [Estados de Lectura](cubit_states_read.md) (`ReadItemRefreshing` conserva datos previos), [Paginación Infinita y Búsqueda](pagination_infinite_scroll.md) (búsqueda server-side que conserva datos) y el `BlocListener` raíz de [Estructura de Pantalla](screen_structure.md).

## 6. Adaptación por capacidad, no por dispositivo

Decidir el layout por **ancho disponible**, no por tipo de dispositivo:

- Breakpoints para variantes, o un layout único que se reacomoda (proporciones, columnas según espacio).
- Ocultar/mostrar acciones según el espacio disponible, no según "es celular/tablet".
- No duplicar pantallas completas si el mismo layout funciona adaptándose.
- Regla pragmática: si un dispositivo no es objetivo del módulo, igualmente el layout debe **degradar sin romperse** (sin overflow, sin contenido inaccesible).

> Ver [Diseño Responsive](responsive_design.md): `context.isMobile()` usa el ancho (600px), y las secciones se comparten entre `mobile/` y `web/` con part files — solo se duplica lo que realmente diverge.

## 7. Reutilizar lo genérico, copiar lo que diverge

- Los componentes de propósito general (selectores, visualizadores, formateadores, diálogos de confirmación) van al nivel **compartido**.
- Si un módulo necesita una variante específica del comportamiento, **cópiala local**, hazla exclusiva del módulo y modifícala ahí.
- **Nunca sobrecargues** el componente compartido con flags del caso particular: cada flag nuevo es un acoplamiento nuevo. Un diálogo compartido que acumula opciones para "este módulo" se vuelve inmantenible.
- Si la variante es genuinamente general, primero generalízala en el nivel compartido; si solo sirve a un contexto, que viva en ese contexto.

> En Flutter: los widgets compartidos del módulo van en `widgets/` raíz; los exclusivos de plataforma en `mobile/widgets/` o `web/widgets/` (ver [Diseño Responsive](responsive_design.md)).

## 8. Feedback siempre visible

Todo estado debe ser evidente para el usuario:

- Estados bien definidos: **vacío, carga, error, sin resultados, deshabilitado**.
- Si un botón se deshabilita por una condición (falta de datos, valores inválidos), **explica el porqué** con un tooltip o mensaje contextual.
- Los estados vacíos invitan a la acción ("agrega un producto", "reintenta").
- Los errores ofrecen una salida (reintentar, volver).

> En Flutter: los estados del cubit ya distinguen vacío/carga/error (ver [Estados de Lectura](cubit_states_read.md)); los errores se muestran con SnackBar en el listener raíz (ver [Estructura de Pantalla](screen_structure.md) y [Manejo de Excepciones](exception_handling_flutter.md)).

```dart
Tooltip(
  message: 'Agrega al menos un producto para guardar',
  child: ElevatedButton(
    onPressed: canSave ? _save : null,  // Deshabilitado + explicado
    child: const Text('Guardar'),
  ),
)
```

## 9. Regla de oro: una sola razón de cambio

Antes de escribir o extraer código, pregúntate:

- ¿Este cambio toca más de una responsabilidad? → divide.
- ¿Esta unidad tendría que modificarse por motivos no relacionados? → la frontera está mal.
- ¿Puedo mover, reordenar o eliminar esta región sin tocar las demás? → el diseño es correcto.

La segmentación no es un fin: es el medio para que un cambio local sea local.

## Reglas

- La raíz es un **orquestador** legible de corrido (10-40 líneas), sin lógica de negocio
- Una unidad se extrae si: >200 líneas, estado propio, o se reutiliza
- Internos privados (`_`), solo el orquestador se expone
- Estado por **contexto**, nunca prop-drilling
- Un objeto de estado por pantalla: setters con side effects, getters derivados, suscripción granular
- Lectura y escritura en **canales separados**; los filtros nunca vacían la pantalla
- Layout por **ancho disponible**, no por dispositivo; degradar sin romperse
- Lo genérico al nivel compartido, la variante local se copia — nunca flags de módulo en lo compartido
- Feedback de todo estado: vacío, carga, error, sin resultados, deshabilitado (explicado)
