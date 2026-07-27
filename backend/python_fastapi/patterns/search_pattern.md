---
type: Pattern
title: Búsqueda por Texto (searchPrefixes + $text index)
description: Sistema de búsqueda con searchPrefixes, índices de texto MongoDB y aggregation pipeline con scoring.
tags: [mongodb, search, texto, indice, patron]
---

# Búsqueda por Texto

Patrón para búsqueda eficiente en entidades como productos, pacientes, doctores, exámenes.

## searchPrefixes

Se genera un campo `searchPrefixes` con todas las combinaciones de prefijos del nombre:

```python
def generate_prefixes(text: str) -> list[str]:
    words = text.lower().split()
    prefixes = set()
    for word in words:
        for i in range(1, len(word) + 1):
            prefixes.add(word[:i])
    return list(prefixes)
```

## Índice de texto

Se crea un índice `$text` compuesto en la collection:

```python
await self._collection.create_index(
    [("name", "text"), ("searchPrefixes", "text")],
    weights={"name": 10, "searchPrefixes": 5},
    default_language="spanish",
)
```

## Query con aggregation

```python
async def search(self, query: str) -> list[dict[str, Any]]:
    pipeline = [
        {"$match": {"$text": {"$search": query}}},
        {"$addFields": {"score": {"$meta": "textScore"}}},
        {"$sort": {"score": -1}},
        {"$match": {"isDeleted": {"$ne": True}}},
    ]
    cursor = self.__collection.aggregate(pipeline)
    result = await cursor.to_list(length=None)
    await cursor.close()
    return result
```

## Reglas

- `searchPrefixes` se genera y almacena en cada `create`/`update`
- El `$text` index es compuesto: mayor peso en `name` que en `searchPrefixes`
- Se usa `default_language="spanish"` para stem español
- El aggregation pipeline añade `textScore` y ordena por relevancia
