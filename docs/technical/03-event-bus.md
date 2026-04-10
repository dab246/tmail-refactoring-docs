# Technical: EventBus là gì?

> Giải pháp cho **Vấn đề B — Inter-Controller Communication**

---

## Định nghĩa

EventBus là một **Design Pattern** (không phải framework cụ thể) — một cơ chế trung gian cho phép các thành phần trong app giao tiếp với nhau mà **không cần biết nhau tồn tại**.

```
 Không có EventBus (tight coupling):
 ──────────────────────────────────
 Controller A   →  biết về  →  Controller B
 Controller A   →  biết về  →  Controller C
 Controller B   →  biết về  →  Controller A
 (Web of dependencies)


 Có EventBus (loose coupling):
 ──────────────────────────────
 Controller A  →  dispatch(EventX)  →  EventBus
                                           │
                                    filter by type
                                           │
                                    Controller B (subscribed to EventX)
                                    Controller C (subscribed to EventX)
 Controller A không biết B và C tồn tại.
 B và C không biết A dispatch.
```

---

## Cơ chế hoạt động trong codebase này

```
 AppEventBus (GetxService · permanent singleton)
 ═══════════════════════════════════════════════
 1. Tất cả events được type vào sealed class AppEvent
    └─ MailboxSelectedEvent
    └─ EmailMovedEvent
    └─ SessionInitializedEvent
    └─ ...

 2. Publisher gọi dispatch():
    MailboxNavigationController:
      _eventBus.dispatch(MailboxSelectedEvent(mailbox))
                    │
                    ▼
            StreamController.broadcast()
                    │
          filter by whereType<T>()
                    │
         ┌──────────┴──────────┐
         ▼                     ▼
    MailboxController    ThreadController
    (subscribed to       (subscribed to
    MailboxSelected)     MailboxSelected)

 3. Subscriber dùng whereType<T>():
    _eventBus.stream
        .whereType<MailboxSelectedEvent>()  ← chỉ nhận đúng type
        .listen(_onMailboxSelected)
        .addTo(compositeSubscription)       ← tự cleanup khi dispose
```

---

## Implementation

### AppEventBus

```dart
// lib/features/base/service/app_event_bus.dart
class AppEventBus extends GetxService {
  final _controller = StreamController<AppEvent>.broadcast();

  Stream<AppEvent> get stream => _controller.stream;

  void dispatch(AppEvent event) {
    _controller.add(event);
  }

  @override
  void onClose() {
    _controller.close();
    super.onClose();
  }
}
```

### AppEvent sealed class

```dart
// lib/features/base/event/app_event.dart
sealed class AppEvent {}

// Session events
final class SessionInitializedEvent extends AppEvent {
  final Session session;
  final AccountId accountId;
  SessionInitializedEvent(this.session, this.accountId);
}

final class SessionExpiredEvent extends AppEvent {}

// Mailbox events
final class MailboxSelectedEvent extends AppEvent {
  final PresentationMailbox mailbox;
  MailboxSelectedEvent(this.mailbox);
}

final class MailboxesLoadedEvent extends AppEvent {
  final Map<Role, MailboxId> byRole;
  final Map<MailboxId, PresentationMailbox> byId;
  MailboxesLoadedEvent(this.byRole, this.byId);
}

// Email events
final class EmailMovedEvent extends AppEvent {
  final List<EmailId> emailIds;
  EmailMovedEvent(this.emailIds);
}

final class EmailDeletedEvent extends AppEvent {
  final List<EmailId> emailIds;
  EmailDeletedEvent(this.emailIds);
}

final class EmailBulkActionRequestedEvent extends AppEvent {
  final EmailActionType actionType;
  final List<PresentationEmail> emails;
  EmailBulkActionRequestedEvent({required this.actionType, required this.emails});
}

// Search events
final class SearchActivatedEvent extends AppEvent {}
final class SearchDeactivatedEvent extends AppEvent {}

// Composer events
final class ComposerOpenedEvent extends AppEvent {
  final ComposerArguments arguments;
  ComposerOpenedEvent(this.arguments);
}

final class ComposerClosedEvent extends AppEvent {}
```

### EventBusSubscriberMixin — auto cleanup

```dart
// lib/features/base/mixin/event_bus_subscriber_mixin.dart
mixin EventBusSubscriberMixin on GetxController {
  final compositeSubscription = CompositeSubscription();

  @override
  void onClose() {
    compositeSubscription.dispose(); // tự cancel tất cả subscriptions
    super.onClose();
  }
}

// Extension tiện lợi
extension StreamSubscriptionX<T> on StreamSubscription<T> {
  void addTo(CompositeSubscription composite) {
    composite.add(this);
  }
}
```

---

## Cách subscribe trong controller

```dart
class ThreadController extends BaseController with EventBusSubscriberMixin {
  final AppEventBus _eventBus;

  ThreadController(this._eventBus, ...);

  @override
  void onInit() {
    super.onInit();
    _subscribeToMailboxEvents();
    _subscribeToSearchEvents();
  }

  void _subscribeToMailboxEvents() {
    _eventBus.stream
        .whereType<MailboxSelectedEvent>()   // ← chỉ nhận event này
        .listen(_onMailboxSelected)
        .addTo(compositeSubscription);       // ← tự cleanup
  }

  void _subscribeToSearchEvents() {
    _eventBus.stream
        .whereType<SearchActivatedEvent>()
        .listen((_) => _onSearchActivated())
        .addTo(compositeSubscription);
  }

  void _onMailboxSelected(MailboxSelectedEvent event) {
    // xử lý 1 event type duy nhất — không còn if-else chain
    _loadEmails(event.mailbox);
  }
}
```

---

## So sánh với approach hiện tại (Rxn\<UIAction\>)

```
 TRƯỚC: ever(rxn<UIAction>, callback) — Tight Coupling
 ════════════════════════════════════════════════════════
 MDC (Dashboard)       MailboxController       ThreadController
       │                      │                      │
       │  Rxn<UIAction>        │                      │
       │  mailboxUIAction      │                      │
       │◄─── ever(mailboxUIAction, callback) ─────────┤ ← 7 action types
       │◄─── ever(dashBoardAction, callback) ─────────┤ ← 13 action types · 85 lines
       │                      │                      │
       │  dispatch(OpenMailboxAction())               │
       ├─────────────────────►│ fires callback        │
       │                      │ → if-else chain...   │
       ├──────────────────────────────────────────────► fires (unrelated — noise)

 ⚠ Vấn đề:
   - Mỗi controller subscribe toàn bộ observable của Dashboard
   - Callback là 85–159 line closures với if-else chain
   - Mọi controller bị ảnh hưởng dù action không liên quan


 SAU: AppEventBus — Decoupled
 ════════════════════════════════════════════════════════
 AppEventBus    MNC (dispatch)   MailboxController   ThreadController
       │               │               │                   │
       │◄── subscribe MailboxSelectedEvent ───────────────►│
       │◄── subscribe MailboxSelectedEvent ────────────────────────►│ (TC)
       │               │               │                   │
       │  User chọn mailbox Inbox      │                   │
       │◄── dispatch(MailboxSelectedEvent(inbox))          │
       │               │               │                   │
       ├───────────────────────────────► ✅ _onMailboxSelected
       ├──────────────────────────────────────────────────► ✅ load emails
       │               │               │                   │
       │  User xóa emails              │                   │
       │◄── dispatch(EmailBulkActionRequestedEvent)        │
       │               │               │                   │
       │  (MC không subscribe → không nhận)                │
       │  (TC không subscribe → không nhận)                │

 ✅ Lợi ích:
   - Mỗi controller chỉ nhận event type mình cần
   - Không còn 159-line closures, không còn if-else chain
   - Thêm event mới không ảnh hưởng controller không liên quan
```

---

## Testability

```dart
// Test ThreadController với mock EventBus
test('loads emails when mailbox selected', () async {
  final mockEventBus = MockAppEventBus();
  final controller = ThreadController(mockEventBus, ...);

  // Emit test event trực tiếp
  final mailboxStream = Stream.value(MailboxSelectedEvent(inboxMailbox));
  when(mockEventBus.stream).thenAnswer((_) => mailboxStream);

  controller.onInit();

  verify(controller._loadEmails(inboxMailbox)).called(1);
});
// Không cần mock toàn bộ MailboxDashBoardController
```
