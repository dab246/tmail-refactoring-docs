# Solution: MailboxController

**Vấn đề:** Feature Envy (75+ refs đến dashboard), `_registerObxStreamListener()` 159 lines, Low Cohesion scroll methods, Excess Data Declarations.

---

## Vấn đề cụ thể

```
MailboxController (1,602 lines)
├── Feature Envy: 75+ references đến MailboxDashBoardController
├── _registerObxStreamListener(): 159 lines, 2 nested ever() với 7+ action types mỗi
├── handleMailboxAction(): 112 lines, 18 switch cases
├── handleSuccessViewState(): 32 lines, 12 if-else branches
├── Low Cohesion: autoScrollTop/Bottom/Stop — không dùng business fields
├── Excess Declarations: _activeScrollTop + _activeScrollBottom (2 bools cho 1 concept)
├── Get.find<MailboxDashBoardController>() trong field declaration
└── Temporal Coupling: _registerObxStreamListener() phải trước onReady()
```

---

## Giải pháp 1 — Fix Feature Envy via Domain Contracts

**Thay vì depend trực tiếp vào MailboxDashBoardController:**
```dart
// Trước
final mailboxDashBoardController = Get.find<MailboxDashBoardController>();
// ... 75 chỗ access: mailboxDashBoardController.selectedMailbox.value
```

**Dùng fine-grained contracts:**
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
// MailboxController nhận contracts qua constructor
class MailboxController extends BaseController {
  final IMailboxNavigationContract _navigation;
  final IEmailListStateContract _emailListState;
  final AppEventBus _eventBus;

  // Business interactors
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

## Giải pháp 2 — Fix `_registerObxStreamListener()` 159 lines

**Trước — 1 method, 2 nested `ever()` với 15+ action types:**
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

**Sau — AppEventBus + tách nhỏ:**
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

// Mỗi handler ≤ 15 lines, 1 concern
void _onMailboxSelected(MailboxSelectedEvent event) {
  _currentSelectedMailbox = event.mailbox;
  _updateMailboxTreeSelection(event.mailbox);
}

void _onSessionInitialized(SessionInitializedEvent event) {
  _loadAllMailboxes(event.session, event.accountId);
}
```

---

## Giải pháp 3 — Fix `handleMailboxAction()` 112 lines, 18 cases

**Trước — 1 switch với 18 cases:**
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
    // ... 8 nữa
  }
}
```

**Sau — nhóm theo category + command pattern:**
```dart
// Tách thành 3 nhóm nhỏ
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

## Giải pháp 4 — Fix Low Cohesion: Scroll Methods

```dart
// Trước: scroll methods trong MailboxController
void autoScrollTop() {
  mailboxListScrollController.animateTo(
    mailboxListScrollController.position.minScrollExtent, ...);
}
void autoScrollBottom() { ... }
void stopAutoScroll() { ... }
```

```dart
// Sau: tách ra MailboxScrollController hoặc extension
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

MailboxController inject `MailboxScrollController` qua constructor.

---

## Giải pháp 5 — Fix Excess Data Declarations

```dart
// Trước: 2 RxBool riêng lẻ
final _activeScrollTop = RxBool(false);
final _activeScrollBottom = RxBool(true);

// Sau: 1 enum
enum MailboxScrollEdge { atTop, middle, atBottom }
final scrollEdge = Rx<MailboxScrollEdge>(MailboxScrollEdge.atBottom);

// Trước: unused field
final isMailboxListScrollable = false.obs; // không ai dùng

// Xóa field này.

// Trước: locally-scoped state stored as class fields
MailboxId? _newFolderId;
NavigationRouter? _navigationRouter;

// Sau: chuyển thành local variables trong methods cần dùng
// Chỉ giữ lại nếu thực sự cần persist across method calls
```

---

## handleSuccessViewState — Fix Bumpy Road

```dart
// Sau: chỉ handle mailbox operation success types
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

## MailboxController sau refactor

```dart
class MailboxController extends BaseController {
  // Contracts (không còn MailboxDashBoardController reference)
  final IMailboxNavigationContract _navigation;
  final IEmailListStateContract _emailListState;
  final AppEventBus _eventBus;

  // Business interactors — chỉ mailbox-related
  final GetAllMailboxInteractor _getAllMailbox;
  final CreateNewMailboxInteractor _createMailbox;
  final DeleteMultipleMailboxInteractor _deleteMailbox;
  final RenameMailboxInteractor _renameMailbox;
  final MoveMailboxInteractor _moveMailbox;
  final SubscribeMailboxInteractor _subscribeMailbox;

  // Scroll helper (không còn scroll methods inline)
  final MailboxScrollController _scrollController;

  // Mailbox tree builder
  final MailboxTreeBuilder _treeBuilder;

  // State — chỉ mailbox-specific
  final mailboxTree = Rxn<MailboxNode>();
  final expandMode = Rx<ExpandMode>(ExpandMode.EXPAND);
  final scrollEdge = Rx<MailboxScrollEdge>(MailboxScrollEdge.atBottom); // fix primitive obsession
}
```

**Ước tính ~600 lines** (từ 1,602 lines) sau khi:
- Xóa 75+ dashboard references
- Tách scroll logic
- Fix `_registerObxStreamListener()` 159 → ~40 lines
- Fix `handleMailboxAction()` 112 → ~50 lines

---

## Metrics

| | Trước | Sau |
|---|---|---|
| Lines | 1,602 | ~600 |
| Dashboard references | 75+ | 0 (contracts only) |
| `_registerObxStreamListener()` | 159 lines | ~40 lines |
| `handleMailboxAction()` | 112 lines, 18 cases | 3 methods ~30 lines each |
| Low cohesion methods | 3 | 0 (moved to MailboxScrollController) |
| `Get.find<T>()` in fields | 1 | 0 |
