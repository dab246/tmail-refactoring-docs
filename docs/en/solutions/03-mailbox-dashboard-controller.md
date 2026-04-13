# Solution: MailboxDashboardController

**Problem:** God Object 3,508 lines, 43 deps, 42 Rx fields, 15 bounded contexts.

---

## Decomposition Map

```
 MailboxDashBoardController  (3,508 lines · 43 deps · 42 Rx fields)
 │
 ├── [Phase 2] ──► EmailListStateService        (GetxService · permanent)
 │                  emails · isLoading · selectedMailbox · mailboxMaps · sortOrder
 │
 ├── [Phase 2] ──► ComposerStateService         (GetxService · permanent)
 │                  isOpen · arguments
 │
 ├── [Phase 2] ──► DragDropStateService         (GetxService · permanent)
 │                  isDragging · dropTarget · draggingEmail
 │
 ├── [Phase 3] ──► EmailActionController        (~300 lines · 7 interactors)
 │                  move · delete · mark · star · restore · unsubscribe
 │
 ├── [Phase 3] ──► SendingQueueController       (~200 lines · 5 interactors)
 │                  store · update · send · delete · getAll
 │
 ├── [Phase 3] ──► MailboxNavigationController  (~250 lines)
 │                  selectMailbox · openDefault · navigateToEmail
 │
 ├── [Phase 3] ──► SessionController            (~200 lines)
 │                  session · accountId · listIdentities
 │
 └── [Phase 3 end] MailboxDashBoardController   (~200 lines)
                    Orchestrator only — no business logic · ≤7 deps
```

---

## Phase 2 — GetxService State Decomposition

### EmailListStateService

```dart
// lib/features/mailbox_dashboard/presentation/service/email_list_state_service.dart
class EmailListStateService extends GetxService {
  final RxList<PresentationEmail> emails = <PresentationEmail>[].obs;
  final RxBool isLoading = false.obs;
  final Rxn<PresentationMailbox> selectedMailbox = Rxn();
  final Rx<EmailSortOrder> sortOrder = Rx(EmailSortOrder.mostRecent);
  final Rx<FilterMessageOption> filterOption =
      Rx(FilterMessageOption.all);

  // Mailbox maps — moved out of MailboxDashBoardController
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
    _mapDefaultMailboxIdByRole
      ..clear()
      ..addAll(byRole);
    _mapMailboxById
      ..clear()
      ..addAll(byId);
  }

  void updateEmails(List<PresentationEmail> list) {
    emails.assignAll(list);
  }

  void clearFilter() {
    filterOption.value = FilterMessageOption.all;
  }
}
```

### ComposerStateService

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

### DragDropStateService

```dart
// lib/features/mailbox_dashboard/presentation/service/drag_drop_state_service.dart
class DragDropStateService extends GetxService {
  final RxBool isDraggingEmail = false.obs;
  final RxBool isDraggingMailbox = false.obs;
  final Rxn<PresentationEmail> draggingEmail = Rxn();
  final Rxn<PresentationMailbox> dropTarget = Rxn();

  void startDragging(PresentationEmail email) {
    draggingEmail.value = email;
    isDraggingEmail.value = true;
  }

  void stopDragging() {
    isDraggingEmail.value = false;
    draggingEmail.value = null;
    dropTarget.value = null;
  }
}
```

---

## Phase 3 — Sub-Controller Extraction

### EmailActionController

```dart
// lib/features/mailbox_dashboard/presentation/controller/email_action_controller.dart
class EmailActionController extends GetxController {
  final MoveToMailboxInteractor _moveToMailbox;
  final DeleteEmailPermanentlyInteractor _deleteEmail;
  final MarkAsEmailReadInteractor _markAsRead;
  final MarkAsStarEmailInteractor _markAsStar;
  final MoveMultipleEmailToMailboxInteractor _moveMultiple;
  final DeleteMultipleEmailsPermanentlyInteractor _deleteMultiple;
  final MarkAsMultipleEmailReadInteractor _markMultipleRead;
  final AppEventBus _eventBus;

  EmailActionController(
    this._moveToMailbox,
    this._deleteEmail,
    this._markAsRead,
    this._markAsStar,
    this._moveMultiple,
    this._deleteMultiple,
    this._markMultipleRead,
    this._eventBus,
  );

  // Fix Mixed Abstraction: the decision of which interactor to use lives here
  void moveEmail({
    required Session session,
    required AccountId accountId,
    required MoveToMailboxRequest request,
    required Map<EmailId, bool> readStatus,
  }) {
    final isSingleEmail = request.isSingleEmail;
    final stream = isSingleEmail
        ? _moveToMailbox.execute(session, accountId, request)
        : _moveMultiple.execute(session, accountId, request);

    consumeState(stream);
  }

  // Fix Bumpy Road: handleSuccess only processes email action success types
  void _handleSuccess(Success success) {
    switch (success) {
      case MoveToMailboxSuccess():
        _eventBus.dispatch(EmailMovedEvent(success.emailIds));
      case DeleteEmailPermanentlySuccess():
        _eventBus.dispatch(EmailDeletedEvent(success.emailIds));
      case MarkAsEmailReadSuccess():
        _eventBus.dispatch(EmailReadStatusChangedEvent(success.emailIds));
      default:
        break;
    }
  }
}
```

### SendingQueueController

```dart
// lib/features/mailbox_dashboard/presentation/controller/sending_queue_controller.dart
class SendingQueueController extends GetxController {
  final StoreSendingEmailInteractor _store;
  final UpdateSendingEmailInteractor _update;
  final GetAllSendingEmailInteractor _getAll;
  final DeleteSendingEmailInteractor _delete;
  final SendEmailInteractor _send;
  final AppEventBus _eventBus;

  final RxList<SendingEmail> sendingEmails = <SendingEmail>[].obs;

  SendingQueueController(
    this._store, this._update, this._getAll,
    this._delete, this._send, this._eventBus,
  );

  void loadSendingEmails(Session session, AccountId accountId) {
    consumeState(_getAll.execute(session, accountId));
  }

  void _handleSuccess(Success success) {
    switch (success) {
      case GetAllSendingEmailSuccess():
        sendingEmails.assignAll(success.sendingEmails);
      case SendEmailSuccess():
        _eventBus.dispatch(EmailSentEvent(success.emailId));
      default:
        break;
    }
  }
}
```

### MailboxNavigationController

```dart
// lib/features/mailbox_dashboard/presentation/controller/mailbox_navigation_controller.dart
class MailboxNavigationController extends GetxController {
  final EmailListStateService _emailListState;
  final AppEventBus _eventBus;

  final Rx<DashboardRoutes> currentRoute =
      Rx(DashboardRoutes.thread);

  MailboxNavigationController(this._emailListState, this._eventBus);

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

### SessionController

```dart
// lib/features/mailbox_dashboard/presentation/controller/session_controller.dart
class SessionController extends GetxController {
  final StoreSessionInteractor _storeSession;
  final GetAllIdentitiesInteractor _getAllIdentities;
  final IAuthService _authService;
  final AppEventBus _eventBus;

  final Rxn<Session> session = Rxn();
  final Rxn<AccountId> accountId = Rxn();
  final RxList<Identity> listIdentities = <Identity>[].obs;

  SessionController(
    this._storeSession,
    this._getAllIdentities,
    this._authService,
    this._eventBus,
  );

  void initialize(Session newSession, AccountId newAccountId) {
    session.value = newSession;
    accountId.value = newAccountId;
    _eventBus.dispatch(SessionInitializedEvent(newSession, newAccountId));
    _loadIdentities(newSession, newAccountId);
  }
}
```

---

## Fix: handleSuccessViewState — God Dispatcher

**Before:**
```dart
// 1 method handling 31 types from 15 bounded contexts
void handleSuccessViewState(Success success) {
  if (success is GetAllMailboxSuccess) { _handleGetAllMailboxSuccess(success); }
  else if (success is MoveToMailboxSuccess) { ... }
  // ... 29 more branches
}
```

**After:** Each sub-controller overrides `handleSuccessViewState()` and only handles the types belonging to it.

```dart
// EmailActionController
@override
void handleSuccessViewState(Success success) {
  switch (success) {
    case MoveToMailboxSuccess(): _handleMoveSuccess(success);
    case DeleteEmailPermanentlySuccess(): _handleDeleteSuccess(success);
    case MarkAsEmailReadSuccess(): _handleMarkReadSuccess(success);
    default: super.handleSuccessViewState(success); // pass up if not handled
  }
}

// SendingQueueController
@override
void handleSuccessViewState(Success success) {
  switch (success) {
    case SendEmailSuccess(): _handleSendSuccess(success);
    case StoreSendingEmailSuccess(): _handleStoreSuccess(success);
    default: super.handleSuccessViewState(success);
  }
}
```

---

## Fix: Excess Data Declarations

**Before — 13+ scattered RxBool fields:**
```dart
final _isDraggingMailbox = RxBool(false);
final searchMailboxActivated = RxBool(false);
final isRecoveringDeletedMessage = RxBool(false);
final isAppGridDialogDisplayed = RxBool(false);
// ... 9 more
```

**After — grouped by bounded context in services:**
```dart
// DragDropStateService
final isDraggingMailbox = RxBool(false);
final isDraggingEmail = RxBool(false);

// EmailListStateService
final isLoading = RxBool(false);
final isRecovering = RxBool(false);

// SearchStateService (or SearchEmailController)
final isSearchActivated = RxBool(false);
```

---

## Fix: Force Unwrap Pattern

**Before — repeated 50+ times:**
```dart
if (accountId.value != null && sessionCurrent != null) {
  consumeState(_interactor.execute(sessionCurrent!, accountId.value!, ...));
}
```

**After — extension method + guard:**
```dart
// lib/features/base/extension/session_extension.dart
extension SessionGuard on SessionController {
  void withSession(void Function(Session session, AccountId accountId) action) {
    final s = session.value;
    final id = accountId.value;
    if (s != null && id != null) {
      action(s, id);
    }
  }
}

// Usage:
withSession((session, accountId) {
  consumeState(_interactor.execute(session, accountId, ...));
});
```

---

## MailboxDashBoardController after refactor (~200 lines)

```dart
// Orchestrator only — no business logic
class MailboxDashBoardController extends ReloadableController {
  final EmailListStateService emailListState;
  final ComposerStateService composerState;
  final DragDropStateService dragDropState;
  final AppEventBus eventBus;

  MailboxDashBoardController(
    this.emailListState,
    this.composerState,
    this.dragDropState,
    this.eventBus,
    ISessionTokenRefresher tokenRefresher,
    ILanguageSettingService languageService,
    IAuthService authService,
  ) : super(tokenRefresher, languageService, authService, ...);

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
}
```

---

## DashboardBindings after refactor

```dart
class MailboxDashboardBindings extends Bindings {
  @override
  void dependencies() {
    // GetxServices (permanent)
    Get.put<EmailListStateService>(EmailListStateService(), permanent: true);
    Get.put<ComposerStateService>(ComposerStateService(), permanent: true);
    Get.put<DragDropStateService>(DragDropStateService(), permanent: true);

    // Sub-controllers with explicit constructor injection
    Get.lazyPut(() => EmailActionController(
      Get.find(), Get.find(), Get.find(), Get.find(),
      Get.find(), Get.find(), Get.find(),
      Get.find<AppEventBus>(),
    ));

    Get.lazyPut(() => SendingQueueController(
      Get.find(), Get.find(), Get.find(),
      Get.find(), Get.find(),
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

    // Orchestrator — receives services via constructor
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

## Metrics

| | Before | After |
|---|---|---|
| Lines | 3,508 | ~200 (orchestrator) |
| Constructor deps | 43 | 7 |
| Rx fields | 42+ | ≤5 (orchestrator) |
| handleSuccessViewState branches | 31 | 0 (moved to sub-controllers) |
| Bounded contexts | 15 | 1 (orchestration) |