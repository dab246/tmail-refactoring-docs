# Technical: Quyết định & Rủi ro

> Tổng hợp từ [`04-shared-state-options.md`](04-shared-state-options.md) và [`05-communication-options.md`](05-communication-options.md)

---

## Quyết định 1: Dùng GetxService cho Shared State

```
 QUYẾT ĐỊNH: Tạo EmailListStateService, ComposerStateService,
             DragDropStateService extends GetxService

 LÝ DO:
   1. Là GetX-native — không cần thêm dependency, không vi phạm constraint
   2. Thiết kế chính xác cho use case này (Flutter/GetX documentation)
   3. Reactive hoàn toàn với .obs — Obx() widgets vẫn hoạt động
   4. Tách concerns rõ ràng: mỗi service = 1 domain
   5. Constructor injectable → testable

 CÁI KHÔNG CHỌN VÀ TẠI SAO:
   - GetxController permanent: true → semantic không rõ, dễ bị misuse
   - Static singleton → không reactive, không testable
   - InheritedWidget/Provider → vi phạm constraint "giữ GetX"

 TRADE-OFF CHẤP NHẬN:
   - GetxService tồn tại mãi → cần cẩn thận về memory
   - Giải pháp: mỗi service chỉ chứa state thuộc 1 domain nhỏ
```

---

## Quyết định 2: Dùng AppEventBus cho Inter-Controller Communication

```
 QUYẾT ĐỊNH: Tạo AppEventBus extends GetxService
             với sealed class AppEvent + whereType<T>()

 LÝ DO:
   1. Zero coupling: controllers không import nhau
   2. Type-safe: sealed class → compiler bắt lỗi
   3. Testable: inject mock event bus, emit test events
   4. OCP: thêm event type mới không sửa code cũ
   5. Single stream → dễ debug, dễ log all events
   6. Không cần package ngoài — dùng Dart Stream thuần

 CÁI KHÔNG CHỌN VÀ TẠI SAO:
   - Rxn<UIAction>: không type-safe, callback spaghetti → CHÍNH LÀ VẤN ĐỀ CẦN FIX
   - Get.find() direct: tight coupling → CHÍNH LÀ VẤN ĐỀ CẦN FIX
   - RxDart: cần thêm package, Dart Stream đã đủ
   - Callback injection: chỉ 1:1, không 1:N

 TRADE-OFF CHẤP NHẬN:
   - Cần thiết kế AppEvent hierarchy cẩn thận upfront
   - "Event spaghetti" nếu lạm dụng → giảm thiểu bằng rule:
     chỉ dùng EventBus cho cross-feature events,
     dùng Rx .obs trực tiếp cho intra-controller state
```

---

## Tổng hợp tất cả quyết định

```
 Vấn đề                    Giải pháp chọn          Thay thế loại bỏ
 ──────────────────────────────────────────────────────────────────────
 Shared State              GetxService             God Object,
 (5 controllers cần         (permanent,             Static singleton,
 cùng đọc state)            injectable,             InheritedWidget
                            reactive .obs)

 Inter-Controller           AppEventBus             Rxn<UIAction> + ever(),
 Communication              (GetxService,           Get.find() direct,
 (cross-feature events)     sealed AppEvent,        RxDart
                            whereType<T>())

 Constructor Over-Injection Domain Contracts        Fat interfaces,
 (43 deps → Feature Envy)  (fine-grained           Monolithic contract
                            interfaces)

 BaseController God Class   ISessionTokenRefresher  Direct data-layer
 (import data layer)        ILanguageSettingService  imports
                            IAuthService
                            (abstractions in domain)
```

---

## Rủi ro của GetxService

| Rủi ro | Mô tả | Giảm thiểu |
|---|---|---|
| **Memory** | GetxService không bao giờ bị dispose → memory leak nếu giữ quá nhiều data | Mỗi service chỉ giữ state cần thiết. Clear data khi logout. |
| **God Service** | GetxService trở thành God Object mới | Rule: mỗi service ≤ 10 Rx fields, 1 domain duy nhất |
| **State stale** | State từ session cũ vẫn còn khi đăng nhập tài khoản mới | `SessionController` dispatch `SessionExpiredEvent` → services clear state |
| **Circular inject** | Service A inject Service B inject Service A | Map dependency graph trước Phase 1 |

---

## Rủi ro của AppEventBus

| Rủi ro | Mô tả | Giảm thiểu |
|---|---|---|
| **Event Spaghetti** | Quá nhiều events, khó trace ai dispatch gì | Giới hạn: chỉ cross-feature events. Intra-controller dùng .obs. |
| **Event ordering** | Không guaranteed order khi nhiều dispatches đồng thời | Thiết kế events idempotent. Không dispatch event phụ thuộc order. |
| **Circular dispatch** | A dispatch EventX → B nhận, B dispatch EventY → A nhận, loop | Rule: không dispatch event trong event handler của cùng loại |
| **Memory leak subscription** | Quên cancel subscription | `compositeSubscription` + `addTo()` → auto cancel khi controller dispose |
| **Event lost** | Dispatch trước khi subscribe | `broadcast()` stream: OK — chỉ nhận events AFTER subscribe. Document this. |

---

## Quy tắc áp dụng để tránh anti-patterns

```
 DOs:                                     DON'Ts:
 ──────────────────────────────────       ──────────────────────────────────
 ✅ Dùng EventBus cho cross-feature       ❌ Dispatch event trong event handler
    events (A → B, khác feature)             của cùng loại event (loop)

 ✅ Dùng .obs trực tiếp cho               ❌ Subscribe toàn bộ AppEvent stream
    intra-controller state                   rồi if-else filter (giống cũ)

 ✅ Mỗi GetxService = 1 domain,           ❌ Tạo 1 GetxService chứa tất cả
    ≤ 10 Rx fields                           state (God Service)

 ✅ Subscribe với whereType<T>()          ❌ Cast event thủ công (as MyEvent)
    → compile-time safety

 ✅ addTo(compositeSubscription)          ❌ Giữ StreamSubscription thủ công
    → auto cleanup                           mà không cancel

 ✅ EventBus chỉ cho domain events        ❌ Dùng EventBus cho UI events
    (business actions)                       (button tap, scroll)
```

---

## Tóm tắt quyết định kiến trúc

```
 ┌─────────────────────────────────────────────────────────────────┐
 │                    QUYẾT ĐỊNH CUỐI CÙNG                         │
 │                                                                 │
 │  1. SHARED STATE: GetxService (permanent)                       │
 │     → EmailListStateService, ComposerStateService,              │
 │       DragDropStateService                                      │
 │     → Lý do: GetX-native, reactive, injectable, 1 domain/svc   │
 │                                                                 │
 │  2. INTER-CONTROLLER COMM: AppEventBus (GetxService)            │
 │     → sealed class AppEvent + whereType<T>()                    │
 │     → Lý do: Zero coupling, type-safe, testable, no extra pkg   │
 │                                                                 │
 │  3. DEPENDENCY INJECTION: Domain Contracts                      │
 │     → IEmailListStateContract, IMailboxNavigationContract...    │
 │     → Lý do: Tách abstraction khỏi implementation, ISP          │
 │                                                                 │
 │  4. GIỮ NGUYÊN: GetX state management                          │
 │     → GetxController, .obs, Obx(), GetPage, Get.toNamed         │
 │     → Lý do: Constraint từ đầu, team cost quá cao nếu đổi      │
 └─────────────────────────────────────────────────────────────────┘
```
