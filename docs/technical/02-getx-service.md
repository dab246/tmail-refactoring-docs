# Technical: GetxService là gì?

> Giải pháp cho **Vấn đề A — Shared State**

---

## Định nghĩa

`GetxService` là một class trong framework GetX, kế thừa từ `GetLifeCycleMixin`, được thiết kế để giữ **state hoặc logic tồn tại suốt vòng đời của app** — không bị dispose khi route thay đổi.

```
 GetxController                      GetxService
 ─────────────────────────────────   ─────────────────────────────────
 Gắn với route lifecycle             Gắn với app lifecycle

 App start                           App start
   └─ Route push → onInit()            └─ Register once → onInit()
       └─ onReady()                         └─ onReady()
           └─ Business logic                    └─ Tồn tại mãi
               └─ Route pop → onClose()
                   └─ ❌ DISPOSED              ✅ NEVER disposed
                                               (trừ khi gọi Get.delete
                                                với force: true)
```

---

## Vòng đời chi tiết

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
 │      └─► onInit() lại từ đầu — state reset                     │
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
 │              ✅ Multiple controllers can Obx() vào              │
 │              │                                                  │
 │          App killed                                             │
 │              └─► onClose() — cleanup                           │
 └─────────────────────────────────────────────────────────────────┘
```

---

## Khi nào dùng GetxService?

| Câu hỏi | GetxController | GetxService |
|---|---|---|
| State có cần persist khi navigate away? | Không | **Có** |
| State được đọc bởi nhiều controllers? | Không | **Có** |
| Logic gắn với 1 screen cụ thể? | **Có** | Không |
| Cần test riêng lẻ không phụ thuộc route? | Khó | **Dễ** |

---

## Cách đăng ký

```dart
// Trong CoreBindings (chạy 1 lần khi app start)
class CoreBindings extends Bindings {
  @override
  void dependencies() {
    // permanent: true — không bao giờ bị auto-dispose
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

## Các GetxService được tạo trong refactoring này

```
 GetxService shard (permanent singletons)
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

## Ví dụ — EmailListStateService

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

## Controller nhận GetxService qua constructor injection

```dart
// ĐÚNG — constructor injection, testable
class ThreadController extends BaseController {
  final EmailListStateService _emailListState;

  ThreadController(this._emailListState, ...);

  void _onEmailsUpdated() {
    // Truy cập qua injected service
    final emails = _emailListState.emails;
    final mailbox = _emailListState.selectedMailbox.value;
  }
}

// SAI — Get.find() trong field declaration, hard to test
class ThreadController extends BaseController {
  final _emailListState = Get.find<EmailListStateService>(); // ❌
}
```

---

## So sánh với approach hiện tại (God Object)

```
 TRƯỚC: God Object làm shared state container
 ───────────────────────────────────────────────────────────
 MailboxDashBoardController {
   final emails = <PresentationEmail>[].obs;      // ThreadController reads
   final isLoading = false.obs;                   // MailboxController reads
   final selectedMailbox = Rxn<PresentationMailbox>(); // 5 controllers read
   // ... 39 Rx fields nữa từ 15 bounded contexts
 }
 → Không thể tách God Object vì tất cả controllers phụ thuộc

 SAU: GetxService tách biệt mối quan tâm
 ───────────────────────────────────────────────────────────
 EmailListStateService {
   final emails = <PresentationEmail>[].obs;
   final isLoading = false.obs;
   final selectedMailbox = Rxn<PresentationMailbox>();
 }
 → MailboxDashBoardController không còn là điểm tập trung state
 → Có thể inject service vào bất kỳ controller nào
 → Có thể test service độc lập
```
