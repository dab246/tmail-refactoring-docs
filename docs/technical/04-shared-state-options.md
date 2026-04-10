# Technical: So sánh các phương án Shared State

> **Bài toán:** 5 controllers cùng cần đọc email list, selected mailbox, loading state.
> **Ràng buộc:** Giữ nguyên GetX framework.

---

## Phương án được xem xét

```
 Phương án 1: God Object Controller (hiện tại)
 Phương án 2: GetxService (đề xuất)
 Phương án 3: GetxController permanent: true
 Phương án 4: Static Singleton class
 Phương án 5: InheritedWidget / Provider
```

---

## So sánh chi tiết

| Tiêu chí | God Object (hiện tại) | GetxService | GetxController permanent | Static Singleton | InheritedWidget |
|---|:---:|:---:|:---:|:---:|:---:|
| **Persist qua route changes** | ✅ (permanent) | ✅ | ✅ | ✅ | ❌ |
| **Tích hợp với GetX .obs** | ✅ | ✅ | ✅ | ❌ | ❌ |
| **Obx() reactive UI** | ✅ | ✅ | ✅ | ❌ | ❌ |
| **Dispose cleanup** | ❌ (không dispose) | ✅ (onClose) | ⚠️ (cần force) | ❌ (manual) | ✅ |
| **Dễ test** | ❌ (mock 43 deps) | ✅ (inject trực tiếp) | ⚠️ | ❌ | ⚠️ |
| **Single Responsibility** | ❌ (15 concerns) | ✅ (1 concern/service) | ⚠️ | ⚠️ | ✅ |
| **Thay đổi State Management** | Không cần | Không cần | Không cần | Không cần | **Cần** |
| **Team learning curve** | 0 (đã biết) | 0 (là GetX) | 0 (là GetX) | Thấp | Cao |
| **Constructor injection** | ✅ | ✅ | ✅ | ❌ | ❌ |
| **Phù hợp constraint "giữ GetX"** | ✅ | ✅ | ✅ | ⚠️ | ❌ |

---

## Phân tích từng phương án

### Phương án 1 — God Object Controller (hiện tại)

```
 Ưu điểm:
   + Không cần thay đổi gì
   + Team đã quen

 Nhược điểm:
   - Chính là nguyên nhân gây ra toàn bộ vấn đề
   - 3,508 lines, 43 deps — không thể maintain
   - Test cần mock 40+ dependencies
   - Mọi thứ coupled vào 1 class

 Kết luận: ❌ Loại bỏ
```

### Phương án 2 — GetxService (đề xuất)

```
 Ưu điểm:
   + Là GetX native — không cần thêm dependency
   + Designed exactly for this use case (shared app-level state)
   + Tích hợp hoàn hảo với .obs, Obx(), ever()
   + onClose() đảm bảo cleanup
   + Constructor injectable → testable
   + Tách concerns: mỗi service = 1 domain

 Nhược điểm:
   - Team cần học thêm GetxService (learning curve thấp)
   - Cần thiết kế careful — tránh biến GetxService thành God Object mới

 Kết luận: ✅ Chọn
```

### Phương án 3 — GetxController với permanent: true

```
 Ưu điểm:
   + Tương tự GetxService về behavior
   + Team đã biết GetxController

 Nhược điểm:
   - GetxController không được thiết kế cho permanent state
   - Get.delete() có thể vô tình xóa nếu cùng type
   - Không có semantic clarity: "đây là service, không phải controller"
   - GetX docs khuyên dùng GetxService cho use case này

 Kết luận: ⚠️ Không dùng — dễ bị misuse, semantic không rõ
```

### Phương án 4 — Static Singleton

```
 Ưu điểm:
   + Simple

 Nhược điểm:
   - Không reactive — không dùng được với Obx()
   - Phải tự manage lifecycle (memory leak risk)
   - Không injectable → không testable
   - Vi phạm DIP
   - Đi ngược hoàn toàn với GetX patterns

 Kết luận: ❌ Loại bỏ
```

### Phương án 5 — InheritedWidget / Provider

```
 Ưu điểm:
   + Flutter standard approach
   + Testable

 Nhược điểm:
   - YÊU CẦU THAY ĐỔI STATE MANAGEMENT → vi phạm constraint
   - Team learning curve cao
   - Không tích hợp với .obs, Obx()
   - Cần rewrite toàn bộ UI layer
   - Không khả thi trong timeframe

 Kết luận: ❌ Loại bỏ (vi phạm constraint "giữ GetX")
```

---

## Decision Matrix — Shared State

```
 Tiêu chí              Trọng số   God Object  GetxService  GetxCtrl permanent  Static
 ─────────────────────────────────────────────────────────────────────────────────────
 Giữ nguyên GetX           30%        ✅ 3        ✅ 3           ✅ 3            ⚠ 1
 Reactive (.obs/Obx)        25%        ✅ 3        ✅ 3           ✅ 3            ❌ 0
 Testability                20%        ❌ 0        ✅ 3           ⚠ 2            ❌ 0
 Single Responsibility      15%        ❌ 0        ✅ 3           ⚠ 2            ⚠ 1
 Lifecycle safety           10%        ❌ 0        ✅ 3           ⚠ 1            ❌ 0
 ─────────────────────────────────────────────────────────────────────────────────────
 Điểm tổng                 100%        1.35       3.00           2.45            0.30
 ─────────────────────────────────────────────────────────────────────────────────────
                                        ❌         ✅ WINNER       ⚠               ❌
```

**Kết quả: GetxService với điểm 3.00 — cao nhất, đáp ứng tất cả tiêu chí.**

Xem quyết định cuối cùng và rủi ro tại [`06-decisions-and-risks.md`](06-decisions-and-risks.md).
