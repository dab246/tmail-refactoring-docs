# Technical: What Is GetxService?

> Solution for **Problem A — Shared State**

---

## Definition

`GetxService` is a class in the GetX framework, inheriting from `GetLifeCycleMixin`, designed to hold **state or logic that persists for the entire app lifecycle** — it is not disposed when the route changes.

```
 GetxController                      GetxService
 ─────────────────────────────────   ─────────────────────────────────
 Tied to route lifecycle             Tied to app lifecycle

 App start                           App start
   └─ Route push → onInit()            └─ Register once → onInit()
       └─ onReady()                         └─ onReady()
           └─ Business logic                    └─ Lives forever
               └─ Route pop → onClose()
                   └─ ❌ DISPOSED              ✅ NEVER disposed
                                               (unless Get.delete
                                                is called with force: true)
```

---

## Detailed Lifecycle

```
 ┌─────────────────────────────────────────────────────────────────┐
 │                    GetxController lifecycle                     │
 │                                                                 │
 │  App Start                                                      │
 │      │                                                          │
 │      ├─► Get.lazyPut(() => MailboxController(...))              │
 │      │       │                                                  │
 │      │   Route /mailbox push                                    │
 │      │       ├─► onInit()     ← setup subscriptions            │
 │      │       ├─► onReady()    ← load initial data              │
 │      │       │                                                  │
 │      │   [User navigates away]                                  │
 │      │       └─► onClose()   ← cleanup                         │
 │      │           ❌ State LOST — Rx fields destroyed            │
 │      │                                                          │
 │  Route /mailbox push again                                      │
 │      └─► onInit() from the beginning again — state reset        │
 └─────────────────────────────────────────────────────────────────┘

 ┌─────────────────────────────────────────────────────────────────┐
 │                    GetxService lifecycle                        │
 │                                                                 │
 │  App Start                                                      │
 │      │                                                          │
 │      └─► Get.put(EmailListStateService(), permanent: true)      │
 │              ├─► onInit()    ← setup once                      │
 │              ├─► onReady()   ← initialize once                 │
 │              │                                                  │
 │          [Navigate anywhere, push/pop routes]                   │
 │              │                                                  │
 │              ✅ State PRESERVED — Rx fields still alive         │
 │              ✅ Observers still active                          │
 │              ✅ Multiple controllers can Obx() into it          │
 │              │                                                  │
 │          App killed                                             │
 │              └─► onClose() — cleanup                           │
 └─────────────────────────────────────────────────────────────────┘
```

---

## When to Use GetxService?

| Question | GetxController | GetxService |
|---|---|---|
| Does state need to persist when navigating away? | No | **Yes** |
| Is state read by multiple controllers? | No | **Yes** |
| Is logic tied to a specific screen? | **Yes** | No |
| Does it need to be tested independently without route dependency? | Hard | **Easy** |

---

## How to Register

```dart
// In CoreBindings (runs once when app starts)
class CoreBindings extends Bindings {
  @override
  void dependencies() {
    // permanent: true — never auto-disposed
    Get.put<EmailListStateService>(
      EmailListStateService(),
      permanent: true,
    );
    Get.put<ComposerStateService>(
      ComposerStateService(),
      permanent: true,
    );
    Get.put<DragDropStateService>(
      DragDropStateService(),
      permanent: true,
    );
    Get.put<AppEventBus>(
      AppEventBus(),
      permanent: true,
    );
  }
}
```

---

## GetxServices Created in This Refactoring

```
 GetxService shared (permanent singletons)
 ═══════════════════════════════════════════════════════════════
 ┌────────────────────────┐
 │  EmailListStateService │  ← emails, isLoading, selectedMailbox,
 │                        │    sortOrder, filterOption, mailboxMaps
 └────────────────────────┘

 ┌────────────────────────┐
 │  ComposerStateService  │  ← isOpen, arguments, isDraftSaving
 └────────────────────────┘

 ┌────────────────────────┐
 │  DragDropStateService  │  ← isDraggingEmail, isDraggingMailbox,
 │                        │    draggingEmail, dropTarget
 └────────────────────────┘

 ┌────────────────────────┐
 │  AppEventBus           │  ← broadcast stream, dispatch(), whereType<T>()
 └────────────────────────┘
```

---

## Example — EmailListStateService

```dart
// lib/features/mailbox_dashboard/presentation/service/email_list_state_service.dart
class EmailListStateService extends GetxService {
  final RxList<PresentationEmail> emails = <PresentationEmail>[].obs;
  final RxBool isLoading = false.obs;
  final Rxn<PresentationMailbox> selectedMailbox = Rxn();
  final Rx<EmailSortOrder> sortOrder = Rx(EmailSortOrder.mostRecent);
  final Rx<FilterMessageOption> filterOption = Rx(FilterMessageOption.all);

  // Mailbox maps — private, exposed via unmodifiable getters
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

---

## Controllers Receive GetxService via Constructor Injection

```dart
// CORRECT — constructor injection, testable
class ThreadController extends BaseController {
  final EmailListStateService _emailListState;

  ThreadController(this._emailListState, ...);

  void _onEmailsUpdated() {
    // Access via injected service
    final emails = _emailListState.emails;
    final mailbox = _emailListState.selectedMailbox.value;
  }
}

// WRONG — Get.find() in field declaration, hard to test
class ThreadController extends BaseController {
  final _emailListState = Get.find<EmailListStateService>(); // ❌
}
```

---

## Comparison with the Current Approach (God Object)

```
 BEFORE: God Object as shared state container
 ───────────────────────────────────────────────────────────
 MailboxDashBoardController {
   final emails = <PresentationEmail>[].obs;      // ThreadController reads
   final isLoading = false.obs;                   // MailboxController reads
   final selectedMailbox = Rxn<PresentationMailbox>(); // 5 controllers read
   // ... 39 more Rx fields from 15 bounded contexts
 }
 → God Object cannot be split because all controllers depend on it

 AFTER: GetxService separates concerns
 ───────────────────────────────────────────────────────────
 EmailListStateService {
   final emails = <PresentationEmail>[].obs;
   final isLoading = false.obs;
   final selectedMailbox = Rxn<PresentationMailbox>();
 }
 → MailboxDashBoardController is no longer the state concentration point
 → Service can be injected into any controller
 → Service can be tested independently
```
