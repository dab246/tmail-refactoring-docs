# Technical: What Is EventBus?

> Solution for **Problem B — Inter-Controller Communication**

---

## Definition

EventBus is a **Design Pattern** (not a specific framework) — a mediator mechanism that allows components within an app to communicate with each other **without needing to know each other exists**.

```
 Without EventBus (tight coupling):
 ──────────────────────────────────
 Controller A   →  knows about  →  Controller B
 Controller A   →  knows about  →  Controller C
 Controller B   →  knows about  →  Controller A
 (Web of dependencies)


 With EventBus (loose coupling):
 ──────────────────────────────────
 Controller A  →  dispatch(EventX)  →  EventBus
                                           │
                                    filter by type
                                           │
                                    Controller B (subscribed to EventX)
                                    Controller C (subscribed to EventX)
 Controller A does not know B and C exist.
 B and C do not know A dispatches.
```

---

## How It Works in This Codebase

```
 AppEventBus (GetxService · permanent singleton)
 ═══════════════════════════════════════════════
 1. All events are typed into a sealed class AppEvent
    └─ MailboxSelectedEvent
    └─ EmailMovedEvent
    └─ SessionInitializedEvent
    └─ ...

 2. Publisher calls dispatch():
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

 3. Subscriber uses whereType<T>():
    _eventBus.stream
        .whereType<MailboxSelectedEvent>()  ← receives only this type
        .listen(_onMailboxSelected)
        .addTo(compositeSubscription)       ← auto cleanup on dispose
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
    compositeSubscription.dispose(); // auto cancels all subscriptions
    super.onClose();
  }
}

// Convenience extension
extension StreamSubscriptionX<T> on StreamSubscription<T> {
  void addTo(CompositeSubscription composite) {
    composite.add(this);
  }
}
```

---

## How to Subscribe in a Controller

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
        .whereType<MailboxSelectedEvent>()   // ← receives only this event
        .listen(_onMailboxSelected)
        .addTo(compositeSubscription);       // ← auto cleanup
  }

  void _subscribeToSearchEvents() {
    _eventBus.stream
        .whereType<SearchActivatedEvent>()
        .listen((_) => _onSearchActivated())
        .addTo(compositeSubscription);
  }

  void _onMailboxSelected(MailboxSelectedEvent event) {
    // handles a single event type — no more if-else chains
    _loadEmails(event.mailbox);
  }
}
```

---

## Comparison with the Current Approach (Rxn\<UIAction\>)

```
 BEFORE: ever(rxn<UIAction>, callback) — Tight Coupling
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

 ⚠ Problems:
   - Each controller subscribes to the entire observable of Dashboard
   - Callbacks are 85–159 line closures with if-else chains
   - Every controller is affected even if the action is unrelated


 AFTER: AppEventBus — Decoupled
 ════════════════════════════════════════════════════════
 AppEventBus    MNC (dispatch)   MailboxController   ThreadController
       │               │               │                   │
       │◄── subscribe MailboxSelectedEvent ───────────────►│
       │◄── subscribe MailboxSelectedEvent ────────────────────────►│ (TC)
       │               │               │                   │
       │  User selects Inbox mailbox   │                   │
       │◄── dispatch(MailboxSelectedEvent(inbox))          │
       │               │               │                   │
       ├───────────────────────────────► ✅ _onMailboxSelected
       ├──────────────────────────────────────────────────► ✅ load emails
       │               │               │                   │
       │  User deletes emails          │                   │
       │◄── dispatch(EmailBulkActionRequestedEvent)        │
       │               │               │                   │
       │  (MC not subscribed → does not receive)           │
       │  (TC not subscribed → does not receive)           │

 ✅ Benefits:
   - Each controller only receives the event types it needs
   - No more 159-line closures, no more if-else chains
   - Adding a new event does not affect unrelated controllers
```

---

## Testability

```dart
// Test ThreadController with mock EventBus
test('loads emails when mailbox selected', () async {
  final mockEventBus = MockAppEventBus();
  final controller = ThreadController(mockEventBus, ...);

  // Emit test event directly
  final mailboxStream = Stream.value(MailboxSelectedEvent(inboxMailbox));
  when(mockEventBus.stream).thenAnswer((_) => mailboxStream);

  controller.onInit();

  verify(controller._loadEmails(inboxMailbox)).called(1);
});
// No need to mock the entire MailboxDashBoardController
```
