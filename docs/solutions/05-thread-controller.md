# Solution: ThreadController

**Vấn đề:** Feature Envy nặng, Deep Nesting 4 levels, Bumpy Road 15 cases, Duplicate fields, Low Cohesion methods.

---

## Vấn đề cụ thể

```
ThreadController (1,625 lines)
├── _registerObxStreamListener(): 85 lines, 13 action types, 3-4 nesting levels
├── pressEmailSelectionAction(): 15 switch cases — unrelated concerns
├── handleEmailActionType(): 18 switch cases
├── Feature Envy: 10+ accesses mutate MailboxDashBoardController state
├── getSelectMode() — pure utility, zero class field usage
├── isInArchiveMailbox() — pure utility, zero class field usage
├── Duplicate fields: canLoadMore + loadingMoreStatus (same concept)
├── Temporal Coupling: _registerObxStreamListener() phải trước onReady()
└── _getAllEmailSuccess(): 5 concerns trong 1 method
```

---

## Giải pháp 1 — Fix Feature Envy: Email state không thuộc ThreadController

**Vấn đề:** `_getAllEmailSuccess()` gọi `mailboxDashBoardController.updateEmailList()` — Thread controller đang mutate state của controller khác.

```dart
// Trước — Feature Envy nghiêm trọng
void _getAllEmailSuccess(GetAllEmailSuccess success) {
  // ...
  mailboxDashBoardController.updateEmailList(newListEmail);    // mutate state khác
  mailboxDashBoardController.updateRefreshAllEmailState();     // mutate state khác
  mailboxDashBoardController.isSelectionEnabled();             // read state khác
  mailboxDashBoardController.listEmailSelected;                // read state khác
}
```

**Sau — ThreadController own email list state:**
```dart
// ThreadController có email list state của chính nó
// Qua EmailListStateService inject qua constructor
class ThreadController extends BaseController {
  final EmailListStateService _emailListState; // inject
  final AppEventBus _eventBus;

  void _getAllEmailSuccess(GetAllEmailSuccess success) {
    final synced = success.emailList.syncPresentationEmail(...);
    _emailListState.updateEmails(synced);              // update shared service
    _eventBus.dispatch(EmailListLoadedEvent(synced));  // notify listeners
    _jumpToFirstEmailIfNeeded();
  }
}
```

---

## Giải pháp 2 — Fix `_registerObxStreamListener()` 85 lines

**Trước:**
```dart
void _registerObxStreamListener() {
  ever(mailboxDashBoardController.dashBoardAction, (action) {  // ever()
    if (action is SelectEmailAction) {
      if (action.email != null) { _handleSelectedEmail(action.email!); }
    } else if (action is RefreshEmailAction) {
      if (mailboxDashBoardController.isLoading) {
        if (canRefresh) { _refreshEmails(); }
      }
    }
    // ... 11 types nữa
  });
}
```

**Sau — AppEventBus + tách listener methods:**
```dart
@override
void onInit() {
  super.onInit();
  _subscribeToEmailEvents();
  _subscribeToMailboxEvents();
  _subscribeToSessionEvents();
  _subscribeToSearchEvents();
}

void _subscribeToEmailEvents() {
  _eventBus.stream
      .whereType<EmailOpenedEvent>()
      .listen(_onEmailOpened)
      .addTo(compositeSubscription);

  _eventBus.stream
      .whereType<EmailListRefreshRequestedEvent>()
      .listen((_) => _refreshEmails())
      .addTo(compositeSubscription);

  _eventBus.stream
      .whereType<EmailSelectionChangedEvent>()
      .listen(_onSelectionChanged)
      .addTo(compositeSubscription);
}

void _subscribeToMailboxEvents() {
  _eventBus.stream
      .whereType<MailboxSelectedEvent>()
      .listen(_onMailboxSelected)
      .addTo(compositeSubscription);
}

void _subscribeToSessionEvents() {
  _eventBus.stream
      .whereType<SessionInitializedEvent>()
      .listen(_onSessionInitialized)
      .addTo(compositeSubscription);
}

void _subscribeToSearchEvents() {
  _eventBus.stream
      .whereType<SearchActivatedEvent>()
      .listen(_onSearchActivated)
      .addTo(compositeSubscription);

  _eventBus.stream
      .whereType<SearchClearedEvent>()
      .listen((_) => _onSearchCleared())
      .addTo(compositeSubscription);
}

// Mỗi handler ≤ 15 lines
void _onMailboxSelected(MailboxSelectedEvent event) {
  if (event.mailbox.id == _currentMailboxId) return;
  _currentMailboxId = event.mailbox.id;
  _resetAndLoadEmails();
}

void _onEmailOpened(EmailOpenedEvent event) {
  _selectedEmail = event.email;
}
```

---

## Giải pháp 3 — Fix Bumpy Road: `pressEmailSelectionAction()` 15 cases

**Vấn đề:** 1 switch xử lý 15 unrelated email action types — email state, navigation, deletion, spam, composer.

```dart
// Trước
switch(actionType) {
  case EmailActionType.markAsRead:
  case EmailActionType.markAsStarred:
  case EmailActionType.moveToMailbox:
  case EmailActionType.deletePermanently:
  case EmailActionType.moveToSpam:
  case EmailActionType.archiveMessage:
  case EmailActionType.compose:   // ← này hoàn toàn không liên quan
  // ... 8 nữa
}
```

**Sau — dispatch action qua EventBus, sub-controller xử lý:**
```dart
// ThreadController không xử lý action, chỉ dispatch
void pressEmailSelectionAction(EmailActionType actionType, List<PresentationEmail> emails) {
  _eventBus.dispatch(EmailBulkActionRequestedEvent(
    actionType: actionType,
    emails: emails,
    sourceMailbox: _currentMailboxId,
  ));
}

// EmailActionController xử lý trong handler của nó:
void _onEmailBulkActionRequested(EmailBulkActionRequestedEvent event) {
  switch (event.actionType) {
    case EmailActionType.markAsRead:
    case EmailActionType.markAsUnread:
      _handleMarkRead(event);
    case EmailActionType.markAsStarred:
    case EmailActionType.unMarkAsStarred:
      _handleMarkStar(event);
    case EmailActionType.moveToMailbox:
    case EmailActionType.moveToTrash:
    case EmailActionType.moveToSpam:
    case EmailActionType.archiveMessage:
      _handleMove(event);
    case EmailActionType.deletePermanently:
      _handleDelete(event);
    default:
      break;
  }
}
// composer actions → ComposerStateService.open(...)
```

---

## Giải pháp 4 — Fix `_getAllEmailSuccess()` 5 concerns

```dart
// Trước — 5 concerns
void _getAllEmailSuccess(GetAllEmailSuccess success, {bool shouldJumpToFirstEmail = true}) {
  if (currentMailboxId != null && ...) { return; }          // 1. guard
  final newList = success.emailList.syncPresentationEmail(...); // 2. transform
  logTrace('ThreadController::_getAllEmailSuccess()...');        // 3. logging
  mailboxDashBoardController.updateEmailList(newList);           // 4. state mutation (wrong controller)
  if (shouldJumpToFirstEmail && listEmailController.hasClients) {
    listEmailController.jumpTo(0);                               // 5. UI command
  }
}
```

```dart
// Sau — tách concerns rõ ràng
void _getAllEmailSuccess(GetAllEmailSuccess success, {bool shouldJumpToFirstEmail = true}) {
  if (_shouldIgnoreResult()) return;  // 1. guard — extracted to named method

  final emails = _transformEmailList(success.emailList);  // 2. transform
  _updateEmailState(emails);                               // 3. state update
  if (shouldJumpToFirstEmail) _scrollToFirstEmail();      // 4. UI command
  // logging tự động qua BaseController stream hooks
}

bool _shouldIgnoreResult() {
  return _currentMailboxId != null &&
      (_isVirtualFolder || _currentMailboxId != _selectedMailboxId);
}

List<PresentationEmail> _transformEmailList(List<Email> raw) {
  return raw.syncPresentationEmail(
    mapMailboxById: _emailListState.mailboxById,
    selectedMailbox: _emailListState.selectedMailbox.value,
  );
}

void _updateEmailState(List<PresentationEmail> emails) {
  _emailListState.updateEmails(emails);
  _emailListState.isLoading.value = false;
}

void _scrollToFirstEmail() {
  if (listEmailController.hasClients) {
    listEmailController.jumpTo(0);
  }
}
```

---

## Giải pháp 5 — Fix Low Cohesion Methods

```dart
// Trước: trong ThreadController
SelectMode getSelectMode(PresentationEmail email, PresentationEmail? selected) {
  return email.id == selected?.id ? SelectMode.ACTIVE : SelectMode.INACTIVE;
}

bool isInArchiveMailbox(PresentationEmail email) =>
    email.mailboxContain?.isArchive == true;
```

```dart
// Sau: extension methods trên domain model
// lib/features/thread/domain/extension/presentation_email_extension.dart
extension PresentationEmailSelectionExt on PresentationEmail {
  SelectMode selectionMode(PresentationEmail? selected) =>
      id == selected?.id ? SelectMode.ACTIVE : SelectMode.INACTIVE;

  bool get isInArchiveMailbox => mailboxContain?.isArchive == true;
}

// Dùng trong View (không qua controller):
Obx(() => EmailTile(
  selectMode: email.selectionMode(controller.selectedEmail),
))
```

---

## Giải pháp 6 — Fix Duplicate Fields

```dart
// Trước: 2 fields cho cùng 1 concept
bool canLoadMore = true;           // line 99
bool canSearchMore = true;         // line 100
Rx<LoadingMoreStatus> loadingMoreStatus = Rx(LoadingMoreStatus.idle); // line 97

// Sau: 1 state object
enum EmailListLoadState {
  idle,
  loading,
  loadingMore,
  canLoadMore,
  allLoaded,
}

final loadState = Rx<EmailListLoadState>(EmailListLoadState.idle);

// Computed getters thay cho duplicate booleans:
bool get canLoadMore => loadState.value == EmailListLoadState.canLoadMore;
bool get canSearchMore => loadState.value == EmailListLoadState.canLoadMore;
```

---

## Fix Temporal Coupling

```dart
// Trước: implicit ordering
@override
void onInit() {
  _registerObxStreamListener(); // PHẢI gọi trước
  super.onInit();
}

// Sau: không còn temporal coupling vì AppEventBus là singleton permanent
// Bất kỳ lúc nào subscribe cũng được — event dispatch sau khi subscribe mới catch được
// Nhưng subscribe sớm trong onInit() là pattern chuẩn
@override
void onInit() {
  super.onInit();
  // Thứ tự không còn quan trọng vì AppEventBus không dispatch gì cho đến khi
  // controller gọi tác vụ đầu tiên trong onReady()
  _subscribeToEmailEvents();
  _subscribeToMailboxEvents();
  _subscribeToSessionEvents();
}

@override
void onReady() {
  super.onReady();
  // Safe: subscriptions đã ready
  _loadInitialEmails();
}
```

---

## ThreadController sau refactor

```dart
class ThreadController extends BaseController {
  // Dependencies — constructor injection
  final EmailListStateService _emailListState;
  final AppEventBus _eventBus;
  final GetEmailsInMailboxInteractor _getEmails;
  final RefreshChangesEmailsInMailboxInteractor _refreshEmails;
  final LoadMoreEmailsInMailboxInteractor _loadMore;
  final CleanAndGetEmailsInMailboxInteractor _cleanAndGet;

  ThreadController(
    this._emailListState,
    this._eventBus,
    this._getEmails,
    this._refreshEmails,
    this._loadMore,
    this._cleanAndGet,
    ISessionTokenRefresher tokenRefresher,
    ILanguageSettingService languageService,
    IAuthService authService,
  ) : super(tokenRefresher, languageService, authService, ...);

  // State — chỉ thread-specific
  final loadState = Rx<EmailListLoadState>(EmailListLoadState.idle);
  final Rxn<PresentationEmail> _selectedEmail = Rxn();
  MailboxId? _currentMailboxId;

  // Không còn: canLoadMore, canSearchMore, loadingMoreStatus (merged vào loadState)
  // Không còn: getSelectMode(), isInArchiveMailbox() (moved to extensions)
  // Không còn: pressEmailSelectionAction() large switch (dispatch via EventBus)
}
```

---

## Metrics

| | Trước | Sau |
|---|---|---|
| Lines | 1,625 | ~500 |
| `_registerObxStreamListener()` | 85 lines | ~40 lines (split into 4 methods) |
| `pressEmailSelectionAction()` | 15 switch cases | 1 line (dispatch to EventBus) |
| Feature envy (dashboard accesses) | 10+ mutations | 0 |
| Duplicate fields | 2 (canLoadMore/canSearchMore) | 0 (merged vào loadState) |
| Low cohesion methods | 2 | 0 (moved to extension) |
