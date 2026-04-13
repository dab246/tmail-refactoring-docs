# Technical: Comparing Shared State Options

> **Problem:** 5 controllers all need to read email list, selected mailbox, loading state.
> **Constraint:** Keep the GetX framework.

---

## Options Considered

```
 Option 1: God Object Controller (current)
 Option 2: GetxService (proposed)
 Option 3: GetxController permanent: true
 Option 4: Static Singleton class
 Option 5: InheritedWidget / Provider
```

---

## Detailed Comparison

| Criterion | God Object (current) | GetxService | GetxController permanent | Static Singleton | InheritedWidget |
|---|:---:|:---:|:---:|:---:|:---:|
| **Persists across route changes** | ✅ (permanent) | ✅ | ✅ | ✅ | ❌ |
| **Integrates with GetX .obs** | ✅ | ✅ | ✅ | ❌ | ❌ |
| **Obx() reactive UI** | ✅ | ✅ | ✅ | ❌ | ❌ |
| **Dispose cleanup** | ❌ (no dispose) | ✅ (onClose) | ⚠️ (requires force) | ❌ (manual) | ✅ |
| **Easy to test** | ❌ (mock 43 deps) | ✅ (inject directly) | ⚠️ | ❌ | ⚠️ |
| **Single Responsibility** | ❌ (15 concerns) | ✅ (1 concern/service) | ⚠️ | ⚠️ | ✅ |
| **State Management change needed** | No | No | No | No | **Yes** |
| **Team learning curve** | 0 (already known) | 0 (it's GetX) | 0 (it's GetX) | Low | High |
| **Constructor injection** | ✅ | ✅ | ✅ | ❌ | ❌ |
| **Fits "keep GetX" constraint** | ✅ | ✅ | ✅ | ⚠️ | ❌ |

---

## Analysis of Each Option

### Option 1 — God Object Controller (current)

```
 Pros:
   + No changes needed
   + Team is already familiar with it

 Cons:
   - Is the root cause of all the problems
   - 3,508 lines, 43 deps — unmaintainable
   - Tests require mocking 40+ dependencies
   - Everything coupled into 1 class

 Conclusion: ❌ Rejected
```

### Option 2 — GetxService (proposed)

```
 Pros:
   + GetX native — no additional dependency needed
   + Designed exactly for this use case (shared app-level state)
   + Integrates perfectly with .obs, Obx(), ever()
   + onClose() ensures cleanup
   + Constructor injectable → testable
   + Separates concerns: each service = 1 domain

 Cons:
   - Team needs to learn GetxService (low learning curve)
   - Requires careful design — avoid turning GetxService into a new God Object

 Conclusion: ✅ Chosen
```

### Option 3 — GetxController with permanent: true

```
 Pros:
   + Similar behavior to GetxService
   + Team already knows GetxController

 Cons:
   - GetxController was not designed for permanent state
   - Get.delete() can accidentally remove it if the same type is used
   - Lacks semantic clarity: "this is a service, not a controller"
   - GetX docs recommend using GetxService for this use case

 Conclusion: ⚠️ Not used — prone to misuse, unclear semantics
```

### Option 4 — Static Singleton

```
 Pros:
   + Simple

 Cons:
   - Not reactive — cannot be used with Obx()
   - Must manually manage lifecycle (memory leak risk)
   - Not injectable → not testable
   - Violates DIP
   - Completely contrary to GetX patterns

 Conclusion: ❌ Rejected
```

### Option 5 — InheritedWidget / Provider

```
 Pros:
   + Flutter standard approach
   + Testable

 Cons:
   - REQUIRES CHANGING STATE MANAGEMENT → violates constraint
   - High team learning curve
   - Does not integrate with .obs, Obx()
   - Requires rewriting the entire UI layer
   - Not feasible within the timeframe

 Conclusion: ❌ Rejected (violates "keep GetX" constraint)
```

---

## Decision Matrix — Shared State

```
 Criterion              Weight     God Object  GetxService  GetxCtrl permanent  Static
 ─────────────────────────────────────────────────────────────────────────────────────
 Keep GetX                  30%        ✅ 3        ✅ 3           ✅ 3            ⚠ 1
 Reactive (.obs/Obx)        25%        ✅ 3        ✅ 3           ✅ 3            ❌ 0
 Testability                20%        ❌ 0        ✅ 3           ⚠ 2            ❌ 0
 Single Responsibility      15%        ❌ 0        ✅ 3           ⚠ 2            ⚠ 1
 Lifecycle safety           10%        ❌ 0        ✅ 3           ⚠ 1            ❌ 0
 ─────────────────────────────────────────────────────────────────────────────────────
 Total score               100%        1.35       3.00           2.45            0.30
 ─────────────────────────────────────────────────────────────────────────────────────
                                        ❌         ✅ WINNER       ⚠               ❌
```

**Result: GetxService with a score of 3.00 — the highest, satisfying all criteria.**

See the final decision and risks at [`06-decisions-and-risks.md`](06-decisions-and-risks.md).
