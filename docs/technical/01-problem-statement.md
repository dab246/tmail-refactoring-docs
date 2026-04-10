# Technical: Bài toán cần giải quyết

**Date:** 2026-04-10
**Context:** Refactoring tmail-flutter — giải quyết God Object, Feature Envy, Tight Coupling

---

## 2 vấn đề kỹ thuật cốt lõi

Có **2 vấn đề kỹ thuật cốt lõi** cần giải quyết đồng thời:

```
 Vấn đề A: SHARED STATE                Vấn đề B: INTER-CONTROLLER COMMUNICATION
 ──────────────────────────────────    ──────────────────────────────────────────
 "State của app nằm ở đâu?"            "Controller A thông báo cho B thế nào?"

 Hiện tại: MailboxDashBoardController  Hiện tại: Get.find<MailboxDashBoardController>()
 giữ toàn bộ state (42 Rx fields)      + ever(dashboard.rxnAction, callback)

 Hệ quả:                               Hệ quả:
  - 5 controllers phụ thuộc vào         - Callback là 85–159 line closures
    1 God Object để đọc state           - Tight coupling: A phải biết B tồn tại
  - God Object không thể xóa           - Không type-safe: Rxn<UIAction>
  - Test cần mock toàn bộ God Object    - Mọi controller nhận noise từ unrelated actions
```

Cả 2 vấn đề này **liên quan nhau** — giải quyết A không xong nếu chưa giải quyết B.

---

## Minh họa vấn đề A — Shared State hiện tại

```
 TRƯỚC: Tất cả state tập trung vào God Object
 ═══════════════════════════════════════════════════════════════
          ┌──────────────────────────────────────────┐
          │       MailboxDashBoardController          │
          │  emailList · isLoading · selectedMailbox  │
          │  accountId · session · listIdentities     │
          │  42 Rx fields tổng cộng                   │
          └───────────────┬──────────────────────────┘
                          │
         ┌────────────────┼────────────────┐
         │                │                │
   Get.find()       Get.find()       Get.find()
         │                │                │
         ▼                ▼                ▼
 MailboxController  ThreadController  SingleEmailController
 đọc selectedMailbox đọc emailList    đọc listIdentities
 ghi mailboxMaps     ghi emailList!   đọc accountId

 ⚠ Hệ quả: Không thể xóa/tách God Object vì 5 controllers
            đang phụ thuộc trực tiếp vào nó
```

---

## Minh họa vấn đề B — Inter-Controller Communication hiện tại

```
 TRƯỚC: ever(rxnAction, callback) — 85 đến 159 lines
 ═══════════════════════════════════════════════════════════════
 ThreadController:
   ever(mailboxDashBoardController.dashBoardAction, (action) {
     if (action is SelectEmailAction) { ... }       // email concern
     else if (action is RefreshEmailAction) { ... } // email concern
     else if (action is StartSearchEmailAction) { ... } // search concern
     else if (action is CancelSearchEmailAction) { ... } // search concern
     else if (action is FilterMessageAction) { ... }    // filter concern
     // ... 8 action types nữa — 85 lines tổng
   });

 MailboxController:
   ever(mailboxDashBoardController.mailboxUIAction, (action) {
     // ... 7 action types — 159 lines
   });

 ⚠ Vấn đề:
   - Mỗi callback xử lý nhiều concerns không liên quan
   - Consumer phải biết tên field của God Object (tight coupling)
   - Không type-safe: Rxn<UIAction> — runtime cast, có thể crash
   - Thêm action type mới → phải sửa closure đã có (vi phạm OCP)
```

---

## Tại sao phải giải quyết cả 2 đồng thời?

```
 Nếu chỉ giải quyết A (tách state ra GetxService):
   GetxService.emailList  ←  ThreadController vẫn Get.find() GetxService
   GetxService.selectedMailbox  ←  MailboxController vẫn Get.find()
   → Coupling vẫn còn, chỉ là God Object nhỏ hơn

 Nếu chỉ giải quyết B (thêm EventBus):
   EventBus.dispatch(MailboxSelectedEvent)
   → Nhưng state vẫn nằm trong God Object
   → Controller vẫn cần Get.find() để đọc state

 Giải quyết cả 2:
   State → GetxService (injectable, reactive)
   Communication → AppEventBus (type-safe, decoupled)
   → Controller không cần biết controller khác tồn tại
   → God Object có thể được tách dần (Strangler Fig Pattern)
```

---

## Các files tiếp theo

| File | Nội dung |
|---|---|
| `02-getx-service.md` | GetxService là gì, lifecycle, khi nào dùng |
| `03-event-bus.md` | EventBus là gì, cơ chế hoạt động |
| `04-shared-state-options.md` | So sánh 5 phương án Shared State + decision matrix |
| `05-communication-options.md` | So sánh 6 phương án Inter-Controller Communication + decision matrix |
| `06-decisions-and-risks.md` | Quyết định cuối, lý do, rủi ro, rules |
