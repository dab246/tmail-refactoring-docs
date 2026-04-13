# Technical: Decisions & Risks

> Summary from [`04-shared-state-options.md`](04-shared-state-options.md) and [`05-communication-options.md`](05-communication-options.md)

---

## Decision 1: Use GetxService for Shared State

```
 DECISION: Create EmailListStateService, ComposerStateService,
            DragDropStateService extends GetxService

 RATIONALE:
   1. GetX-native — no extra dependency, does not violate the constraint
   2. Exactly the right design for this use case (Flutter/GetX documentation)
   3. Fully reactive with .obs — Obx() widgets still work
   4. Clear separation of concerns: each service = 1 domain
   5. Constructor injectable → testable

 WHAT WAS NOT CHOSEN AND WHY:
   - GetxController permanent: true → unclear semantic, prone to misuse
   - Static singleton → not reactive, not testable
   - InheritedWidget/Provider → violates the "keep GetX" constraint

 ACCEPTED TRADE-OFFS:
   - GetxService lives forever → must be careful about memory
   - Mitigation: each service only holds state belonging to 1 small domain
```

---

## Decision 2: Use AppEventBus for Inter-Controller Communication

```
 DECISION: Create AppEventBus extends GetxService
            with sealed class AppEvent + whereType<T>()

 RATIONALE:
   1. Zero coupling: controllers do not import each other
   2. Type-safe: sealed class → compiler catches errors
   3. Testable: inject mock event bus, emit test events
   4. OCP: adding new event types does not modify old code
   5. Single stream → easy to debug, easy to log all events
   6. No external package needed — uses pure Dart Stream

 WHAT WAS NOT CHOSEN AND WHY:
   - Rxn<UIAction>: not type-safe, callback spaghetti → THIS IS THE PROBLEM TO FIX
   - Get.find() direct: tight coupling → THIS IS THE PROBLEM TO FIX
   - RxDart: requires extra package, Dart Stream is sufficient
   - Callback injection: only 1:1, not 1:N

 ACCEPTED TRADE-OFFS:
   - AppEvent hierarchy must be carefully designed upfront
   - "Event spaghetti" if overused → mitigated by the rule:
     use EventBus only for cross-feature events,
     use Rx .obs directly for intra-controller state
```

---

## Summary of All Decisions

```
 Problem                    Chosen Solution         Discarded Alternatives
 ──────────────────────────────────────────────────────────────────────
 Shared State               GetxService             God Object,
 (5 controllers need         (permanent,             Static singleton,
 to read the same state)     injectable,             InheritedWidget
                             reactive .obs)

 Inter-Controller            AppEventBus             Rxn<UIAction> + ever(),
 Communication               (GetxService,           Get.find() direct,
 (cross-feature events)      sealed AppEvent,        RxDart
                             whereType<T>())

 Constructor Over-Injection Domain Contracts        Fat interfaces,
 (43 deps → Feature Envy)  (fine-grained           Monolithic contract
                            interfaces)

 BaseController God Class   ISessionTokenRefresher  Direct data-layer
 (imports data layer)       ILanguageSettingService  imports
                            IAuthService
                            (abstractions in domain)
```

---

## Risks of GetxService

| Risk | Description | Mitigation |
|---|---|---|
| **Memory** | GetxService is never disposed → memory leak if too much data is held | Each service only holds the state it needs. Clear data on logout. |
| **God Service** | GetxService becomes the new God Object | Rule: each service ≤ 10 Rx fields, 1 domain only |
| **Stale state** | State from an old session persists when a new account logs in | `SessionController` dispatches `SessionExpiredEvent` → services clear state |
| **Circular inject** | Service A injects Service B which injects Service A | Map the dependency graph before Phase 1 |

---

## Risks of AppEventBus

| Risk | Description | Mitigation |
|---|---|---|
| **Event Spaghetti** | Too many events, hard to trace who dispatches what | Limit: cross-feature events only. Intra-controller uses .obs. |
| **Event ordering** | Order is not guaranteed when multiple dispatches happen simultaneously | Design events to be idempotent. Do not dispatch events that depend on order. |
| **Circular dispatch** | A dispatches EventX → B handles it, B dispatches EventY → A handles it, loop | Rule: do not dispatch an event inside an event handler of the same type |
| **Memory leak subscription** | Forgetting to cancel a subscription | `compositeSubscription` + `addTo()` → auto-cancel when the controller disposes |
| **Event lost** | Dispatch happens before subscribe | `broadcast()` stream: OK — only receives events AFTER subscribing. Document this. |

---

## Rules to Apply to Avoid Anti-patterns

```
 DOs:                                     DON'Ts:
 ──────────────────────────────────       ──────────────────────────────────
 ✅ Use EventBus for cross-feature        ❌ Dispatch an event inside an event
    events (A → B, different feature)        handler of the same event type (loop)

 ✅ Use .obs directly for                 ❌ Subscribe to the entire AppEvent stream
    intra-controller state                   then filter with if-else (same as before)

 ✅ Each GetxService = 1 domain,          ❌ Create 1 GetxService holding everything
    ≤ 10 Rx fields                           (God Service)

 ✅ Subscribe with whereType<T>()         ❌ Manually cast events (as MyEvent)
    → compile-time safety

 ✅ addTo(compositeSubscription)          ❌ Hold a StreamSubscription manually
    → auto cleanup                           without cancelling it

 ✅ EventBus for domain events only       ❌ Use EventBus for UI events
    (business actions)                       (button tap, scroll)
```

---

## Summary of Architectural Decisions

```
 ┌─────────────────────────────────────────────────────────────────┐
 │                    FINAL DECISIONS                               │
 │                                                                 │
 │  1. SHARED STATE: GetxService (permanent)                       │
 │     → EmailListStateService, ComposerStateService,              │
 │       DragDropStateService                                      │
 │     → Reason: GetX-native, reactive, injectable, 1 domain/svc  │
 │                                                                 │
 │  2. INTER-CONTROLLER COMM: AppEventBus (GetxService)            │
 │     → sealed class AppEvent + whereType<T>()                    │
 │     → Reason: Zero coupling, type-safe, testable, no extra pkg  │
 │                                                                 │
 │  3. DEPENDENCY INJECTION: Domain Contracts                      │
 │     → IEmailListStateContract, IMailboxNavigationContract...    │
 │     → Reason: Separate abstraction from implementation, ISP     │
 │                                                                 │
 │  4. KEEP AS-IS: GetX state management                           │
 │     → GetxController, .obs, Obx(), GetPage, Get.toNamed         │
 │     → Reason: Constraint from the start, team cost too high     │
 │               to change                                         │
 └─────────────────────────────────────────────────────────────────┘
```