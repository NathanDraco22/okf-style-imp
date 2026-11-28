---
type: Pattern
title: Policy Object (Decisiones de Negocio Centralizadas)
description: Patrón para extraer decisiones de negocio densas o repetidas a un objeto puro que recibe un contexto y devuelve bool, enum, objeto opcional o DTO de respuesta, compuesto desde el Cubit y sin I/O.
tags: [flutter, dart, policy, decision, enum, value-object, cubit, core, patron]
---

# Policy Object (Decisiones de Negocio Centralizadas)

Patrón para sacar las decisiones de negocio de las pantallas y de los bloques de `if` dispersos, y concentrarlas en un objeto puro que responde preguntas con resultados tipados.

## El problema

Una decisión suele depender de varias banderas de configuración o de estado:

```dart
final shouldCharge =
    settings.useShifts && settings.chargeConvertedOrdersToShift;
final requiresShift =
    shouldCharge && settings.requireShiftForConvertedOrders;
```

Ese cálculo se repite en varias pantallas. Cada consumidor puede interpretarlo distinto y, cuando aparece una condición nueva, hay que buscar todos los lugares donde se combinaron las banderas.

Los síntomas típicos:

- combinaciones de banderas repetidas en varios widgets o cubits;
- métodos con demasiados `if` anidados;
- decisiones que devuelven datos y se reconstruyen en cada pantalla;
- tests que tienen que montar UI para validar una regla de negocio;
- inconsistencias entre la vista, el ticket, el PDF y el backend.

## La solución

Un **Policy Object** es un objeto puro que:

- recibe el contexto que necesita (configuración, entidad, estado);
- encapsula una familia de decisiones;
- devuelve resultados tipados;
- no hace I/O, no conoce Flutter y no muta estado.

La pantalla deja de saber cómo se combinan las banderas:

```dart
final policy = getIt<WorkspaceSettingsCubit>().policy;
final approval = policy.approvalPolicy;

if (approval.requiresApproval) {
  // ...
}
```

El nombre del método expresa la intención de negocio. Si mañana cambia la regla, se cambia **solo** en la policy.

## Regla de retorno

| Situación | Retorno | Ejemplo |
| :--- | :--- | :--- |
| Una decisión binaria | `bool` | `bool get hasExchangeRate` |
| Varias decisiones excluyentes | `enum` | `ApprovalPolicy get approvalPolicy` |
| Un resultado que puede existir o no | Objeto opcional (`T?`) | `PriceSuggestion? suggestionFor(...)` |
| Un resultado compuesto que siempre existe | Objeto respuesta | `InvoiceDisplayConfig get displayConfig` |
| Una validación pura que necesita explicar el rechazo | Objeto decisión (`PolicyDecision`) | `PolicyDecision validateWineExtension()` |

Los enums pueden llevar comportamiento para que la UI no lo reinterprete:

```dart
enum ApprovalPolicy {
  never(requiresApproval: false, linksToApprover: false),
  optional(requiresApproval: false, linksToApprover: true),
  required(requiresApproval: true, linksToApprover: true);

  const ApprovalPolicy({
    required this.requiresApproval,
    required this.linksToApprover,
  });

  final bool requiresApproval;
  final bool linksToApprover;
}
```

Los objetos de resultado deben ser inmutables y llevar solo los datos que la decisión produce:

```dart
class PriceSuggestion {
  const PriceSuggestion({
    required this.costInCents,
    required this.priceInCents,
    required this.marginPercent,
    required this.unitFactor,
  });

  final int costInCents;
  final int priceInCents;
  final double marginPercent;
  final int unitFactor;

  int get totalPriceInCents => priceInCents * unitFactor;
}
```

Para validaciones puras donde `bool` no alcanza porque la pantalla o el flujo necesita mostrar el motivo del rechazo, usa un objeto de decisión inmutable en lugar de lanzar excepciones:

```dart
final class PolicyDecision {
  const PolicyDecision._({
    required this.allowed,
    this.code,
    this.reason,
  });

  const PolicyDecision.allow() : this._(allowed: true);

  const PolicyDecision.deny({
    required String code,
    required String reason,
  }) : this._(allowed: false, code: code, reason: reason);

  final bool allowed;
  final String? code;
  final String? reason;
}
```

### Composición de Policies

Una policy puede delegar reglas transversales en policies más pequeñas manteniendo pureza estricta (sin I/O y sin dependencias circulares):

```dart
final class AgePolicy {
  const AgePolicy(this.age);
  final int age;

  bool get isAdult => age >= 18;
}

final class ReservationPolicy {
  const ReservationPolicy(this.reservation);
  final Reservation reservation;

  PolicyDecision validateWineExtension() {
    if (!AgePolicy(reservation.guestAge).isAdult) {
      return const PolicyDecision.deny(
        code: 'GUEST_UNDERAGE',
        reason: 'El huésped debe tener al menos 18 años.',
      );
    }
    return const PolicyDecision.allow();
  }
}
```

## Composición con el Cubit

El Policy **no es un singleton global**. El cubit es el dueño del estado y lo construye a partir del estado actual:

```dart
class WorkspaceSettingsCubit extends Cubit<WorkspaceSettingsState> {
  WorkspaceSettingsPolicy get policy =>
      WorkspaceSettingsPolicy(currentSettings ?? WorkspaceSettings.defaults);
}
```

Los consumidores acceden con:

```dart
final policy = getIt<WorkspaceSettingsCubit>().policy;
```

o, en un widget reactivo:

```dart
final policy = context.watch<WorkspaceSettingsCubit>().policy;
```

El cubit **no debe replicar cada bandera como getter**. Expón `policy` y el estado crudo necesario; los consumidores preguntan por la decisión. Si agregas `bool get useApprovals => policy.useApprovals`, vuelves a tener dos formas de acceder a lo mismo y una superficie pública que crece con cada configuración.

### Por qué no un singleton

- **Una sola fuente de verdad:** el estado del cubit; la policy se deriva de él.
- **Sin sincronización manual:** no hay que acordarse de actualizar un global en cada `load`, `save`, `error` o `clear`.
- **Tests deterministas:** se instancia la policy directamente, sin `reset()` global.
- **Dependencia explícita:** el consumidor sabe de dónde viene la decisión.

Si una función pura necesita la decisión, recibe la policy como parámetro en lugar de buscarla en un global.

## Transiciones de estado

La policy es una vista calculada de la configuración vigente. No introduce un estado paralelo:

| Estado | Policy |
| :--- | :--- |
| `Initial` / primera carga | defaults del modelo |
| `Loading` con caché previa | última configuración confirmada |
| `Loaded` | nueva configuración |
| `Saving` | configuración anterior (no aplica cambios a medias) |
| `Error` al guardar | `lastSettings` o última configuración confirmada |
| `clear` / logout | defaults |

El formulario de configuración mantiene su propio draft local y solo se sincroniza cuando llega `Loaded`. Así puede mostrar cambios sin que el resto de la app los vea antes de guardar.

## Ejemplo genérico

```dart
final class WorkspaceSettingsPolicy {
  const WorkspaceSettingsPolicy([
    this.settings = WorkspaceSettings.defaults,
  ]);

  final WorkspaceSettings settings;

  bool get usesApprovals => settings.useApprovals;
  bool get hasExchangeRate => settings.exchangeRate > 0;

  ApprovalPolicy get approvalPolicy {
    if (!settings.useApprovals || !settings.chargeApprovalsToReviewer) {
      return ApprovalPolicy.never;
    }
    return settings.requireApproval
        ? ApprovalPolicy.required
        : ApprovalPolicy.optional;
  }

  PriceSuggestion? suggestionFor({
    required int costInCents,
    int unitFactor = 1,
  }) {
    if (!settings.useDefaultMargin || costInCents <= 0) {
      return null;
    }
    final factor = unitFactor <= 0 ? 1 : unitFactor;
    final price = (costInCents * (1 + settings.defaultMargin / 100)).round();
    return PriceSuggestion(
      costInCents: costInCents,
      priceInCents: price,
      marginPercent: settings.defaultMargin,
      unitFactor: factor,
    );
  }
}
```

## Dónde vive

En un paquete de negocio puro, no en la capa de UI:

```text
business_package/
└── lib/src/core/
    └── policies/
        └── workspace_settings/
            ├── workspace_settings_policy.dart
            ├── approval_policy.dart
            └── price_suggestion.dart
```

En Flutter, el paquete **no depende de Flutter ni de `GetIt`**. El cubit de la app compone la policy y la expone.

Para los modelos profundos que no tienen acceso al cubit (impresoras, PDFs, formatters) se mantiene el patrón de **resolver inyectado**:

```dart
MoneyFormatter.configureCurrencyResolver(() => policy.currency);
```

## Cuándo usarlo

- La misma decisión se repite en varias pantallas o flujos.
- Hay demasiados `if` o combinaciones de banderas.
- La decisión devuelve datos, no solo un booleano.
- Se quiere testear sin montar UI, sin red y sin base de datos.
- La regla puede cambiar de forma independiente al transporte.

## Cuándo no usarlo

- Un `if` local y simple que no se repite.
- Lógica que necesita I/O, transacciones o repositorios: eso es un **Core / Orchestrator**, no una policy.
- Una decisión sin consumidores claros; primero extrae la función, luego formaliza la policy.

## Checklist de adopción

1. Identifica una decisión repetida o con muchos `if`.
2. Define el contexto mínimo que necesita.
3. Elige el retorno: `bool`, `enum`, objeto opcional, objeto respuesta o `PolicyDecision`.
4. Crea la policy en `core/policies/<dominio>/`.
5. Expórtala desde el barrel del paquete.
6. Compón la policy desde el cubit y expón un getter `policy`.
7. Sustituye las combinaciones de banderas en los consumidores.
8. Agrega tests con la tabla de decisión completa.
9. Documenta la tabla en la wiki del módulo.
10. Si aplica, replica la misma tabla en el backend como fuente de verdad.

## Proyección backend

El patrón cuenta con su guía equivalente para Python/FastAPI en [Policy Object (Backend)](../../../backend/python_fastapi/patterns/policy_object.md), compartiendo exactamente la misma taxonomía de retornos y separación estricta:

- **Policy (`core/policies/`):** decisión o validación pura; recibe modelos ya cargados y devuelve `bool`, `Enum`, `T | None`, `@dataclass(frozen=True)` o `PolicyDecision`.
- **Repository / Pipeline-Steps / Orchestrator:** consulta la base de datos, maneja transacciones y traduce rechazos de la policy a `BusinessRuleException`.

## Implementación de referencia

- `kardex_app_front/packages/kardex_business/lib/src/core/policies/app_settings/app_settings_policy.dart`
- `kardex_app_front/lib/cubits/app_settings/app_settings_cubit.dart`
