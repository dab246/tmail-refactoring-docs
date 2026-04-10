# Solution: Shared Infrastructure — AppEventBus + GetxService

**Đây là nền tảng cần implement trước (Phase 0). Tất cả solutions trong các files khác đều phụ thuộc vào infrastructure này.**

---

## 1. AppEventBus

### Sơ đồ tổng quan AppEventBus

```
 AppEventBus — Publish / Subscribe Pattern
 ════════════════════════════════════════════════════════════════════
          PUBLISHERS                         SUBSCRIBERS
 ─────────────────────────────────────────────────────────────────────
 MailboxNavigationController                MailboxController
   dispatch(MailboxSelectedEvent)      ──►  whereType<MailboxSelectedEvent>
                                            ThreadController
                                       ──►  whereType<MailboxSelectedEvent>

 EmailActionController
   dispatch(EmailMovedEvent)           ──►  ThreadController
   dispatch(EmailDeletedEvent)              whereType<EmailDeletedEvent>

 SessionController
   dispatch(SessionInitializedEvent)   ──►  MailboxController
                                            ThreadController
                                            SearchEmailController

 SendingQueueController
   dispatch(EmailSentEvent)            ──►  (controllers liên quan)
 ─────────────────────────────────────────────────────────────────────
                      ┌───────────────────────┐
                      │      AppEventBus       │
                      │  GetxService·permanent │
                      │  broadcast stream      │
                      │  dispatch(AppEvent)    │
                      │  stream.whereType<T>() │
                      └───────────────────────┘
 Nguyên tắc: Publishers không biết ai subscribe.
             Subscribers không biết ai dispatch.
             Coupling = 0.
```

### Sơ đồ luồng sự kiện — Sequence Diagram

```
 Sequence: CoreBindings setup → User chọn mailbox → User xóa emails
 ════════════════════════════════════════════════════════════════════════
 CoreBindings   AppEventBus     MNC           MC             TC      EAC
      │               │          │             │              │        │
      │─ put(EB,      │          │             │              │        │
      │   permanent)─►│          │             │              │        │
      │               │          │             │              │        │
      │               │◄─ subscribe MailboxSelectedEvent ────►│        │
      │               │◄─ subscribe MailboxSelectedEvent ──────────────►│ (TC)
      │               │◄─ subscribe EmailBulkActionRequested ──────────────────►│
      │               │          │             │              │        │
      │               │          │ User chọn Inbox            │        │
      │               │◄── dispatch(MailboxSelectedEvent(inbox))        │
      │               │          │             │              │        │
      │               ├──────────────────────►│ _onMailboxSelected()  │
      │               ├────────────────────────────────────►│ _onMailboxSelected()
      │               │          │             │  load emails │        │
      │               │          │             │              │        │
      │               │          │   User chọn 3 emails và xóa          │
      │               │◄─ dispatch(EmailBulkActionRequestedEvent(delete, 3 emails))
      │               │          │             │              │        │
      │               ├──────────────────────────────────────────────►│
      │               │          │             │           [execute delete]
      │               │◄─ dispatch(EmailDeletedEvent(3 emailIds)) ─────┤
      │               │          │             │              │        │
      │               ├──────────────────────────────────────►│ _onEmailDeleted
      │               │          │             │    update list│        │
```

### File structure

```
lib/core/event_bus/
├── app_event_bus.dart        # GetxService singleton
├── app_event.dart            # sealed class + subtypes
└── composite_subscription.dart # subscription lifecycle helper
```

### Implementation

```dart
// lib/core/event_bus/app_event_bus.dart
class AppEventBus extends GetxService {
  final _controller = StreamController<AppEvent>.broadcast();

  Stream<AppEvent> get stream => _controller.stream;

  void dispatch(AppEvent event) {
    if (!_controller.isClosed) {
      _controller.add(event);
    }
  }

  @override
  void onClose() {
    _controller.close();
    super.onClose();
  }
}
```

```dart
// lib/core/event_bus/app_event.dart
sealed class AppEvent {}

// ── Session Events ──────────────────────────────
class SessionInitializedEvent extends AppEvent {
  final Session session;
  final AccountId accountId;
  SessionInitializedEvent(this.session, this.accountId);
}

class SessionExpiredEvent extends AppEvent {}
class SessionLoggedOutEvent extends AppEvent {}

// ── Mailbox Events ──────────────────────────────
class MailboxSelectedEvent extends AppEvent {
  final PresentationMailbox mailbox;
  MailboxSelectedEvent(this.mailbox);
}

class MailboxRefreshRequestedEvent extends AppEvent {}
class DefaultMailboxOpenedEvent extends AppEvent {}

class MailboxCreatedEvent extends AppEvent {
  final PresentationMailbox mailbox;
  MailboxCreatedEvent(this.mailbox);
}

class MailboxDeletedEvent extends AppEvent {
  final MailboxId mailboxId;
  MailboxDeletedEvent(this.mailboxId);
}

class MailboxRenamedEvent extends AppEvent {
  final MailboxId mailboxId;
  final String newName;
  MailboxRenamedEvent(this.mailboxId, this.newName);
}

class MailboxMovedEvent extends AppEvent {
  final MailboxId mailboxId;
  MailboxMovedEvent(this.mailboxId);
}

// ── Email List Events ────────────────────────────
class EmailListLoadedEvent extends AppEvent {
  final List<PresentationEmail> emails;
  EmailListLoadedEvent(this.emails);
}

class EmailListRefreshRequestedEvent extends AppEvent {}

// ── Email Action Events ──────────────────────────
class EmailActionRequestedEvent extends AppEvent {
  final EmailActionType actionType;
  final PresentationEmail email;
  final EmailActionSourceContext sourceContext;
  EmailActionRequestedEvent({
    required this.actionType,
    required this.email,
    required this.sourceContext,
  });
}

class EmailBulkActionRequestedEvent extends AppEvent {
  final EmailActionType actionType;
  final List<PresentationEmail> emails;
  final EmailActionSourceContext sourceContext;
  EmailBulkActionRequestedEvent({
    required this.actionType,
    required this.emails,
    required this.sourceContext,
  });
}

class EmailMovedEvent extends AppEvent {
  final List<EmailId> emailIds;
  final MailboxId destination;
  EmailMovedEvent(this.emailIds, this.destination);
}

class EmailDeletedEvent extends AppEvent {
  final List<EmailId> emailIds;
  EmailDeletedEvent(this.emailIds);
}

class EmailReadStatusChangedEvent extends AppEvent {
  final List<EmailId> emailIds;
  final bool isRead;
  EmailReadStatusChangedEvent(this.emailIds, {required this.isRead});
}

class EmailReadRequestedEvent extends AppEvent {
  final EmailId emailId;
  final ReadActions action;
  EmailReadRequestedEvent(this.emailId, this.action);
}

class EmailNavigationRequestedEvent extends AppEvent {
  final EmailId emailId;
  EmailNavigationRequestedEvent(this.emailId);
}

class EmailSentEvent extends AppEvent {
  final EmailId? emailId;
  EmailSentEvent(this.emailId);
}

// ── Search Events ────────────────────────────────
class SearchActivatedEvent extends AppEvent {
  final SearchQuery query;
  SearchActivatedEvent(this.query);
}

class SearchClearedEvent extends AppEvent {}

// ── Composer Events ──────────────────────────────
class ComposerOpenRequestedEvent extends AppEvent {
  final ComposerArguments arguments;
  ComposerOpenRequestedEvent(this.arguments);
}

class ComposerClosedEvent extends AppEvent {}

// ── Calendar Events ──────────────────────────────
class CalendarEventRepliedEvent extends AppEvent {
  final CalendarEventId eventId;
  CalendarEventRepliedEvent(this.eventId);
}

// ── Filter Events ────────────────────────────────
class DateFilterAppliedEvent extends AppEvent {
  final DateRangeFilter range;
  DateFilterAppliedEvent(this.range);
}

class DateFilterRemovedEvent extends AppEvent {}
class LabelFilterRemovedEvent extends AppEvent {}
class MailboxFilterRemovedEvent extends AppEvent {}

class ContactFilterRemovedEvent extends AppEvent {
  final SearchFilterType filterType;
  ContactFilterRemovedEvent(this.filterType);
}

class KeywordFilterRemovedEvent extends AppEvent {
  final SearchFilterType filterType;
  KeywordFilterRemovedEvent(this.filterType);
}
```

### Subscription Lifecycle Helper

```dart
// lib/core/event_bus/composite_subscription.dart
extension CompositeSubscriptionExt on StreamSubscription {
  void addTo(List<StreamSubscription> subscriptions) {
    subscriptions.add(this);
  }
}

// Mixin để controllers tự động cleanup subscriptions
mixin EventBusSubscriberMixin on GetxController {
  final List<StreamSubscription> _subscriptions = [];

  List<StreamSubscription> get compositeSubscription => _subscriptions;

  @override
  void onClose() {
    for (final sub in _subscriptions) {
      sub.cancel();
    }
    _subscriptions.clear();
    super.onClose();
  }
}
```

### Cách dùng trong controller

```dart
class ThreadController extends BaseController with EventBusSubscriberMixin {
  final AppEventBus _eventBus;

  @override
  void onInit() {
    super.onInit();
    _eventBus.stream
        .whereType<MailboxSelectedEvent>()
        .listen(_onMailboxSelected)
        .addTo(compositeSubscription);  // tự cleanup khi controller dispose
  }
}
```

---

## 2. GetxService Shared State

### EmailListStateService

```dart
// lib/features/mailbox_dashboard/presentation/service/email_list_state_service.dart
class EmailListStateService extends GetxService {
  // Email list state
  final RxList<PresentationEmail> emails = <PresentationEmail>[].obs;
  final RxBool isLoading = false.obs;
  final Rxn<PresentationMailbox> selectedMailbox = Rxn();
  final Rx<EmailSortOrder> sortOrder = Rx(EmailSortOrder.mostRecent);
  final Rx<FilterMessageOption> filterOption = Rx(FilterMessageOption.all);

  // Mailbox maps (moved from MailboxDashBoardController)
  Map<Role, MailboxId> _mailboxIdByRole = {};
  Map<MailboxId, PresentationMailbox> _mailboxById = {};

  Map<Role, MailboxId> get mailboxIdByRole =>
      Map.unmodifiable(_mailboxIdByRole);
  Map<MailboxId, PresentationMailbox> get mailboxById =>
      Map.unmodifiable(_mailboxById);

  void updateEmails(List<PresentationEmail> list) {
    emails.assignAll(list);
  }

  void updateMailboxMaps({
    required Map<Role, MailboxId> byRole,
    required Map<MailboxId, PresentationMailbox> byId,
  }) {
    _mailboxIdByRole = Map.unmodifiable(byRole);
    _mailboxById = Map.unmodifiable(byId);
  }

  MailboxId? getMailboxIdByRole(Role role) => _mailboxIdByRole[role];
}
```

### ComposerStateService

```dart
// lib/features/mailbox_dashboard/presentation/service/composer_state_service.dart
class ComposerStateService extends GetxService {
  final RxBool isOpen = false.obs;
  final Rxn<ComposerArguments> arguments = Rxn();

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

  void startEmailDrag(PresentationEmail email) {
    draggingEmail.value = email;
    isDraggingEmail.value = true;
  }

  void startMailboxDrag() {
    isDraggingMailbox.value = true;
  }

  void endDrag() {
    isDraggingEmail.value = false;
    isDraggingMailbox.value = false;
    draggingEmail.value = null;
    dropTarget.value = null;
  }

  void setDropTarget(PresentationMailbox? mailbox) {
    dropTarget.value = mailbox;
  }
}
```

### GetxService vs GetxController — Khi nào dùng gì?

```
 GetxController                    │  GetxService (permanent: true)
 ────────────────────────────────  │  ────────────────────────────────
 [onInit]                          │  [onInit]
    │                              │     │
 [onReady]                         │  [onReady]
    │                              │     │
 Business logic / UI state         │  Shared state / App-level services
    │                              │     │
 [onClose]                         │  Không có onClose tự động
    │                              │     │
 Disposed khi route pop            │  Tồn tại suốt app lifetime
    │                              │     │
 ❌ State mất khi navigate away    │  ✅ State preserved across routes
 ────────────────────────────────  │  ────────────────────────────────
 Dùng cho:                         │  Dùng cho:
   MailboxController                │    AppEventBus
   ThreadController                 │    EmailListStateService
   SingleEmailController            │    ComposerStateService
   SearchEmailController            │    DragDropStateService
   Feature-specific controllers     │    (SessionController nếu cần)
```

---

## 3. CoreBindings — đăng ký tất cả

```dart
// lib/main/bindings/core_bindings.dart
class CoreBindings extends Bindings {
  @override
  void dependencies() {
    // ── Event Bus ──────────────────────────────────
    Get.put<AppEventBus>(AppEventBus(), permanent: true);

    // ── Auth Abstractions ───────────────────────────
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
        Get.find<DeleteAuthorityOidcInteractor>(),
      ),
      permanent: true,
    );

    // ── Shared State Services ────────────────────────
    // (Đăng ký trong DashboardBindings, không ở đây
    //  vì chỉ cần khi vào màn hình chính)
  }
}
```

```dart
// lib/features/mailbox_dashboard/presentation/bindings/dashboard_bindings.dart
class MailboxDashboardBindings extends Bindings {
  @override
  void dependencies() {
    // Shared state services — permanent, không bị dispose
    Get.put<EmailListStateService>(EmailListStateService(), permanent: true);
    Get.put<ComposerStateService>(ComposerStateService(), permanent: true);
    Get.put<DragDropStateService>(DragDropStateService(), permanent: true);

    // Sub-controllers
    Get.lazyPut(() => EmailActionController(
      Get.find(), Get.find(), Get.find(),
      Get.find(), Get.find(), Get.find(), Get.find(),
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

    // Orchestrator
    Get.lazyPut(() => MailboxDashBoardController(
      Get.find<EmailListStateService>(),
      Get.find<ComposerStateService>(),
      Get.find<DragDropStateService>(),
      Get.find<AppEventBus>(),
      Get.find<ISessionTokenRefresher>(),
      Get.find<ILanguageSettingService>(),
      Get.find<IAuthService>(),
    ));

    // Feature controllers
    Get.lazyPut(() => MailboxController(
      Get.find<EmailListStateService>(), // implements IEmailListStateContract
      Get.find<MailboxNavigationController>(), // implements IMailboxNavigationContract
      Get.find<AppEventBus>(),
      Get.find(), Get.find(), Get.find(), Get.find(), Get.find(),
      Get.find<ISessionTokenRefresher>(),
      Get.find<ILanguageSettingService>(),
      Get.find<IAuthService>(),
    ));

    Get.lazyPut(() => ThreadController(
      Get.find<EmailListStateService>(),
      Get.find<AppEventBus>(),
      Get.find(), Get.find(), Get.find(), Get.find(),
      Get.find<ISessionTokenRefresher>(),
      Get.find<ILanguageSettingService>(),
      Get.find<IAuthService>(),
    ));
  }
}
```

---

## 4. Domain Contracts

```dart
// lib/features/mailbox_dashboard/domain/contracts/email_list_contract.dart
abstract class IEmailListStateContract {
  RxList<PresentationEmail> get emails;
  RxBool get isLoading;
  Rxn<PresentationMailbox> get selectedMailbox;
  Map<Role, MailboxId> get mailboxIdByRole;
  Map<MailboxId, PresentationMailbox> get mailboxById;
  void updateEmails(List<PresentationEmail> list);
  void updateMailboxMaps({
    required Map<Role, MailboxId> byRole,
    required Map<MailboxId, PresentationMailbox> byId,
  });
}

// lib/features/mailbox_dashboard/domain/contracts/mailbox_navigation_contract.dart
abstract class IMailboxNavigationContract {
  Rxn<PresentationMailbox> get selectedMailbox;
  void selectMailbox(PresentationMailbox mailbox);
  void openDefaultMailbox();
  void navigateToEmail(EmailId emailId);
}

// lib/features/mailbox_dashboard/domain/contracts/composer_visibility_contract.dart
abstract class IComposerVisibilityContract {
  RxBool get isOpen;
  void open(ComposerArguments args);
  void close();
}
```

`EmailListStateService` implement `IEmailListStateContract`.
`MailboxNavigationController` implement `IMailboxNavigationContract`.
`ComposerStateService` implement `IComposerVisibilityContract`.

### Sơ đồ phân cấp Domain Contracts

```
 Domain Contracts — Dependency Inversion
 ════════════════════════════════════════════════════════════════════

 domain/contracts/ (abstractions — không có implementation)
 ┌────────────────────────────┐  ┌──────────────────────────────┐
 │  IEmailListStateContract   │  │  IMailboxNavigationContract  │
 │   emails                   │  │   selectedMailbox            │
 │   isLoading                │  │   selectMailbox()            │
 │   selectedMailbox          │  │   openDefaultMailbox()       │
 │   updateEmails()           │  │   navigateToEmail()          │
 └─────────────┬──────────────┘  └─────────────┬────────────────┘
               │ implements                     │ implements
               ▼                               ▼
 EmailListStateService              MailboxNavigationController
  (GetxService · permanent)          (GetxController)

 Consumers depend on contracts only — KHÔNG depend vào implementations:
 ┌─────────────────────────────────────────────────────────────────┐
 │ ThreadController(IEmailListStateContract emailListState, ...)    │
 │ MailboxController(IEmailListStateContract, IMailboxNavContract)  │
 └─────────────────────────────────────────────────────────────────┘

 Lợi ích:
   - Swap implementation mà không đổi consumer
   - Test: inject mock contract thay vì mock toàn bộ service
   - Feature A không import trực tiếp Feature B
```

---

## 5. Lợi ích tổng hợp

### Trước → Sau

| Metric | Trước | Sau |
|---|---|---|
| Inter-controller coupling | `Get.find<MailboxDashBoardController>()` 75+ times | 0 — contracts + EventBus |
| `Rxn<UIAction>` observables | ~15 trong codebase | 0 |
| `ever(dashboard.someAction, ...)` | ~30 calls | 0 |
| Test setUp (mỗi controller) | 40+ `Get.put()` | ≤5 mock assignments |
| Circular dependencies risk | High (A→B→A) | None (A→EventBus←B) |
| Subscription cleanup | Manual, error-prone | `compositeSubscription` auto-cleanup |

### Testing trở nên đơn giản

```dart
// Trước: cần mock toàn bộ MailboxDashBoardController
setUp(() {
  Get.put<MailboxDashBoardController>(mockDashboard); // complex mock
  Get.put<AuthorizationInterceptors>(mockAuth);
  // ... 38 lines nữa
});

// Sau: chỉ cần mock EventBus và service
setUp(() {
  final eventBus = AppEventBus();
  final emailListState = EmailListStateService();
  Get.put<AppEventBus>(eventBus);
  Get.put<EmailListStateService>(emailListState);

  controller = ThreadController(
    emailListState,
    eventBus,
    mockGetEmails,
    mockRefreshEmails,
    mockLoadMore,
    mockCleanGet,
    mockTokenRefresher,
    mockLanguageService,
    mockAuthService,
  );
});

// Test event flow:
test('should load emails when mailbox selected', () async {
  when(mockGetEmails.execute(any, any, any))
      .thenAnswer((_) => Stream.value(Right(GetAllEmailSuccess(emails: testEmails))));

  eventBus.dispatch(MailboxSelectedEvent(testMailbox));
  await Future.delayed(Duration.zero); // let async complete

  expect(emailListState.emails, equals(testEmails));
});
```
