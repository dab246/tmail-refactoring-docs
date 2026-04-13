# MailboxDashboardController — Phân tích sâu & Giải pháp chi tiết

---

## I. Vấn đề — Số liệu thực tế

| Chỉ số | Giá trị |
|---|---|
| Số dòng | **3,498** |
| Constructor params (positional) | **29** |
| `Get.find()` trực tiếp tại class level | **14** |
| Rx fields | **42+** |
| `handleSuccessViewState` branches | **35+** |
| Force-unwrap (`sessionCurrent!`, `accountId.value!`) | **67 lần** |
| Extension files dưới `extensions/` | **35 files** |
| Bounded contexts | **15+** |

---

## II. Các vấn đề cụ thể

### 1. God Object — 15 bounded contexts trong 1 class

`MailboxDashBoardController` đang xử lý đồng thời:

```
Session & authentication     → _storeSessionInteractor, getAuthenticationInfoInteractor
Email actions                → move, delete, mark read, mark star, unsubscribe, restore
Sending queue                → store, update, getAll, delete, send
Mailbox operations           → markAsRead, clear, empty trash, empty spam
Vacation                     → getAllVacation, updateVacation
Drag & drop                  → _isDraggingMailbox, attachmentDraggableAppState, localFileDraggableAppState
Search                       → searchController, listResultSearch, filterMessageOption
Navigation routing           → dashboardRoute, dispatchRoute()
Identities                   → _getAllIdentitiesInteractor, _identities
Composer cache               → _getEmailCacheOnWebInteractor, _removeComposerCacheByIdOnWebInteractor
Server settings              → getServerSettingInteractor, isSenderImportantFlagEnabled
AI Scribe                    → getAIScribeConfigInteractor, cachedAIScribeConfig, isAINeedsActionSettingEnabled
Push notifications / deep links → _notificationManager, _deepLinksManager, _fcmService
Labels                       → labelController, subscribeLabelViewStateSuccess()
App grid / paywall           → appGridDashboardController, paywallController
```

---

### 2. Constructor Injection Hell — 29 params positional

```dart
// Lines 369–399: không thể nhìn mà biết param nào là gì
MailboxDashBoardController(
  this._moveToMailboxInteractor,
  this._deleteEmailPermanentlyInteractor,
  this._markAsMailboxReadInteractor,
  this._getEmailCacheOnWebInteractor,
  // ... 25 params nữa
);
```

Thêm vào đó **14 trường dùng `Get.find()` trực tiếp** tại class level (lines 246–259) — trộn lẫn hai cơ chế injection, không nhất quán và không test được.

---

### 3. `handleSuccessViewState` — God Dispatcher (35 nhánh)

```dart
// Lines 452–563: 1 method xử lý 35 Success types từ 15 contexts khác nhau
void handleSuccessViewState(Success success) {
  if (success is SendEmailLoading) { ... }
  else if (success is SendEmailSuccess) { ... }
  else if (success is SaveEmailAsDraftsSuccess) { ... }
  else if (success is MoveToMailboxSuccess) { ... }
  else if (success is MarkAsMailboxReadAllSuccess || ...) { ... }
  else if (success is GetAllVacationSuccess) { ... }
  // ... 29 nhánh nữa
}
```

Vấn đề: Mỗi lần thêm feature mới = thêm `else if` vào đây. O(n) type-check, vi phạm Open/Closed.

---

### 4. `Rxn<UIAction>` + `ever()` — Implicit Message Bus

```dart
// Lines 306–309: 4 "kênh" UIAction riêng biệt
final dashBoardAction      = Rxn<UIAction>();
final mailboxUIAction      = Rxn<MailboxUIAction>();
final emailUIAction        = Rxn<EmailUIAction>();
final threadDetailUIAction = Rxn<ThreadDetailUIAction>();
```

Controller con muốn giao tiếp → set `.value` → controller cha `ever()` lắng nghe → ẩn luồng dữ liệu, không trace được, phải null-reset thủ công (`DashBoardAction.idle`).

---

### 5. Force-Unwrap Pattern — 67 lần lặp

```dart
// Lặp đi lặp lại khắp nơi trong file
if (accountId.value == null || sessionCurrent == null) return;
consumeState(_someInteractor.execute(sessionCurrent!, accountId.value!, ...));
```

67 lần — lỗi runtime nếu quên check, boilerplate cực kỳ nhiễu.

---

### 6. Rx Fields rời rạc — 42+ fields không có nhóm logic

```dart
// Lines 303–338: các field thuộc nhiều contexts khác nhau nằm chung 1 chỗ
final _isDraggingMailbox          = RxBool(false);          // drag/drop
final isRecoveringDeletedMessage  = RxBool(false);          // email recovery
final isAppGridDialogDisplayed    = RxBool(false);          // app grid UI
final isAINeedsActionSettingEnabled = RxBool(false);        // AI scribe
final cachedAIScribeConfig        = AIScribeConfig.initial().obs; // AI scribe
final octetsQuota                 = Rxn<Quota>();           // storage quota
```

---

### 7. Extension-based Code Splitting — giải pháp sai hướng

35 files extension (`handle_*.dart`) dưới `extensions/` — cố chia nhỏ code nhưng tất cả vẫn là extension của `MailboxDashBoardController`. Vẫn cùng `this`, cùng state, cùng deps. Đây là **code splitting, không phải decomposition**.

---

## III. Giải pháp — Chi tiết từng bước

---

### Phase 2 — Tách State ra GetxService (3 services)

#### Bước 2.1: `EmailListStateService`

Tách các field liên quan đến email list và mailbox maps:

```
selectedMailbox           → EmailListStateService.selectedMailbox
emailsInCurrentMailbox    → EmailListStateService.emails
filterMessageOption       → EmailListStateService.filterOption
mapDefaultMailboxIdByRole → EmailListStateService._mapDefaultMailboxIdByRole
mapMailboxById            → EmailListStateService._mapMailboxById
currentSortOrder          → EmailListStateService.sortOrder
```

```dart
// lib/features/mailbox_dashboard/presentation/service/email_list_state_service.dart
class EmailListStateService extends GetxService {
  final RxList<PresentationEmail> emails = <PresentationEmail>[].obs;
  final RxBool isLoading = false.obs;
  final Rxn<PresentationMailbox> selectedMailbox = Rxn();
  final Rx<EmailSortOrderType> sortOrder = Rx(SearchEmailFilter.defaultSortOrder);
  final Rx<FilterMessageOption> filterOption = Rx(FilterMessageOption.all);

  final _mapDefaultMailboxIdByRole = <Role, MailboxId>{};
  final _mapMailboxById = <MailboxId, PresentationMailbox>{};

  Map<Role, MailboxId> get mailboxIdByRole =>
      Map.unmodifiable(_mapDefaultMailboxIdByRole);
  Map<MailboxId, PresentationMailbox> get mailboxById =>
      Map.unmodifiable(_mapMailboxById);

  void updateMailboxMaps({
    required Map<Role, MailboxId> byRole,
    required Map<MailboxId, PresentationMailbox> byId,
  }) {
    _mapDefaultMailboxIdByRole..clear()..addAll(byRole);
    _mapMailboxById..clear()..addAll(byId);
  }
}
```

Register permanent (trong `MailboxDashboardBindings`):
```dart
Get.put<EmailListStateService>(EmailListStateService(), permanent: true);
```

Controller con (`ThreadController`, `MailboxController`) đọc state qua `Get.find<EmailListStateService>()` thay vì `Get.find<MailboxDashBoardController>()`.

---

#### Bước 2.2: `ComposerStateService`

```dart
// lib/features/mailbox_dashboard/presentation/service/composer_state_service.dart
class ComposerStateService extends GetxService {
  final RxBool isOpen = false.obs;
  final Rxn<ComposerArguments> arguments = Rxn();
  final RxBool isDraftSaving = false.obs;

  void open(ComposerArguments args) {
    arguments.value = args;
    isOpen.value = true;
  }

  void close() {
    isOpen.value = false;
    arguments.value = null;
  }
}
```

---

#### Bước 2.3: `DragDropStateService`

```dart
// lib/features/mailbox_dashboard/presentation/service/drag_drop_state_service.dart
class DragDropStateService extends GetxService {
  final RxBool isDraggingMailbox = false.obs;
  final RxBool isDraggingEmail = false.obs;
  final Rxn<DraggableAppState> attachmentState = Rxn();
  final Rxn<DraggableAppState> localFileState = Rxn();
  final Rxn<PresentationEmail> draggingEmail = Rxn();

  void startDraggingEmail(PresentationEmail email) {
    draggingEmail.value = email;
    isDraggingEmail.value = true;
  }

  void stopDragging() {
    isDraggingEmail.value = false;
    isDraggingMailbox.value = false;
    draggingEmail.value = null;
  }
}
```

---

### Phase 3 — Tách Sub-Controllers

#### Bước 3.1: `EmailActionController` (~300 lines)

Nhận 7 interactors liên quan đến email actions:

```
_moveToMailboxInteractor
_deleteEmailPermanentlyInteractor
_markAsEmailReadInteractor
_markAsStarEmailInteractor
_moveMultipleEmailToMailboxInteractor
_deleteMultipleEmailsPermanentlyInteractor
_markAsMultipleEmailReadInteractor
```

Override `handleSuccessViewState` chỉ cho các Success types của mình:

```dart
// lib/features/mailbox_dashboard/presentation/controller/email_action_controller.dart
class EmailActionController extends GetxController {
  // ... interactors + AppEventBus

  @override
  void handleSuccessViewState(Success success) {
    switch (success) {
      case MoveToMailboxSuccess():
        _eventBus.dispatch(EmailMovedEvent(success.emailIds));
      case DeleteEmailPermanentlySuccess():
        _eventBus.dispatch(EmailDeletedEvent(success.emailIds));
      case MarkAsEmailReadSuccess():
        _eventBus.dispatch(EmailReadStatusChangedEvent(success.emailIds));
      case MoveMultipleEmailToMailboxAllSuccess():
      case MoveMultipleEmailToMailboxHasSomeEmailFailure():
        _handleMoveMultipleSuccess(success);
      default:
        super.handleSuccessViewState(success);
    }
  }
}
```

Giao tiếp ra ngoài qua `AppEventBus.dispatch(...)` thay vì set `dashBoardAction.value`.

---

#### Bước 3.2: `SendingQueueController` (~200 lines)

```dart
// lib/features/mailbox_dashboard/presentation/controller/sending_queue_controller.dart
class SendingQueueController extends GetxController {
  final StoreSendingEmailInteractor _store;
  final UpdateSendingEmailInteractor _update;
  final GetAllSendingEmailInteractor _getAll;
  final DeleteSendingEmailInteractor _delete;
  final SendEmailInteractor _send;
  final AppEventBus _eventBus;

  // Rx field chuyển về đây, không còn trong Dashboard
  final sendingEmails = RxList<SendingEmail>();

  @override
  void handleSuccessViewState(Success success) {
    switch (success) {
      case GetAllSendingEmailSuccess():
        sendingEmails.assignAll(success.sendingEmails);
      case StoreSendingEmailSuccess():
        _eventBus.dispatch(SendingEmailStoredEvent(success.sendingEmail));
      case SendEmailSuccess():
        _eventBus.dispatch(EmailSentEvent(success.emailId));
      case DeleteSendingEmailSuccess():
        loadSendingEmails();
      default:
        super.handleSuccessViewState(success);
    }
  }
}
```

---

#### Bước 3.3: `MailboxNavigationController` (~250 lines)

Chịu trách nhiệm toàn bộ navigation state:

```dart
// lib/features/mailbox_dashboard/presentation/controller/mailbox_navigation_controller.dart
class MailboxNavigationController extends GetxController {
  final EmailListStateService _emailListState;
  final AppEventBus _eventBus;

  // dashboardRoute chuyển về đây
  final currentRoute = DashboardRoutes.waiting.obs;

  void selectMailbox(PresentationMailbox mailbox) {
    _emailListState.selectedMailbox.value = mailbox;
    _eventBus.dispatch(MailboxSelectedEvent(mailbox));
  }

  void openDefaultMailbox() {
    currentRoute.value = DashboardRoutes.thread;
    _eventBus.dispatch(DefaultMailboxOpenedEvent());
  }

  void navigateToEmail(EmailId emailId) {
    currentRoute.value = DashboardRoutes.emailDetailed;
    _eventBus.dispatch(EmailNavigationRequestedEvent(emailId));
  }
}
```

Thay thế toàn bộ `dashBoardAction.value = ...` + `mailboxUIAction.value = ...` bằng event dispatch.

---

#### Bước 3.4: `SessionController` (~200 lines)

```dart
// lib/features/mailbox_dashboard/presentation/controller/session_controller.dart
class SessionController extends GetxController {
  final StoreSessionInteractor _storeSession;
  final GetAllIdentitiesInteractor _getAllIdentities;
  final IAuthService _authService;
  final AppEventBus _eventBus;

  // Tách ra khỏi ReloadableController / MailboxDashBoardController
  final session      = Rxn<Session>();
  final accountId    = Rxn<AccountId>();
  final listIdentities = RxList<Identity>();

  void initialize(Session newSession, AccountId newAccountId) {
    session.value = newSession;
    accountId.value = newAccountId;
    _eventBus.dispatch(SessionInitializedEvent(newSession, newAccountId));
    _loadIdentities(newSession, newAccountId);
  }
}
```

---

#### Bước 3.5: Fix Force-Unwrap bằng extension

```dart
// lib/features/base/extension/session_extension.dart
extension SessionGuard on SessionController {
  void withSession(void Function(Session session, AccountId accountId) action) {
    final s = session.value;
    final id = accountId.value;
    if (s != null && id != null) action(s, id);
  }
}
```

```dart
// Trước — lặp 67 lần:
if (accountId.value == null || sessionCurrent == null) return;
consumeState(_interactor.execute(sessionCurrent!, accountId.value!, ...));

// Sau:
withSession((session, accountId) {
  consumeState(_interactor.execute(session, accountId, ...));
});
```

---

#### Bước 3.6: Thay `Rxn<UIAction>` + `ever()` bằng `AppEventBus`

```dart
// Xóa 4 field này khỏi MailboxDashBoardController:
// dashBoardAction, mailboxUIAction, emailUIAction, threadDetailUIAction

// Controller con dispatch event thay vì set .value:
_eventBus.dispatch(OpenEmailDetailEvent(emailId));
_eventBus.dispatch(MailboxSelectedEvent(mailbox));
_eventBus.dispatch(ComposerOpenedEvent(args));

// Orchestrator subscribe:
eventBus.stream
  .whereType<OpenEmailDetailEvent>()
  .listen((e) => _navigationController.navigateToEmail(e.emailId))
  .addTo(compositeSubscription);
```

Không còn `DashBoardAction.idle` null-reset, không còn `workerObxVariables`.

---

#### Bước 3.7: `MailboxDashBoardController` sau refactor (~200 lines)

```dart
// Orchestrator only — không có business logic
class MailboxDashBoardController extends ReloadableController {
  final EmailListStateService emailListState;
  final ComposerStateService composerState;
  final DragDropStateService dragDropState;
  final AppEventBus eventBus;

  // 7 deps thay vì 43
  MailboxDashBoardController(
    this.emailListState,
    this.composerState,
    this.dragDropState,
    this.eventBus,
    ISessionTokenRefresher tokenRefresher,
    ILanguageSettingService languageService,
    IAuthService authService,
  ) : super(tokenRefresher, languageService, authService);

  @override
  void onInit() {
    super.onInit();
    _subscribeToSessionEvents();
    _subscribeToNavigationEvents();
  }

  void _subscribeToSessionEvents() {
    eventBus.stream
        .whereType<SessionInitializedEvent>()
        .listen(_onSessionInitialized)
        .addTo(compositeSubscription);
  }

  void _subscribeToNavigationEvents() {
    eventBus.stream
        .whereType<MailboxSelectedEvent>()
        .listen((e) => emailListState.selectedMailbox.value = e.mailbox)
        .addTo(compositeSubscription);
  }

  // handleSuccessViewState: không có gì ở đây — tất cả đã moved to sub-controllers
}
```

---

### DashboardBindings sau refactor

```dart
class MailboxDashboardBindings extends Bindings {
  @override
  void dependencies() {
    // GetxServices (permanent — sống suốt app lifecycle)
    Get.put<EmailListStateService>(EmailListStateService(), permanent: true);
    Get.put<ComposerStateService>(ComposerStateService(), permanent: true);
    Get.put<DragDropStateService>(DragDropStateService(), permanent: true);

    // Sub-controllers với constructor injection rõ ràng
    Get.lazyPut(() => EmailActionController(
      Get.find(), Get.find(), Get.find(), Get.find(),
      Get.find(), Get.find(), Get.find(),
      Get.find<AppEventBus>(),
    ));

    Get.lazyPut(() => SendingQueueController(
      Get.find(), Get.find(), Get.find(), Get.find(), Get.find(),
      Get.find<AppEventBus>(),
    ));

    Get.lazyPut(() => MailboxNavigationController(
      Get.find<EmailListStateService>(),
      Get.find<AppEventBus>(),
    ));

    Get.lazyPut(() => SessionController(
      Get.find(), Get.find(),
      Get.find<IAuthService>(),
      Get.find<AppEventBus>(),
    ));

    // Orchestrator — chỉ nhận services
    Get.lazyPut(() => MailboxDashBoardController(
      Get.find<EmailListStateService>(),
      Get.find<ComposerStateService>(),
      Get.find<DragDropStateService>(),
      Get.find<AppEventBus>(),
      Get.find<ISessionTokenRefresher>(),
      Get.find<ILanguageSettingService>(),
      Get.find<IAuthService>(),
    ));
  }
}
```

---

## IV. Kết quả sau refactor

| | Trước | Sau |
|---|---|---|
| Lines | 3,498 | ~200 (orchestrator) |
| Constructor deps | 29 positional + 14 Get.find() | 7 |
| Rx fields | 42+ | ≤5 |
| `handleSuccessViewState` branches | 35 | 0 |
| `Rxn<UIAction>` channels | 4 (+ workerObxVariables) | 0 |
| Force-unwrap (`!`) | 67 | 0 |
| Extension files (coupled) | 35 | moved vào sub-controllers |
| Testability | Không test được độc lập | Mỗi sub-controller test riêng |
