---
type: Pattern
title: GoRouter (AppRouter + StatefulShellRoute + Stream Redirect + Location Preservation)
description: Configuración de go_router con clase AppRouter estática, StatefulShellRoute.indexedStack, redirección reactiva a streams multi-estado (GoRouterRefreshStream), preservación de ubicación operativa (lastKnownLocation) y subrutas a pantalla completa (parentNavigatorKey).
tags: [flutter, router, go_router, navegacion, reactivo, stream, patron]
---

# GoRouter Reactivo

Configuración integral de `go_router` que combina una clase estática `AppRouter`, un shell de navegación persistente (`StatefulShellRoute.indexedStack`), **reacción atómica a streams de sesión global** (`GoRouterRefreshStream`), **preservación de ubicación operativa** ante cambios de contexto y **subrutas a pantalla completa**.

## Flujo del Pipeline de Redirección

El router escucha de forma reactiva el stream de `AppSessionCubit`. Ante cualquier emisión de estado global (expiración de token, bloqueo remoto, conmutación de espacio de trabajo o logout), GoRouter reevalúa su redirección sin recrear la instancia del router ni recargar la aplicación:

```
[Inicio de App] ──> /splash (Inicialización de persistencia/tokens)
                         │
                         ├── Terminal/Instancia sin configurar ──> /onboarding
                         │
                         ├── Cuenta/Tenant suspendido o inactivo ─> /inactive
                         │
                         ├── No autenticado ──────────────────────> /login
                         │
                         └── Autenticado ─────────────────────────> / (Dashboard)
                                                                       │
                                                       (Ruta preservada vía lastKnownLocation)
```

## Implementación de `AppRouter`

```dart
import 'dart:async';
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

class AppRouter {
  AppRouter._();

  // Constantes de rutas públicas y de ciclo de vida
  static const String splash = '/splash';
  static const String onboarding = '/onboarding';
  static const String inactive = '/inactive';
  static const String login = '/login';

  // Rutas operativas del Shell
  static const String dashboard = '/';
  static const String items = '/items';
  static const String reports = '/reports';
  static const String settings = '/settings';

  // Subrutas a pantalla completa (fuera del Shell visual)
  static const String itemCreate = '/items/new';
  static const String itemDetail = '/items/:id';

  // Navigator keys
  static final _rootNavigatorKey = GlobalKey<NavigatorState>();
  static final _dashboardNavKey = GlobalKey<NavigatorState>(debugLabel: 'dashboard');
  static final _itemsNavKey = GlobalKey<NavigatorState>(debugLabel: 'items');
  static final _reportsNavKey = GlobalKey<NavigatorState>(debugLabel: 'reports');
  static final _settingsNavKey = GlobalKey<NavigatorState>(debugLabel: 'settings');

  // Memoria de última ubicación para cambios de contexto global
  static String? _lastKnownLocation;
  static String? get lastKnownLocation => _lastKnownLocation;
  static void setLastKnownLocation(String? location) => _lastKnownLocation = location;

  static GoRouter createRouter(
    AppSessionCubit sessionCubit, {
    String? initialLocation,
  }) {
    return GoRouter(
      navigatorKey: _rootNavigatorKey,
      initialLocation: initialLocation ?? splash,
      refreshListenable: GoRouterRefreshStream(sessionCubit.stream),
      redirect: (context, state) {
        final sessionState = sessionCubit.state;
        final currentLoc = state.matchedLocation;

        final isSplash = currentLoc == splash;
        final isOnboarding = currentLoc == onboarding;
        final isInactive = currentLoc == inactive;
        final isLogin = currentLoc == login;

        // 1. Inicialización en progreso -> Redirigir a /splash
        if (sessionState.isInitial) {
          return isSplash ? null : splash;
        }

        // 2. Terminal o dispositivo no configurado -> Redirigir a /onboarding
        if (sessionState.isOnboardingRequired) {
          return isOnboarding ? null : onboarding;
        }

        // 3. Suspensión remota o cuenta inactiva -> Redirigir a /inactive
        if (sessionState.isInstanceInactive) {
          return isInactive ? null : inactive;
        }

        // 4. No autenticado -> Redirigir a /login y limpiar memoria de ruta
        if (!sessionState.isAuthenticated) {
          _lastKnownLocation = null;
          return isLogin ? null : login;
        }

        // 5. Preservar la última ubicación operativa navegada
        if (!isSplash && !isLogin && !isOnboarding && !isInactive) {
          _lastKnownLocation = currentLoc;
        }

        // 6. Autenticado: Si intenta entrar a pantallas de acceso, dirigir a ubicación previa o Dashboard
        if (isSplash || isLogin || isOnboarding || isInactive) {
          return _lastKnownLocation ?? dashboard;
        }

        return null;
      },
      routes: [
        // Rutas de ciclo de vida (fuera del Shell)
        GoRoute(
          path: splash,
          builder: (context, state) => const SplashScreen(),
        ),
        GoRoute(
          path: onboarding,
          builder: (context, state) => const OnboardingScreen(),
        ),
        GoRoute(
          path: inactive,
          builder: (context, state) => const InactiveScreen(),
        ),
        GoRoute(
          path: login,
          builder: (context, state) => const LoginScreen(),
        ),

        // Subrutas a pantalla completa (cubren el Shell usando parentNavigatorKey)
        GoRoute(
          path: itemCreate,
          parentNavigatorKey: _rootNavigatorKey,
          builder: (context, state) => const ItemCreateScreen(),
        ),
        GoRoute(
          path: itemDetail,
          parentNavigatorKey: _rootNavigatorKey,
          builder: (context, state) {
            final id = state.pathParameters['id'] ?? '';
            return ItemDetailScreen(itemId: id);
          },
        ),

        // Shell principal con navegación por pestañas indexadas
        StatefulShellRoute.indexedStack(
          builder: (context, state, navigationShell) {
            return MainShellLayout(navigationShell: navigationShell);
          },
          branches: [
            StatefulShellBranch(
              navigatorKey: _dashboardNavKey,
              routes: [
                GoRoute(path: dashboard, builder: (_, __) => const DashboardScreen()),
              ],
            ),
            StatefulShellBranch(
              navigatorKey: _itemsNavKey,
              routes: [
                GoRoute(path: items, builder: (_, __) => const ItemsScreen()),
              ],
            ),
            StatefulShellBranch(
              navigatorKey: _reportsNavKey,
              routes: [
                GoRoute(path: reports, builder: (_, __) => const ReportsScreen()),
              ],
            ),
            StatefulShellBranch(
              navigatorKey: _settingsNavKey,
              routes: [
                GoRoute(path: settings, builder: (_, __) => const SettingsScreen()),
              ],
            ),
          ],
        ),
      ],
    );
  }
}
```

## `GoRouterRefreshStream`

Convierte cualquier `Stream<T>` de Dart en un `ChangeNotifier` compatible con `refreshListenable` de GoRouter. Cuando el stream emite un evento, notifica a los listeners de GoRouter para re-evaluar la función `redirect`:

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

## Modelo de Estado y `AppSessionCubit`

El cubit expone el estado de sesión global y banderas calculadas para simplificar los checks del router:

```dart
enum AppSessionStatus {
  initial,
  onboardingRequired,
  instanceInactive,
  unauthenticated,
  authenticated,
}

class AppSessionState {
  final AppSessionStatus status;
  final UserInDb? currentUser;
  final WorkspaceInDb? currentWorkspace;

  const AppSessionState({
    this.status = AppSessionStatus.initial,
    this.currentUser,
    this.currentWorkspace,
  });

  bool get isInitial => status == AppSessionStatus.initial;
  bool get isOnboardingRequired => status == AppSessionStatus.onboardingRequired;
  bool get isInstanceInactive => status == AppSessionStatus.instanceInactive;
  bool get isAuthenticated => status == AppSessionStatus.authenticated && currentUser != null;

  AppSessionState copyWith({
    AppSessionStatus? status,
    UserInDb? currentUser,
    WorkspaceInDb? currentWorkspace,
  }) {
    return AppSessionState(
      status: status ?? this.status,
      currentUser: currentUser ?? this.currentUser,
      currentWorkspace: currentWorkspace ?? this.currentWorkspace,
    );
  }
}

class AppSessionCubit extends Cubit<AppSessionState> {
  AppSessionCubit() : super(const AppSessionState());

  void setAuthenticated(UserInDb user) => emit(state.copyWith(
    status: AppSessionStatus.authenticated,
    currentUser: user,
  ));

  void logout() => emit(const AppSessionState(status: AppSessionStatus.unauthenticated));

  void setInstanceBlocked() => emit(state.copyWith(status: AppSessionStatus.instanceInactive));
}
```

## Uso en `main.dart`

```dart
void main() {
  final sessionCubit = AppSessionCubit();
  final router = AppRouter.createRouter(sessionCubit);

  runApp(
    BlocProvider.value(
      value: sessionCubit,
      child: MaterialApp.router(
        routerConfig: router,
      ),
    ),
  );
}
```

## Ventajas Clave

| Característica | Beneficio Arquitectónico |
|---|---|
| **Reacción Atómica por Streams** | Si la sesión caduca o el tenant se desactiva, GoRouter reacciona inmediatamente sin requerir `context.go()` manuales dispersos por la app. |
| **`lastKnownLocation`** | Al conmutar de sucursal/organización o reconectar red, el usuario permanece exactamente en la pantalla donde estaba trabajando. |
| **`parentNavigatorKey: rootNavigatorKey`** | Formularios complejos o wizards de creación ocupan el 100% de la pantalla sin la barra lateral/inferior del shell, manteniendo URLs limpias y botón Atrás nativo. |
| **StatefulShellRoute** | Cada pestaña (`StatefulShellBranch`) mantiene su pila de navegación y scroll de forma independiente al alternar entre menús. |

## Reglas

- **AppRouter es estática:** Constructor privado `AppRouter._()` y métodos/constantes de rutas estáticas.
- **GoRouterRefreshStream siempre con cancelación:** Asegurar que el listener del stream se cancele al llamar `dispose()`.
- **Preservación pasiva de ruta:** Actualizar `_lastKnownLocation` en cada navegación exitosa dentro de áreas operativas; resetear a `null` al desautenticar.
- **Rutas Overlay completas:** Toda ruta de creación profunda o modal a pantalla completa debe usar `parentNavigatorKey: _rootNavigatorKey`.
- **Rutas fuera del Shell:** Las pantallas de ciclo de vida (`splash`, `onboarding`, `inactive`, `login`) deben registrarse como rutas hermanas directas en `routes: [...]`, fuera del `StatefulShellRoute`.

## Relacionados

- [Cross-Cubit Event Sync](cross_cubit_event_sync.md) — sincronización de estado global mediante streams de repositorios
- [Flujo Onion en Flutter](flutter_onion_flow.md) — Screen → Cubit → Repository → DataSource → API
- [Patrón Dialog/Modal](dialog_show_functions.md) — show functions para vistas emergentes fuera del router
