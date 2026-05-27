# Migration Plan: GetX → Riverpod

> **Scope:** Migrate all state management in tmail-flutter from GetX (GetxController + Bindings + Get.find) to Riverpod (Notifier + Provider).  
> **Strategy:** Strangler Fig — replace feature by feature, no big-bang rewrite.  
> **Existing foundation:** `flutter_riverpod ^2.6.1` already in pubspec; `ProviderContainer` singleton in place; 3 provider modules already working.

---

## 1. Why Migrate to Riverpod — And Why NOW?

### 1.1 Evidence from the Codebase Itself

The project **has already started migrating on its own** without a formal ADR:

```
Current state (May 2026):
═════════════════════════════════════════════════════════════
lib/
  main/providers/
    app_provider_container.dart      ← ProviderContainer singleton
  features/
    composer/presentation/providers/
      composer_cache_providers.dart  ← 5 Provider<T> (ADR-0086)
      composer_auto_save_notifier.dart ← StateNotifierProvider.family
    manage_account/presentation/providers/
      local_settings_notifier.dart   ← StateNotifierProvider
    network_connection/presentation/providers/
      (expanding)

pubspec.yaml:
  flutter_riverpod: ^2.6.1          ← dependency already present
```

**Clear signal:** The team has chosen Riverpod for the newest features (ADR-0086 — composer auto-save, local settings). GetX is only used in legacy code. Without a migration plan, the codebase will remain in a dual-framework limbo indefinitely — the worst outcome.

### 1.2 Technical Problems That Need Solving

```
Problem 1 — God Object that cannot be broken up with GetX
─────────────────────────────────────────────────────────
MailboxDashBoardController: 3,528 lines
  42 Rx<T> fields — all always active
  43 injected dependencies — Get.find() everywhere
  31 extension files — business logic scattered uncontrollably

→ Riverpod: each Notifier focuses on 1 domain, compiler catches missing state

Problem 2 — Get.find() global service locator
─────────────────────────────────────────────
Get.find<MailboxDashBoardController>() — 75+ times across the codebase
  → Not compile-time safe
  → Runtime crash if controller not yet registered
  → Impossible to test in isolation

→ Riverpod: ref.watch(provider) — compile-time, lazy, auto-dispose

Problem 3 — Bindings class proliferation
──────────────────────────────────────────
78 Binding files — each containing only Get.put/lazyPut boilerplate
  → No type safety for missing dependencies
  → Hidden initialization order, easy to crash

→ Riverpod: Provider graph resolves dependency order automatically

Problem 4 — Testing is too expensive
──────────────────────────────────────
Testing ThreadController currently requires mocking 43 deps from DashboardController
(because ThreadController calls Get.find<DashboardController>() to invoke methods)

→ Riverpod: override provider in test, inject mock deps cleanly
```

### 1.3 Why Riverpod (not Provider, BLoC, etc.)

```
Criterion              Riverpod   Provider   BLoC    Redux   Keep GetX
──────────────────────────────────────────────────────────────────────
Already in codebase    ✅         ❌         ❌      ❌      ✅ (current)
Compile-time safety    ✅         ⚠          ✅      ❌      ❌
Auto-dispose           ✅         ❌         ⚠       ❌      ⚠
Family providers       ✅         ❌         ❌      ❌      ❌
Testability            ✅         ✅         ✅      ✅      ❌
Flutter-independent    ✅         ❌         ❌      ✅      ❌
Learning curve         Medium     Low        High    High    ✅ (known)
Async state built-in   ✅         ❌         ✅      ❌      ⚠
Code generation opt.   ✅         ❌         ❌      ❌      ❌
──────────────────────────────────────────────────────────────────────
The team chose Riverpod in ADR-0086. This is the consistent conclusion.
```

---

## 2. Current GetX State — What Needs to Change

### 2.1 GetX Pattern Categories in the Codebase

```
Category A — GetxController lifecycle
══════════════════════════════════════
62 controller files: extends GetxController / ReloadableController / BaseController
  → Must become: Notifier<S> / AsyncNotifier<S>
  → Lifecycle onInit/onReady/onClose → ref.onDispose() / WidgetRef hooks

Category B — Reactive state (Rx<T>)
══════════════════════════════════════
~400+ reactive fields across the codebase
  Rxn<PresentationMailbox>     → StateProvider<PresentationMailbox?>
  RxBool(false)                → StateProvider<bool>
  RxList<PresentationEmail>    → StateProvider<List<PresentationEmail>>
  Rx<Either<Failure,Success>>  → AsyncNotifierProvider / AsyncValue<T>
  .obs                         → StateProvider / NotifierProvider

Category C — Dependency Injection (Bindings + Get.find)
══════════════════════════════════════════════════════════
78 binding files + 500+ Get.find() calls
  Get.put<T>(impl)             → Provider<T>((ref) => impl)
  Get.lazyPut<T>(() => impl)   → Provider<T>((ref) => impl) [lazy by default]
  Get.find<T>()                → ref.read(tProvider)
  GetxService (permanent)      → keepAlive: true

Category D — Reactive effects (ever)
══════════════════════════════════════
~50 ever() calls across the codebase
  ever(rxField, callback)      → ref.listen(provider, (prev, next) => callback)
  Obx(() => widget)            → Consumer(builder: (ctx, ref, _) => widget)
  GetBuilder<C>                → Consumer / ref.watch()

Category E — Navigation (GetX routing)
══════════════════════════════════════
Get.to / Get.off / Get.toNamed → keep as-is in phases 1–4
  → Phase 5 (optional): migrate to go_router
  → This is SEPARATE from the state management migration
```

### 2.2 GetX → Riverpod Pattern Mapping

```dart
// ═══════════════════════════════════════════════════════════════
// MAPPING REFERENCE: GetX → Riverpod
// ═══════════════════════════════════════════════════════════════

// --- REACTIVE STATE ---

// GetX:
final selectedMailbox = Rxn<PresentationMailbox>();
// Riverpod:
final selectedMailboxProvider = StateProvider<PresentationMailbox?>((ref) => null);

// GetX:
final isDrawerOpened = RxBool(false);
// Riverpod:
final isDrawerOpenedProvider = StateProvider<bool>((ref) => false);

// GetX:
final listEmailSelected = <PresentationEmail>[].obs;
// Riverpod:
final listEmailSelectedProvider = StateProvider<List<PresentationEmail>>((ref) => []);

// --- DEPENDENCY INJECTION ---

// GetX:
class SomeBindings extends Bindings {
  @override
  void dependencies() {
    Get.lazyPut(() => SomeRepository(Get.find<SomeDatasource>()));
    Get.lazyPut(() => SomeInteractor(Get.find<SomeRepository>()));
  }
}
// Riverpod:
final someRepositoryProvider = Provider<SomeRepository>(
  (ref) => SomeRepository(ref.read(someDatasourceProvider)),
);
final someInteractorProvider = Provider<SomeInteractor>(
  (ref) => SomeInteractor(ref.read(someRepositoryProvider)),
);

// --- EFFECTS (ever → listen) ---

// GetX:
ever(selectedMailbox, (mailbox) {
  if (mailbox != null) _loadEmails(mailbox);
});
// Riverpod:
ref.listen<PresentationMailbox?>(
  selectedMailboxProvider,
  (prev, next) {
    if (next != null) _loadEmails(next);
  },
);

// --- CONTROLLERS → NOTIFIERS ---

// GetX:
class ThreadController extends GetxController {
  final emailsInThread = <PresentationEmail>[].obs;
  void loadThread(MailboxId id) { ... }
}

// Riverpod:
class ThreadState {
  final List<PresentationEmail> emails;
  final AsyncValue<void> loadStatus;
  const ThreadState({this.emails = const [], this.loadStatus = const AsyncData(null)});
  ThreadState copyWith({...}) => ...;
}

class ThreadNotifier extends AsyncNotifier<ThreadState> {
  @override
  Future<ThreadState> build() async => const ThreadState();

  Future<void> loadThread(MailboxId id) async {
    state = const AsyncLoading();
    final result = await ref.read(getEmailsInteractorProvider).execute(id);
    state = result.fold(
      (failure) => AsyncError(failure, StackTrace.current),
      (success) => AsyncData(ThreadState(emails: success.emails)),
    );
  }
}

final threadNotifierProvider = AsyncNotifierProvider<ThreadNotifier, ThreadState>(
  ThreadNotifier.new,
);
```

---

## 3. Target Architecture

### 3.1 Provider Tree Structure

```
ProviderScope (main.dart)
│
├─ Infrastructure Layer (keepAlive: true)
│   ├─ authInterceptorProvider         (AuthorizationInterceptors)
│   ├─ dynamicUrlInterceptorProvider   (DynamicUrlInterceptors)
│   ├─ cachingManagerProvider          (CachingManager)
│   ├─ languageCacheManagerProvider    (LanguageCacheManager)
│   └─ dioProvider                     (Dio)
│
├─ Repository Layer (auto-dispose when not needed)
│   ├─ emailRepositoryProvider
│   ├─ mailboxRepositoryProvider
│   └─ ...
│
├─ Interactor Layer (thin wrappers, Provider<T>)
│   ├─ sendEmailInteractorProvider
│   ├─ markAsReadInteractorProvider
│   └─ ...
│
├─ Session Layer (keepAlive: true, reset on logout)
│   ├─ accountIdProvider               (StateProvider<AccountId?>)
│   ├─ sessionProvider                 (StateProvider<Session?>)
│   └─ userIdentitiesProvider          (NotifierProvider)
│
├─ Feature State (auto-dispose with screen)
│   ├─ dashboardNotifierProvider       (AsyncNotifierProvider)
│   ├─ threadNotifierProvider          (AsyncNotifierProvider)
│   ├─ mailboxNotifierProvider         (AsyncNotifierProvider)
│   └─ composerAutoSaveProvider        (StateNotifierProvider.family) ← already exists
│
└─ Cross-feature Shared State (keepAlive: true)
    ├─ selectedMailboxProvider         (StateProvider)
    ├─ selectedEmailProvider           (StateProvider)
    └─ localSettingsNotifierProvider   ← already exists
```

### 3.2 Bridge Pattern (Transition Period)

During migration, GetX controllers and Riverpod providers coexist. The bridge pattern is already established:

```dart
// Already in codebase: app_provider_container.dart
final appProviderContainer = ProviderContainer();

// In a GetX controller that needs to read Riverpod state:
final threadConfig = appProviderContainer
    .read(localSettingsNotifierProvider)
    .threadConfig; // ← currently used at thread_controller.dart:138

// Rule: GetX → Riverpod only (one direction)
// NEVER: Riverpod → GetX (signals the GetX component must be migrated next)
```

---

## 4. Migration Phases

### Phase 0 — Setup (1–2 days) ✅ Mostly Done

```yaml
# pubspec.yaml — add:
dependencies:
  flutter_riverpod: ^2.6.1      # ✅ already present
  riverpod_annotation: ^2.6.1   # add

dev_dependencies:
  riverpod_generator: ^2.6.1    # add
  build_runner: ^2.4.0          # add if not present
```

```dart
// main.dart — wrap with ProviderScope:
runApp(
  ProviderScope(
    parent: appProviderContainer, // ← bridge to singleton
    child: const TMailApp(),
  ),
);
```

---

### Phase 1 — Infrastructure Providers (1 week)

**Goal:** Migrate singleton services out of GetX Bindings.

```dart
// lib/main/providers/infrastructure_providers.dart

final dioProvider = Provider<Dio>((ref) {
  final dio = Dio();
  ref.onDispose(dio.close);
  return dio;
});

final authInterceptorProvider = Provider<AuthorizationInterceptors>(
  (ref) => AuthorizationInterceptors(ref.read(dioProvider)),
);

final cachingManagerProvider = Provider<CachingManager>(
  (ref) => CachingManager(),
);
```

```dart
// lib/main/providers/session_providers.dart
final accountIdProvider = StateProvider<AccountId?>((ref) => null);
final sessionProvider   = StateProvider<Session?>((ref) => null);

// On logout — reset all session state:
// ref.read(accountIdProvider.notifier).state = null;
// ref.read(sessionProvider.notifier).state   = null;
// → All dependent providers auto-invalidate
```

---

### Phase 2 — Repository & Interactor Providers (1 week)

**Goal:** Replace 78 Bindings classes with provider files.

```dart
// BEFORE: email_action_interactor_bindings.dart
class EmailActionInteractorBindings extends Bindings {
  @override
  void dependencies() {
    Get.lazyPut(() => AddALabelToAnEmailInteractor(_emailRepository));
    Get.lazyPut(() => RemoveALabelFromAnEmailInteractor(_emailRepository));
  }
}

// AFTER: email_action_providers.dart
final addLabelToEmailProvider = Provider<AddALabelToAnEmailInteractor>(
  (ref) => AddALabelToAnEmailInteractor(ref.read(emailRepositoryProvider)),
);

final removeLabelFromEmailProvider = Provider<RemoveALabelFromAnEmailInteractor>(
  (ref) => RemoveALabelFromAnEmailInteractor(ref.read(emailRepositoryProvider)),
);
```

**Platform overrides** (replaces platform-specific Binding subclasses):

```dart
// lib/main/providers/platform_overrides.dart
List<Override> buildPlatformOverrides() {
  if (PlatformInfo.isWeb) {
    return [
      composerCacheDatasourceProvider.overrideWith(
        (ref) => ComposerSessionCacheDatasourceImpl(
          ref.read(cacheExceptionThrowerProvider),
        ),
      ),
    ];
  }
  return [
    composerCacheDatasourceProvider.overrideWith(
      (ref) => ComposerPersistentCacheDatasourceImpl(
        ref.read(composerHiveCacheClientProvider),
        ref.read(cacheExceptionThrowerProvider),
      ),
    ),
  ];
}
```

---

### Phase 3 — Leaf Controllers (2–3 weeks)

**Goal:** Migrate simple, lightly-coupled controllers first.

**Priority order:**

```
Priority 1 — Standalone (no DashboardController dependency):
  SpamReportController       → SpamReportNotifier
  AppGridDashboardController → AppGridNotifier
  DownloadController         → DownloadNotifier

Priority 2 — Feature controllers:
  AdvancedFilterController   → AdvancedFilterNotifier
  SearchController           → SearchNotifier

Priority 3 — Core feature controllers:
  MailboxController          → MailboxNotifier
  ThreadController           → ThreadNotifier
  SingleEmailController      → SingleEmailNotifier
```

**Controller migration template:**

```dart
// BEFORE — GetX:
class SpamReportController extends GetxController {
  final spamReportState = Rxn<SpamReportState>();
  final unreadSpamCount = RxInt(0);

  @override
  void onInit() {
    super.onInit();
    _loadSpamState();
  }
}

// AFTER — Riverpod:
class SpamReportState {
  final SpamReportData? data;
  final int unreadCount;
  const SpamReportState({this.data, this.unreadCount = 0});
}

class SpamReportNotifier extends AsyncNotifier<SpamReportState> {
  @override
  Future<SpamReportState> build() async => _loadSpamState();

  Future<SpamReportState> _loadSpamState() async {
    final result = await ref
        .read(getSpamReportStateInteractorProvider)
        .execute();
    return result.fold(
      (failure) => throw failure,
      (success) => SpamReportState(
        data: success.state,
        unreadCount: success.unreadCount,
      ),
    );
  }
}

final spamReportNotifierProvider =
    AsyncNotifierProvider<SpamReportNotifier, SpamReportState>(
  SpamReportNotifier.new,
);
```

**Lifecycle mapping:**

```dart
// onInit()  → build() in Notifier (runs once when provider is created)
// onClose() → ref.onDispose() inside build()
// ever(rx, callback) → ref.listen(provider, (prev, next) => callback)
// Obx(() => widget)  → Consumer(builder: (ctx, ref, _) => widget)
//                    → or ref.watch() in ConsumerWidget.build()
```

---

### Phase 4 — Core Controllers: Dashboard + Composer (3–4 weeks)

**This is the hardest phase.** `MailboxDashBoardController` (3,528 lines) must be decomposed into multiple focused Notifiers.

#### 4.1 Decompose MailboxDashBoardController

```
MailboxDashBoardController (3,528 lines)
→ Split by domain:

  DashboardSessionNotifier         (~150 lines)
    selectedMailbox, accountId, vacationResponse

  DashboardUINotifier              (~200 lines)
    dashboardRoute, currentSelectMode, isDrawerOpened,
    isContextMenuOpened, filterMessageOption

  EmailSelectionNotifier           (~150 lines)
    listEmailSelected, emailsInCurrentMailbox

  EmailFlagNotifier                (~200 lines)
    mark-as-read, mark-as-starred (replaces UIAction dispatch)

  ComposerOrchestrationNotifier    (~250 lines)
    openComposer, closeComposer, restoreFromCache, listSendingEmails

  DragDropNotifier                 (~100 lines)
    isDraggingMailbox, attachmentDraggableAppState

  VacationNotifier                 (~100 lines)
    vacationResponse, updateVacation
```

**Cross-notifier communication** (replaces `Rxn<UIAction>` + `ever()`):

```dart
// ThreadNotifier reacts to selectedMailbox changes automatically:
class ThreadNotifier extends AsyncNotifier<ThreadState> {
  @override
  Future<ThreadState> build() async {
    // ref.watch() re-runs build() when selectedMailbox changes
    final mailbox = ref.watch(selectedMailboxProvider);
    if (mailbox == null) return const ThreadState();
    return _loadEmails(mailbox.id);
  }
}
```

#### 4.2 Decompose ComposerController

```
ComposerController →
  ComposerAutoSaveNotifier    ✅ done (ADR-0086)
  ComposerDraftNotifier
  ComposerRecipientsNotifier
  ComposerAttachmentNotifier
  ComposerSendNotifier
```

#### 4.3 Replace BaseController

`BaseController` (659 lines) handles logout, session, network — cross-cutting concerns:

```dart
// Replace with reactive session pattern:
class SomeNotifier extends AsyncNotifier<SomeState> {
  @override
  Future<SomeState> build() async {
    final session = ref.watch(sessionProvider); // reacts to logout
    if (session == null) return const SomeState();
    return _loadData(session.accountId);
  }
}
// On logout: sessionProvider.state = null
// → All dependent Notifiers rebuild → state clears automatically
```

---

### Phase 5 — Routing & Cleanup (1–2 weeks)

**Goal:** Remove GetX entirely from the dependency stack (optional).

```dart
// go_router integration
final routerProvider = Provider<GoRouter>((ref) {
  final session = ref.watch(sessionProvider);
  return GoRouter(
    initialLocation: session != null ? '/dashboard' : '/login',
    routes: [
      GoRoute(path: '/login',     builder: (ctx, state) => const LoginView()),
      GoRoute(path: '/dashboard', builder: (ctx, state) => const DashboardView()),
    ],
  );
});

// MaterialApp.router — no more GetMaterialApp needed:
MaterialApp.router(routerConfig: ref.watch(routerProvider))
```

---

## 5. Testing Strategy

```dart
// Riverpod testing: override providers with mocks
test('loads emails when mailbox selected', () async {
  final container = ProviderContainer(
    overrides: [
      getEmailsInteractorProvider.overrideWithValue(
        MockGetEmailsInteractor(),
      ),
      selectedMailboxProvider.overrideWith((ref) => mockMailbox),
    ],
  );
  addTearDown(container.dispose);

  final notifier = container.read(threadNotifierProvider.notifier);
  await notifier.loadEmails();

  final state = container.read(threadNotifierProvider);
  expect(state.value?.emails, hasLength(5));
});
```

---

## 6. Risk Management

```
Risk 1 — Breaking changes when deleting a GetX controller
──────────────────────────────────────────────────────────
Mitigation: Only delete Binding after ALL consumers have migrated.
            Use feature flags to toggle old/new implementation.

Risk 2 — Memory leaks from keepAlive providers after logout
────────────────────────────────────────────────────────────
Mitigation: All session providers depend on sessionProvider.
            Logout: invalidate sessionProvider → cascade invalidation.

Risk 3 — Performance regression from over-rebuilding
──────────────────────────────────────────────────────
Mitigation: Use select() to watch a slice of state:
              ref.watch(threadProvider.select((s) => s.emails.length))
            Use Consumer scope narrowly, not at the screen level.

Risk 4 — State desync between two frameworks in transition
──────────────────────────────────────────────────────────
Mitigation: Enforce one direction: GetX → Riverpod via appProviderContainer.
            Never let GetX widgets read Riverpod state in build().
```

---

## 7. Timeline

```
PHASE               DURATION    DELIVERABLE
══════════════════════════════════════════════════════════════
Phase 0 (Setup)     Complete    pubspec, ProviderContainer, 3 providers
Phase 1 (Infra)     1 week      ~10 infrastructure providers
Phase 2 (DI)        1 week      78 Bindings → Provider files
Phase 3 (Leaf)      2–3 weeks   10 leaf controllers → Notifiers
Phase 4 (Core)      3–4 weeks   Dashboard + Composer → Notifiers
Phase 5 (Routing)   1–2 weeks   (optional) go_router migration
──────────────────────────────────────────────────────────────
TOTAL               8–11 weeks  100% GetX state management removed
```

---

## 8. Before vs. After

```
Metric                    GetX (current)       Riverpod (target)    Change
══════════════════════════════════════════════════════════════════════════
Dependency safety         Runtime crash        Compile-time error   ✅ Better
Test isolation            Hard (mock GetX)     Easy (ProviderScope) ✅ Better
Auto-dispose              Manual / binding     Automatic (scoped)   ✅ Better
State reset on logout     Manual clearState    Invalidate cascade   ✅ Better
Platform overrides        Binding subclasses   Provider.override()  ✅ Cleaner
Learning curve (new dev)  Know GetX            Learn Riverpod       ⚠ Harder
DI lines                  78 Binding files     ~30 provider files   ✅ Fewer
Reactive UI perf          Granular (Obx)       Granular (ref.watch) = Same
Navigation                Get.to/off           go_router (Phase 5)  → To migrate
Framework lock-in         GetX tightly         Riverpod loosely     ✅ Less
```

---

## Related Documents

| Document | Content |
|---|---|
| [`01-problem-statement.md`](01-problem-statement.md) | Root causes in detail |
| [`07-why-we-chose-getxservice-eventbus.md`](07-why-we-chose-getxservice-eventbus.md) | Interim solution (ADR-0076) |

**Code references (actual codebase):**
- `lib/main/providers/app_provider_container.dart` — ProviderContainer singleton
- `lib/features/composer/presentation/providers/` — First provider pattern in the project
- `lib/features/manage_account/presentation/providers/local_settings_notifier.dart` — StateNotifierProvider
- `lib/features/mailbox_dashboard/presentation/controller/mailbox_dashboard_controller.dart` — God Object to decompose (3,528 lines)
