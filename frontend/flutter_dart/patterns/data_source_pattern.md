---
type: Pattern
title: DataSource Pattern (Singleton + HttpService)
description: Las DataSources son singletons con constructor privado y factory, usan el mixin HttpService para hacer llamadas HTTP.
tags: [flutter, datasource, singleton, http, patron]
---

# DataSource Pattern

Cada entidad tiene una clase DataSource que encapsula las llamadas HTTP. Es análoga al mismo patrón del backend.

> **CLI:** El comando `onion dart {entity}` genera el DataSource con singleton (factory + constructor privado + mixin HttpService) y métodos CRUD placeholder. El desarrollador define los endpoints y la lógica de llamadas HTTP.

## Estructura

```dart
class ItemDataSource with HttpService {
  ItemDataSource._();
  static final ItemDataSource instance = ItemDataSource._();
  factory ItemDataSource() => instance;

  final _endpoint = "/items";

  Future<Map<String, dynamic>> createItem(Map<String, dynamic> item) async {
    final uri = HttpTools.generateUri(_endpoint);
    final headers = HttpTools.generateAuthHeaders();
    final res = await postQuery(uri, item, headers: headers);
    return res;
  }

  Future<Map<String, dynamic>> getAllItems() async {
    final uri = HttpTools.generateUri(_endpoint);
    final headers = HttpTools.generateAuthHeaders();
    final res = await getQuery(uri, headers: headers);
    return res;
  }

  Future<Map<String, dynamic>> getItemById(String id) async {
    final uri = HttpTools.generateUri("$_endpoint/$id");
    final headers = HttpTools.generateAuthHeaders();
    final res = await getQuery(uri, headers: headers);
    return res;
  }

  Future<Map<String, dynamic>> updateItemById(String id, Map<String, dynamic> data) async {
    final uri = HttpTools.generateUri("$_endpoint/$id");
    final headers = HttpTools.generateAuthHeaders();
    final res = await patchQuery(uri, body: data, headers: headers);
    return res;
  }

  Future<Map<String, dynamic>> deleteItemById(String id) async {
    final uri = HttpTools.generateUri("$_endpoint/$id");
    final headers = HttpTools.generateAuthHeaders();
    final res = await deleteQuery(uri, headers: headers);
    return res;
  }
}
```

## Singleton Pattern

```dart
class ItemDataSource with HttpService {
  ItemDataSource._();          // Constructor privado
  static final ItemDataSource instance = ItemDataSource._();  // Única instancia
  factory ItemDataSource() => instance;  // Factory devuelve siempre la misma
}
```

## HttpTools

```dart
class HttpTools {
  static Uri generateUri(String path, {int version = 1, Map<String, String>? queryParameters}) {
    final baseUrl = KardexClient.instance.config.baseUrl;
    return Uri.parse('$baseUrl/api/v$version$path').replace(queryParameters: queryParameters);
  }

  static Map<String, String> generateAuthHeaders() {
    return {
      'Authorization': 'Bearer ${KardexClient.instance.token}',
      'Content-Type': 'application/json',
      // Custom headers para multi-tenant
      'X-Servin': KardexClient.instance.servin,
    };
  }
}
```

## Reglas

- **Singleton** — constructor privado + factory que retorna la instancia única
- **Mixin HttpService** — hereda get/post/put/patch/delete/multipart
- **Métodos** reciben `Map<String, dynamic>` y devuelven `Future<Map<String, dynamic>>`
- **Sin lógica de negocio** — solo arma URI, headers, y llama al HTTP method
- **Endpoint** como constante privada al inicio de la clase
