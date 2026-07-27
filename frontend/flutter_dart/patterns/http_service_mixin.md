---
type: Pattern
title: HttpService Mixin
description: Mixin que provee métodos HTTP (get, post, put, patch, delete, multipart) con manejo de códigos de estado y jerarquía de excepciones.
tags: [flutter, http, mixin, api, patron]
---

# HttpService Mixin

Mixin que cualquier DataSource puede usar para obtener métodos HTTP sin duplicación de código.

## Implementación

```dart
mixin HttpService {
  Future<Map<String, dynamic>> getQuery(Uri url, {Map<String, String>? headers}) async {
    try {
      final response = await http.get(url, headers: headers);
      final body = jsonDecode(utf8.decode(response.bodyBytes));
      _processStatusCode(response.statusCode, body);
      return body;
    } on SocketException {
      throw NoInternetException();
    }
  }

  Future<Map<String, dynamic>> postQuery(
    Uri url, Map<String, dynamic> body, {
    Map<String, String>? headers,
  }) async {
    try {
      final response = await http.post(
        url,
        headers: {'Content-Type': 'application/json', ...?headers},
        body: jsonEncode(body),
      );
      final result = jsonDecode(utf8.decode(response.bodyBytes));
      _processStatusCode(response.statusCode, result);
      return result;
    } on SocketException {
      throw NoInternetException();
    }
  }

  Future<Map<String, dynamic>> putQuery(...) async { /* similar */ }
  Future<Map<String, dynamic>> patchQuery(...) async { /* similar */ }
  Future<Map<String, dynamic>> deleteQuery(...) async { /* similar */ }
  Future<Map<String, dynamic>> multipartQuery(...) async { /* file uploads */ }

  void _processStatusCode(int statusCode, Map<String, dynamic> body) {
    if (statusCode >= 500) throw ServerException(body);
    if (statusCode == 401) throw UnauthorizedException(body);
    if (statusCode == 403) throw ForbiddenException(body);
    if (statusCode >= 400) throw BadRequestException(body, statusCode);
  }
}
```

## Uso

```dart
class ItemDataSource with HttpService {
  Future<Map<String, dynamic>> getAllItems() async {
    final uri = HttpTools.generateUri("/patients");
    return await getQuery(uri, headers: HttpTools.generateAuthHeaders());
  }
}
```

## Excepciones mapeadas

| Código HTTP | Excepción | Acción en UI |
|-------------|-----------|--------------|
| 400+ | `BadRequestException` | Mostrar error de validación |
| 401 | `UnauthorizedException` | Redirigir a login |
| 403 | `ForbiddenException` | Mostrar sin permisos |
| 500+ | `ServerException` | Mostrar error del servidor |
| Sin conexión | `NoInternetException` | Mostrar sin internet |
