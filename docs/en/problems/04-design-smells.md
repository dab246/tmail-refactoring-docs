# Problems: Minor Design Smells

> Issue group: **#7 Excess Data Declarations · #9 Code Health Degradations · #10 Excess Function Arguments · #11 Primitive Obsession · #12 Temporal Coupling · #13 Mixed Abstraction Levels**

---

## 7. Excess Data Declarations

> **Definition:** Too many fields in a class, many fields are unnecessary, or Rx wrapping is excessive.

### MailboxDashboardController — 42+ reactive fields

**13+ RxBool fields not grouped together (lines 319–334):**
```dart
final _isDraggingMailbox = RxBool(false);
final searchMailboxActivated = RxBool(false);
final isRecoveringDeletedMessage = RxBool(false);
final isAppGridDialogDisplayed = RxBool(false);
final isAllEmailSelected = RxBool(false);
final isComposerShowing = RxBool(false);
final isEmailDetailed = RxBool(false);
// ... 6+ more RxBool
```

**Force unwrap pattern repeated 50+ times:**
```dart
if (accountId.value != null && sessionCurrent != null) {
  consumeState(_someInteractor.execute(sessionCurrent!, accountId.value!, ...));
}
```

### MailboxController — Fields used only once or locally-scoped

```dart
// Lines 129-130: 2 booleans for 1 concept (scroll state)
final _activeScrollTop = RxBool(false);
final _activeScrollBottom = RxBool(true);

// Line 115: unused reactive field
final isMailboxListScrollable = false.obs;

// Lines 133-135: locally-scoped state stored as class fields
MailboxId? _newFolderId;
NavigationRouter? _navigationRouter;
WebSocketQueueHandler? _webSocketQueueHandler;
```

### SingleEmailController — 12 Rx fields, many single-use fields

```dart
final emailContents = RxnString();
final attachments = <Attachment>[].obs;
final isDisplayAllAttachments = RxBool(false);
final emlPreviewer = <Attachment>[].obs;
final blobCalendarEvent = Rxn<BlobCalendarEvent>();
final emailLoadedViewState = Rx<Either<Failure, Success>>(Right(UIState.idle));
final emailUnsubscribe = Rxn<EmailUnsubscribe>();
final attachmentsViewState = RxMap<Id, Either<Failure, Success>>();
final isEmailContentHidden = RxBool(false);
final currentEmailLoaded = Rxn<EmailLoaded>();
final isEmailContentClipped = RxBool(false);
final attendanceStatus = Rxn<AttendanceStatus>();
```

---

## 9. Code Health Degradations

> **Definition:** Patterns that show code has been "patched" many times and is no longer clean architecture.

### Optional Late Dependencies — SingleEmailController

```dart
// 6 optional interactors injected lazily — unclear when they are initialized
late CreateNewEmailRuleFilterInteractor? _createNewEmailRuleFilterInteractor;
late SendReceiptToSenderInteractor? _sendReceiptToSenderInteractor;
late ParseCalendarEventInteractor? _parseCalendarEventInteractor;
late AcceptCalendarEventInteractor? _acceptCalendarEventInteractor;
late MaybeCalendarEventInteractor? _maybeCalendarEventInteractor;
late RejectCalendarEventInteractor? _rejectCalendarEventInteractor;
```

Combined with runtime checks:
```dart
if (_parseCalendarEventInteractor != null) {
  consumeState(_parseCalendarEventInteractor!.execute(...));
}
```

### Dynamic Binding Injection — SingleEmailController (lines 1570–1622)

```dart
void onViewEntireMessage() {
  final interactor = getBinding<GetEntireMessageAsDocumentInteractor>();
  if (interactor == null) return;
  // ...
}
```

### Force Unwrap Pattern — MailboxDashboardController (50+ occurrences)

```dart
if (accountId.value != null && sessionCurrent != null) {
  consumeState(_interactor.execute(sessionCurrent!, accountId.value!, ...));
}
```

---

## 10. Excess Number of Function Arguments

> **Definition:** A method (not a constructor) has 5+ parameters.

### MailboxController — `_handleMovingMailbox()` (lines 1096–1113)

```dart
void _handleMovingMailbox(
  BuildContext context,
  Session session,
  AccountId accountId,
  MoveAction moveAction,
  PresentationMailbox mailboxSelected,
  {PresentationMailbox? destinationMailbox}   // 6 params total
)
```

### MailboxDashboardController — `moveToMailbox()` (lines 1125–1147)

```dart
void moveToMailbox(
  Session session,
  AccountId accountId,
  MoveToMailboxRequest moveRequest,
  Map<EmailId, bool> emailIdsWithReadStatus,
)
```

---

## 11. Primitive Obsession

> **Definition:** Using primitive types (bool, String, int) instead of domain objects.

### SearchEmailController

```dart
// 2 fields for the same concept "can load more or not"
late SearchMoreState searchMoreState;  // enum — correct
late bool canSearchMore;               // redundant boolean — should be removed

// Should be consolidated as:
// canSearchMore = searchMoreState == SearchMoreState.idle
```

### MailboxController

```dart
// 2 separate RxBools for scroll state
final _activeScrollTop = RxBool(false);
final _activeScrollBottom = RxBool(true);

// Should be:
enum MailboxScrollEdge { top, middle, bottom }
final scrollEdge = Rx<MailboxScrollEdge>(MailboxScrollEdge.bottom);
```

---

## 12. Temporal Coupling

> **Definition:** Methods must be called in a specific order but there is no mechanism to enforce this.

### ThreadController

`_registerObxStreamListener()` must be called before `onReady()`. If the lifecycle order is changed, observers will not be set up and `onReady()` will silently fail.

```dart
@override
void onInit() {
  _registerObxStreamListener(); // MUST come before onReady
  super.onInit();
}

@override
void onReady() {
  super.onReady();
  getAllEmailAction(); // assumes observers are already set up
}
```

---

## 13. Mixed Abstraction Levels

> **Definition:** A single method contains both high-level orchestration and low-level implementation detail.

### MailboxDashboardController — `moveToMailbox()` (lines 1125–1147)

```dart
void moveToMailbox(...) {
  final currentMailboxes = moveRequest.currentMailboxes;  // low-level extraction
  if (currentMailboxes.length == 1 &&                     // low-level counting
      currentMailboxes.values.first.length == 1) {
    consumeState(_moveToMailboxInteractor.execute(...));   // high-level
  } else {
    consumeState(_moveMultipleEmailToMailboxInteractor.execute(...)); // high-level
  }
}
```

### SingleEmailController — `_handleOpenEmailDetailedView()` (lines 393–415)

```dart
void _handleOpenEmailDetailedView() {
  emailLoadedViewState.value = Right(GetEmailContentLoading()); // state update
  _resetToOriginalValue();                    // state reset
  _createSingleEmailView(currentEmail!.id!); // delegate to interactor
  if (!currentEmail!.hasRead) {
    markAsEmailRead(...);                     // side effect: mark as read
  }
  if (mailboxDashBoardController.listIdentities.isEmpty) {
    _getAllIdentities();                       // async fetch — completely different concern
  } else {
    _initializeSelectedIdentity(...);         // setup identity
  }
}
```

6 concerns of different abstraction levels crammed into 1 method.
