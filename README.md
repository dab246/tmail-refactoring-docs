# Tmail Flutter — Architecture Refactoring

> **ADR-0076** · Tái cấu trúc kiến trúc presentation layer, giữ nguyên GetX framework.
> **Ngày:** 2026-04-10 · **Scope:** 7 controllers (~10,522 lines tổng)

---

## Xem tài liệu

```bash
# Clone repo
git clone https://github.com/<username>/tmail-refactoring-docs.git
cd tmail-refactoring-docs

# Chạy local server
npx serve .
# hoặc
python -m http.server 8000
```

Mở `http://localhost:3000` (hoặc port tương ứng) trên browser.

**GitHub Pages:** `https://<username>.github.io/tmail-refactoring-docs/`

**Keyboard shortcuts:**
- `←` `→` — điều hướng prev/next
- `S` — ẩn/hiện sidebar
- `Home` / `End` — nhảy đầu/cuối

---

## Bối cảnh

Twake Mail (tmail-flutter) là email client đa nền tảng (Android/iOS/Web) xây dựng bằng Flutter + JMAP. Sau nhiều vòng phát triển, presentation layer tích lũy nhiều vấn đề code health nghiêm trọng:

```
 Tổng Lines of Code — 7 controllers
 ─────────────────────────────────────────────────────────────────────
 MailboxDashboard ████████████████████████████████████  3,508  ← God Object
 SingleEmail      ████████████████                      1,630
 ThreadController ████████████████                      1,625
 MailboxController████████████████                      1,602
 SearchEmail      ████████████                          1,198
 BaseController   ██████                                  647
 ThreadDetail     ███                                     312
 ─────────────────────────────────────────────────────────────────────
 Tổng             ~10,522 lines  |  Mục tiêu: ≤ 300 lines / controller
```

---

## Cấu trúc tài liệu

Tài liệu chia làm 3 phần, đọc theo thứ tự:

```
 docs/
 ├── problems/      ← 1. Vấn đề hiện tại là gì?
 ├── technical/     ← 2. Tại sao chọn giải pháp này?
 └── solutions/     ← 3. Giải pháp cụ thể như thế nào?
```

---

## 1. Problems — 14 vấn đề code health

> **Đọc trước để hiểu ngữ cảnh.** Mỗi vấn đề có code evidence cụ thể với line numbers.

```
 docs/problems/
 ├── 01-overview.md            ← Index: heatmap severity, LoC chart, bảng tổng hợp
 ├── 02-size-and-complexity.md ← #1 Constructor Over-Injection (43 deps)
 │                                #2 Large Method (113-line dispatcher)
 │                                #3 Complex Method (CC > 15)
 │                                #14 BaseController God Class (7 domains)
 ├── 03-coupling-and-cohesion.md ← #4 Bumpy Road (31 branches)
 │                                  #5 Deep Nested Complexity (4+ levels)
 │                                  #6 Low Cohesion (methods không dùng class fields)
 │                                  #8 Feature Envy (75+ Get.find refs)
 └── 04-design-smells.md       ← #7 Excess Data Declarations (42+ Rx fields)
                                  #9 Code Health Degradations (late?, dynamic inject)
                                  #10 Excess Function Arguments
                                  #11 Primitive Obsession (redundant bool + enum)
                                  #12 Temporal Coupling
                                  #13 Mixed Abstraction Levels
```

**Severity heatmap:**

```
                  Dashboard  Mailbox  Thread  SingleEmail  Search  Base
 Constructor OI      🔴        🟠       ⚪        🟠          ⚪      🔴
 Large Method        🔴        🟠       🔴        🔴          🟡     ⚪
 Bumpy Road          🔴        🔴       🔴        🔴          🔴     ⚪
 Feature Envy        🟠        🔴       🔴        🔴          🟠     ⚪
 BaseCtrl God        🔴        🔴       🔴        🔴          🔴     🔴
```

---

## 2. Technical — Tại sao chọn GetxService + AppEventBus?

> **Quyết định kỹ thuật có căn cứ.** So sánh đầy đủ các phương án trước khi chọn.

```
 docs/technical/
 ├── 01-problem-statement.md    ← 2 bài toán cốt lõi cần giải đồng thời
 │                                 A: Shared State nằm ở đâu?
 │                                 B: Controller giao tiếp thế nào?
 ├── 02-getx-service.md         ← GetxService là gì, lifecycle vs GetxController
 ├── 03-event-bus.md            ← AppEventBus là gì, cơ chế pub/sub
 ├── 04-shared-state-options.md ← So sánh 5 phương án Shared State
 │                                 → God Object / GetxService / GetxCtrl permanent
 │                                 → Static Singleton / InheritedWidget
 ├── 05-communication-options.md ← So sánh 6 phương án Inter-Controller Comm
 │                                  → Rxn<UIAction> / Get.find() / AppEventBus
 │                                  → GetX Workers / RxDart / Callback injection
 └── 06-decisions-and-risks.md  ← Quyết định cuối + rủi ro + DOs/DON'Ts rules
```

**Decision matrices (tóm tắt):**

```
 Shared State                    Inter-Controller Communication
 ─────────────────────────────   ──────────────────────────────────
 God Object       1.35  ❌        Rxn<UIAction>     0.30  ❌
 GetxService      3.00  ✅ WIN    Get.find() direct 0.90  ❌
 GetxCtrl perm    2.45  ⚠         AppEventBus       3.00  ✅ WIN
 Static Singleton 0.30  ❌        GetX Workers      1.15  ⚠
 InheritedWidget  N/A   ❌        RxDart            2.55  ❌ (needs pkg)
```

---

## 3. Solutions — Giải pháp cụ thể cho từng controller

> **Kế hoạch thực thi.** Mỗi file là blueprint cho 1 controller/component.

```
 docs/solutions/
 ├── 01-overview.md                    ← Kiến trúc tổng quan trước/sau + roadmap
 ├── 02-base-controller.md             ← Tách domain contracts, clean arch fix
 ├── 03-mailbox-dashboard-controller.md ← God Object → 4 sub-controllers + 3 services
 ├── 04-mailbox-controller.md          ← Feature Envy fix, 159-line listener split
 ├── 05-thread-controller.md           ← Deep nesting, bumpy road fix
 ├── 06-single-email-controller.md     ← Optional deps → CalendarEmailController
 ├── 07-search-email-controller.md     ← 5 mixins → SearchFilterController
 └── 08-shared-infrastructure.md       ← AppEventBus + GetxService implementation
```

**Kiến trúc sau refactor:**

```
 TRƯỚC                              SAU
 ──────────────────────────────     ──────────────────────────────────────
 MailboxDashBoardController         AppEventBus (GetxService · permanent)
   3,508 lines · 43 deps              sealed AppEvent · dispatch/subscribe
   42 Rx fields · 15 contexts               │
   Get.find() 75+ times             ┌───────┼───────────┐
        │                           ▼       ▼           ▼
   MailboxController          EmailAction  Mailbox   Sending
   ThreadController           Controller  Navigation Queue
   SingleEmailController      ~300 lines  Controller Controller
   SearchEmailController
                              GetxService (permanent shared state)
                              EmailListStateService · ComposerStateService
                              DragDropStateService

                              MailboxDashBoardController ~200 lines
                              (orchestrator only · ≤7 deps)
```

---

## Roadmap

```
       Apr        May        Jun        Jul-Aug     Sep-Oct
       W15-W18    W19-W24    W25-W28    W29-W36     W37-W44
 ──────────────────────────────────────────────────────────
 Phase 0  ████████
 Foundation
   AppEventBus + sealed AppEvent
   GetxService shells
   Domain contracts
   BaseController cleanup

 Phase 1           ████████████████
 Constructor Injection
   Leaf: ThreadDetail, SearchEmail
   Mid: Thread, SingleEmail, Mailbox

 Phase 2                    ████████████
 State Decomposition
   .obs fields → GetxService
   ever(dashboard.X) → EventBus
   Remove Rxn<UIAction>

 Phase 3                               ████████████████
 God Object Split              ⚠ HIGH RISK
   Extract 4 sub-controllers
   Extract CalendarEmailController

 Phase 4                                              ████████
 Cleanup & Regression
 ──────────────────────────────────────────────────────────
 Tổng: ~5–8 tháng · chạy song song với roadmap sản phẩm
```

---

## Ràng buộc

- **Giữ nguyên GetX** — GetxController, `.obs`, `Obx()`, `GetPage`, `Get.toNamed()`
- **Không migrate framework** — không Riverpod, không Provider
- **Strangler Fig Pattern** — refactor dần, không rewrite toàn bộ
- **Không block roadmap** — Phase 3 (HIGH RISK) tránh release window
