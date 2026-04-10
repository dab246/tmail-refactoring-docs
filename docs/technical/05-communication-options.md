# Technical: So sánh các phương án Inter-Controller Communication

> **Bài toán:** Khi user chọn mailbox, làm sao MailboxController thông báo cho ThreadController, SearchController biết?
> **Ràng buộc:** Giữ nguyên GetX, không tạo circular dependencies.

---

## Phương án được xem xét

```
 Phương án 1: Rxn<UIAction> + ever() (hiện tại)
 Phương án 2: Get.find() trực tiếp (hiện tại)
 Phương án 3: AppEventBus / StreamController (đề xuất)
 Phương án 4: GetX Workers (ever, debounce, once)
 Phương án 5: RxDart BehaviorSubject / PublishSubject
 Phương án 6: Callback / Function injection
```

---

## So sánh chi tiết

| Tiêu chí | Rxn\<UIAction\> | Get.find() direct | AppEventBus | GetX Workers | RxDart | Callback injection |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| **Type safety** | ❌ (runtime cast) | ✅ | ✅ (sealed class) | ⚠️ | ✅ | ✅ |
| **Decoupled (A không biết B)** | ❌ | ❌ | ✅ | ❌ | ✅ | ❌ |
| **Dễ test** | ❌ | ❌ | ✅ | ⚠️ | ✅ | ✅ |
| **Tự cleanup khi dispose** | ⚠️ (manual) | N/A | ✅ (compositeSubscription) | ✅ | ⚠️ | ✅ |
| **Không cần thêm dependency** | ✅ | ✅ | ✅ (GetX stream) | ✅ | ❌ | ✅ |
| **Tích hợp với GetX** | ✅ | ✅ | ✅ | ✅ | ⚠️ | ✅ |
| **Multiple subscribers** | ✅ (ever × N) | N/A | ✅ (broadcast) | ✅ | ✅ | ❌ (1:1) |
| **Xử lý 1 event type** | ❌ (if-else chain) | N/A | ✅ (whereType) | ❌ | ✅ | ✅ |
| **Circular dep risk** | ❌ (cao) | ❌ (cao) | ✅ (zero) | ❌ (cao) | ✅ | ⚠️ |
| **Thêm package** | Không | Không | Không | Không | **Có** | Không |
| **Debug/trace event** | Khó | N/A | ✅ (single stream) | Khó | ✅ | Khó |

---

## Phân tích từng phương án

### Phương án 1 — `Rxn<UIAction>` + `ever()` (hiện tại)

```
 Cơ chế:
   ever(dashboard.rxnMailboxUIAction, (action) {
     if (action is OpenMailboxAction) { ... }
     else if (action is RefreshAction) { ... }
     // 7+ loại action trong 1 closure
   });

 Ưu điểm:
   + Có sẵn trong GetX, không cần code thêm
   + Simple với 1–2 action types

 Nhược điểm:
   - Rxn<UIAction> không type-safe → runtime cast, có thể crash
   - 1 observable chứa tất cả action types → callback là 85–159 lines
   - Mọi subscriber nhận TẤT CẢ events, kể cả không liên quan (noise)
   - Consumer phải biết tên field (mailboxUIAction) trong God Object
   - Impossible to add new action type mà không sửa closure

 Kết luận: ❌ Đây chính là vấn đề cần fix
```

### Phương án 2 — `Get.find<T>()` trực tiếp

```
 Cơ chế:
   final dashboard = Get.find<MailboxDashBoardController>();
   dashboard.doSomething(); // gọi trực tiếp

 Ưu điểm:
   + Simple, rõ ràng

 Nhược điểm:
   - A phải import và biết B tồn tại → tight coupling
   - Get.find() có thể crash nếu B chưa được register
   - 75+ references → God Object không thể tách
   - Test cần mock toàn bộ B
   - Circular dependency risk: A→B→A

 Kết luận: ❌ Đây chính là vấn đề cần fix
```

### Phương án 3 — AppEventBus (đề xuất)

```
 Cơ chế:
   // Publisher (không biết ai subscribe):
   _eventBus.dispatch(MailboxSelectedEvent(mailbox))

   // Subscriber (không biết ai dispatch):
   _eventBus.stream
       .whereType<MailboxSelectedEvent>()
       .listen(_onMailboxSelected)
       .addTo(compositeSubscription)

 Ưu điểm:
   + Zero coupling: A không import B, B không import A
   + Type-safe: sealed class + whereType<T>() → compile error nếu sai
   + Mỗi subscriber chỉ nhận event type mình cần → no noise
   + Testable: inject mockEventBus vào constructor, emit test events
   + Auto-cleanup qua compositeSubscription.dispose()
   + Dễ add event type mới mà không sửa subscriber cũ (OCP)
   + Không cần thêm package — dùng Dart Stream thuần
   + Single stream → dễ debug/log

 Nhược điểm:
   - Cần thiết kế AppEvent hierarchy cẩn thận
   - Event dispatch order không guaranteed trong một số edge cases
   - Nếu lạm dụng: "Event spaghetti" — khó trace ai dispatch gì

 Kết luận: ✅ Chọn
```

### Phương án 4 — GetX Workers (ever, debounce, once)

```
 Cơ chế:
   ever(someRxVariable, (value) { ... });
   debounce(searchQuery, (q) { search(q); });

 Ưu điểm:
   + Native GetX, không cần code thêm
   + Tốt cho reactive state changes (debounce, throttle)

 Nhược điểm:
   - Worker.ever() vẫn cần biết observable của controller khác
   - Không giải quyết được vấn đề coupling
   - Không phải event bus — không support 1 event → N subscribers
   - Thích hợp cho local state changes, không phải cross-controller events

 Kết luận: ⚠️ Dùng bổ sung (debounce cho search input),
            không dùng cho inter-controller communication
```

### Phương án 5 — RxDart BehaviorSubject / PublishSubject

```
 Cơ chế:
   final _subject = BehaviorSubject<MailboxSelectedEvent>();
   _subject.stream.listen(handler);
   _subject.add(MailboxSelectedEvent(mailbox));

 Ưu điểm:
   + Powerful: BehaviorSubject giữ last value
   + Type-safe per subject
   + Tốt nếu đã dùng RxDart trong project

 Nhược điểm:
   - CẦN THÊM PACKAGE rxdart → thêm dependency
   - Project hiện tại không dùng RxDart → learning curve
   - Nhiều subjects riêng lẻ thay vì 1 unified bus → khó manage
   - BehaviorSubject lưu last value → có thể gây side effects ngoài ý muốn

 Kết luận: ❌ Không cần thiết — Dart Stream thuần đã đủ
            Tránh thêm dependency không cần thiết
```

### Phương án 6 — Callback / Function injection

```
 Cơ chế:
   class ThreadController {
     final Function(PresentationMailbox) onMailboxSelected;
     ThreadController(this.onMailboxSelected, ...);
   }

 Ưu điểm:
   + Explicit, compile-time safe
   + Dễ test: truyền lambda vào

 Nhược điểm:
   - Chỉ hỗ trợ 1:1 (1 producer → 1 consumer)
   - Không support 1:N (1 event → nhiều subscribers)
   - Bindings trở nên phức tạp khi nhiều callbacks
   - Không scalable khi thêm controller mới

 Kết luận: ❌ Phù hợp cho 1:1 delegation, không phải cross-cutting events
```

---

## Decision Matrix — Inter-Controller Communication

```
 Tiêu chí              Trọng số   Rxn<UIAction>  Get.find()  EventBus  GetX Workers  RxDart
 ──────────────────────────────────────────────────────────────────────────────────────────
 Decoupling                 30%        ❌ 0          ❌ 0       ✅ 3        ❌ 0        ✅ 3
 Type safety                25%        ❌ 0          ✅ 3       ✅ 3        ⚠ 2        ✅ 3
 Testability                20%        ❌ 0          ❌ 0       ✅ 3        ⚠ 1        ✅ 3
 No extra dependency        15%        ✅ 3          ✅ 3       ✅ 3        ✅ 3        ❌ 0
 Multiple subscribers       10%        ✅ 3          ❌ 0       ✅ 3        ✅ 3        ✅ 3
 ──────────────────────────────────────────────────────────────────────────────────────────
 Điểm tổng                 100%        0.30          0.90       3.00       1.15        2.55
 ──────────────────────────────────────────────────────────────────────────────────────────
                                        ❌            ❌         ✅ WINNER   ⚠          ✅*
                                                              (* RxDart cần thêm package)
```

**Kết quả: AppEventBus với điểm 3.00 — cao nhất, đáp ứng tất cả tiêu chí mà không cần thêm dependency.**

Xem quyết định cuối cùng và rủi ro tại [`06-decisions-and-risks.md`](06-decisions-and-risks.md).
