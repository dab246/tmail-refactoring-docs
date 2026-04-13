# Problems Overview — Code Health Report

**Date:** 2026-04-10
**Scope:** MailboxDashboardController, MailboxController, ThreadController, SingleEmailController, ThreadDetailController, SearchEmailController, BaseController
**Source:** CodeScene static analysis + manual review

---

## Mục lục vấn đề

| # | Issue | Mô tả | Severity | Controllers bị ảnh hưởng |
|---|---|---|---|---|
| 1 | **Constructor Over-Injection** | Constructor nhận quá nhiều dependencies — dấu hiệu trực tiếp của class đang đảm nhận quá nhiều trách nhiệm | Critical | MailboxDashboard (43 deps), ThreadDetail (15 deps) |
| 2 | **Large Method** | Method quá dài (>30 lines) — khó đọc, khó test, khó maintain, thường chứa nhiều concern lẫn lộn | Critical | MailboxDashboard, Thread, SingleEmail |
| 3 | **Complex Method** | Method có quá nhiều nhánh rẽ (cyclomatic complexity >10) — khó trace flow, dễ có bug ẩn | Critical | MailboxDashboard, Mailbox, Thread, SingleEmail |
| 4 | **Bumpy Road** | Method có 5+ nhánh if-else/switch liên tiếp, mỗi nhánh xử lý concern hoàn toàn khác nhau — như đường gồ ghề, khó đi | High | Tất cả controllers |
| 5 | **Deep Nested Complexity** | Code lồng nhau 4+ cấp — càng sâu càng khó hiểu, khó test từng nhánh | High | Thread, SingleEmail, Mailbox |
| 6 | **Low Cohesion** | Các methods trong class không liên quan đến nhau, không dùng chung state — class không có "chủ đề" rõ ràng | High | Tất cả controllers |
| 7 | **Excess Data Declarations** | Khai báo quá nhiều fields, nhiều fields không cần thiết, Rx wrapping thừa hoặc fields chỉ dùng 1 lần | High | MailboxDashboard, Mailbox, SearchEmail |
| 8 | **Feature Envy** | Class A truy cập data/methods của class B nhiều hơn của chính mình — A "ghen tị" với B, logic đang ở sai chỗ | High | Mailbox, Thread, SingleEmail, Search |
| 9 | **Code Health Degradations** | Code bị "vá" nhiều lần: optional late deps, dynamic injection, force unwrap (!), mixed concerns — dấu hiệu kiến trúc xuống cấp dần | Medium | MailboxDashboard, SingleEmail |
| 10 | **Excess Number of Function Arguments** | Method (không phải constructor) có 5+ tham số — khó gọi, khó test, thường là dấu hiệu thiếu domain object | Medium | Mailbox, MailboxDashboard |
| 11 | **Primitive Obsession** | Dùng kiểu nguyên thủy (bool, String, int) thay cho domain object — mất đi ngữ nghĩa, dễ dùng sai | Medium | SearchEmail, Mailbox |
| 12 | **Temporal Coupling** | Các methods phải gọi đúng thứ tự mới hoạt động đúng, nhưng không có cơ chế nào bắt buộc điều này | Medium | Thread, SingleEmail |
| 13 | **Mixed Abstraction Levels** | Một method vừa chứa logic orchestration cấp cao vừa xử lý chi tiết cấp thấp — trộn lẫn 2 tầng tư duy khác nhau | Medium | MailboxDashboard, Mailbox |
| 14 | **BaseController God Class** | Class cơ sở đảm nhận 7 domains khác nhau, import thẳng data layer — mọi controller con đều "kế thừa" toàn bộ nợ kỹ thuật này | Critical | Tất cả (kế thừa) |

---

## Sơ đồ phân bổ vấn đề

> 14 vấn đề phân bố theo 3 vòng quỹ đạo quanh dự án — vòng trong cùng là nghiêm trọng nhất.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1100 1100" style="width:100%;max-width:900px;display:block;margin:1.5rem auto;border-radius:12px;">
  <defs>
    <radialGradient id="cg" cx="50%" cy="50%" r="50%">
      <stop offset="0%" stop-color="#3b82f6"/>
      <stop offset="100%" stop-color="#1d4ed8"/>
    </radialGradient>
  </defs>
  <rect width="1100" height="1100" fill="#0f172a" rx="16"/>
  <!-- Orbit rings -->
  <circle cx="550" cy="550" r="150" fill="none" stroke="#ef444430" stroke-width="1.5" stroke-dasharray="8,5"/>
  <circle cx="550" cy="550" r="280" fill="none" stroke="#f9731630" stroke-width="1.5" stroke-dasharray="8,5"/>
  <circle cx="550" cy="550" r="405" fill="none" stroke="#eab30830" stroke-width="1.5" stroke-dasharray="8,5"/>
  <!-- Ring labels -->
  <text x="550" y="397" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#ef444488">● Critical</text>
  <text x="550" y="267" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#f9731688">● High</text>
  <text x="550" y="142" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#eab30888">● Medium</text>
  <!-- Lines center → Critical -->
  <line x1="550" y1="550" x2="550" y2="400" stroke="#ef444428" stroke-width="1"/>
  <line x1="550" y1="550" x2="700" y2="550" stroke="#ef444428" stroke-width="1"/>
  <line x1="550" y1="550" x2="550" y2="700" stroke="#ef444428" stroke-width="1"/>
  <line x1="550" y1="550" x2="400" y2="550" stroke="#ef444428" stroke-width="1"/>
  <!-- Lines center → High -->
  <line x1="550" y1="550" x2="720" y2="330" stroke="#f9731628" stroke-width="1"/>
  <line x1="550" y1="550" x2="800" y2="650" stroke="#f9731628" stroke-width="1"/>
  <line x1="550" y1="550" x2="550" y2="835" stroke="#f9731628" stroke-width="1"/>
  <line x1="550" y1="550" x2="300" y2="650" stroke="#f9731628" stroke-width="1"/>
  <line x1="550" y1="550" x2="380" y2="330" stroke="#f9731628" stroke-width="1"/>
  <!-- Lines center → Medium -->
  <line x1="550" y1="550" x2="930" y2="435" stroke="#eab30828" stroke-width="1"/>
  <line x1="550" y1="550" x2="790" y2="875" stroke="#eab30828" stroke-width="1"/>
  <line x1="550" y1="550" x2="310" y2="875" stroke="#eab30828" stroke-width="1"/>
  <line x1="550" y1="550" x2="160" y2="435" stroke="#eab30828" stroke-width="1"/>
  <line x1="550" y1="550" x2="550" y2="145" stroke="#eab30828" stroke-width="1"/>
  <!-- Center node -->
  <circle cx="550" cy="550" r="70" fill="url(#cg)" stroke="#60a5fa" stroke-width="2.5"/>
  <text x="550" y="540" text-anchor="middle" font-family="monospace,ui-monospace" font-size="15" font-weight="bold" fill="white">Tmail</text>
  <text x="550" y="558" text-anchor="middle" font-family="monospace,ui-monospace" font-size="13" fill="#93c5fd">Flutter</text>
  <text x="550" y="575" text-anchor="middle" font-family="monospace,ui-monospace" font-size="10" fill="#bfdbfe">7 controllers</text>
  <!-- CRITICAL nodes (r≈150) -->
  <!-- #1 Constructor Over-Injection — top -->
  <g transform="translate(550,400)">
    <rect x="-64" y="-26" width="128" height="52" rx="8" fill="#3b0808" stroke="#ef4444" stroke-width="1.5"/>
    <text x="0" y="-9" text-anchor="middle" font-family="monospace,ui-monospace" font-size="10" font-weight="bold" fill="#fca5a5">#1 · Critical</text>
    <text x="0" y="5" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fecaca">Constructor</text>
    <text x="0" y="18" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fecaca">Over-Injection</text>
  </g>
  <!-- #2 Large Method — right -->
  <g transform="translate(700,550)">
    <rect x="-64" y="-26" width="128" height="52" rx="8" fill="#3b0808" stroke="#ef4444" stroke-width="1.5"/>
    <text x="0" y="-9" text-anchor="middle" font-family="monospace,ui-monospace" font-size="10" font-weight="bold" fill="#fca5a5">#2 · Critical</text>
    <text x="0" y="7" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fecaca">Large Method</text>
    <text x="0" y="19" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fecaca">&gt;30 lines</text>
  </g>
  <!-- #3 Complex Method — bottom -->
  <g transform="translate(550,700)">
    <rect x="-64" y="-26" width="128" height="52" rx="8" fill="#3b0808" stroke="#ef4444" stroke-width="1.5"/>
    <text x="0" y="-9" text-anchor="middle" font-family="monospace,ui-monospace" font-size="10" font-weight="bold" fill="#fca5a5">#3 · Critical</text>
    <text x="0" y="5" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fecaca">Complex Method</text>
    <text x="0" y="18" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fecaca">cyclomatic &gt;10</text>
  </g>
  <!-- #14 BaseController God Class — left -->
  <g transform="translate(400,550)">
    <rect x="-64" y="-26" width="128" height="52" rx="8" fill="#3b0808" stroke="#ef4444" stroke-width="1.5"/>
    <text x="0" y="-9" text-anchor="middle" font-family="monospace,ui-monospace" font-size="10" font-weight="bold" fill="#fca5a5">#14 · Critical</text>
    <text x="0" y="5" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fecaca">BaseController</text>
    <text x="0" y="18" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fecaca">God Class</text>
  </g>
  <!-- HIGH nodes (r≈280) -->
  <!-- #4 Bumpy Road — upper-right -->
  <g transform="translate(720,330)">
    <rect x="-64" y="-26" width="128" height="52" rx="8" fill="#3b1206" stroke="#f97316" stroke-width="1.5"/>
    <text x="0" y="-9" text-anchor="middle" font-family="monospace,ui-monospace" font-size="10" font-weight="bold" fill="#fdba74">#4 · High</text>
    <text x="0" y="7" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fed7aa">Bumpy Road</text>
    <text x="0" y="19" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fed7aa">5+ if-else chain</text>
  </g>
  <!-- #5 Deep Nested Complexity — right-bottom -->
  <g transform="translate(800,650)">
    <rect x="-64" y="-26" width="128" height="52" rx="8" fill="#3b1206" stroke="#f97316" stroke-width="1.5"/>
    <text x="0" y="-9" text-anchor="middle" font-family="monospace,ui-monospace" font-size="10" font-weight="bold" fill="#fdba74">#5 · High</text>
    <text x="0" y="5" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fed7aa">Deep Nested</text>
    <text x="0" y="18" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fed7aa">Complexity 4+</text>
  </g>
  <!-- #6 Low Cohesion — bottom -->
  <g transform="translate(550,835)">
    <rect x="-64" y="-26" width="128" height="52" rx="8" fill="#3b1206" stroke="#f97316" stroke-width="1.5"/>
    <text x="0" y="-9" text-anchor="middle" font-family="monospace,ui-monospace" font-size="10" font-weight="bold" fill="#fdba74">#6 · High</text>
    <text x="0" y="7" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fed7aa">Low Cohesion</text>
    <text x="0" y="19" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fed7aa">tất cả controllers</text>
  </g>
  <!-- #7 Excess Data Declarations — left-bottom -->
  <g transform="translate(300,650)">
    <rect x="-64" y="-26" width="128" height="52" rx="8" fill="#3b1206" stroke="#f97316" stroke-width="1.5"/>
    <text x="0" y="-9" text-anchor="middle" font-family="monospace,ui-monospace" font-size="10" font-weight="bold" fill="#fdba74">#7 · High</text>
    <text x="0" y="5" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fed7aa">Excess Data</text>
    <text x="0" y="18" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fed7aa">Declarations</text>
  </g>
  <!-- #8 Feature Envy — upper-left -->
  <g transform="translate(380,330)">
    <rect x="-64" y="-26" width="128" height="52" rx="8" fill="#3b1206" stroke="#f97316" stroke-width="1.5"/>
    <text x="0" y="-9" text-anchor="middle" font-family="monospace,ui-monospace" font-size="10" font-weight="bold" fill="#fdba74">#8 · High</text>
    <text x="0" y="7" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fed7aa">Feature Envy</text>
    <text x="0" y="19" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fed7aa">logic sai chỗ</text>
  </g>
  <!-- MEDIUM nodes (r≈405) -->
  <!-- #9 Code Health Degradations — right -->
  <g transform="translate(930,435)">
    <rect x="-64" y="-26" width="128" height="52" rx="8" fill="#3b2000" stroke="#eab308" stroke-width="1.5"/>
    <text x="0" y="-9" text-anchor="middle" font-family="monospace,ui-monospace" font-size="10" font-weight="bold" fill="#fde047">#9 · Medium</text>
    <text x="0" y="5" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fef08a">Code Health</text>
    <text x="0" y="18" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fef08a">Degradations</text>
  </g>
  <!-- #10 Excess Function Arguments — lower-right -->
  <g transform="translate(790,875)">
    <rect x="-64" y="-26" width="128" height="52" rx="8" fill="#3b2000" stroke="#eab308" stroke-width="1.5"/>
    <text x="0" y="-9" text-anchor="middle" font-family="monospace,ui-monospace" font-size="10" font-weight="bold" fill="#fde047">#10 · Medium</text>
    <text x="0" y="5" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fef08a">Excess Func</text>
    <text x="0" y="18" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fef08a">Arguments 5+</text>
  </g>
  <!-- #11 Primitive Obsession — lower-left -->
  <g transform="translate(310,875)">
    <rect x="-64" y="-26" width="128" height="52" rx="8" fill="#3b2000" stroke="#eab308" stroke-width="1.5"/>
    <text x="0" y="-9" text-anchor="middle" font-family="monospace,ui-monospace" font-size="10" font-weight="bold" fill="#fde047">#11 · Medium</text>
    <text x="0" y="5" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fef08a">Primitive</text>
    <text x="0" y="18" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fef08a">Obsession</text>
  </g>
  <!-- #12 Temporal Coupling — left -->
  <g transform="translate(160,435)">
    <rect x="-64" y="-26" width="128" height="52" rx="8" fill="#3b2000" stroke="#eab308" stroke-width="1.5"/>
    <text x="0" y="-9" text-anchor="middle" font-family="monospace,ui-monospace" font-size="10" font-weight="bold" fill="#fde047">#12 · Medium</text>
    <text x="0" y="5" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fef08a">Temporal</text>
    <text x="0" y="18" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fef08a">Coupling</text>
  </g>
  <!-- #13 Mixed Abstraction Levels — top -->
  <g transform="translate(550,145)">
    <rect x="-64" y="-26" width="128" height="52" rx="8" fill="#3b2000" stroke="#eab308" stroke-width="1.5"/>
    <text x="0" y="-9" text-anchor="middle" font-family="monospace,ui-monospace" font-size="10" font-weight="bold" fill="#fde047">#13 · Medium</text>
    <text x="0" y="5" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fef08a">Mixed Abstraction</text>
    <text x="0" y="18" text-anchor="middle" font-family="monospace,ui-monospace" font-size="9" fill="#fef08a">Levels</text>
  </g>
  <!-- Legend -->
  <rect x="28" y="28" width="198" height="95" rx="8" fill="#1e293b" stroke="#334155" stroke-width="1"/>
  <text x="127" y="50" text-anchor="middle" font-family="monospace,ui-monospace" font-size="11" font-weight="bold" fill="#94a3b8">Severity</text>
  <rect x="43" y="60" width="12" height="12" rx="3" fill="#ef4444"/>
  <text x="61" y="71" font-family="monospace,ui-monospace" font-size="10" fill="#fca5a5">Critical — 4 issues</text>
  <rect x="43" y="78" width="12" height="12" rx="3" fill="#f97316"/>
  <text x="61" y="89" font-family="monospace,ui-monospace" font-size="10" fill="#fdba74">High — 5 issues</text>
  <rect x="43" y="96" width="12" height="12" rx="3" fill="#eab308"/>
  <text x="61" y="107" font-family="monospace,ui-monospace" font-size="10" fill="#fde047">Medium — 5 issues</text>
</svg>

---

## Sơ đồ mức độ nghiêm trọng theo controller

> 🔴 Critical &nbsp; 🟠 High &nbsp; 🟡 Medium &nbsp; ⚪ Không ảnh hưởng

| Issue | Dashboard | Mailbox | Thread | SingleEmail | Search | ThreadDetail | Base |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| Constructor Over-Injection | 🔴 | 🟠 | ⚪ | 🟠 | ⚪ | 🟠 | 🔴 |
| Large Method | 🔴 | 🟠 | 🔴 | 🔴 | 🟡 | ⚪ | ⚪ |
| Complex Method | 🔴 | 🔴 | 🔴 | 🔴 | 🟡 | ⚪ | 🟠 |
| Bumpy Road | 🔴 | 🔴 | 🔴 | 🔴 | 🔴 | ⚪ | ⚪ |
| Deep Nested Complexity | 🔴 | 🔴 | 🔴 | 🔴 | 🟡 | ⚪ | ⚪ |
| Low Cohesion | 🔴 | 🟠 | 🟠 | 🟠 | 🟠 | ⚪ | 🔴 |
| Excess Data Declarations | 🔴 | 🟠 | 🟡 | 🟠 | 🟠 | ⚪ | ⚪ |
| Feature Envy | 🟠 | 🔴 | 🔴 | 🔴 | 🟠 | 🟠 | ⚪ |
| Code Health Degradations | 🔴 | ⚪ | ⚪ | 🔴 | ⚪ | ⚪ | ⚪ |
| Excess Function Arguments | 🟡 | 🟠 | ⚪ | ⚪ | ⚪ | ⚪ | ⚪ |
| Primitive Obsession | ⚪ | 🟡 | 🟡 | ⚪ | 🟠 | ⚪ | ⚪ |
| Temporal Coupling | ⚪ | ⚪ | 🟠 | 🟠 | ⚪ | ⚪ | ⚪ |
| Mixed Abstraction Levels | 🟠 | 🟠 | ⚪ | 🟠 | ⚪ | ⚪ | ⚪ |
| BaseController God Class | 🔴 | 🔴 | 🔴 | 🔴 | 🔴 | 🔴 | 🔴 |

---

## So sánh kích thước controllers hiện tại

```
 Số dòng code hiện tại (Lines of Code)
 ─────────────────────────────────────────────────────────────────────
 Dashboard    ████████████████████████████████████  3,508 ← God Object
 SingleEmail  ████████████████                      1,630
 Thread       ████████████████                      1,625
 Mailbox      ████████████████                      1,602
 Search       ████████████                          1,198
 Base         ██████                                  647
 ThreadDetail ███                                     312
 ─────────────────────────────────────────────────────────────────────
              0        1,000     2,000     3,000    4,000
                                             (mỗi █ ≈ 100 lines)
 Mục tiêu sau refactor: ≤ 300 lines / controller
```

---

## Tóm tắt số liệu

| Controller | Lines | Constructor Deps | Rx Fields | Issues chính |
|---|---|---|---|---|
| MailboxDashboard | 3,508 | 43 | 42+ | God Object, 29 interactors, 31-branch dispatcher |
| MailboxController | 1,602 | 13 | 6 | Feature envy (75+ refs), _registerObxStreamListener 159 lines |
| ThreadController | 1,625 | 7 | 8 | Deep nesting, feature envy, bumpy road 15 branches |
| SingleEmailController | 1,630 | 6 mandatory + 6 optional | 12 | Optional deps, 20-branch switch, temporal coupling |
| SearchEmailController | 1,198 | 6 | 11 | 5 mixins, primitive obsession, search state redundancy |
| ThreadDetailController | 312 | 10 | — | 15 total deps (10 constructor + 5 Get.find) |
| BaseController | ~647 | 14 | 1 | God Class, imports data layer, 7 domains |

**Tổng line count của 7 controllers: ~10,522 lines**
**Mục tiêu sau refactor: ≤ 300 lines/controller**

---

## Chi tiết theo nhóm vấn đề

| File | Nhóm | Vấn đề |
|---|---|---|
| `02-size-and-complexity.md` | Kích thước & độ phức tạp | #1 Constructor Over-Injection, #2 Large Method, #3 Complex Method, #14 BaseController God Class |
| `03-coupling-and-cohesion.md` | Liên kết & gắn kết | #4 Bumpy Road, #5 Deep Nested Complexity, #6 Low Cohesion, #8 Feature Envy |
| `04-design-smells.md` | Thiết kế nhỏ lẻ | #7 Excess Data Declarations, #9 Code Health Degradations, #10 Excess Function Arguments, #11 Primitive Obsession, #12 Temporal Coupling, #13 Mixed Abstraction Levels |
