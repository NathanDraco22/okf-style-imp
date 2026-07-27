---
type: Pattern
title: Manejo de Excepciones HTTP (Jerarquía Sellada)
description: Jerarquía sellada de excepciones para errores HTTP, con tipos específicos para cada código de estado.
tags: [flutter, http, excepciones, errores, patron]
---

# Manejo de Excepciones HTTP

Las excepciones HTTP usan una jerarquía sellada de clases para representar cada tipo de error.

## Jerarquía

```dart
sealed class HttpServiceException implements Exception {}

final class NoInternetException extends HttpServiceException {
  final String message = 'Sin conexión a internet';
}

final class ServerException extends HttpServiceException {
  final Map<String, dynamic> body;
  ServerException(this.body);
}

final class BadRequestException extends HttpServiceException {
  final Map<String, dynamic> body;
  final int statusCode;
  BadRequestException(this.body, this.statusCode);
}

final class UnauthorizedException extends HttpServiceException {
  final Map<String, dynamic> body;
  UnauthorizedException(this.body);
}

final class ForbiddenException extends HttpServiceException {
  final Map<String, dynamic> body;
  ForbiddenException(this.body);
}

final class InternalRequestException extends HttpServiceException {
  final String message;
  InternalRequestException(this.message);
}
```

## Mapeo en HttpService

```dart
mixin HttpService {
  void _processStatusCode(int statusCode, Map<String, dynamic> body) {
    switch (statusCode) {
      case >= 500:
        throw ServerException(body);
      case 401:
        throw UnauthorizedException(body);
      case 403:
        throw ForbiddenException(body);
      case >= 400:
        throw BadRequestException(body, statusCode);
    }
  }
}
```

## Manejo en Repository

```dart
class ItemRepository {
  Future<List<Item>> getAllItems() async {
    try {
      final result = await dataSource.getAllItems();
      final response = ListResponse<Item>.fromJson(result, Item.fromJson);
      return response.data;
    } on UnauthorizedException {
      // El SessionCubit detectará el 401 y redirigirá a login
      rethrow;
    } on NoInternetException {
      rethrow;
    }
  }
}
```

## Manejo en Cubit

```dart
Future<void> getAll() async {
  emit(ReadItemLoading());
  try {
    final items = await itemRepository.getAllItems();
    emit(ReadItemSuccess(items));
  } on UnauthorizedException {
    emit(ReadItemError('Sesión expirada'));
  } on NoInternetException {
    emit(ReadItemError('Sin conexión a internet'));
  } on ServerException {
    emit(ReadItemError('Error del servidor'));
  } catch (e) {
    emit(ReadItemError('Error inesperado: $e'));
  }
}
```

## Reglas

- Jerarquía **sellada** — todos los casos están cubiertos
- `NoInternetException` se lanza al capturar `SocketException`
- Las excepciones se lanzan desde `HttpService` y se manejan donde corresponda
- `UnauthorizedException` puede escucharse globalmente para hacer logout automático
- Las excepciones se traducen a mensajes de usuario en los Cubits, no en los DataSources
