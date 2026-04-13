# Technical: Problem Statement

**Date:** 2026-04-10
**Context:** Refactoring tmail-flutter — resolving God Object, Feature Envy, Tight Coupling

---

## 2 Core Technical Problems

There are **2 core technical problems** that must be solved simultaneously:

```
 Problem A: SHARED STATE                Problem B: INTER-CONTROLLER COMMUNICATION
 ──────────────────────────────────    ──────────────────────────────────────────
 "Where does app state live?"           "How does Controller A notify B?"

 Currently: MailboxDashBoardController  Currently: Get.find<MailboxDashBoardController>()
 holds all state (42 Rx fields)         + ever(dashboard.rxnAction, callback)

 Consequences:                          Consequences:
  - 5 controllers depend on             - Callbacks are 85–159 line closures
    1 God Object to read state          - Tight coupling: A must know B exists
  - God Object cannot be deleted        - Not type-safe: Rxn<UIAction>
  - Tests must mock the entire          - Every controller receives noise from
    God Object                            unrelated actions
```

Both problems are **related** — Problem A cannot be fully solved without also solving Problem B.

---

## Illustration of Problem A — Current Shared State

```
 BEFORE: All state concentrated in the God Object
 ═══════════════════════════════════════════════════════════════
          ┌──────────────────────────────────────────┐
          │       MailboxDashBoardController          │
          │  emailList · isLoading · selectedMailbox  │
          │  accountId · session · listIdentities     │
          │  42 Rx fields total                       │
          └───────────────┬──────────────────────────┘
                          │
         ┌────────────────┼────────────────┐
         │                │                │
   Get.find()       Get.find()       Get.find()
         │                │                │
         ▼                ▼                ▼
 MailboxController  ThreadController  SingleEmailController
 reads selectedMailbox reads emailList  reads listIdentities
 writes mailboxMaps   writes emailList! reads accountId

 ⚠ Consequence: God Object cannot be removed/split because 5 controllers
                are directly dependent on it
```

---

## Illustration of Problem B — Current Inter-Controller Communication

```
 BEFORE: ever(rxnAction, callback) — 85 to 159 lines
 ═══════════════════════════════════════════════════════════════
 ThreadController:
   ever(mailboxDashBoardController.dashBoardAction, (action) {
     if (action is SelectEmailAction) { ... }       // email concern
     else if (action is RefreshEmailAction) { ... } // email concern
     else if (action is StartSearchEmailAction) { ... } // search concern
     else if (action is CancelSearchEmailAction) { ... } // search concern
     else if (action is FilterMessageAction) { ... }    // filter concern
     // ... 8 more action types — 85 lines total
   });

 MailboxController:
   ever(mailboxDashBoardController.mailboxUIAction, (action) {
     // ... 7 action types — 159 lines
   });

 ⚠ Problems:
   - Each callback handles multiple unrelated concerns
   - Consumers must know the field names of the God Object (tight coupling)
   - Not type-safe: Rxn<UIAction> — runtime cast, potential crash
   - Adding a new action type → must modify existing closures (violates OCP)
```

---

## Why Both Must Be Solved Simultaneously?

```
 If only Problem A is solved (extracting state into GetxService):
   GetxService.emailList  ←  ThreadController still Get.find() GetxService
   GetxService.selectedMailbox  ←  MailboxController still Get.find()
   → Coupling remains, just a smaller God Object

 If only Problem B is solved (adding EventBus):
   EventBus.dispatch(MailboxSelectedEvent)
   → But state still lives inside the God Object
   → Controllers still need Get.find() to read state

 Solving both:
   State → GetxService (injectable, reactive)
   Communication → AppEventBus (type-safe, decoupled)
   → Controllers do not need to know other controllers exist
   → God Object can be gradually decomposed (Strangler Fig Pattern)
```

---

## Next Files

| File | Content |
|---|---|
| `02-getx-service.md` | What GetxService is, lifecycle, when to use it |
| `03-event-bus.md` | What EventBus is, how it works |
| `04-shared-state-options.md` | Comparison of 5 Shared State options + decision matrix |
| `05-communication-options.md` | Comparison of 6 Inter-Controller Communication options + decision matrix |
| `06-decisions-and-risks.md` | Final decisions, rationale, risks, rules |
