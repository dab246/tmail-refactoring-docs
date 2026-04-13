# Solution: BaseController

**Problem:** God Class — 7 responsibility domains, directly imports data layer, every controller inherits 43 deps.

---

## Specific problems

```dart
// Current: BaseController imports data layer — violates Clean Architecture
import 'package:tmail_ui_user/features/login/data/network/interceptors/authorization_interceptors.dart';
import 'package:tmail_ui_user/features/caching/caching_manager.dart'; // data layer

abstract class BaseController extends GetxController {
  final CachingManager cachingManager;           // data layer
  final LanguageCacheManager languageCacheManager; // data layer
  final AuthorizationInterceptors authorizationInterceptors; // data layer
  // ... 11 other deps
}
```

---

## Solution

### Step 1 — Create abstractions in the domain layer

```dart
// lib/features/base/domain/service/i_session_token_refresher.dart
abstract class ISessionTokenRefresher {
  Future<void> refreshIfNeeded();
  Future<void> refreshToken();
}

// lib/features/base/domain/service/i_language_setting_service.dart
abstract class ILanguageSettingService {
  String get currentLanguageTag;
  Future<void> setLanguage(String tag);
}

// lib/features/base/domain/service/i_auth_service.dart
abstract class IAuthService {
  Future<void> logout();
  Future<void> logoutToSignInNewAccount();
  bool get isAuthenticated;
}

// lib/features/base/domain/service/i_notification_service.dart
abstract class INotificationService {
  Future<void> registerFCM();
  Future<void> unregisterFCM();
}
```

### Step 2 — BaseController only accepts abstractions

```dart
// lib/features/base/base_controller.dart
abstract class BaseController extends GetxController {
  // Domain-layer abstractions only — no more data-layer imports
  final ISessionTokenRefresher _tokenRefresher;
  final ILanguageSettingService _languageService;
  final IAuthService _authService;

  // Core UI utilities — acceptable because they are pure utilities
  final AppToast appToast;
  final ImagePaths imagePaths;
  final ResponsiveUtils responsiveUtils;

  BaseController(
    this._tokenRefresher,
    this._languageService,
    this._authService,
    this.appToast,
    this.imagePaths,
    this.responsiveUtils,
  );

  // State management — unchanged
  final viewState = Rx<Either<Failure, Success>>(Right(UIState.idle));

  void consumeState(Stream<Either<Failure, Success>> stream) { ... }
  void dispatchState(Either<Failure, Success> state) { ... }

  // Auth — delegate to IAuthService
  void logout() => _authService.logout();

  // Language — delegate to ILanguageSettingService
  String get currentLanguage => _languageService.currentLanguageTag;
}
```

### Step 3 — Extract FCM/Notification into `NotificationCapabilityMixin`

```dart
// lib/features/base/mixin/notification_capability_mixin.dart
mixin NotificationCapabilityMixin on GetxController {
  INotificationService get notificationService; // abstract getter

  void injectFCMBindings() {
    notificationService.registerFCM();
  }
}

// Controller usage:
class MailboxDashBoardController extends ReloadableController
    with NotificationCapabilityMixin {
  @override
  final INotificationService notificationService;

  MailboxDashBoardController(this.notificationService, ...);
}
```

### Step 4 — Extract Capability Injection into a dedicated service

```dart
// Current: BaseController contains 4 inject methods
// injectAutoCompleteBindings(), injectMdnBindings(),
// injectForwardBindings(), injectRuleFilterBindings()

// Solution: CapabilityInjector service
// lib/features/base/service/capability_injector.dart
class CapabilityInjector {
  void injectAutoComplete(AccountId accountId, Session session) { ... }
  void injectMdn(AccountId accountId, Session session) { ... }
  void injectForward(AccountId accountId, Session session) { ... }
  void injectRuleFilter(AccountId accountId, Session session) { ... }
}

// BaseController receives CapabilityInjector instead of implementing inline
abstract class BaseController extends GetxController {
  final CapabilityInjector _capabilityInjector;

  void injectCapabilities(AccountId accountId, Session session) {
    _capabilityInjector.injectAutoComplete(accountId, session);
    _capabilityInjector.injectMdn(accountId, session);
  }
}
```

### Step 5 — Data-layer implementations registered in CoreBindings

```dart
// lib/main/bindings/core_bindings.dart
class CoreBindings extends Bindings {
  @override
  void dependencies() {
    // Implementations (data layer) — only registered here
    Get.put<ISessionTokenRefresher>(
      AuthorizationInterceptorsAdapter(Get.find<AuthorizationInterceptors>()),
      permanent: true,
    );
    Get.put<ILanguageSettingService>(
      LanguageCacheManagerAdapter(Get.find<LanguageCacheManager>()),
      permanent: true,
    );
    Get.put<IAuthService>(
      AuthServiceImpl(
        Get.find<LogoutOidcInteractor>(),
        Get.find<DeleteCredentialInteractor>(),
      ),
      permanent: true,
    );

    // AppEventBus
    Get.put<AppEventBus>(AppEventBus(), permanent: true);
  }
}
```

---

## Results

| Metric | Before | After |
|---|---|---|
| BaseController deps | 14 | 6 |
| Data-layer imports in presentation | Yes | No |
| Domain abstractions | 0 | 4 interfaces |
| Test setUp (per controller) | ~14 base mocks | ~3-4 base mocks |

---

## Priority: Phase 0 — implement before everything else

This is the foundation. Without extracting BaseController, it is impossible to reduce constructor injection for all other controllers.