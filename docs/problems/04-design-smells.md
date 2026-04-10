# Problems: Thiết kế nhỏ lẻ (Design Smells)

> Nhóm vấn đề: **#7 Excess Data Declarations · #9 Code Health Degradations · #10 Excess Function Arguments · #11 Primitive Obsession · #12 Temporal Coupling · #13 Mixed Abstraction Levels**

---

## 7. Excess Data Declarations

> **Định nghĩa:** Quá nhiều fields trong class, nhiều fields không cần thiết, hoặc Rx wrapping thừa.

### MailboxDashboardController — 42+ reactive fields

**13+ RxBool fields không được nhóm lại (lines 319–334):**
```dart
final _isDraggingMailbox = RxBool(false);
final searchMailboxActivated = RxBool(false);
final isRecoveringDeletedMessage = RxBool(false);
final isAppGridDialogDisplayed = RxBool(false);
final isAllEmailSelected = RxBool(false);
final isComposerShowing = RxBool(false);
final isEmailDetailed = RxBool(false);
// ... 6+ RxBool nữa
```

**Map fields mutable, thay đổi liên tục (lines 337–338):**
```dart
Map<Role, MailboxId> mapDefaultMailboxIdByRole = {};
Map<MailboxId, PresentationMailbox> mapMailboxById = {};
```
Những map này được mutate trực tiếp thay vì bất biến — gây khó debug state.

**Null-force pattern lặp lại 50+ lần:**
```dart
// Lặp lại ở lines 440, 441, 449-450, 522, 966, 1212, 1240, 1253...
if (accountId.value != null && sessionCurrent != null) {
  consumeState(_someInteractor.execute(sessionCurrent!, accountId.value!, ...));
  //                                               ↑ force unwrap sau khi null check
}
```

### MailboxController — Fields dùng 1 lần hoặc locally-scoped

```dart
// Lines 129-130: 2 booleans cho 1 concept (scroll state)
final _activeScrollTop = RxBool(false);
final _activeScrollBottom = RxBool(true);

// Line 115: unused reactive field
final isMailboxListScrollable = false.obs;

// Lines 133-135: locally-scoped state stored as class fields
MailboxId? _newFolderId;         // chỉ dùng trong vài methods tạo folder
NavigationRouter? _navigationRouter; // chỉ dùng trong _handleDataFromNavigationRouter
WebSocketQueueHandler? _webSocketQueueHandler;
```

### SingleEmailController — 12 Rx fields, nhiều fields single-use

```dart
// Lines 134-145: 12 Rx fields trong 1 controller
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

// Lines 148-150: fields single-use
Identity? _identitySelected;        // set once
ButtonState? _printEmailButtonState; // chỉ dùng trong print-related methods
GlobalKey? attachmentListKey;        // init once, web-only
```

### SearchEmailController

```dart
// Lines 131-132: StreamController + Subscription là 2 fields riêng
// nên được gói trong 1 abstraction
StreamController<MailListShortcutActionViewEvent>? shortcutActionEventController;
StreamSubscription<MailListShortcutActionViewEvent>? shortcutActionEventSubscription;
```

---

## 9. Code Health Degradations

> **Định nghĩa:** Patterns cho thấy code đã bị "vá" nhiều lần, không còn clean architecture.

### Optional Late Dependencies — SingleEmailController

```dart
// 6 optional interactors inject lazily, không rõ khi nào được khởi tạo
late CreateNewEmailRuleFilterInteractor? _createNewEmailRuleFilterInteractor;
late SendReceiptToSenderInteractor? _sendReceiptToSenderInteractor;
late ParseCalendarEventInteractor? _parseCalendarEventInteractor;
late AcceptCalendarEventInteractor? _acceptCalendarEventInteractor;
late MaybeCalendarEventInteractor? _maybeCalendarEventInteractor;
late RejectCalendarEventInteractor? _rejectCalendarEventInteractor;
```

Cùng với check ở runtime:
```dart
if (_parseCalendarEventInteractor != null) {
  consumeState(_parseCalendarEventInteractor!.execute(...));
}
```
Đây là dấu hiệu tính năng được "nhét vào" sau — không được thiết kế từ đầu.

### Dynamic Binding Injection — SingleEmailController (lines 1570–1622)

```dart
// onViewEntireMessage() load interactor dynamically — patch rõ ràng
void onViewEntireMessage() {
  final interactor = getBinding<GetEntireMessageAsDocumentInteractor>();
  if (interactor == null) return;
  // ...
}
```
Interactor không được inject vào constructor mà được lấy từ binding tại runtime.

### Force Unwrap Pattern — MailboxDashboardController (50+ lần)

```dart
// Lặp lại ở hơn 50 vị trí trong file
if (accountId.value != null && sessionCurrent != null) {
  consumeState(_interactor.execute(sessionCurrent!, accountId.value!, ...));
  //                                            ↑ unsafe — có thể crash nếu logic thay đổi
}
```

### Mixed Concerns trong 1 Method — MailboxDashboardController

```dart
// _handleMessageFromNotification() — lines 2462-2484
// Concern 1: logging
logInfo('_handleMessageFromNotification: ...');
// Concern 2: JSON parsing (low-level)
final email = jsonDecode(message.data['email']!)['id'];
// Concern 3: business logic
if (email != null && accountId.value != null) {
// Concern 4: routing/navigation
  dispatchRoute(DashboardRoutes.emailDetailed);
}
```

---

## 10. Excess Number of Function Arguments

> **Định nghĩa:** Method (không phải constructor) có 5+ parameters.

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
  Map<EmailId, bool> emailIdsWithReadStatus,  // 4 params, nhưng MoveToMailboxRequest đã chứa nhiều thứ
)
```

---

## 11. Primitive Obsession

> **Định nghĩa:** Dùng primitive types (bool, String, int) thay vì domain objects.

### SearchEmailController

```dart
// Lines 127-128: 2 fields cho cùng 1 concept "có thể tải thêm không"
late SearchMoreState searchMoreState;  // enum — đúng
late bool canSearchMore;               // redundant boolean — nên xóa

// Nên gộp thành:
// canSearchMore = searchMoreState == SearchMoreState.idle
```

### MailboxController

```dart
// Lines 129-130: 2 RxBool riêng lẻ cho scroll state
final _activeScrollTop = RxBool(false);
final _activeScrollBottom = RxBool(true);

// Nên là:
enum MailboxScrollEdge { top, middle, bottom }
final scrollEdge = Rx<MailboxScrollEdge>(MailboxScrollEdge.bottom);
```

---

## 12. Temporal Coupling

> **Định nghĩa:** Methods phải được gọi theo thứ tự nhất định nhưng không có cơ chế enforce.

### ThreadController

`_registerObxStreamListener()` phải được gọi trước `onReady()`. Nếu thứ tự lifecycle bị thay đổi, observers sẽ không được set up và `onReady()` sẽ silently fail.

```dart
@override
void onInit() {
  _registerObxStreamListener(); // PHẢI trước onReady
  super.onInit();
}

@override
void onReady() {
  super.onReady();
  getAllEmailAction(); // assumes observers đã được set
}
```

### SingleEmailController — Calendar interactors

`_injectAndGetInteractorBindings()` được gọi trong `_registerObxStreamListener()` khi `accountId != null`, nhưng code sau đó tại lines 1221–1226 check `_parseCalendarEventInteractor != null` mà không có gì đảm bảo accountId đã non-null khi method đó được gọi.

---

## 13. Mixed Abstraction Levels

> **Định nghĩa:** Một method vừa chứa high-level orchestration vừa chứa low-level implementation detail.

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

Quyết định "dùng interactor nào" dựa trên low-level data structure của `moveRequest` — logic phân nhánh này lẽ ra phải nằm trong một factory hoặc trong chính `MoveToMailboxRequest`.

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

6 concerns khác nhau về mặt abstraction được nhét vào 1 method.
