---
type: Pattern
title: Search Cubit
description: Cubit separado para búsqueda por texto con debounce, filtra la lista del ReadCubit según el query de búsqueda.
tags: [flutter, cubit, busqueda, search, patron]
---

# Search Cubit

Las entidades que requieren búsqueda tienen un tercer cubit dedicado. Trabaja junto al ReadCubit: el ReadCubit mantiene la lista completa, el SearchCubit filtra sobre ella.

## Estructura

```dart
sealed class SearchItemState {}
final class SearchItemInitial extends SearchItemState {}
final class SearchItemSuccess extends SearchItemState {
  final List<Item> items;
  SearchItemSuccess(this.items);
}

class SearchItemCubit extends Cubit<SearchItemState> {
  final ItemRepository itemRepository;

  SearchItemCubit({required this.itemRepository})
    : super(SearchItemInitial());

  Future<void> search(String query) async {
    if (query.isEmpty) {
      emit(SearchItemInitial());
      return;
    }
    try {
      final results = await itemRepository.searchItems(query);
      emit(SearchItemSuccess(results));
    } catch (e) {
      emit(SearchItemInitial());
    }
  }
}
```

## Uso en UI

```dart
Column(
  children: [
    TextField(
      decoration: InputDecoration(labelText: 'Buscar...'),
      onChanged: (query) => context.read<SearchItemCubit>().search(query),
    ),
    Expanded(
      child: BlocBuilder<SearchItemCubit, SearchItemState>(
        builder: (context, state) {
          return switch (state) {
            SearchItemInitial() => _ReadItemList(),
            SearchItemSuccess() => _ItemsList(items: state.items),
          };
        },
      ),
    ),
  ],
)
```

## Registro en Screen

```dart
MultiBlocProvider(
  providers: [
    BlocProvider(create: (ctx) => ReadItemCubit(repo: repo)..getAll()),
    BlocProvider(create: (ctx) => WriteItemCubit(repo: repo)),
    BlocProvider(create: (ctx) => SearchItemCubit(repo: repo)),
  ],
  child: const _RootScaffold(),
)
```

## Reglas

- SearchCubit es **independiente** del ReadCubit (no lo modifica)
- Entra en escena **solo cuando hay query** — si está vacío, muestra el ReadCubit
- No tiene estado Loading (la búsqueda debe ser rápida)
- El backend usa `$text` index + aggregation (ver [search_pattern](../../../backend/python_fastapi/patterns/search_pattern.md))
