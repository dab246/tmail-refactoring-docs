# Solution: BaseController

**Vấn đề:** God Class — 7 responsibility domains, import data layer trực tiếp, mọi controller kế thừa 43 deps.

---

## Vấn đề cụ thể

```dart
// Hiện tại: BaseController import data layer — vi phạm Clean Architecture
import 'package:tmail_ui_user/features/login/data/network/interceptors/authorization_interceptors.dart';
import 'package:tmail_ui_user/features/caching/caching_manager.dart'; // data layer

abstract class BaseController extends GetxController {
  final CachingManager cachingManager;           // data layer
  final LanguageCacheManager languageCacheManager; // data layer
  final AuthorizationInterceptors authorizationInterceptors; // data layer
  // ... 11 deps khác
}
```

---

## Giải pháp

### Bước 1 — Tạo abstractions trong domain layer

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

### Bước 2 — BaseController chỉ nhận abstractions

```dart
// lib/features/base/base_controller.dart
abstract class BaseController extends GetxController {
  // Chỉ domain-layer abstractions — không còn data-layer imports
  final ISessionTokenRefresher _tokenRefresher;
  final ILanguageSettingService _languageService;
  final IAuthService _authService;

  // Core UI utilities — acceptable vì chúng là pure utilities
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

  // State management — giữ nguyên
  final viewState = Rx<Either<Failure, Success>>(Right(UIState.idle));

  void consumeState(Stream<Either<Failure, Success>> stream) { ... }
  void dispatchState(Either<Failure, Success> state) { ... }

  // Auth — delegate sang IAuthService
  void logout() => _authService.logout();

  // Language — delegate sang ILanguageSettingService
  String get currentLanguage => _languageService.currentLanguageTag;
}
```

### Bước 3 — Tách FCM/Notification ra `NotificationCapabilityMixin`

```dart
// lib/features/base/mixin/notification_capability_mixin.dart
mixin NotificationCapabilityMixin on GetxController {
  INotificationService get notificationService; // abstract getter

  void injectFCMBindings() {
    notificationService.registerFCM();
  }
}

// Controller sử dụng:
class MailboxDashBoardController extends ReloadableController
    with NotificationCapabilityMixin {
  @override
  final INotificationService notificationService;

  MailboxDashBoardController(this.notificationService, ...);
}
```

### Bước 4 — Tách Capability Injection ra service riêng

```dart
// Hiện tại: BaseController chứa 4 inject methods
// injectAutoCompleteBindings(), injectMdnBindings(),
// injectForwardBindings(), injectRuleFilterBindings()

// Giải pháp: CapabilityInjector service
// lib/features/base/service/capability_injector.dart
class CapabilityInjector {
  void injectAutoComplete(AccountId accountId, Session session) { ... }
  void injectMdn(AccountId accountId, Session session) { ... }
  void injectForward(AccountId accountId, Session session) { ... }
  void injectRuleFilter(AccountId accountId, Session session) { ... }
}

// BaseController nhận CapabilityInjector thay vì implement inline
abstract class BaseController extends GetxController {
  final CapabilityInjector _capabilityInjector;

  void injectCapabilities(AccountId accountId, Session session) {
    _capabilityInjector.injectAutoComplete(accountId, session);
    _capabilityInjector.injectMdn(accountId, session);
  }
}
```

### Bước 5 — Data-layer implementations đăng ký trong CoreBindings

```dart
// lib/main/bindings/core_bindings.dart
class CoreBindings extends Bindings {
  @override
  void dependencies() {
    // Implementations (data layer) — chỉ đăng ký ở đây
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

## Kết quả

| Metric | Trước | Sau |
|---|---|---|
| BaseController deps | 14 | 6 |
| Data-layer imports trong presentation | Có | Không |
| Domain abstractions | 0 | 4 interfaces |
| Test setUp (mỗi controller) | ~14 base mocks | ~3-4 base mocks |

---

## Priority: Phase 0 — implement trước tất cả

Đây là nền tảng. Không tách được BaseController thì không thể giảm constructor injection cho các controller khác.
