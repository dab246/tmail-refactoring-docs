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
