---
type: Pattern
title: ListResponse Genérico (Flutter)
description: Wrapper genérico para respuestas paginadas de la API, con count, data y un factory fromJson que recibe un converter.
tags: [flutter, response, list, generico, patron]
---

# ListResponse

Respuesta estándar para endpoints que devuelven listas, mismo patrón que en el backend.

## Implementación

```dart
class ListResponse<T> {
  final List<T> data;
  final int count;

  ListResponse({required this.data, required this.count});

  factory ListResponse.fromJson(
    Map<String, dynamic> json,
    T Function(Map<String, dynamic>) fromJsonT,
  ) {
    final dataList = (json['data'] as List<dynamic>)
        .map((e) => fromJsonT(e as Map<String, dynamic>))
        .toList();
    return ListResponse(
      count: json['count'] as int,
      data: dataList,
    );
  }
}
```

## Uso en Repository

```dart
Future<List<Item>> getAllItems() async {
  final result = await dataSource.getAllItems();
  final response = ListResponse<Item>.fromJson(
    result,
    Item.fromJson,
  );
  return response.data;
}
```
