# Solution Overview — Architecture Refactoring

**Date:** 2026-04-10
**Constraint:** Keep GetX as-is (GetxController, .obs, Obx, GetPage, Get.toNamed)

---

## Detailed solution files

| File | Controller | Content |
|---|---|---|
| `02-base-controller.md` | BaseController | Domain separation, clean architecture fix |
| `03-mailbox-dashboard-controller.md` | MailboxDashboardController | God Object decomposition, EventBus |
| `04-mailbox-controller.md` | MailboxController | Feature envy fix, sub-responsibilities |
| `05-thread-controller.md` | ThreadController | Bumpy road, deep nesting, feature envy |
| `06-single-email-controller.md` | SingleEmailController | Optional deps, large switch |
| `07-search-email-controller.md` | SearchEmailController | State model, mixin cleanup |
| `08-shared-infrastructure.md` | AppEventBus + GetxService | Cross-cutting infrastructure |

---

## Proposed high-level architecture

### Before — Tight Coupling, God Object

```
 BEFORE (current) — Tight Coupling, God Object
 ════════════════════════════════════════════════════════════════════
 ┌─────────────────────────────────────────────────────────────────┐
 │           MailboxDashBoardController  (3,508 lines)             │
 │  Email Actions · Sending Queue · Session · Composer Cache       │
 │  Spam · Recovery · Mailbox Ops · Preferences · Drag&Drop        │
 │  42 Rx fields · Rxn<UIAction> mailboxUIAction                   │
 │  29 interactors · Rxn<UIAction> dashBoardAction                 │
 │                  Rxn<UIAction> emailUIAction                     │
 └──────────────────────────┬──────────────────────────────────────┘
          ▲                  ▲                   ▲
   Get.find()          Get.find()           Get.find()
   75+ refs            ever(X) 85 lines     20+ refs
   ever(X) 159 lines   Feature Envy         delegate actions
          │                  │                   │
  MailboxController    ThreadController    SingleEmailController
   (1,602 lines)        (1,625 lines)       (1,630 lines)
                              ▲
                        Get.find()
                              │
                     SearchEmailController
                       (1,198 lines)
```

### After — Decoupled, Single Responsibility

```
 AFTER (target) — Decoupled, Event-Driven, Single Responsibility
 ════════════════════════════════════════════════════════════════════
                   ┌─────────────────────────────┐
                   │         AppEventBus          │ ← GetxService · permanent
                   │   sealed class AppEvent      │
                   │   dispatch / whereType<T>    │
                   └──────────────┬──────────────┘
          ┌───────────────────────┼───────────────────────┐
   dispatch│                subscribe│                subscribe│
          │                      │                      │
          ▼                      ▼                      ▼
 ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
 │ EmailAction      │  │ MailboxNavigation │  │  SendingQueue    │
 │ Controller       │  │ Controller        │  │  Controller      │
 │ ~300 lines       │  │ ~250 lines        │  │  ~200 lines      │
 └──────────────────┘  └──────────────────┘  └──────────────────┘

 GetxService (permanent) — Shared State
 ┌────────────────────┐  ┌───────────────────┐  ┌───────────────┐
 │ EmailListState     │  │ ComposerState      │  │ DragDrop      │
 │ Service            │  │ Service            │  │ StateService  │
 │ emails · isLoading │  │ isOpen · arguments │  │ isDragging    │
 │ selectedMailbox    │  │                   │  │ dropTarget    │
 └────────────────────┘  └───────────────────┘  └───────────────┘

 ┌────────────────┐  ┌────────────────┐  ┌────────────────┐
 │MailboxDashBoard│  │MailboxController│  │ThreadController│
 │Controller      │  │~600 lines       │  │~500 lines      │
 │~200 lines      │  │inject ELS       │  │inject ELS      │
 │(orchestrator)  │  │subscribe EventBus│  │subscribe EventBus│
 └────────────────┘  └────────────────┘  └────────────────┘
```

---

## 2 Core Patterns — AppEventBus + GetxService

### Pattern 1: AppEventBus replaces `Rxn<UIAction>`

**Current problem:**
```dart
// Implicit, not type-safe, tight coupling
ever(mailboxDashBoardController.dashBoardAction, (action) {
  if (action is SelectEmailAction) { ... }
  else if (action is RefreshAction) { ... }
  // ... 11 more types in the same closure
});
```

**Solution:**
```dart
// AppEventBus — registered once in CoreBindings
class AppEventBus extends GetxService {
  final _stream = StreamController<AppEvent>.broadcast();
  Stream<AppEvent> get stream => _stream.stream;
  void dispatch(AppEvent event) => _stream.add(event);
}

// Each controller subscribes only to the event types it needs
class ThreadController extends BaseController {
  @override
  void onInit() {
    super.onInit();
    _eventBus.stream.whereType<MailboxSelectedEvent>()
        .listen(_onMailboxSelected)
        .addTo(compositeSubscription);
    _eventBus.stream.whereType<EmailSearchActivatedEvent>()
        .listen(_onSearchActivated)
        .addTo(compositeSubscription);
  }
}
```

### Pattern 2: GetxService for shared state replaces God Object

**Current problem:**
```dart
// All state lives in MailboxDashBoardController
mailboxDashBoardController.emailList     // read by ThreadController
mailboxDashBoardController.isLoading     // read by MailboxController
mailboxDashBoardController.selectedMailbox // read by 5 controllers
```

**Solution:**
```dart
// State is distributed into permanent GetxService singletons
class EmailListStateService extends GetxService {
  final RxList<PresentationEmail> emails = <PresentationEmail>[].obs;
  final RxBool isLoading = false.obs;
  final Rxn<PresentationMailbox> selectedMailbox = Rxn();
}

// Controllers receive it via constructor injection
class ThreadController extends BaseController {
  final EmailListStateService _emailListState;
  ThreadController(this._emailListState, ...);
}
```

---

## General principles applied to all solutions

1. **Constructor injection only** — no more `Get.find<T>()` outside `*_bindings.dart`
2. **Each controller ≤ 300 lines** — split if exceeded
3. **Each controller ≤ 8 constructor dependencies**
4. **Each controller ≤ 10 Rx fields**
5. **No more `Rxn<UIAction>` observables** — replaced by `AppEventBus`
6. **`handleSuccessViewState()` only handles success types belonging to that controller's bounded context**
7. **`_registerObxStreamListener()` is split into multiple `_subscribeToXEvents()` methods**
8. **No more `Get.find<T>()` in field declarations**

---

## Implementation roadmap

```
 Refactoring Roadmap — ADR-0076
 ──────────────────────────────────────────────────────────────────────
       Apr       May        Jun        Jul-Aug     Sep-Oct     Nov
       W15-W18   W19-W24   W25-W28    W29-W36     W37-W44     W45+
 ──────────────────────────────────────────────────────────────────────
 Phase 0  ████████
 Foundation
   AppEventBus + AppEvent sealed class
   GetxService shells (EmailListState, ComposerState, DragDrop)
   domain/contracts/ interfaces
   BaseController clean-up (separate data layer deps)
 ──────────────────────────────────────────────────────────────────────
 Phase 1           ████████████████
 Constructor
 Injection
   Leaf controllers: ThreadDetail, SearchEmail
   Mid-tier: Thread, SingleEmail, Mailbox
   (Defer MailboxDashboard + Composer → Phase 3)
 ──────────────────────────────────────────────────────────────────────
 Phase 2                    ████████████
 Decompose
 State
   Move .obs fields → GetxService
   Replace ever(dashboard.X) → EventBus
   Remove Rxn<UIAction>
 ──────────────────────────────────────────────────────────────────────
 Phase 3                               ████████████████
 God Object                            ⚠ HIGH RISK
 Decomposition                         Do not do during a release window
   Extract EmailActionController
   Extract SendingQueueController
   Extract MailboxNavigationController
   Extract SessionController
   Extract CalendarEmailController
 ──────────────────────────────────────────────────────────────────────
 Phase 4                                              ████████
 Cleanup
   Remove all remaining Get.find in field declarations
   Remove all Rxn<UIAction>
   Full regression pass: Android + iOS + Web
 ──────────────────────────────────────────────────────────────────────
 Total: ~5–8 months · Roadmap is non-blocking (runs in parallel)
```

## Controller communication comparison: Before vs After

### Before — `Rxn<UIAction>` creates Tight Coupling

```
 BEFORE — Rxn<UIAction>: Tight Coupling
 ════════════════════════════════════════════════════════════════════
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
       │                      │                      │
 ⚠ Problems:
   - Each controller subscribes to the entire Dashboard observable
   - Callbacks are 85–159 line closures with if-else chains
   - Every controller is affected even when the action is unrelated
```

### After — `AppEventBus` Decoupled

```
 AFTER — AppEventBus: Decoupled
 ════════════════════════════════════════════════════════════════════
 AppEventBus    MNC (dispatch)   MailboxController   ThreadController
       │               │               │                   │
       │◄── subscribe MailboxSelectedEvent ───────────────►│
       │◄── subscribe MailboxSelectedEvent ────────────────────────►│ (TC)
       │               │               │                   │
       │  User selects mailbox Inbox   │                   │
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
       │                                                   │
 ✅ Benefits:
   - Each controller only receives the event types it needs
   - No more 159-line closures, no more if-else chains
   - Adding new events does not affect unrelated controllers
```

See each controller in detail in files `02-` through `08-` in the `solutions/` directory.