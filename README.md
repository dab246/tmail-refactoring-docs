# Tmail Flutter — Architecture Refactoring

> **ADR-0076** · Refactoring the presentation layer architecture while keeping the GetX framework.
> **Date:** 2026-04-10 · **Scope:** 7 controllers (~10,522 lines total)

---

## Viewing the Documentation

```bash
# Clone repo
git clone https://github.com/<username>/tmail-refactoring-docs.git
cd tmail-refactoring-docs

# Run local server
npx serve .
# or
python -m http.server 8000
```

Open `http://localhost:3000` (or the corresponding port) in your browser.

**GitHub Pages:** `https://<username>.github.io/tmail-refactoring-docs/`

**Keyboard shortcuts:**
- `←` `→` — navigate prev/next
- `S` — toggle sidebar
- `Home` / `End` — jump to first/last doc

---

## Background

Twake Mail (tmail-flutter) is a cross-platform email client (Android/iOS/Web) built with Flutter + JMAP. After multiple development cycles, the presentation layer has accumulated serious code health issues:

```
 Total Lines of Code — 7 controllers
 ─────────────────────────────────────────────────────────────────────
 MailboxDashboard ████████████████████████████████████  3,508  ← God Object
 SingleEmail      ████████████████                      1,630
 ThreadController ████████████████                      1,625
 MailboxController████████████████                      1,602
 SearchEmail      ████████████                          1,198
 BaseController   ██████                                  647
 ThreadDetail     ███                                     312
 ─────────────────────────────────────────────────────────────────────
 Total            ~10,522 lines  |  Target: ≤ 300 lines / controller
```

---

## Documentation Structure

The documentation is divided into 3 sections, intended to be read in order:

```
 docs/
 ├── problems/      ← 1. What are the current problems?
 ├── technical/     ← 2. Why was this solution chosen?
 └── solutions/     ← 3. What does the concrete solution look like?
```

---

## 1. Problems — 14 Code Health Issues

> **Read first for context.** Each issue includes concrete code evidence with line numbers.

```
 docs/problems/
 ├── 01-overview.md            ← Index: severity heatmap, LoC chart, summary table
 ├── 02-size-and-complexity.md ← #1 Constructor Over-Injection (43 deps)
 │                                #2 Large Method (113-line dispatcher)
 │                                #3 Complex Method (CC > 15)
 │                                #14 BaseController God Class (7 domains)
 ├── 03-coupling-and-cohesion.md ← #4 Bumpy Road (31 branches)
 │                                  #5 Deep Nested Complexity (4+ levels)
 │                                  #6 Low Cohesion (methods not using class fields)
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

## 2. Technical — Why GetxService + AppEventBus?

> **Evidence-based technical decisions.** Full comparison of alternatives before selection.

```
 docs/technical/
 ├── 01-problem-statement.md    ← 2 core problems that must be solved simultaneously
 │                                 A: Where does Shared State live?
 │                                 B: How do controllers communicate?
 ├── 02-getx-service.md         ← What is GetxService, lifecycle vs GetxController
 ├── 03-event-bus.md            ← What is AppEventBus, pub/sub mechanism
 ├── 04-shared-state-options.md ← Comparison of 5 Shared State options
 │                                 → God Object / GetxService / GetxCtrl permanent
 │                                 → Static Singleton / InheritedWidget
 ├── 05-communication-options.md ← Comparison of 6 Inter-Controller Communication options
 │                                  → Rxn<UIAction> / Get.find() / AppEventBus
 │                                  → GetX Workers / RxDart / Callback injection
 └── 06-decisions-and-risks.md  ← Final decisions + risks + DOs/DON'Ts rules
```

**Decision matrices (summary):**

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

## 3. Solutions — Concrete Plan per Controller

> **Implementation blueprints.** Each file is a blueprint for one controller/component.

```
 docs/solutions/
 ├── 01-overview.md                    ← Overall before/after architecture + roadmap
 ├── 02-base-controller.md             ← Extract domain contracts, clean arch fix
 ├── 03-mailbox-dashboard-controller.md ← God Object → 4 sub-controllers + 3 services
 ├── 04-mailbox-controller.md          ← Feature Envy fix, 159-line listener split
 ├── 05-thread-controller.md           ← Deep nesting, bumpy road fix
 ├── 06-single-email-controller.md     ← Optional deps → CalendarEmailController
 ├── 07-search-email-controller.md     ← 5 mixins → SearchFilterController
 └── 08-shared-infrastructure.md       ← AppEventBus + GetxService implementation
```

**Post-refactor architecture:**

```
 BEFORE                             AFTER
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
 Total: ~5–8 months · running in parallel with the product roadmap
```

---

## Constraints

- **Keep GetX** — GetxController, `.obs`, `Obx()`, `GetPage`, `Get.toNamed()`
- **No framework migration** — no Riverpod, no Provider
- **Strangler Fig Pattern** — incremental refactor, no full rewrite
- **Don't block the roadmap** — Phase 3 (HIGH RISK) avoids release windows
