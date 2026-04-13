# Technical: Comparison of Inter-Controller Communication Options

> **The problem:** When a user selects a mailbox, how does MailboxController notify ThreadController and SearchController?
> **Constraint:** Keep GetX as-is; do not create circular dependencies.

---

## Options Considered

```
 Option 1: Rxn<UIAction> + ever() (current)
 Option 2: Get.find() direct (current)
 Option 3: AppEventBus / StreamController (proposed)
 Option 4: GetX Workers (ever, debounce, once)
 Option 5: RxDart BehaviorSubject / PublishSubject
 Option 6: Callback / Function injection
```

---

## Detailed Comparison

| Criteria | Rxn\<UIAction\> | Get.find() direct | AppEventBus | GetX Workers | RxDart | Callback injection |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| **Type safety** | ❌ (runtime cast) | ✅ | ✅ (sealed class) | ⚠️ | ✅ | ✅ |
| **Decoupled (A does not know B)** | ❌ | ❌ | ✅ | ❌ | ✅ | ❌ |
| **Easy to test** | ❌ | ❌ | ✅ | ⚠️ | ✅ | ✅ |
| **Auto-cleanup on dispose** | ⚠️ (manual) | N/A | ✅ (compositeSubscription) | ✅ | ⚠️ | ✅ |
| **No extra dependency needed** | ✅ | ✅ | ✅ (GetX stream) | ✅ | ❌ | ✅ |
| **Integration with GetX** | ✅ | ✅ | ✅ | ✅ | ⚠️ | ✅ |
| **Multiple subscribers** | ✅ (ever × N) | N/A | ✅ (broadcast) | ✅ | ✅ | ❌ (1:1) |
| **Handle 1 event type** | ❌ (if-else chain) | N/A | ✅ (whereType) | ❌ | ✅ | ✅ |
| **Circular dep risk** | ❌ (high) | ❌ (high) | ✅ (zero) | ❌ (high) | ✅ | ⚠️ |
| **Extra package** | No | No | No | No | **Yes** | No |
| **Debug/trace event** | Difficult | N/A | ✅ (single stream) | Difficult | ✅ | Difficult |

---

## Analysis of Each Option

### Option 1 — `Rxn<UIAction>` + `ever()` (current)

```
 Mechanism:
   ever(dashboard.rxnMailboxUIAction, (action) {
     if (action is OpenMailboxAction) { ... }
     else if (action is RefreshAction) { ... }
     // 7+ action types in 1 closure
   });

 Pros:
   + Already available in GetX, no extra code needed
   + Simple with 1–2 action types

 Cons:
   - Rxn<UIAction> is not type-safe → runtime cast, may crash
   - 1 observable holds all action types → callback is 85–159 lines
   - Every subscriber receives ALL events, including unrelated ones (noise)
   - Consumer must know the field name (mailboxUIAction) inside the God Object
   - Impossible to add new action type without modifying the closure

 Conclusion: ❌ This is exactly the problem to fix
```

### Option 2 — `Get.find<T>()` direct

```
 Mechanism:
   final dashboard = Get.find<MailboxDashBoardController>();
   dashboard.doSomething(); // direct call

 Pros:
   + Simple, explicit

 Cons:
   - A must import and know B exists → tight coupling
   - Get.find() may crash if B has not been registered
   - 75+ references → God Object cannot be split
   - Tests require mocking all of B
   - Circular dependency risk: A→B→A

 Conclusion: ❌ This is exactly the problem to fix
```

### Option 3 — AppEventBus (proposed)

```
 Mechanism:
   // Publisher (unaware of who subscribes):
   _eventBus.dispatch(MailboxSelectedEvent(mailbox))

   // Subscriber (unaware of who dispatches):
   _eventBus.stream
       .whereType<MailboxSelectedEvent>()
       .listen(_onMailboxSelected)
       .addTo(compositeSubscription)

 Pros:
   + Zero coupling: A does not import B, B does not import A
   + Type-safe: sealed class + whereType<T>() → compile error if wrong
   + Each subscriber only receives the event type it needs → no noise
   + Testable: inject mockEventBus into constructor, emit test events
   + Auto-cleanup via compositeSubscription.dispose()
   + Easy to add new event types without modifying existing subscribers (OCP)
   + No extra package needed — uses pure Dart Stream
   + Single stream → easy to debug/log

 Cons:
   - AppEvent hierarchy must be carefully designed
   - Event dispatch order is not guaranteed in some edge cases
   - If overused: "Event spaghetti" — hard to trace who dispatches what

 Conclusion: ✅ Selected
```

### Option 4 — GetX Workers (ever, debounce, once)

```
 Mechanism:
   ever(someRxVariable, (value) { ... });
   debounce(searchQuery, (q) { search(q); });

 Pros:
   + Native GetX, no extra code needed
   + Good for reactive state changes (debounce, throttle)

 Cons:
   - Worker.ever() still needs to know the observable of another controller
   - Does not solve the coupling problem
   - Not an event bus — does not support 1 event → N subscribers
   - Suitable for local state changes, not cross-controller events

 Conclusion: ⚠️ Use as supplement (debounce for search input),
             not for inter-controller communication
```

### Option 5 — RxDart BehaviorSubject / PublishSubject

```
 Mechanism:
   final _subject = BehaviorSubject<MailboxSelectedEvent>();
   _subject.stream.listen(handler);
   _subject.add(MailboxSelectedEvent(mailbox));

 Pros:
   + Powerful: BehaviorSubject retains last value
   + Type-safe per subject
   + Good if RxDart is already used in the project

 Cons:
   - REQUIRES ADDING PACKAGE rxdart → extra dependency
   - Current project does not use RxDart → learning curve
   - Many separate subjects instead of 1 unified bus → harder to manage
   - BehaviorSubject stores last value → may cause unintended side effects

 Conclusion: ❌ Not necessary — pure Dart Stream is sufficient
             Avoid adding an unnecessary dependency
```

### Option 6 — Callback / Function injection

```
 Mechanism:
   class ThreadController {
     final Function(PresentationMailbox) onMailboxSelected;
     ThreadController(this.onMailboxSelected, ...);
   }

 Pros:
   + Explicit, compile-time safe
   + Easy to test: pass a lambda in

 Cons:
   - Only supports 1:1 (1 producer → 1 consumer)
   - Does not support 1:N (1 event → multiple subscribers)
   - Bindings become complex when there are many callbacks
   - Not scalable when adding new controllers

 Conclusion: ❌ Suitable for 1:1 delegation, not cross-cutting events
```

---

## Decision Matrix — Inter-Controller Communication

```
 Criteria              Weight     Rxn<UIAction>  Get.find()  EventBus  GetX Workers  RxDart
 ──────────────────────────────────────────────────────────────────────────────────────────
 Decoupling                 30%        ❌ 0          ❌ 0       ✅ 3        ❌ 0        ✅ 3
 Type safety                25%        ❌ 0          ✅ 3       ✅ 3        ⚠ 2        ✅ 3
 Testability                20%        ❌ 0          ❌ 0       ✅ 3        ⚠ 1        ✅ 3
 No extra dependency        15%        ✅ 3          ✅ 3       ✅ 3        ✅ 3        ❌ 0
 Multiple subscribers       10%        ✅ 3          ❌ 0       ✅ 3        ✅ 3        ✅ 3
 ──────────────────────────────────────────────────────────────────────────────────────────
 Total score               100%        0.30          0.90       3.00       1.15        2.55
 ──────────────────────────────────────────────────────────────────────────────────────────
                                        ❌            ❌         ✅ WINNER   ⚠          ✅*
                                                              (* RxDart requires extra package)
```

**Result: AppEventBus with a score of 3.00 — the highest, meeting all criteria without any additional dependency.**

See the final decision and risks at [`06-decisions-and-risks.md`](06-decisions-and-risks.md).