# Solution: SingleEmailController

**Problem:** Optional late dependencies (Code Health Degradation), 20-case switch (Bumpy Road), `_registerObxStreamListener()` 108 lines (Deep Nesting), Feature Envy (20+ dashboard refs), 12 ungrouped Rx fields.

---

## Specific Problems

```
SingleEmailController (1,630 lines)
├── 6 optional late interactors — Code Health Degradation, unclear init
├── _registerObxStreamListener(): 108 lines, 4 nesting levels
├── handleEmailAction(): 20 switch cases — unrelated features
├── Feature Envy: 20+ refs to mailboxDashBoardController
├── 12 Rx fields not grouped by concern
├── _handleOpenEmailDetailedView(): 6 concerns in 1 method
├── Low cohesion: openNewTabAction(), _isBelongToTeamMailboxes()
├── Temporal Coupling: calendar interactors depend on accountId availability
└── getBinding<>() in business method — dynamic injection patch
```

---

## Solution 1 — Fix Optional Late Dependencies: Extract CalendarEmailController

**Root cause:** Calendar event handling was "stuffed into" `SingleEmailController` after the feature was added, resulting in 6 optional interactors.

```dart
// Before: 6 optional interactors in SingleEmailController
late ParseCalendarEventInteractor? _parseCalendarEventInteractor;
late AcceptCalendarEventInteractor? _acceptCalendarEventInteractor;
late MaybeCalendarEventInteractor? _maybeCalendarEventInteractor;
late RejectCalendarEventInteractor? _rejectCalendarEventInteractor;
late SendReceiptToSenderInteractor? _sendReceiptToSenderInteractor;
late CreateNewEmailRuleFilterInteractor? _createNewEmailRuleFilterInteractor;
```

**After — Extract into 2 dedicated controllers:**

```dart
// lib/features/email/presentation/controller/calendar_email_controller.dart
class CalendarEmailController extends GetxController {
  final ParseCalendarEventInteractor _parse;
  final AcceptCalendarEventInteractor _accept;
  final MaybeCalendarEventInteractor _maybe;
  final RejectCalendarEventInteractor _reject;
  final AppEventBus _eventBus;

  final Rxn<BlobCalendarEvent> blobCalendarEvent = Rxn();
  final Rxn<AttendanceStatus> attendanceStatus = Rxn();

  CalendarEmailController(
    this._parse, this._accept, this._maybe, this._reject, this._eventBus,
  );

  void parseCalendarEvent(Session session, AccountId accountId, Email email) {
    consumeState(_parse.execute(session, accountId, email));
  }

  void replyCalendarEvent(
    Session session,
    AccountId accountId,
    EventActionType actionType,
    CalendarEventId eventId,
  ) {
    final stream = switch (actionType) {
      EventActionType.yes => _accept.execute(session, accountId, eventId),
      EventActionType.maybe => _maybe.execute(session, accountId, eventId),
      EventActionType.no => _reject.execute(session, accountId, eventId),
      _ => null,
    };
    if (stream != null) consumeState(stream);
  }

  // Fix Bumpy Road: only 3 cases instead of being stuffed into a 20-case switch
  @override
  void handleSuccessViewState(Success success) {
    switch (success) {
      case ParseCalendarEventSuccess():
        blobCalendarEvent.value = success.blobCalendarEvent;
      case AcceptCalendarEventSuccess():
      case MaybeCalendarEventSuccess():
      case RejectCalendarEventSuccess():
        attendanceStatus.value = success.attendanceStatus;
        _eventBus.dispatch(CalendarEventRepliedEvent(success.eventId));
      default:
        super.handleSuccessViewState(success);
    }
  }
}
```

```dart
// lib/features/email/presentation/controller/email_rule_controller.dart
class EmailRuleController extends GetxController {
  final CreateNewEmailRuleFilterInteractor _createRule;
  final AppEventBus _eventBus;

  EmailRuleController(this._createRule, this._eventBus);

  void createRuleFromEmail(Session session, AccountId accountId, EmailAddress from) {
    consumeState(_createRule.execute(session, accountId, from));
  }
}
```

---

## Solution 2 — Fix `handleEmailAction()` 20-case Switch

**Before — 1 switch, 20 cases, 20 different concerns:**
```dart
switch(actionType) {
  case EmailActionType.markAsUnread: ...
  case EmailActionType.moveToMailbox: ...
  case EmailActionType.createRule: ...     // rule filter — different feature
  case EmailActionType.unsubscribe: ...    // unsubscribe — different feature
  case EmailActionType.printAll: ...       // print — different feature
  case EmailActionType.downloadMessageAsEML: ... // download — different feature
  case EmailActionType.reply: ...          // composer — different feature
  case EmailActionType.labelAs: ...        // label — different feature
  // ... 12 more
}
```

**After — Route to EventBus, each controller handles its own:**
```dart
// SingleEmailController only handles email-core actions
void handleEmailAction(EmailActionType actionType, PresentationEmail email) {
  _eventBus.dispatch(EmailActionRequestedEvent(
    actionType: actionType,
    email: email,
    sourceContext: EmailActionSourceContext.singleView,
  ));
}

// EmailActionController handles: move, delete, mark, star
// CalendarEmailController handles: calendar event reply
// ComposerStateService handles: reply, forward, compose
// EmailRuleController handles: createRule
// PrintEmailController handles: printAll, downloadEML
```

---

## Solution 3 — Fix `_registerObxStreamListener()` 108 lines

**Before — 1 method, 10 action types, 4 nesting levels:**
```dart
obxListeners.add(ever(mailboxDashBoardController.emailUIAction, (action) {
  if (action is HideEmailContentViewAction) { ... }
  else if (action is PerformEmailActionInThreadDetailAction) {
    if (action.presentationEmail.id != _currentEmailId) return;
    pressEmailAction(...);
  }
  else if (action is DisposePreviousExpandedEmailAction) {
    if (_currentEmailId == null || _currentEmailId != action.emailId) return;
    for (var worker in obxListeners) {       // 4 levels
      worker.dispose();
    }
    Get.delete<SingleEmailController>(tag: action.emailId.id.value);
  }
  // ... 7 more action types
}));
```

**After — AppEventBus + split listeners:**
```dart
@override
void onInit() {
  super.onInit();
  _subscribeToEmailContentEvents();
  _subscribeToEmailVisibilityEvents();
  _subscribeToActionEvents();
}

void _subscribeToEmailContentEvents() {
  _eventBus.stream
      .whereType<EmailNavigationRequestedEvent>()
      .where((e) => e.emailId == _currentEmailId)
      .listen(_onEmailNavigated)
      .addTo(compositeSubscription);
}

void _subscribeToEmailVisibilityEvents() {
  _eventBus.stream
      .whereType<EmailContentHiddenEvent>()
      .listen((_) => isEmailContentHidden.value = true)
      .addTo(compositeSubscription);

  _eventBus.stream
      .whereType<EmailContentShownEvent>()
      .listen((_) => isEmailContentHidden.value = false)
      .addTo(compositeSubscription);
}

void _subscribeToActionEvents() {
  _eventBus.stream
      .whereType<EmailActionRequestedEvent>()
      .where((e) => e.email.id == _currentEmailId)
      .listen(_onEmailActionRequested)
      .addTo(compositeSubscription);
}

// Each handler is concise:
void _onEmailNavigated(EmailNavigationRequestedEvent event) {
  _currentEmailId = event.emailId;
  _loadEmailContent();
}
```

---

## Solution 4 — Fix `_handleOpenEmailDetailedView()` 6 concerns

```dart
// Before — 6 concerns
void _handleOpenEmailDetailedView() {
  emailLoadedViewState.value = Right(GetEmailContentLoading()); // state
  _resetToOriginalValue();        // reset
  _createSingleEmailView(...);    // load content
  if (!currentEmail!.hasRead) {
    markAsEmailRead(...);         // mark read
  }
  if (mailboxDashBoardController.listIdentities.isEmpty) {
    _getAllIdentities();           // fetch identities
  } else {
    _initializeSelectedIdentity(...); // setup identity
  }
}
```

```dart
// After — separated concerns
void _handleOpenEmailDetailedView() {
  _prepareForLoading();
  _loadEmailContent();
  _markAsReadIfNeeded();
  _initializeIdentity();
}

void _prepareForLoading() {
  emailLoadedViewState.value = Right(GetEmailContentLoading());
  _resetToOriginalValue();
}

void _loadEmailContent() {
  final emailId = _currentEmailId;
  if (emailId == null) return;
  consumeState(_getEmailContent.execute(_currentSession!, _currentAccountId!, emailId));
}

void _markAsReadIfNeeded() {
  final email = _currentEmail.value;
  if (email != null && !email.hasRead) {
    _eventBus.dispatch(EmailReadRequestedEvent(email.id!, ReadActions.markAsRead));
  }
}

void _initializeIdentity() {
  // SingleEmailController no longer fetches identities — reads from SessionController
  final identities = _sessionController.listIdentities;
  if (identities.isNotEmpty) {
    _selectedIdentity = identities.first;
  }
}
```

---

## Solution 5 — Fix Low Cohesion Methods

```dart
// openNewTabAction — pure wrapper → move to View/utility
// Before:
void openNewTabAction(String link) {
  AppUtils.launchLink(link);
}

// After: called directly from View
// InkWell(onTap: () => AppUtils.launchLink(link))
// Or:
// onTap: () => context.launchLink(link)  // extension on BuildContext

// _isBelongToTeamMailboxes — pure utility → extension
extension PresentationMailboxExt on PresentationMailbox {
  bool get isTeamMailbox => !isPersonal;
}
```

---

## Solution 6 — Fix 12 Rx Fields: Group by Concern

```dart
// Before: 12 individual Rx fields
final emailContents = RxnString();
final attachments = <Attachment>[].obs;
final isDisplayAllAttachments = RxBool(false);
final emlPreviewer = <Attachment>[].obs;
final blobCalendarEvent = Rxn<BlobCalendarEvent>();    // → CalendarEmailController
final emailLoadedViewState = Rx<Either<...>>(Right(UIState.idle));
final emailUnsubscribe = Rxn<EmailUnsubscribe>();
final attachmentsViewState = RxMap<Id, Either<...>>();
final isEmailContentHidden = RxBool(false);
final currentEmailLoaded = Rxn<EmailLoaded>();
final isEmailContentClipped = RxBool(false);
final attendanceStatus = Rxn<AttendanceStatus>();      // → CalendarEmailController

// After: once CalendarEmailController is extracted, SingleEmailController retains
final emailContents = RxnString();                     // email display
final isEmailContentHidden = RxBool(false);            // email display
final isEmailContentClipped = RxBool(false);           // email display
final currentEmailLoaded = Rxn<EmailLoaded>();         // email loading
final emailLoadedViewState = Rx<Either<...>>(Right(UIState.idle)); // email loading
final attachments = <Attachment>[].obs;                // attachments
final emlPreviewer = <Attachment>[].obs;               // attachments
final attachmentsViewState = RxMap<Id, Either<...>>(); // attachments
final emailUnsubscribe = Rxn<EmailUnsubscribe>();      // unsubscribe feature
// → 9 fields instead of 12, and could further extract EmailAttachmentController
```

---

## Fix Dynamic Binding Injection (Code Health Degradation)

```dart
// Before: patch pattern — fetch interactor from binding at runtime
void onViewEntireMessage() {
  final interactor = getBinding<GetEntireMessageAsDocumentInteractor>();
  if (interactor == null) return;
  consumeState(interactor.execute(...));
}

// After: inject via constructor — explicit, testable
class SingleEmailController extends BaseController {
  final GetEntireMessageAsDocumentInteractor _getEntireMessage;

  SingleEmailController(
    this._getEntireMessage,
    // ...
  );

  void onViewEntireMessage() {
    withSession((session, accountId) {
      consumeState(_getEntireMessage.execute(session, accountId, _currentEmailId!));
    });
  }
}
```

---

## SingleEmailController after refactor

```dart
class SingleEmailController extends BaseController {
  // Core dependencies — calendar/rule extracted to other controllers
  final GetEmailContentInteractor _getEmailContent;
  final PrintEmailInteractor _printEmail;
  final StoreOpenedEmailInteractor _storeOpenedEmail;
  final GetEntireMessageAsDocumentInteractor _getEntireMessage;

  // Services/contracts (no longer directly references MailboxDashBoardController)
  final CalendarEmailController _calendarController;
  final EmailRuleController _ruleController;
  final ComposerStateService _composerState;
  final AppEventBus _eventBus;

  // State — email display concern only (from 12 → 6 Rx fields)
  final emailContents = RxnString();
  final isEmailContentHidden = RxBool(false);
  final isEmailContentClipped = RxBool(false);
  final currentEmailLoaded = Rxn<EmailLoaded>();
  final emailLoadedViewState = Rx<Either<Failure, Success>>(Right(UIState.idle));
  final emailUnsubscribe = Rxn<EmailUnsubscribe>();
}
```

---

## EmailBindings after refactor

```dart
class EmailBindings extends Bindings {
  @override
  void dependencies() {
    Get.lazyPut(() => CalendarEmailController(
      Get.find(), Get.find(), Get.find(), Get.find(),
      Get.find<AppEventBus>(),
    ));

    Get.lazyPut(() => EmailRuleController(
      Get.find(),
      Get.find<AppEventBus>(),
    ));

    Get.lazyPut(() => SingleEmailController(
      Get.find(), Get.find(), Get.find(), Get.find(),
      Get.find<CalendarEmailController>(),
      Get.find<EmailRuleController>(),
      Get.find<ComposerStateService>(),
      Get.find<AppEventBus>(),
      Get.find<ISessionTokenRefresher>(),
      Get.find<ILanguageSettingService>(),
      Get.find<IAuthService>(),
    ));
  }
}
```

---

## Metrics

| | Before | After |
|---|---|---|
| Lines (SingleEmailController) | 1,630 | ~400 |
| Optional late dependencies | 6 | 0 |
| Rx fields | 12 | 6 |
| `handleEmailAction()` switch cases | 20 | 0 (dispatch via EventBus) |
| `_registerObxStreamListener()` | 108 lines | ~30 lines |
| Feature envy (dashboard refs) | 20+ | 0 |
| Controllers after split | 1 | 3 (SingleEmail + Calendar + Rule) |