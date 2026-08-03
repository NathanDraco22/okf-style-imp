---
type: Pattern
title: GoRouter (AppRouter + StatefulShellRoute + Session Redirect)
description: Configuración de go_router con clase AppRouter estática, StatefulShellRoute.indexedStack y redirects basados en AppSessionCubit.
tags: [flutter, router, go_router, navegacion, patron]
---

# GoRouter

Se usa `go_router` con una clase `AppRouter` que expone rutas como constantes estáticas y un factory `createRouter(AppSessionCubit)`.

## AppRouter

```dart
class AppRouter {
  AppRouter._();

  // Constantes de rutas
  static const String login = '/login';
  static const String selectCashRegister = '/select-cash-register';
  static const String dashboard = '/';
  static const String items = '/items';
  static const String itemsList = '/items-list';
  static const String itemDetail = '/items/{id}';
  static const String patients = '/patients';
  static const String doctors = '/doctors';
  static const String orders = '/orders';

  // Navigator keys para cada branch
  static final _rootNavigatorKey = GlobalKey<NavigatorState>();
  static final _dashboardNavKey = GlobalKey<NavigatorState>(debugLabel: 'dashboard');
  static final _itemsNavKey = GlobalKey<NavigatorState>(debugLabel: 'items');
  static final _itemsListNavKey = GlobalKey<NavigatorState>(debugLabel: 'itemsList');
  static final _itemDetailNavKey = GlobalKey<NavigatorState>(debugLabel: 'itemDetail');
  static final _patientsNavKey = GlobalKey<NavigatorState>(debugLabel: 'patients');
  static final _doctorsNavKey = GlobalKey<NavigatorState>(debugLabel: 'doctors');
  static final _ordersNavKey = GlobalKey<NavigatorState>(debugLabel: 'orders');

  static GoRouter createRouter(AppSessionCubit sessionCubit) {
    return GoRouter(
      navigatorKey: _rootNavigatorKey,
      initialLocation: dashboard,
      refreshListenable: GoRouterRefreshStream(sessionCubit.stream),
      redirect: (context, state) {
        final sessionState = sessionCubit.state;
        final isAuthenticated = sessionState.isAuthenticated;
        final hasCashRegister = sessionState.hasCashRegister;
        final isLoggingIn = state.matchedLocation == login;
        final isSelectingCash = state.matchedLocation == selectCashRegister;

        // 1. No autenticado -> Redirigir a /login
        if (!isAuthenticated) {
          return isLoggingIn ? null : login;
        }

        // 2. Autenticado pero sin caja seleccionada -> Redirigir a /select-cash-register
        if (!hasCashRegister) {
          return isSelectingCash ? null : selectCashRegister;
        }

        // 3. Autenticado con caja -> Si intenta estar en /login o /select-cash-register, ir a /
        if (isLoggingIn || isSelectingCash) {
          return dashboard;
        }

        return null;
      },
      routes: [
        GoRoute(
          path: login,
          builder: (context, state) => const LoginScreen(),
        ),
        GoRoute(
          path: selectCashRegister,
          builder: (context, state) => const SelectCashRegisterScreen(),
        ),
        StatefulShellRoute.indexedStack(
          builder: (context, state, navigationShell) {
            return HomeMenusScreen(navigationShell: navigationShell);
          },
          branches: [
            StatefulShellBranch(
              navigatorKey: _dashboardNavKey,
              routes: [
                GoRoute(path: dashboard, builder: (_, __) => const DashboardScreen()),
              ],
            ),
            StatefulShellBranch(
              navigatorKey: _patientsNavKey,
              routes: [
                GoRoute(path: patients, builder: (_, __) => const PatientsScreen()),
              ],
            ),
            StatefulShellBranch(
              navigatorKey: _doctorsNavKey,
              routes: [
                GoRoute(path: doctors, builder: (_, __) => const DoctorsScreen()),
              ],
            ),
            StatefulShellBranch(
              navigatorKey: _ordersNavKey,
              routes: [
                GoRoute(path: orders, builder: (_, __) => const OrdersScreen()),
              ],
            ),
          ],
        ),
      ],
    );
  }
}
```

## GoRouterRefreshStream

```dart
class GoRouterRefreshStream extends ChangeNotifier {
  late final StreamSubscription<dynamic> _subscription;

  GoRouterRefreshStream(Stream<dynamic> stream) {
    notifyListeners();
    _subscription = stream.asBroadcastStream().listen((_) => notifyListeners());
  }

  @override
  void dispose() {
    _subscription.cancel();
    super.dispose();
  }
}
```

## AppSessionCubit

```dart
class AppSessionState {
  final UserInDb? currentUser;
  final CashRegisterInDb? activeCashRegister;

  const AppSessionState({this.currentUser, this.activeCashRegister});

  bool get isAuthenticated => currentUser != null;
  bool get hasCashRegister => activeCashRegister != null;
}

class AppSessionCubit extends Cubit<AppSessionState> {
  AppSessionCubit() : super(const AppSessionState());

  void login(UserInDb user) => emit(state.copyWith(currentUser: user));
  void logout() => emit(const AppSessionState());
  void selectCashRegister(CashRegisterInDb cr) => emit(state.copyWith(activeCashRegister: cr));
}
```

## Uso en main.dart

```dart
final sessionCubit = AppSessionCubit();
final router = AppRouter.createRouter(sessionCubit);
```

## Reglas

- **AppRouter** es una clase estática (constructor privado + métodos static)
- **refreshListenable** usa `GoRouterRefreshStream` conectado al stream del `AppSessionCubit`
- **redirect** evalúa `isAuthenticated` y `hasCashRegister` del `AppSessionState`
- Dos rutas fuera del StatefulShellRoute: `/login` y `/select-cash-register`
- Cada `StatefulShellBranch` tiene su propio `navigatorKey` para mantener estado persistente al cambiar de pestaña
