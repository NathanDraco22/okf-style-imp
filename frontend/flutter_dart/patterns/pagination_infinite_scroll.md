---
type: Pattern
title: Paginación Infinita y Búsqueda en ReadCubit
description: Scroll infinito con ScrollController + umbral, cache de páginas en el cubit, guard de página final y búsqueda que conserva los datos visibles con estado Searching.
tags: [flutter, cubit, paginacion, scroll, busqueda, infinite-scroll, patron]
---

# Paginación Infinita y Búsqueda en ReadCubit

Para listados largos servidos por páginas (offset/limit), el ReadCubit maneja la paginación incremental: carga más datos al acercarse al final del scroll, cachea las páginas ya cargadas y, al buscar, **conserva los datos visibles** mientras llega la respuesta.

> **CLI:** El comando `onion dart-cubit {entity}` genera el ReadCubit base. La paginación se agrega manualmente siguiendo este patrón.

## Estado con variante Searching que conserva datos

```dart
sealed class ReadTerminalProductState {}

final class ReadTerminalProductInitial extends ReadTerminalProductState {}
final class ReadTerminalProductLoading extends ReadTerminalProductState {}

class ReadTerminalProductSuccess extends ReadTerminalProductState {
  final List<ProductInDbInBranch> products;
  ReadTerminalProductSuccess(this.products);
}

/// Búsqueda en curso: mantiene los productos visibles (extiende Success)
final class ReadTerminalProductSearching extends ReadTerminalProductSuccess {
  ReadTerminalProductSearching(super.products);
}

final class ReadTerminalProductError extends ReadTerminalProductState {
  final String message;
  ReadTerminalProductError(this.message);
}
```

## ReadCubit con cache de páginas

```dart
class ReadTerminalProductCubit extends Cubit<ReadTerminalProductState> {
  ReadTerminalProductCubit({
    required this.productsRepository,
    required this.branchId,
  }) : super(ReadTerminalProductInitial());

  final ProductsRepository productsRepository;
  final String branchId;

  bool isLastPage = false;
  bool _isSearching = false;
  List<ProductInDbInBranch> _productsCache = [];

  Future<void> loadProducts() async {
    emit(ReadTerminalProductLoading());
    try {
      final products = await productsRepository.getAllProductsInBranch(
        ProductQueryParams(offset: 0, limit: paginationItems),
        branchId,
      );
      _productsCache = products;
      isLastPage = products.length < paginationItems;
      emit(ReadTerminalProductSuccess(_productsCache));
    } catch (error) {
      emit(ReadTerminalProductError(error.toString()));
    }
  }

  Future<void> getNextPagedProducts() async {
    if (isLastPage || _isSearching) return;
    try {
      final products = await productsRepository.getAllProductsInBranch(
        ProductQueryParams(offset: _productsCache.length, limit: paginationItems),
        branchId,
      );
      if (products.length < paginationItems) isLastPage = true;
      _productsCache = [..._productsCache, ...products];
      emit(ReadTerminalProductSuccess(_productsCache));
    } catch (error) {
      log(error.toString());
    }
  }

  Future<void> searchProducts(String keyword) async {
    if (keyword.trim().isEmpty) {
      _isSearching = false;
      emit(ReadTerminalProductSuccess(_productsCache));  // Restaura cache sin refetch
      return;
    }

    if (state is ReadTerminalProductSuccess) {
      emit(ReadTerminalProductSearching((state as ReadTerminalProductSuccess).products));
    }

    _isSearching = true;
    try {
      final results = await productsRepository.searchProductInBranch(keyword.trim(), branchId);
      emit(ReadTerminalProductSuccess(results));
    } catch (error) {
      emit(ReadTerminalProductError(error.toString()));
    } finally {
      _isSearching = false;
    }
  }
}
```

## Detección de scroll (umbral)

En la sección de la UI, un `ScrollController` dispara la siguiente página cuando falta poco para el final:

```dart
class _ProductsSectionState extends State<_ProductsSection> {
  final ScrollController _scrollController = ScrollController();

  @override
  void initState() {
    super.initState();
    _scrollController.addListener(_onScroll);
  }

  @override
  void dispose() {
    _scrollController.dispose();
    super.dispose();
  }

  void _onScroll() {
    if (_scrollController.position.pixels >=
        _scrollController.position.maxScrollExtent - 200) {
      context.read<ReadTerminalProductCubit>().getNextPagedProducts();
    }
  }
}
```

## UI: indicador discreto, nunca vacía la pantalla

```dart
BlocBuilder<ReadTerminalProductCubit, ReadTerminalProductState>(
  builder: (context, state) {
    return Column(
      children: [
        // Loading inicial o Searching → barra fina, los datos siguen visibles
        if (state is ReadTerminalProductLoading || state is ReadTerminalProductSearching)
          const LinearProgressIndicator(minHeight: 3),
        Expanded(
          child: switch (state) {
            ReadTerminalProductInitial() || ReadTerminalProductLoading() =>
              const Center(child: CircularProgressIndicator()),
            ReadTerminalProductError(:final message) =>
              _ErrorView(message: message, onRetry: () => context.read<...>().loadProducts()),
            ReadTerminalProductSuccess(:final products) =>
              _ProductGrid(products: products, controller: _scrollController),
            ReadTerminalProductSearching() => ...,
          },
        ),
      ],
    );
  },
)
```

## Reglas

- **Guard de página final** — `isLastPage` se marca cuando llegan menos ítems que el limit
- **Guard de concurrencia** — `_isSearching` evita fetchs simultáneos de scroll + búsqueda
- **Cache en el cubit** — `_productsCache` acumula páginas; el offset es `cache.length`
- **Búsqueda vacía restaura la cache** — sin refetch, se re-emite `_productsCache`
- **Searching extiende Success** — los datos visibles nunca se borran mientras se busca
- **Barra de progreso lineal** (`minHeight: 3`) para búsquedas; spinner central solo en carga inicial
- **Los errores de páginas siguientes se loguean** — no se rompe la pantalla (solo `log()`)
- El umbral típico es 200px antes del final del scroll
