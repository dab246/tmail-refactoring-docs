# Kế hoạch Migration: GetX → Riverpod

> **Phạm vi:** Chuyển đổi toàn bộ state management của tmail-flutter từ GetX (GetxController + Bindings + Get.find) sang Riverpod (Notifier + Provider).  
> **Chiến lược:** Strangler Fig — thay dần từng feature, không rewrite đồng loạt.  
> **Nền tảng đã có:** `flutter_riverpod ^2.6.1` đã trong pubspec; `ProviderContainer` singleton đã tồn tại; 3 provider module đã hoạt động.

---

## 1. Tại sao migrate sang Riverpod — và tại sao phải là BÂY GIỜ?

### 1.1 Bằng chứng thực tế từ chính codebase

Dự án **đã tự mình bắt đầu migration** mà không có ADR chính thức:

```
Hiện trạng (tháng 5/2026):
═════════════════════════════════════════════════════════════
lib/
  main/providers/
    app_provider_container.dart      ← ProviderContainer singleton
  features/
    composer/presentation/providers/
      composer_cache_providers.dart  ← 5 Provider<T> (ADR-0086)
      composer_auto_save_notifier.dart ← StateNotifierProvider.family
    manage_account/presentation/providers/
      local_settings_notifier.dart   ← StateNotifierProvider
    network_connection/presentation/providers/
      (đang mở rộng)

pubspec.yaml:
  flutter_riverpod: ^2.6.1          ← đã có dependency
```

**Tín hiệu rõ ràng:** Team đã chọn Riverpod cho các feature mới nhất (ADR-0086 — composer auto-save, local settings). GetX chỉ còn được dùng ở code cũ. Nếu không có migration plan, codebase sẽ rơi vào trạng thái 2 framework song song vô thời hạn — đây là worst case.

### 1.2 Vấn đề kỹ thuật cần giải quyết

```
Vấn đề 1 — God Object không thể phá vỡ với GetX
─────────────────────────────────────────────────
MailboxDashBoardController: 3,528 lines
  42 Rx<T> fields — tất cả luôn active
  43 injected dependencies — Get.find() ở khắp nơi
  31 extension files — logic phân tán không kiểm soát được

→ Riverpod: mỗi Notifier focus 1 domain, compiler bắt lỗi missing state

Vấn đề 2 — Get.find() global service locator
─────────────────────────────────────────────
Get.find<MailboxDashBoardController>() — 75+ lần trong codebase
  → Không compile-time safe
  → Crash runtime nếu controller chưa registered
  → Impossible to test in isolation

→ Riverpod: ref.watch(provider) — compile-time, lazy, auto-dispose

Vấn đề 3 — Bindings class proliferation
──────────────────────────────────────
78 Binding files — mỗi file chỉ có boilerplate Get.put/lazyPut
  → Không có type safety cho missing dependencies
  → Thứ tự initialization ẩn, dễ crash

→ Riverpod: Provider graph tự giải quyết dependency order

Vấn đề 4 — Testing quá đắt
────────────────────────────
Test ThreadController hiện tại cần mock 43 deps của DashboardController
(vì ThreadController làm Get.find<DashboardController>() để gọi methods)

→ Riverpod: override provider trong test, inject mock deps cleanly
```

### 1.3 Lý do chọn Riverpod (không phải Provider, BLoC, etc.)

```
Tiêu chí              Riverpod   Provider   BLoC    Redux   Keep GetX
──────────────────────────────────────────────────────────────────────
Đã có trong codebase  ✅         ❌         ❌      ❌      ✅ (current)
Compile-time safety   ✅         ⚠          ✅      ❌      ❌
Auto-dispose          ✅         ❌         ⚠       ❌      ⚠
Family providers      ✅         ❌         ❌      ❌      ❌
Testability           ✅         ✅         ✅      ✅      ❌
Flutter-independent   ✅         ❌         ❌      ✅      ❌
Learning curve        Trung bình Thấp       Cao     Cao     ✅ (đã biết)
Async state built-in  ✅         ❌         ✅      ❌      ⚠
Code generation opt.  ✅         ❌         ❌      ❌      ❌
──────────────────────────────────────────────────────────────────────
Team đã chọn Riverpod tại ADR-0086. Đây là kết luận nhất quán.
```

---

## 2. Hiện trạng GetX — Những gì cần chuyển đổi

### 2.1 Phân loại các pattern GetX trong codebase

```
Category A — GetxController lifecycle
══════════════════════════════════════
62 controller files: extends GetxController / ReloadableController / BaseController
  → Cần chuyển thành: Notifier<S> / AsyncNotifier<S>
  → Lifecycle onInit/onReady/onClose → ref.onDispose() / WidgetRef hooks

Category B — Reactive state (Rx<T>)
══════════════════════════════════════
~400+ reactive fields trên toàn codebase
  Rxn<PresentationMailbox>     → StateProvider<PresentationMailbox?>
  RxBool(false)                → StateProvider<bool>
  RxList<PresentationEmail>    → StateProvider<List<PresentationEmail>>
  Rx<Either<Failure,Success>>  → AsyncNotifierProvider / AsyncValue<T>
  .obs                         → StateProvider / NotifierProvider

Category C — Dependency Injection (Bindings + Get.find)
══════════════════════════════════════════════════════════
78 binding files + 500+ Get.find() calls
  Get.put<T>(impl)             → Provider<T>((ref) => impl)
  Get.lazyPut<T>(() => impl)   → Provider<T>((ref) => impl) [lazy by default]
  Get.find<T>()                → ref.read(tProvider)
  GetxService (permanent)      → keepAlive: true

Category D — Reactive effects (ever)
══════════════════════════════════════
~50 ever() calls trên codebase
  ever(rxField, callback)      → ref.listen(provider, (prev, next) => callback)
  Obx(() => widget)            → Consumer(builder: (ctx, ref, _) => widget)
  GetBuilder<C>                → Consumer / ref.watch()

Category E — Navigation (GetX routing)
══════════════════════════════════════
Get.to / Get.off / Get.toNamed → giữ nguyên trong phase 1-4
  → Phase 5 (optional): migrate sang go_router
  → Đây là việc TÁCH RIÊNG với state management migration
```

### 2.2 Bản đồ mapping GetX → Riverpod

```dart
// ═══════════════════════════════════════════════════════════════
// MAPPING REFERENCE: GetX → Riverpod
// ═══════════════════════════════════════════════════════════════

// --- REACTIVE STATE ---

// GetX:
final selectedMailbox = Rxn<PresentationMailbox>();
// Riverpod:
final selectedMailboxProvider = StateProvider<PresentationMailbox?>((ref) => null);

// GetX:
final isDrawerOpened = RxBool(false);
// Riverpod:
final isDrawerOpenedProvider = StateProvider<bool>((ref) => false);

// GetX:
final listEmailSelected = <PresentationEmail>[].obs;
// Riverpod:
final listEmailSelectedProvider = StateProvider<List<PresentationEmail>>((ref) => []);

// GetX:
final viewState = Rx<Either<Failure, Success>>(Right(UIState.idle));
// Riverpod (dùng AsyncValue — built-in):
// → Dùng AsyncNotifierProvider thay vì manual Either

// --- DEPENDENCY INJECTION ---

// GetX:
class SomeBindings extends Bindings {
  @override
  void dependencies() {
    Get.lazyPut(() => SomeRepository(Get.find<SomeDatasource>()));
    Get.lazyPut(() => SomeInteractor(Get.find<SomeRepository>()));
  }
}
// Riverpod:
final someRepositoryProvider = Provider<SomeRepository>(
  (ref) => SomeRepository(ref.read(someDatasourceProvider)),
);
final someInteractorProvider = Provider<SomeInteractor>(
  (ref) => SomeInteractor(ref.read(someRepositoryProvider)),
);

// --- EFFECTS (ever → listen) ---

// GetX:
ever(selectedMailbox, (mailbox) {
  if (mailbox != null) _loadEmails(mailbox);
});
// Riverpod:
ref.listen<PresentationMailbox?>(
  selectedMailboxProvider,
  (prev, next) {
    if (next != null) _loadEmails(next);
  },
);

// --- CONTROLLERS → NOTIFIERS ---

// GetX:
class ThreadController extends GetxController {
  final emailsInThread = <PresentationEmail>[].obs;
  void loadThread(MailboxId id) { ... }
}

// Riverpod:
class ThreadState {
  final List<PresentationEmail> emails;
  final AsyncValue<void> loadStatus;
  const ThreadState({this.emails = const [], this.loadStatus = const AsyncData(null)});
  ThreadState copyWith({...}) => ...;
}

class ThreadNotifier extends AsyncNotifier<ThreadState> {
  @override
  Future<ThreadState> build() async => const ThreadState();

  Future<void> loadThread(MailboxId id) async {
    state = const AsyncLoading();
    final result = await ref.read(getEmailsInteractorProvider).execute(id);
    state = result.fold(
      (failure) => AsyncError(failure, StackTrace.current),
      (success) => AsyncData(ThreadState(emails: success.emails)),
    );
  }
}

final threadNotifierProvider = AsyncNotifierProvider<ThreadNotifier, ThreadState>(
  ThreadNotifier.new,
);
```

---

## 3. Kiến trúc đích — Target Architecture

### 3.1 Cấu trúc provider tree

```
ProviderScope (main.dart)
│
├─ Infrastructure Layer (keepAlive: true)
│   ├─ authInterceptorProvider         (AuthorizationInterceptors)
│   ├─ dynamicUrlInterceptorProvider   (DynamicUrlInterceptors)
│   ├─ cachingManagerProvider          (CachingManager)
│   ├─ languageCacheManagerProvider    (LanguageCacheManager)
│   └─ dioProvider                     (Dio)
│
├─ Repository Layer (auto-dispose khi không cần)
│   ├─ emailRepositoryProvider
│   ├─ mailboxRepositoryProvider
│   └─ ...
│
├─ Interactor Layer (thin wrappers, Provider<T>)
│   ├─ sendEmailInteractorProvider
│   ├─ markAsReadInteractorProvider
│   └─ ...
│
├─ Session Layer (keepAlive: true, reset on logout)
│   ├─ accountIdProvider               (StateProvider<AccountId?>)
│   ├─ sessionProvider                 (StateProvider<Session?>)
│   └─ userIdentitiesProvider          (NotifierProvider)
│
├─ Feature State (auto-dispose với screen)
│   ├─ dashboardNotifierProvider       (AsyncNotifierProvider)
│   ├─ threadNotifierProvider          (AsyncNotifierProvider)
│   ├─ mailboxNotifierProvider         (AsyncNotifierProvider)
│   └─ composerAutoSaveProvider        (StateNotifierProvider.family) ← đã có
│
└─ Cross-feature Shared State (keepAlive: true)
    ├─ selectedMailboxProvider         (StateProvider)
    ├─ selectedEmailProvider           (StateProvider)
    └─ localSettingsNotifierProvider   ← đã có
```

### 3.2 Bridge pattern (transition period)

Trong thời gian migration, GetX controllers và Riverpod providers cùng tồn tại. Bridge pattern đã được thiết lập:

```dart
// Đã có trong codebase: app_provider_container.dart
final appProviderContainer = ProviderContainer();

// Trong GetX controller muốn đọc Riverpod state:
final threadConfig = appProviderContainer
    .read(localSettingsNotifierProvider)
    .threadConfig; // ← đang được dùng tại thread_controller.dart:138

// Trong Riverpod widget muốn trigger GetX action:
// → Phải tránh pattern này, thay vào đó migrate GetX action sang provider
```

**Quy tắc bridge:**
- GetX → Riverpod: Dùng `appProviderContainer.read()` (1 chiều, chấp nhận được trong giai đoạn chuyển tiếp)
- Riverpod → GetX: **KHÔNG làm** — đây là dấu hiệu cần migrate GetX component đó sang Riverpod

---

## 4. Kế hoạch migration theo Phase

### Phase 0 — Setup (1-2 ngày) ✅ Đã xong một phần

**Mục tiêu:** Nền tảng kỹ thuật sẵn sàng

```yaml
# pubspec.yaml — thêm:
dependencies:
  flutter_riverpod: ^2.6.1      # ✅ đã có
  riverpod_annotation: ^2.6.1   # thêm mới (code generation)

dev_dependencies:
  riverpod_generator: ^2.6.1    # thêm mới
  build_runner: ^2.4.0          # thêm mới (nếu chưa có)
```

```dart
// main.dart — wrap với ProviderScope:
void main() async {
  // ...
  runApp(
    ProviderScope(
      parent: appProviderContainer, // ← bridge sang singleton
      child: const TMailApp(),
    ),
  );
}
```

**Conventions cần document:**
- Tất cả provider files đặt trong `lib/features/<feature>/presentation/providers/`
- Provider names theo pattern: `<noun><Noun>Provider` (ví dụ: `selectedMailboxProvider`)
- Notifier names: `<Noun>Notifier` với class state: `<Noun>State`
- `keepAlive: true` chỉ cho session-level state và infrastructure

---

### Phase 1 — Infrastructure Providers (1 tuần)

**Mục tiêu:** Migrate tầng singleton services ra khỏi GetX Bindings

Đây là tầng quan trọng nhất vì tất cả các tầng trên phụ thuộc vào nó.

```dart
// lib/main/providers/infrastructure_providers.dart

// --- Network ---
final dioProvider = Provider<Dio>((ref) {
  final dio = Dio();
  ref.onDispose(dio.close);
  return dio;
}, name: 'dioProvider');

final authInterceptorProvider = Provider<AuthorizationInterceptors>(
  (ref) => AuthorizationInterceptors(ref.read(dioProvider)),
  name: 'authInterceptorProvider',
);

// --- Caching ---
final cachingManagerProvider = Provider<CachingManager>(
  (ref) => CachingManager(),
  name: 'cachingManagerProvider',
);

final languageCacheManagerProvider = Provider<LanguageCacheManager>(
  (ref) => LanguageCacheManager(ref.read(cachingManagerProvider)),
  name: 'languageCacheManagerProvider',
);
```

```dart
// lib/main/providers/session_providers.dart

final accountIdProvider = StateProvider<AccountId?>((ref) => null);

final sessionProvider = StateProvider<Session?>((ref) => null);

// Khi logout — reset tất cả session state:
// ref.read(accountIdProvider.notifier).state = null;
// ref.read(sessionProvider.notifier).state = null;
// → Tất cả provider phụ thuộc session sẽ tự invalidate
```

**Checklist Phase 1:**
- [ ] Tạo `lib/main/providers/infrastructure_providers.dart`
- [ ] Tạo `lib/main/providers/session_providers.dart`
- [ ] Tạo `lib/main/providers/network_providers.dart` (Dio, interceptors)
- [ ] Tạo `lib/main/providers/caching_providers.dart`
- [ ] Bridge: các GetX services đọc từ Riverpod providers qua `appProviderContainer`
- [ ] Unit test: verify provider graph không có circular dependencies

---

### Phase 2 — Repository & Interactor Providers (1 tuần)

**Mục tiêu:** Migrate Bindings classes thành provider files

Hiện tại có 78 Binding files với 500+ `Get.lazyPut`. Chuyển đổi từng feature theo thứ tự phụ thuộc.

**Template chuyển đổi:**

```dart
// TRƯỚC: email_action_interactor_bindings.dart
class EmailActionInteractorBindings extends Bindings {
  final EmailRepository _emailRepository;
  EmailActionInteractorBindings(this._emailRepository);

  @override
  void dependencies() {
    Get.lazyPut(() => AddALabelToAnEmailInteractor(_emailRepository));
    Get.lazyPut(() => RemoveALabelFromAnEmailInteractor(_emailRepository));
  }
}

// SAU: email_action_providers.dart
final addLabelToEmailProvider = Provider<AddALabelToAnEmailInteractor>(
  (ref) => AddALabelToAnEmailInteractor(ref.read(emailRepositoryProvider)),
);

final removeLabelFromEmailProvider = Provider<RemoveALabelFromAnEmailInteractor>(
  (ref) => RemoveALabelFromAnEmailInteractor(ref.read(emailRepositoryProvider)),
);
```

**Thứ tự migrate (bottom-up theo dependency graph):**

```
Bước 2a — Data layer (datasource, repository):
  ├─ composer_cache_providers.dart    ✅ đã hoàn thành
  ├─ email_datasource_providers.dart
  ├─ mailbox_datasource_providers.dart
  └─ thread_datasource_providers.dart

Bước 2b — Domain layer (interactors):
  ├─ email_interactor_providers.dart
  ├─ mailbox_interactor_providers.dart
  └─ thread_interactor_providers.dart

Bước 2c — Platform overrides (mobile vs web):
  → Dùng Riverpod override thay vì platform-specific Bindings
```

```dart
// Platform override pattern (thay thế MobileMailboxDashboardBindings)
// lib/main/providers/platform_overrides.dart

List<Override> buildPlatformOverrides() {
  if (PlatformInfo.isWeb) {
    return [
      composerCacheDatasourceProvider.overrideWith(
        (ref) => ComposerSessionCacheDatasourceImpl(
          ref.read(cacheExceptionThrowerProvider),
        ),
      ),
    ];
  } else {
    return [
      composerCacheDatasourceProvider.overrideWith(
        (ref) => ComposerPersistentCacheDatasourceImpl(
          ref.read(composerHiveCacheClientProvider),
          ref.read(cacheExceptionThrowerProvider),
        ),
      ),
    ];
  }
}

// Trong main.dart:
runApp(
  ProviderScope(
    overrides: buildPlatformOverrides(),
    child: const TMailApp(),
  ),
);
```

**Checklist Phase 2:**
- [ ] Migrate email data layer providers
- [ ] Migrate mailbox data layer providers
- [ ] Migrate thread data layer providers
- [ ] Migrate composer data layer providers (đã có một phần)
- [ ] Migrate tất cả interactor providers
- [ ] Xóa binding files đã được replace (từng file, verify tests pass)
- [ ] Kiểm tra platform-specific overrides hoạt động đúng

---

### Phase 3 — Leaf Controllers (2-3 tuần)

**Mục tiêu:** Migrate các controllers đơn giản, ít phụ thuộc trước

**Thứ tự ưu tiên (từ đơn giản → phức tạp):**

```
Priority 1 — Standalone controllers (không phụ thuộc DashboardController):
  ├─ SpamReportController           → SpamReportNotifier
  ├─ AppGridDashboardController     → AppGridNotifier
  ├─ DownloadController             → DownloadNotifier
  └─ NetworkConnectionController    → đã có provider?

Priority 2 — Feature controllers:
  ├─ AdvancedFilterController       → AdvancedFilterNotifier
  ├─ SearchController               → SearchNotifier
  └─ IdentityCreatorController      → IdentityCreatorNotifier

Priority 3 — Core feature controllers:
  ├─ MailboxController              → MailboxNotifier
  ├─ ThreadController               → ThreadNotifier
  └─ SingleEmailController          → SingleEmailNotifier
```

**Template migration controller:**

```dart
// ─────────────────────────────────────────────────
// TRƯỚC: SpamReportController (GetX)
// ─────────────────────────────────────────────────
class SpamReportController extends GetxController {
  final GetSpamReportStateInteractor _getSpamReportState;
  final StoreSpamReportStateInteractor _storeSpamReportState;

  final spamReportState = Rxn<SpamReportState>();
  final unreadSpamCount = RxInt(0);

  SpamReportController(this._getSpamReportState, this._storeSpamReportState);

  @override
  void onInit() {
    super.onInit();
    _loadSpamState();
  }

  Future<void> _loadSpamState() async {
    final result = await _getSpamReportState.execute();
    result.fold(
      (failure) => log('load spam state failed'),
      (success) {
        spamReportState.value = success.state;
        unreadSpamCount.value = success.unreadCount;
      },
    );
  }
}

// ─────────────────────────────────────────────────
// SAU: SpamReportNotifier (Riverpod)
// ─────────────────────────────────────────────────
class SpamReportState {
  final SpamReportData? data;
  final int unreadCount;
  const SpamReportState({this.data, this.unreadCount = 0});
  SpamReportState copyWith({SpamReportData? data, int? unreadCount}) =>
      SpamReportState(data: data ?? this.data, unreadCount: unreadCount ?? this.unreadCount);
}

class SpamReportNotifier extends AsyncNotifier<SpamReportState> {
  @override
  Future<SpamReportState> build() async {
    return _loadSpamState();
  }

  Future<SpamReportState> _loadSpamState() async {
    final result = await ref
        .read(getSpamReportStateInteractorProvider)
        .execute();
    return result.fold(
      (failure) => throw failure,
      (success) => SpamReportState(data: success.state, unreadCount: success.unreadCount),
    );
  }

  Future<void> updateSpamState(bool isEnabled) async {
    state = const AsyncLoading();
    final result = await ref
        .read(storeSpamReportStateInteractorProvider)
        .execute(isEnabled);
    state = result.fold(
      (failure) => AsyncError(failure, StackTrace.current),
      (_) => AsyncData(state.value!.copyWith(data: SpamReportData(isEnabled: isEnabled))),
    );
  }
}

final spamReportNotifierProvider =
    AsyncNotifierProvider<SpamReportNotifier, SpamReportState>(
  SpamReportNotifier.new,
);

// ─────────────────────────────────────────────────
// Widget: Consumer thay Obx
// ─────────────────────────────────────────────────

// TRƯỚC:
Obx(() => SpamBannerWidget(count: controller.unreadSpamCount.value))

// SAU:
Consumer(
  builder: (context, ref, _) {
    final state = ref.watch(spamReportNotifierProvider);
    return state.when(
      loading: () => const SizedBox.shrink(),
      error: (_, __) => const SizedBox.shrink(),
      data: (spam) => SpamBannerWidget(count: spam.unreadCount),
    );
  },
)
```

**Lifecycle mapping:**

```dart
// GetX controller lifecycle → Riverpod equivalents

// onInit() → build() trong Notifier
class MyNotifier extends AsyncNotifier<MyState> {
  @override
  Future<MyState> build() async {
    // Code chạy 1 lần khi provider được create
    // ← Equivalent của onInit()
    final data = await _loadInitialData();
    return MyState(data: data);
  }
}

// onClose() → ref.onDispose() trong build()
class MyNotifier extends AsyncNotifier<MyState> {
  @override
  Future<MyState> build() async {
    ref.onDispose(() {
      // Cleanup resources
      // ← Equivalent của onClose()
      _subscription?.cancel();
    });
    return const MyState();
  }
}

// ever(rxField, callback) → ref.listen()
// Trong widget (ConsumerStatefulWidget):
@override
void initState() {
  super.initState();
  ref.listen<SomeState>(
    someProvider,
    (previous, next) {
      // Xử lý state changes
    },
  );
}
```

**Checklist Phase 3:**
- [ ] Tạo `SpamReportNotifier` + unit tests
- [ ] Tạo `AppGridNotifier` + unit tests
- [ ] Tạo `DownloadNotifier` + unit tests
- [ ] Tạo `AdvancedFilterNotifier` + unit tests
- [ ] Tạo `SearchNotifier` + unit tests
- [ ] Tạo `MailboxNotifier` + unit tests
- [ ] Tạo `ThreadNotifier` + unit tests  ← đang dùng `appProviderContainer.listen` cho localSettings
- [ ] Migrate `SpamReportController bindings` → xóa
- [ ] Integration test: verify toàn bộ flow từ UI → Notifier → Repository

---

### Phase 4 — Core Controllers: Dashboard + Composer (3-4 tuần)

**Đây là phase khó nhất.** `MailboxDashBoardController` (3528 lines) cần được phân tách thành nhiều Notifiers theo domain.

#### 4.1 Phân tách MailboxDashBoardController

```
MailboxDashBoardController (3,528 lines)
→ Phân tách theo domain:

  DashboardSessionNotifier         (~150 lines)
    selectedMailbox, accountId, vacationResponse, routerParameters

  DashboardUINotifier              (~200 lines)
    dashboardRoute, currentSelectMode, isDrawerOpened,
    isContextMenuOpened, isPopupMenuOpened, filterMessageOption

  EmailSelectionNotifier           (~150 lines)
    listEmailSelected, currentSelectMode, emailsInCurrentMailbox

  EmailFlagNotifier                (~200 lines)
    mark-as-read, mark-as-starred (replace UIAction dispatch)

  ComposerOrchestrationNotifier    (~250 lines)
    openComposer, closeComposer, restoreFromCache, listSendingEmails

  DragDropNotifier                 (~100 lines)
    isDraggingMailbox, attachmentDraggableAppState, localFileDraggableAppState

  DownloadNotifier                 (~150 lines)
    (tách từ DownloadController hiện tại)

  VacationNotifier                 (~100 lines)
    vacationResponse, updateVacation
```

**Ví dụ migration DashboardUINotifier:**

```dart
// lib/features/mailbox_dashboard/presentation/providers/dashboard_ui_notifier.dart

class DashboardUIState {
  final DashboardRoutes route;
  final SelectMode selectMode;
  final FilterMessageOption filterOption;
  final bool isDrawerOpened;
  final bool isAppGridDialogDisplayed;
  final bool isContextMenuOpened;
  final bool isRecoveringDeletedMessage;

  const DashboardUIState({
    this.route = DashboardRoutes.waiting,
    this.selectMode = SelectMode.INACTIVE,
    this.filterOption = FilterMessageOption.all,
    this.isDrawerOpened = false,
    this.isAppGridDialogDisplayed = false,
    this.isContextMenuOpened = false,
    this.isRecoveringDeletedMessage = false,
  });

  DashboardUIState copyWith({...}) => DashboardUIState(...);
}

class DashboardUINotifier extends Notifier<DashboardUIState> {
  @override
  DashboardUIState build() => const DashboardUIState();

  void navigateTo(DashboardRoutes route) {
    state = state.copyWith(route: route);
  }

  void toggleDrawer() {
    state = state.copyWith(isDrawerOpened: !state.isDrawerOpened);
  }

  void enterSelectMode() {
    state = state.copyWith(selectMode: SelectMode.ACTIVE);
  }

  void exitSelectMode() {
    state = state.copyWith(selectMode: SelectMode.INACTIVE);
  }
}

final dashboardUIProvider = NotifierProvider<DashboardUINotifier, DashboardUIState>(
  DashboardUINotifier.new,
);
```

**Giải quyết cross-notifier communication:**

```dart
// Trước: MailboxDashBoardController dùng Rxn<UIAction> để communicate
// → Thread/Mailbox controllers gọi Get.find<DashboardController>().mailboxUIAction.value = ...

// Sau: Dùng ref.read() để gọi trực tiếp notifier method
// ThreadNotifier gọi method trên DashboardSessionNotifier:

class ThreadNotifier extends AsyncNotifier<ThreadState> {
  Future<void> selectEmail(PresentationEmail email) async {
    // Cập nhật selected email trong session state
    ref.read(dashboardSessionProvider.notifier).selectEmail(email);
    // Tải email detail
    await _loadEmailDetail(email.id);
  }
}

// HOẶC dùng watch để react theo state thay đổi:
class ThreadNotifier extends AsyncNotifier<ThreadState> {
  @override
  Future<ThreadState> build() async {
    // React khi selectedMailbox thay đổi
    final mailbox = ref.watch(selectedMailboxProvider);
    if (mailbox != null) {
      return _loadEmails(mailbox.id);
    }
    return const ThreadState();
  }
}
// Khi selectedMailboxProvider thay đổi, build() tự được gọi lại → emails reload
```

#### 4.2 Migration ComposerController

```
ComposerController hiện tại:
  ├─ Auto-save logic        → ComposerAutoSaveNotifier ✅ (ADR-0086 đã xong)
  ├─ Draft management       → ComposerDraftNotifier
  ├─ Recipient management   → ComposerRecipientsNotifier
  ├─ Attachment management  → ComposerAttachmentNotifier
  └─ Send email flow        → ComposerSendNotifier
```

#### 4.3 Xử lý phần BaseController

`BaseController` (659 lines) xử lý logout, session, network — đây là cross-cutting concern:

```dart
// Thay BaseController bằng các ref.watch/listen pattern trong mỗi Notifier

// Logout handling: Notifier subscribe vào session provider
class SomeNotifier extends AsyncNotifier<SomeState> {
  @override
  Future<SomeState> build() async {
    // Tự động dispose khi session null (logout)
    final session = ref.watch(sessionProvider);
    if (session == null) return const SomeState();
    return _loadData(session.accountId);
  }
}
// Khi logout: sessionProvider.state = null → tất cả providers rebuild → clear state tự động

// Network error handling: Tạo middleware provider
final networkErrorHandlerProvider = Provider((ref) {
  return NetworkErrorHandler(ref.read(routerProvider));
});
```

**Checklist Phase 4:**
- [ ] Phân tách `MailboxDashBoardController` thành 7 Notifiers
- [ ] Tạo integration tests cho mỗi Notifier
- [ ] Migrate `ComposerController` hoàn toàn sang Riverpod
- [ ] Xóa `Rxn<UIAction>` pattern trên toàn codebase
- [ ] Xóa `ever(dashBoardAction, ...)` callbacks
- [ ] Migrate `BaseController` logic sang provider utilities
- [ ] E2E test: login → compose → send → logout flow

---

### Phase 5 — Routing & Cleanup (1-2 tuần)

**Mục tiêu:** Loại bỏ hoàn toàn GetX khỏi dependency stack (optional — chỉ nếu team quyết định)

```
Option A — Giữ GetX routing, chỉ dùng Riverpod cho state:
  → Ít rủi ro nhất
  → GetMaterialApp vẫn còn
  → Get.to/Get.off vẫn hoạt động
  → Recommend cho giai đoạn đầu

Option B — Migrate routing sang go_router:
  → Loại bỏ hoàn toàn dependency GetX
  → Type-safe routing với GoRoute params
  → Cần thêm 2-3 tuần
  → Thực hiện sau khi Phase 4 ổn định
```

```dart
// Option B - go_router setup
final routerProvider = Provider<GoRouter>((ref) {
  final session = ref.watch(sessionProvider);
  return GoRouter(
    initialLocation: session != null ? '/dashboard' : '/login',
    routes: [
      GoRoute(path: '/login', builder: (ctx, state) => const LoginView()),
      GoRoute(path: '/dashboard', builder: (ctx, state) => const DashboardView()),
      // ...
    ],
  );
});

// MaterialApp.router (không cần GetMaterialApp nữa):
MaterialApp.router(
  routerConfig: ref.watch(routerProvider),
  // ...
)
```

**Cleanup checklist Phase 5:**
- [ ] Remove tất cả `import 'package:get/get.dart'` khỏi business logic
- [ ] Remove tất cả Bindings files đã được thay thế
- [ ] Remove `Get.put`, `Get.lazyPut`, `Get.find` calls
- [ ] Remove `extends GetxController` trên tất cả controllers
- [ ] Verify `get: 4.6.6` có thể xóa hoặc downgrade còn navigation only
- [ ] Chạy `flutter analyze` không còn GetX warnings
- [ ] Performance profiling: so sánh trước/sau migration

---

## 5. Testing Strategy

### 5.1 Unit test với Riverpod

```dart
// Riverpod testing: override providers để inject mocks
void main() {
  group('ThreadNotifier', () {
    test('loads emails when mailbox selected', () async {
      final container = ProviderContainer(
        overrides: [
          // Inject mock thay vì real implementation
          getEmailsInteractorProvider.overrideWithValue(
            MockGetEmailsInteractor(),
          ),
          selectedMailboxProvider.overrideWith(
            (ref) => mockMailbox,
          ),
        ],
      );
      addTearDown(container.dispose);

      final notifier = container.read(threadNotifierProvider.notifier);
      await notifier.loadEmails();

      final state = container.read(threadNotifierProvider);
      expect(state.value?.emails, hasLength(5));
    });
  });
}

// So sánh với GetX testing (phức tạp hơn nhiều):
// GetX cần Get.put mock controllers, mock service locator, etc.
// → Riverpod testing rõ ràng và explicit hơn
```

### 5.2 Widget test

```dart
// Widget test với ConsumerWidget
testWidgets('shows email list when loaded', (tester) async {
  await tester.pumpWidget(
    ProviderScope(
      overrides: [
        threadNotifierProvider.overrideWith(() => MockThreadNotifier()),
      ],
      child: const MaterialApp(home: ThreadView()),
    ),
  );

  await tester.pump();
  expect(find.byType(EmailListItem), findsNWidgets(5));
});
```

---

## 6. Quản lý rủi ro

### 6.1 Rủi ro kỹ thuật

```
Rủi ro 1: Breaking changes khi xóa GetX controller
──────────────────────────────────────────────────
Nguy cơ: Màn hình đang dùng Get.find<Controller>() bị crash sau khi xóa
Giảm thiểu:
  1. Mỗi phase: chỉ xóa Binding sau khi TẤT CẢ consumers đã migrate
  2. Dùng feature flag để toggle giữa old/new implementation
  3. Chạy integration tests sau mỗi PR

Rủi ro 2: Memory leaks từ provider không dispose
──────────────────────────────────────────────────
Nguy cơ: Provider keepAlive:true giữ state stale sau logout
Giảm thiểu:
  1. Session providers phụ thuộc vào sessionProvider
  2. Khi logout: invalidate sessionProvider → cascade invalidation
  3. Viết test verify state = null sau logout

Rủi ro 3: Performance regression
──────────────────────────────────
Nguy cơ: Riverpod rebuild nhiều widget không cần thiết
Giảm thiểu:
  1. Dùng select() để watch một phần state:
     ref.watch(threadNotifierProvider.select((s) => s.emails.length))
  2. Dùng Consumer thay vì ConsumerWidget nếu rebuild scope nhỏ
  3. Profile với Flutter DevTools sau mỗi phase

Rủi ro 4: Xung đột hai framework trong transition period
──────────────────────────────────────────────────────────
Nguy cơ: State không sync giữa GetX và Riverpod providers
Giảm thiểu:
  1. Enforce 1 chiều: GetX → Riverpod qua appProviderContainer.read()
  2. Không bao giờ GetX widget read Riverpod state trực tiếp trong build()
  3. Code review checklist: kiểm tra bidirectional dependencies
```

### 6.2 Rollback strategy

```
Mỗi phase là độc lập và reversible:
  Phase 1: Nếu infrastructure providers có vấn đề
    → Revert về GetxService/Get.find() cũ trong Bindings
    → appProviderContainer vẫn tồn tại, không ảnh hưởng

  Phase 2: Nếu Repository providers sai
    → Bindings vẫn hoạt động song song trong transition
    → Remove provider file, revert về Get.lazyPut

  Phase 3-4: Nếu Notifier bị regression
    → Feature flag: if (useRiverpod) NotifierWidget() else GetxWidget()
    → Không xóa GetX controller cho đến khi Notifier ổn định 2 sprint
```

---

## 7. Timeline tổng thể

```
GIAI ĐOẠN          THỜI GIAN    DELIVERABLE
══════════════════════════════════════════════════════════════
Phase 0 (Setup)    Đã hoàn thành  pubspec, ProviderContainer, 3 providers
Phase 1 (Infra)    1 tuần         ~10 infrastructure providers
Phase 2 (DI)       1 tuần         78 Bindings → Provider files
Phase 3 (Leaf)     2-3 tuần       10 leaf controllers → Notifiers
Phase 4 (Core)     3-4 tuần       Dashboard + Composer → Notifiers
Phase 5 (Routing)  1-2 tuần       (optional) go_router migration
──────────────────────────────────────────────────────────────
TỔNG               8-11 tuần      100% GetX state management removed

Note: Các phases có thể chạy song song nếu team > 1 người.
      Phase 3 và Phase 2 có thể overlap.
```

---

## 8. Bảng đánh giá so sánh — Trước và Sau migration

```
Metric                    GetX (hiện tại)    Riverpod (target)    Thay đổi
══════════════════════════════════════════════════════════════════════════
Dependency safety         Runtime crash      Compile-time error   ✅ Tốt hơn
Test isolation            Khó (mock GetX)    Dễ (ProviderScope)   ✅ Tốt hơn
Auto-dispose              Manual / binding   Tự động (scoped)     ✅ Tốt hơn
State reset on logout     Manual clearState  Invalidate cascade   ✅ Tốt hơn
Platform overrides        Subclass Bindings  Provider.override()  ✅ Gọn hơn
Learning curve (new dev)  Biết GetX          Cần học Riverpod     ⚠ Khó hơn
Code lines (DI)           78 Bindings files  ~30 provider files   ✅ Ít hơn
Reactive UI performance   Granular (Obx)     Granular (ref.watch) = Tương đương
Navigation                Get.to/off         go_router (Phase 5)  → Cần migrate
Framework lock-in         GetX tightly       Riverpod loosely     ✅ Ít lock-in
```

---

## 9. Tài liệu liên quan

| Tài liệu | Nội dung |
|---|---|
| [`01-problem-statement.md`](01-problem-statement.md) | Root causes cần giải quyết |
| [`07-why-we-chose-getxservice-eventbus.md`](07-why-we-chose-getxservice-eventbus.md) | Giải pháp trung gian (ADR-0076) |
| [ADR-0086 — composer auto-save](../solutions/) | First Riverpod adoption: StateNotifierProvider.family |

**Code references (codebase thực tế):**
- `lib/main/providers/app_provider_container.dart` — ProviderContainer singleton hiện tại
- `lib/features/composer/presentation/providers/` — Mẫu provider đầu tiên trong project
- `lib/features/manage_account/presentation/providers/local_settings_notifier.dart` — StateNotifierProvider pattern
- `lib/features/mailbox_dashboard/presentation/controller/mailbox_dashboard_controller.dart` — God Object cần phân tách (3,528 lines)
