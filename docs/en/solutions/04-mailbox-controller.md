# Solution: MailboxController

**Problem:** Feature Envy (75+ refs to dashboard), `_registerObxStreamListener()` 159 lines, Low Cohesion scroll methods, Excess Data Declarations.

---

## Specific problems

```
MailboxController (1,602 lines)
├── Feature Envy: 75+ references to MailboxDashBoardController
├── _registerObxStreamListener(): 159 lines, 2 nested ever() with 7+ action types each
├── handleMailboxAction(): 112 lines, 18 switch cases
├── handleSuccessViewState(): 32 lines, 12 if-else branches
├── Low Cohesion: autoScrollTop/Bottom/Stop — does not use business fields
├── Excess Declarations: _activeScrollTop + _activeScrollBottom (2 bools for 1 concept)
├── Get.find<MailboxDashBoardController>() in field declaration
└── Temporal Coupling: _registerObxStreamListener() must run before onReady()
```

---

## Solution 1 — Fix Feature Envy via Domain Contracts

**Instead of depending directly on MailboxDashBoardController:**
```dart
// Before
final mailboxDashBoardController = Get.find<MailboxDashBoardController>();
// ... 75 access points: mailboxDashBoardController.selectedMailbox.value
```

**Use fine-grained contracts:**
```dart
// lib/features/mailbox_dashboard/domain/contracts/mailbox_navigation_contract.dart
abstract class IMailboxNavigationContract {
  Rxn<PresentationMailbox> get selectedMailbox;
  void selectMailbox(PresentationMailbox mailbox);
  void openDefaultMailbox();
}

// lib/features/mailbox_dashboard/domain/contracts/email_list_contract.dart
abstract class IEmailListStateContract {
  RxList<PresentationEmail> get emails;
  RxBool get isLoading;
  Map<Role, MailboxId> get mailboxIdByRole;
  Map<MailboxId, PresentationMailbox> get mailboxById;
  void updateMailboxMaps({...});
}
```

```dart
// MailboxController receives contracts via constructor
class MailboxController extends BaseController {
  final IMailboxNavigationContract _navigation;
  final IEmailListStateContract _emailListState;
  final AppEventBus _eventBus;

  // Business interactors — mailbox-related only
  final CreateNewMailboxInteractor _createMailbox;
  final DeleteMultipleMailboxInteractor _deleteMailbox;
  final RenameMailboxInteractor _renameMailbox;
  final MoveMailboxInteractor _moveMailbox;
  // ...

  MailboxController(
    this._navigation,
    this._emailListState,
    this._eventBus,
    this._createMailbox,
    this._deleteMailbox,
    this._renameMailbox,
    this._moveMailbox,
    // ... (≤8 total)
    ISessionTokenRefresher tokenRefresher,
    ILanguageSettingService languageService,
    IAuthService authService,
  ) : super(tokenRefresher, languageService, authService, ...);
}
```

---

## Solution 2 — Fix `_registerObxStreamListener()` 159 lines

**Before — 1 method, 2 nested `ever()` with 15+ action types:**
```dart
void _registerObxStreamListener() {
  ever(mailboxDashBoardController.mailboxUIAction, (action) {
    // 7+ action types, if-else chain
  });
  ever(mailboxDashBoardController.viewState, (viewState) {
    // 110+ lines of if-else
  });
}
```

**After — AppEventBus + split into smaller methods:**
```dart
@override
void onInit() {
  super.onInit();
  _subscribeToMailboxEvents();
  _subscribeToNavigationEvents();
}

void _subscribeToMailboxEvents() {
  _eventBus.stream
      .whereType<MailboxSelectedEvent>()
      .listen(_onMailboxSelected)
      .addTo(compositeSubscription);

  _eventBus.stream
      .whereType<MailboxRefreshRequestedEvent>()
      .listen((_) => _refreshAllMailboxes())
      .addTo(compositeSubscription);

  _eventBus.stream
      .whereType<MailboxCreatedEvent>()
      .listen(_onMailboxCreated)
      .addTo(compositeSubscription);
}

void _subscribeToNavigationEvents() {
  _eventBus.stream
      .whereType<DefaultMailboxOpenedEvent>()
      .listen((_) => _selectDefaultMailbox())
      .addTo(compositeSubscription);

  _eventBus.stream
      .whereType<SessionInitializedEvent>()
      .listen(_onSessionInitialized)
      .addTo(compositeSubscription);
}

// Each handler ≤ 15 lines, 1 concern
void _onMailboxSelected(MailboxSelectedEvent event) {
  _currentSelectedMailbox = event.mailbox;
  _updateMailboxTreeSelection(event.mailbox);
}

void _onSessionInitialized(SessionInitializedEvent event) {
  _loadAllMailboxes(event.session, event.accountId);
}
```

---

## Solution 3 — Fix `handleMailboxAction()` 112 lines, 18 cases

**Before — 1 switch with 18 cases:**
```dart
void handleMailboxAction(BuildContext context, MailboxActions actions, PresentationMailbox mailbox) {
  switch(actions) {
    case MailboxActions.delete: _showDeleteDialog(context, mailbox);
    case MailboxActions.rename: _showRenameDialog(context, mailbox);
    case MailboxActions.move: _showMoveDialog(context, mailbox);
    case MailboxActions.markAsRead: _markMailboxAsRead(mailbox);
    case MailboxActions.emptyTrash: _emptyTrash(context);
    case MailboxActions.emptySpam: _emptySpam(context);
    case MailboxActions.newSubfolder: _showCreateSubfolderDialog(context, mailbox);
    case MailboxActions.createFilter: _navigateToCreateFilter(mailbox);
    case MailboxActions.openInNewTab: _openInNewTab(mailbox);
    case MailboxActions.copySubaddress: _copySubaddress(mailbox);
    // ... 8 more
  }
}
```

**After — grouped by category + command pattern:**
```dart
// Split into 3 smaller groups
void handleMailboxAction(BuildContext context, MailboxActions action, PresentationMailbox mailbox) {
  if (_isDialogAction(action)) {
    _handleDialogAction(context, action, mailbox);
  } else if (_isDirectAction(action)) {
    _handleDirectAction(action, mailbox);
  } else {
    _handleNavigationAction(context, action, mailbox);
  }
}

bool _isDialogAction(MailboxActions action) => const {
  MailboxActions.delete,
  MailboxActions.rename,
  MailboxActions.move,
  MailboxActions.emptyTrash,
  MailboxActions.emptySpam,
  MailboxActions.newSubfolder,
}.contains(action);

void _handleDialogAction(BuildContext context, MailboxActions action, PresentationMailbox mailbox) {
  switch (action) {
    case MailboxActions.delete: _showDeleteDialog(context, mailbox);
    case MailboxActions.rename: _showRenameDialog(context, mailbox);
    case MailboxActions.move: _showMoveDialog(context, mailbox);
    case MailboxActions.emptyTrash: _showEmptyTrashDialog(context);
    case MailboxActions.emptySpam: _showEmptySpamDialog(context);
    case MailboxActions.newSubfolder: _showCreateSubfolderDialog(context, mailbox);
    default: break;
  }
}

void _handleDirectAction(MailboxActions action, PresentationMailbox mailbox) {
  switch (action) {
    case MailboxActions.markAsRead: _markMailboxAsRead(mailbox);
    case MailboxActions.copySubaddress: _copySubaddress(mailbox);
    case MailboxActions.allowSubaddressing: _allowSubaddressing(mailbox);
    case MailboxActions.disallowSubaddressing: _disallowSubaddressing(mailbox);
    default: break;
  }
}
```

---

## Solution 4 — Fix Low Cohesion: Scroll Methods

```dart
// Before: scroll methods inside MailboxController
void autoScrollTop() {
  mailboxListScrollController.animateTo(
    mailboxListScrollController.position.minScrollExtent, ...);
}
void autoScrollBottom() { ... }
void stopAutoScroll() { ... }
```

```dart
// After: extracted into MailboxScrollController or extension
// lib/features/mailbox/presentation/controller/mailbox_scroll_controller.dart
class MailboxScrollController {
  final ScrollController scrollController = ScrollController();

  void scrollToTop() {
    scrollController.animateTo(
      scrollController.position.minScrollExtent,
      duration: const Duration(seconds: 1),
      curve: Curves.easeInToLinear,
    );
  }

  void scrollToBottom() {
    scrollController.animateTo(
      scrollController.position.maxScrollExtent,
      duration: const Duration(seconds: 1),
      curve: Curves.easeInToLinear,
    );
  }

  void stopScrolling() {
    scrollController.animateTo(
      scrollController.offset,
      duration: const Duration(milliseconds: 300),
      curve: Curves.fastOutSlowIn,
    );
  }

  void dispose() => scrollController.dispose();
}
```

MailboxController injects `MailboxScrollController` via constructor.

---

## Solution 5 — Fix Excess Data Declarations

```dart
// Before: 2 separate RxBool fields
final _activeScrollTop = RxBool(false);
final _activeScrollBottom = RxBool(true);

// After: 1 enum
enum MailboxScrollEdge { atTop, middle, atBottom }
final scrollEdge = Rx<MailboxScrollEdge>(MailboxScrollEdge.atBottom);

// Before: unused field
final isMailboxListScrollable = false.obs; // nobody uses this

// Delete this field.

// Before: locally-scoped state stored as class fields
MailboxId? _newFolderId;
NavigationRouter? _navigationRouter;

// After: convert to local variables in the methods that need them
// Only retain if truly needed to persist across method calls
```

---

## handleSuccessViewState — Fix Bumpy Road

```dart
// After: only handles mailbox operation success types
@override
void handleSuccessViewState(Success success) {
  switch (success) {
    case GetAllMailboxSuccess():
      _onGetAllMailboxSuccess(success);
    case CreateNewMailboxSuccess():
      _onCreateMailboxSuccess(success);
    case DeleteMultipleMailboxAllSuccess():
    case DeleteMultipleMailboxHasSomeSuccess():
      _onDeleteMailboxSuccess(success);
    case RenameMailboxSuccess():
      _eventBus.dispatch(MailboxRenamedEvent(success.mailboxId));
    case MoveMailboxSuccess():
      _eventBus.dispatch(MailboxMovedEvent(success.mailboxId));
    default:
      super.handleSuccessViewState(success);
  }
}
```

---

## MailboxController after refactor

```dart
class MailboxController extends BaseController {
  // Contracts (no more MailboxDashBoardController reference)
  final IMailboxNavigationContract _navigation;
  final IEmailListStateContract _emailListState;
  final AppEventBus _eventBus;

  // Business interactors — mailbox-related only
  final GetAllMailboxInteractor _getAllMailbox;
  final CreateNewMailboxInteractor _createMailbox;
  final DeleteMultipleMailboxInteractor _deleteMailbox;
  final RenameMailboxInteractor _renameMailbox;
  final MoveMailboxInteractor _moveMailbox;
  final SubscribeMailboxInteractor _subscribeMailbox;

  // Scroll helper (no more inline scroll methods)
  final MailboxScrollController _scrollController;

  // Mailbox tree builder
  final MailboxTreeBuilder _treeBuilder;

  // State — mailbox-specific only
  final mailboxTree = Rxn<MailboxNode>();
  final expandMode = Rx<ExpandMode>(ExpandMode.EXPAND);
  final scrollEdge = Rx<MailboxScrollEdge>(MailboxScrollEdge.atBottom); // fix primitive obsession
}
```

**Estimated ~600 lines** (down from 1,602 lines) after:
- Removing 75+ dashboard references
- Extracting scroll logic
- Fixing `_registerObxStreamListener()` 159 → ~40 lines
- Fixing `handleMailboxAction()` 112 → ~50 lines

---

## Metrics

| | Before | After |
|---|---|---|
| Lines | 1,602 | ~600 |
| Dashboard references | 75+ | 0 (contracts only) |
| `_registerObxStreamListener()` | 159 lines | ~40 lines |
| `handleMailboxAction()` | 112 lines, 18 cases | 3 methods ~30 lines each |
| Low cohesion methods | 3 | 0 (moved to MailboxScrollController) |
| `Get.find<T>()` in fields | 1 | 0 |