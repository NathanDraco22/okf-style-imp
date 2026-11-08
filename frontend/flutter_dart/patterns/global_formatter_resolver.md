---
type: Pattern
title: Formatter Global con Resolver Inyectado (Callback Resolver)
description: Patrón para que un formatter puro lea configuración global (moneda, locale, formato) desde un Cubit mediante un callback registrado una sola vez y limpiado en close, evitando setters manuales en cada flujo y respetando la dirección de dependencias del paquete.
tags: [flutter, dart, formatter, callback, resolver, estado-global, cubit, desacoplamiento, patron]
---

# Formatter Global con Resolver Inyectado

Patrón para que una utilidad de formato viva en un paquete puro y, aun así, use configuración global que pertenece a la capa de aplicación. El formatter no conoce el Cubit ni `GetIt`; recibe una **función** que lee el estado actual cuando la necesita.

## El problema

Un formatter de dinero suele vivir en la capa de dominio o en un paquete Dart compartido:

```text
kardex_business (Dart puro)
  └── NumberFormatter
```

La configuración global vive en la capa de aplicación:

```text
kardex_app_front (Flutter)
  ├── AppSettingsCubit
  └── GetIt
```

Si `NumberFormatter` importa `AppSettingsCubit` o `GetIt`, se rompe la dirección de dependencias: el paquete compartido pasa a depender del frontend y aparece un acoplamiento circular o imposible de compilar.

Las alternativas habituales también tienen problemas:

```text
❌ Setter manual en cada flujo
   loadSettings()  -> NumberFormatter.setCurrency(...)
   updateSettings() -> NumberFormatter.setCurrency(...)
   * Fácil olvidar una ruta (login, caché, error, refresh, tests).

❌ Formatter lee GetIt directamente
   NumberFormatter -> GetIt -> AppSettingsCubit
   * El paquete puro depende del service locator de la app.

❌ Cada pantalla pasa el símbolo manualmente
   Text(formatMoney(total, currencySymbol: cubit.currency.symbol))
   * Repetitivo y fácil de olvidar en tickets, PDFs, CSV y tablas.
```

## La solución: resolver por callback

El formatter guarda una **función** que devuelve el valor actual. El Cubit global registra esa función una sola vez y la limpia cuando se destruye.

```text
AppSettingsCubit
      │ registra una vez
      ▼
NumberFormatter._currencyResolver = () => currency
      │
      │ cada formateo
      ▼
NumberFormatter.convertToMoneyLike(cents)
      │
      ▼
_resolver() ──> AppSettingsCubit.currency ──> estado actual del Cubit
      │
      ▼
"C$1,234.56" / "US$1,234.56" / "₡1,234.56" / ...
```

La clave es que **no se guarda la moneda**, se guarda la forma de preguntarla. Por eso no hace falta actualizar el formatter cuando cambia el estado.

## Implementación

### 1. Value object de configuración

```dart
enum AppCurrency {
  nio(code: 'NIO', displayName: 'Córdoba nicaragüense', symbol: 'C\$'),
  usd(code: 'USD', displayName: 'Dólar estadounidense', symbol: 'US\$'),
  hnl(code: 'HNL', displayName: 'Lempira hondureña', symbol: 'L'),
  gtq(code: 'GTQ', displayName: 'Quetzal guatemalteco', symbol: 'Q'),
  crc(code: 'CRC', displayName: 'Colón costarricense', symbol: '₡'),
  cop(code: 'COP', displayName: 'Peso colombiano', symbol: 'COL\$'),
  mxn(code: 'MXN', displayName: 'Peso mexicano', symbol: 'MX\$'),
  eur(code: 'EUR', displayName: 'Euro', symbol: '€');

  const AppCurrency({
    required this.code,
    required this.displayName,
    required this.symbol,
  });

  final String code;
  final String displayName;
  final String symbol;

  static AppCurrency fromCode(String? code) {
    final normalized = code?.toUpperCase();
    return AppCurrency.values.firstWhere(
      (currency) => currency.code == normalized,
      orElse: () => AppCurrency.nio,
    );
  }
}
```

### 2. Formatter puro con resolver

```dart
import 'package:intl/intl.dart';

typedef CurrencyResolver = AppCurrency Function();

class NumberFormatter {
  static CurrencyResolver? _currencyResolver;

  static final NumberFormat _decimal = NumberFormat.decimalPattern('en_US')
    ..minimumFractionDigits = 2
    ..maximumFractionDigits = 2;

  /// Se registra una sola vez desde la capa de aplicación.
  static void configureCurrencyResolver(CurrencyResolver? resolver) {
    _currencyResolver = resolver;
  }

  /// Lee la moneda actual cada vez que se necesita.
  static AppCurrency get currency =>
      _currencyResolver?.call() ?? AppCurrency.nio;

  static String get currencySymbol => currency.symbol;

  static NumberFormat get currencyFormat =>
      NumberFormat.currency(symbol: currencySymbol);

  static String convertToMoneyLike(int valueInCents) {
    final value = valueInCents / 100;
    return currencySymbol + _decimal.format(value);
  }
}
```

El fallback (`AppCurrency.nio`) cubre el arranque antes de cargar la configuración y cualquier error de inicialización.

### 3. Cubit global que registra el callback

```dart
class AppSettingsCubit extends Cubit<AppSettingsState> {
  final AppSettingsRepository repository;

  AppSettingsCubit(this.repository) : super(AppSettingsInitial()) {
    // Se registra una sola vez, al crear el Cubit global.
    NumberFormatter.configureCurrencyResolver(() => currency);
  }

  AppSettingModel? get currentSettings {
    if (state is AppSettingsLoaded) {
      return (state as AppSettingsLoaded).settings;
    }
    if (state is AppSettingsSaving) {
      return (state as AppSettingsSaving).currentSettings;
    }
    return repository.currentSettings;
  }

  AppCurrency get currency =>
      currentSettings?.currency ?? AppCurrency.nio;

  Future<void> loadSettings() async {
    emit(AppSettingsLoading());
    final settings = await repository.getCurrentSettings();
    emit(AppSettingsLoaded(settings)); // El resolver ya devuelve la nueva moneda.
  }

  Future<bool> updateSettings(UpdateAppSetting update) async {
    final updated = await repository.updateSettings(update);
    emit(AppSettingsLoaded(updated)); // No se toca NumberFormatter.
    return true;
  }

  @override
  Future<void> close() async {
    // Evita que el formatter conserve una referencia al Cubit destruido.
    NumberFormatter.configureCurrencyResolver(null);
    await super.close();
  }
}
```

### 4. Uso en UI, PDFs y tickets

```dart
Text('Total: ${NumberFormatter.convertToMoneyLike(totalCents)}');

final format = NumberFormatter.currencyFormat;
Text(format.format(totalCents / 100));

// Etiquetas que solo necesitan el símbolo:
Text('Precio (${NumberFormatter.currencySymbol})');
```

Todas las capas que usen `NumberFormatter` quedan cubiertas automáticamente: tablas, reportes, exportaciones CSV, PDFs y tickets.

## Por qué no se actualiza el formatter al guardar

El flujo real es:

```text
1. Usuario guarda moneda USD
        │
        ▼
2. Cubit ejecuta emit(AppSettingsLoaded(USD))
        │
        ▼
3. El estado del Cubit ahora contiene USD
        │
        ▼
4. Al formatear un monto, el resolver ejecuta:
   () => cubit.currency
        │
        ▼
5. Devuelve USD y el texto usa "US$"
```

No existe un `NumberFormatter.setCurrency(updated.currency)` porque el valor no se copia al formatter. El formatter pregunta cada vez y el Cubit responde con su estado actual. Esto es **lazy y siempre consistente**.

## Reglas de implementación

1. **Registrar una sola vez:** en el constructor del Cubit global o en el composition root.
2. **Limpiar en `close()`:** `configureCurrencyResolver(null)` para no retener el Cubit.
3. **Nunca importar el Cubit ni `GetIt` dentro del paquete puro:** la función se inyecta desde la capa externa.
4. **Siempre tener un fallback:** evita nulls o valores vacíos antes de cargar la configuración.
5. **Una única fuente de verdad:** el estado del Cubit. No dupliques la moneda en variables estáticas con setter.
6. **Cuidado con múltiples Cubits:** si dos instancias registran resolvers, gana la última. Mantén el Cubit como singleton global o centraliza el registro en un composition root.
7. **Isolates:** el callback vive en el isolate donde se registra. Si generas PDFs o reportes en otro isolate, pasa la moneda explícitamente o configura el resolver en ese isolate.

## Testing

```dart
setUp(() {
  NumberFormatter.configureCurrencyResolver(() => AppCurrency.eur);
});

tearDown(() {
  NumberFormatter.configureCurrencyResolver(null);
});

test('formatea en euros', () {
  expect(NumberFormatter.convertToMoneyLike(123456), '€1,234.56');
});
```

## Checklist para reutilizarlo en otro proyecto

- [ ] Crear el enum/value object con `code`, etiqueta y símbolo.
- [ ] Crear el formatter puro con `typedef Resolver = T Function();` y un resolver estático.
- [ ] Agregar un fallback seguro.
- [ ] Exponer el valor actual como getter en el Cubit global.
- [ ] Registrar `() => getter` en el constructor del Cubit.
- [ ] Limpiar el resolver en `close()`.
- [ ] Reemplazar símbolos/formatos quemados por las utilidades del formatter.
- [ ] Probar todos los valores y el fallback.
- [ ] Si los símbolos no son ASCII (₡, €), verificar que PDFs y tickets tengan una fuente con soporte Unicode.

## Ventajas y trade-offs

| Ventaja | Trade-off |
|---|---|
| Un solo punto de cambio global | Es estado estático mutable |
| No hay setters repartidos por la app | El orden de inicialización importa |
| El paquete puro no depende del frontend | Requiere limpiar el resolver en `close()` |
| Fácil de testear con un resolver falso | En isolates hay que pasarlo explícitamente |
| Funciona para moneda, locale, tema o unidades | No conviene usarlo para estado de alta frecuencia |

## Variantes

El mismo patrón aplica a cualquier configuración global de formato:

```dart
typedef LocaleResolver = String Function();
typedef DateFormatResolver = DateFormat Function();
typedef UnitSystemResolver = UnitSystem Function();
```

Si la configuración crece, en lugar de múltiples callbacks se puede inyectar un único objeto:

```dart
typedef FormatConfigResolver = FormatConfig Function();

class FormatConfig {
  final AppCurrency currency;
  final String locale;
  final DateFormat dateFormat;
}
```

El Cubit registra `() => formatConfig` y el formatter lee todo desde una sola fuente de verdad.
