---
type: Pattern
title: Patrón de Modelos Flutter (Create/Update/InDb)
description: Triple patrón de modelos con CreateEntity, UpdateEntity y EntityInDb, con fromJson/toJson para serialización.
tags: [flutter, dart, modelos, json, patron]
---

# Patrón de Modelos Flutter

Cada entidad define 3 clases de modelo más una base compartida.

> **CLI:** El comando `onion dart-model {entity}` genera las 4 clases (Base, Create, Update, InDb) con `fromJson`/`toJson` vacíos. El desarrollador completa los campos reales de la entidad.

## Estructura

```dart
class BaseItem {
  final String name;
  final String description;
  final double price;

  BaseItem({
    required this.name,
    required this.description,
    required this.price,
  });
}

class CreateItem extends BaseItem {
  CreateItem({
    required super.name,
    required super.description,
    required super.price,
  });

  Map<String, dynamic> toJson() => {
    'name': name,
    'description': description,
    'price': price,
  };
}

class UpdateItem {
  final String? name;
  final String? description;
  final double? price;

  Map<String, dynamic> toJson() {
    final map = <String, dynamic>{};
    if (name != null) map['name'] = name;
    if (description != null) map['description'] = description;
    if (price != null) map['price'] = price;
    return map;
  }
}

class Item extends BaseItem {
  final String id;
  final int createdAt;
  final int? updatedAt;
  final bool isDeleted;

  Item({
    required this.id,
    required this.createdAt,
    this.updatedAt,
    this.isDeleted = false,
    required super.name,
    required super.description,
    required super.price,
  });

  factory Item.fromJson(Map<String, dynamic> json) => Item(
    id: json['id'],
    createdAt: json['createdAt'],
    updatedAt: json['updatedAt'],
    isDeleted: json['isDeleted'] ?? false,
    name: json['name'],
    description: json['description'],
    price: (json['price'] as num).toDouble(),
  );
}
```

## Convenciones

| Clase | Propósito | Serialización |
|-------|-----------|---------------|
| `Create{Entity}` | Creación (POST) | `toJson()` mapea todos los campos |
| `Update{Entity}` | Actualización (PATCH) | `toJson()` omite nulls para merge parcial |
| `{Entity}InDb` | Respuesta de API | `fromJson()` factory + `toJson()` completo |

## Fechas de dominio y epoch negativo

`createdAt` / `updatedAt` son marcas de auditoría (siempre positivas), pero una fecha de dominio como `birthDate` puede ser anterior a 1970 y, por tanto, negativa:

```dart
// 1969-05-13T00:00:00Z
const birthDate = -20131200000;
DateTime.fromMillisecondsSinceEpoch(birthDate, isUtc: true); // válido
```

- `DateTime.fromMillisecondsSinceEpoch` acepta valores negativos; su rango es de ±100 000 000 días respecto al epoch (≈ ±275 000 años).
- En el selector de fecha usar `firstDate: DateTime(1900)` (o anterior) y `lastDate: DateTime.now()`.
- En `fromJson`, no validar `json['birthDate'] > 0`; solo comprobar que no sea `null`.
- No usar `abs()` ni `toUnsigned()` al serializar una fecha de dominio.
