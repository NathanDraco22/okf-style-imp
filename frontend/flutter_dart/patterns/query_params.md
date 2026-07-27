---
type: Pattern
title: Query Params Tipados
description: Objetos tipados que encapsulan parámetros de consulta para filtrar listas desde la API, con método toMap() para serialización.
tags: [flutter, query, params, filtros, patron]
---

# Query Params

Para filtros complejos en endpoints GET, se usan objetos tipados con método `toMap()` que se convierten en query parameters de la URI.

## Implementación

```dart
class CashTransactionFilterParams {
  final String? cashRegisterId;
  final String? type;
  final int? startDate;
  final int? endDate;

  CashTransactionFilterParams({
    this.cashRegisterId,
    this.type,
    this.startDate,
    this.endDate,
  });

  Map<String, String> toMap() {
    final map = <String, String>{};
    if (cashRegisterId != null) map['cashRegisterId'] = cashRegisterId!;
    if (type != null) map['type'] = type!;
    if (startDate != null) map['startDate'] = startDate.toString();
    if (endDate != null) map['endDate'] = endDate.toString();
    return map;
  }
}
```

## Uso en DataSource

```dart
class CashTransactionsDataSource with HttpService {
  Future<Map<String, dynamic>> getAllCashTransactions(
    CashTransactionFilterParams? filter,
  ) async {
    final uri = HttpTools.generateUri(
      "/cash-transactions",
      queryParameters: filter?.toMap(),
    );
    final headers = HttpTools.generateAuthHeaders();
    return await getQuery(uri, headers: headers);
  }
}
```

## Uso en Repository

```dart
Future<List<CashTransactionInDb>> getCashTransactions({
  String? cashRegisterId,
  String? type,
  DateTime? startDate,
  DateTime? endDate,
}) async {
  final filter = CashTransactionFilterParams(
    cashRegisterId: cashRegisterId,
    type: type,
    startDate: startDate?.millisecondsSinceEpoch,
    endDate: endDate?.millisecondsSinceEpoch,
  );
  final result = await dataSource.getAllCashTransactions(filter);
  return ListResponse<CashTransactionInDb>.fromJson(result, CashTransactionInDb.fromJson).data;
}
```

## Reglas

- Cada conjunto de filtros es una clase separada en `domain/query_params/`
- Los campos opcionales son `null` cuando no se aplican
- `toMap()` solo incluye campos no nulos
- Se usa en combinación con `HttpTools.generateUri(path, queryParameters:)`
