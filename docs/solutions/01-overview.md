# Solution Overview — Architecture Refactoring

**Date:** 2026-04-10
**Constraint:** Giữ nguyên GetX (GetxController, .obs, Obx, GetPage, Get.toNamed)

---

## Files giải pháp chi tiết

| File | Controller | Nội dung |
|---|---|---|
| `02-base-controller.md` | BaseController | Tách domains, clean architecture fix |
| `03-mailbox-dashboard-controller.md` | MailboxDashboardController | God Object decomposition, EventBus |
| `04-mailbox-controller.md` | MailboxController | Feature envy fix, sub-responsibilities |
| `05-thread-controller.md` | ThreadController | Bumpy road, deep nesting, feature envy |
| `06-single-email-controller.md` | SingleEmailController | Optional deps, large switch |
| `07-search-email-controller.md` | SearchEmailController | State model, mixin cleanup |
| `08-shared-infrastructure.md` | AppEventBus + GetxService | Cross-cutting infrastructure |

---

## Kiến trúc tổng quan đề xuất

### Trước — Tight Coupling, God Object

```
 TRƯỚC (hiện tại) — Tight Coupling, God Object
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

### Sau — Decoupled, Single Responsibility

```
 SAU (target) — Decoupled, Event-Driven, Single Responsibility
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

## 2 Patterns cốt lõi — AppEventBus + GetxService

### Pattern 1: AppEventBus thay thế `Rxn<UIAction>`

**Vấn đề hiện tại:**
```dart
// Implicit, không type-safe, tight coupling
ever(mailboxDashBoardController.dashBoardAction, (action) {
  if (action is SelectEmailAction) { ... }
  else if (action is RefreshAction) { ... }
  // ... 11 types nữa trong cùng một closure
});
```

**Giải pháp:**
```dart
// AppEventBus — đăng ký 1 lần trong CoreBindings
class AppEventBus extends GetxService {
  final _stream = StreamController<AppEvent>.broadcast();
  Stream<AppEvent> get stream => _stream.stream;
  void dispatch(AppEvent event) => _stream.add(event);
}

// Mỗi controller subscribe chỉ những event type mà nó cần
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

### Pattern 2: GetxService cho shared state thay thế God Object

**Vấn đề hiện tại:**
```dart
// Tất cả state nằm trong MailboxDashBoardController
mailboxDashBoardController.emailList     // ThreadController đọc
mailboxDashBoardController.isLoading     // MailboxController đọc
mailboxDashBoardController.selectedMailbox // 5 controllers đọc
```

**Giải pháp:**
```dart
// State được chia vào các GetxService permanent singletons
class EmailListStateService extends GetxService {
  final RxList<PresentationEmail> emails = <PresentationEmail>[].obs;
  final RxBool isLoading = false.obs;
  final Rxn<PresentationMailbox> selectedMailbox = Rxn();
}

// Controllers nhận via constructor injection
class ThreadController extends BaseController {
  final EmailListStateService _emailListState;
  ThreadController(this._emailListState, ...);
}
```

---

## Nguyên tắc chung áp dụng cho tất cả giải pháp

1. **Constructor injection only** — không còn `Get.find<T>()` ngoài `*_bindings.dart`
2. **Mỗi controller ≤ 300 lines** — split nếu vượt
3. **Mỗi controller ≤ 8 constructor dependencies**
4. **Mỗi controller ≤ 10 Rx fields**
5. **Không còn `Rxn<UIAction>` observables** — thay bằng `AppEventBus`
6. **`handleSuccessViewState()` chỉ handle success types của bounded context của controller đó**
7. **`_registerObxStreamListener()` được tách thành nhiều `_subscribeToXEvents()` methods**
8. **Không còn `Get.find<T>()` trong field declarations**

---

## Lộ trình implement

```
 Lộ trình Refactoring — ADR-0076
 ──────────────────────────────────────────────────────────────────────
       Apr       May        Jun        Jul-Aug     Sep-Oct     Nov
       W15-W18   W19-W24   W25-W28    W29-W36     W37-W44     W45+
 ──────────────────────────────────────────────────────────────────────
 Phase 0  ████████
 Foundation
   AppEventBus + AppEvent sealed class
   GetxService shells (EmailListState, ComposerState, DragDrop)
   domain/contracts/ interfaces
   BaseController clean-up (tách data layer deps)
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
 Decomposition                         Không làm trong release window
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
 Tổng: ~5–8 tháng · Roadmap không bị block (chạy song song)
```

## So sánh giao tiếp giữa controllers: Trước vs Sau

### Trước — `Rxn<UIAction>` tạo Tight Coupling

```
 TRƯỚC — Rxn<UIAction>: Tight Coupling
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
 ⚠ Vấn đề:
   - Mỗi controller subscribe toàn bộ observable của Dashboard
   - Callback là 85–159 line closures với if-else chain
   - Mọi controller bị ảnh hưởng dù action không liên quan
```

### Sau — `AppEventBus` Decoupled

```
 SAU — AppEventBus: Decoupled
 ════════════════════════════════════════════════════════════════════
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
       │                                                   │
 ✅ Lợi ích:
   - Mỗi controller chỉ nhận event type mình cần
   - Không còn 159-line closures, không còn if-else chain
   - Thêm event mới không ảnh hưởng controller không liên quan
```

Xem chi tiết từng controller trong các files `02-` đến `08-` trong thư mục `solutions/`.
