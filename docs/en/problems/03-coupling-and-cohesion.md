# Problems: Coupling & Cohesion

> Issue group: **#4 Bumpy Road · #5 Deep Nested Complexity · #6 Low Cohesion · #8 Feature Envy**

---

## 4. Bumpy Road

> **Definition:** A method with 5+ sequential branches (if-else/switch), each handling a completely different concern — a "bumpy" road through many unrelated pieces of logic.

### MailboxDashboardController

| Method | Branch count | Lines |
|---|---|---|
| `handleSuccessViewState()` | 31 | 453–566 |
| `handleFailureViewState()` | 22 | 568–619 |
| `onDeleteSearchFilterAction()` | 6+ | 2384–2413 |
| `_navigateToScreen()` | 8+ | 3113–3172 |

### ThreadController

```dart
// pressEmailSelectionAction() — lines 1271–1330: 15 branches
switch(actionType) {
  case EmailActionType.markAsRead:       // concern: email state
  case EmailActionType.markAsStarred:    // concern: email state
  case EmailActionType.moveToMailbox:    // concern: navigation
  case EmailActionType.moveToTrash:      // concern: navigation
  case EmailActionType.deletePermanently: // concern: deletion
  case EmailActionType.moveToSpam:       // concern: spam
  case EmailActionType.archiveMessage:   // concern: archive
  case EmailActionType.compose:          // concern: composer — completely unrelated
  // ... 7 more cases
}
```

### SingleEmailController

```dart
// handleEmailAction() — lines 874–962: 20 branches
switch(actionType) {
  case EmailActionType.markAsUnread:         // state
  case EmailActionType.markAsStarred:        // state
  case EmailActionType.moveToMailbox:        // navigation
  case EmailActionType.deletePermanently:    // deletion
  case EmailActionType.createRule:           // rule filter feature
  case EmailActionType.unsubscribe:          // unsubscribe feature
  case EmailActionType.printAll:             // print feature
  case EmailActionType.downloadMessageAsEML: // download feature
  case EmailActionType.reply:                // composer
  case EmailActionType.replyAll:             // composer
  case EmailActionType.forward:              // composer
  case EmailActionType.labelAs:              // label feature
  // ... 8 more cases — each case is an independent feature
}
```

### SearchEmailController

```dart
// onDeleteSearchFilterAction() — 10 branches for 10 different filter types
switch(filterType) {
  case SearchFilterType.from: ...
  case SearchFilterType.to: ...
  case SearchFilterType.hasKeyword: ...
  case SearchFilterType.notKeyword: ...
  case SearchFilterType.mailbox: ...
  case SearchFilterType.date: ...
  // ...
}
```

### Bumpy Road Diagram — `handleSuccessViewState` Dashboard (31 branches)

```
 handleSuccessViewState()  ── 113 lines · 31 branches ─────────────────
 │
 ├──▶ GetAllMailboxSuccess         📁 Mailbox domain
 ├──▶ MoveToMailboxSuccess         📧 Email action domain
 ├──▶ DeleteEmailPermanentlySuccess 📧 Email action domain
 ├──▶ MarkAsEmailReadSuccess       📧 Email action domain
 ├──▶ SendEmailSuccess             📤 Sending queue domain
 ├──▶ StoreSendingEmailSuccess     📤 Sending queue domain
 ├──▶ GetAllIdentitiesSuccess      🔐 Session domain
 ├──▶ EmptyTrashFolderSuccess      🗑 Mailbox domain
 ├──▶ UnsubscribeEmailSuccess      🚫 Unsubscribe domain
 ├──▶ RestoreDeletedSuccess        🔄 Recovery domain
 └──▶ ... 21 more branches (9 different bounded contexts)

 ⚠ Problem: Every time a new feature is added, this method must be modified
            → violates Open/Closed Principle
```

---

## 5. Deep Nested Complexity

> **Definition:** Code with 4+ consecutive nesting levels — hard to trace flow, hard to test.

### MailboxDashboardController — `dragSelectedMultipleEmailToMailboxAction()` (lines 1433–1471)

```dart
if (searchController.isSearchEmailRunning ||
    selectedMailbox.value?.isVirtualFolder == true) {   // level 1
  final Map<...> mapListEmail = {};
  for (var element in listEmails) {                      // level 2
    final mailbox = element.findMailboxContain(...);
    if (mailbox != null && element.id != null) {         // level 3
      if (mapListEmail.containsKey(mailbox.id)) {        // level 4
        mapListEmail[mailbox.id]?.add(element.id!);
      } else {
        mapListEmail.addAll({mailbox.id: [element.id!]});
      }
    }
  }
}
```

### ThreadController — `_registerObxStreamListener()` (lines 292–352)

```dart
ever(mailboxDashBoardController.dashBoardAction, (action) {  // level 1
  if (action is SelectEmailAction) {                          // level 2
    if (action.email != null) {                              // level 3
      _handleSelectedEmail(action.email!);
    }
  } else if (action is RefreshEmailAction) {                 // level 2
    if (mailboxDashBoardController.isLoading) {              // level 3
      if (canRefresh) {                                      // level 4
        _refreshEmails();
      }
    }
  }
  // ... 9 more action types, each with its own nesting
});
```

### SingleEmailController — `_registerObxStreamListener()` (lines 284–391)

```dart
obxListeners.add(ever(mailboxDashBoardController.emailUIAction, (action) {  // level 1
  if (action is HideEmailContentViewAction) {                 // level 2
    isEmailContentHidden.value = true;
  } else if (action is PerformEmailActionInThreadDetailAction) { // level 2
    if (action.presentationEmail.id != _currentEmailId) return;  // level 3
    pressEmailAction(action.emailActionType, action.presentationEmail);
  } else if (action is DisposePreviousExpandedEmailAction) {  // level 2
    if (_currentEmailId == null ||
        _currentEmailId != action.emailId) return;           // level 3
    for (var worker in obxListeners) {                        // level 4
      worker.dispose();
    }
    Get.delete<SingleEmailController>(tag: action.emailId.id.value); // level 4
  }
}));
```

---

## 6. Low Cohesion

> **Definition:** Methods in a class do not use that class's own fields — they do not "belong" to this class.

### MailboxDashboardController — Wrapper methods with no state logic

```dart
// Lines 2005-2006: just sets 1 field — doesn't need to be a method
void clearFilterMessageOption() {
  filterMessageOption.value = FilterMessageOption.all;
}

// Lines 2437-2438: pure wrapper for a single dispatch
void emptyTrashAction() {
  dispatchAction(EmptyTrashAction());
}

// Lines 2782-2785: doesn't use any field of the class
void openDefaultMailbox() {
  dispatchRoute(DashboardRoutes.thread);
  dispatchMailboxUIAction(SelectMailboxDefaultAction());
}
```

### ThreadController

```dart
// Lines 979-983: pure utility, zero class state usage
SelectMode getSelectMode(PresentationEmail email, PresentationEmail? selected) {
  return email.id == selected?.id ? SelectMode.ACTIVE : SelectMode.INACTIVE;
}

// Lines 1573: pure utility
bool isInArchiveMailbox(PresentationEmail email) =>
    email.mailboxContain?.isArchive == true;
```

### MailboxController

```dart
// Lines 1317-1336: scroll utilities — don't use any business fields
void autoScrollTop() {
  mailboxListScrollController.animateTo(
    mailboxListScrollController.position.minScrollExtent, ...);
}
void autoScrollBottom() {
  mailboxListScrollController.animateTo(
    mailboxListScrollController.position.maxScrollExtent, ...);
}
void stopAutoScroll() {
  mailboxListScrollController.animateTo(
    mailboxListScrollController.offset, ...);
}
```

---

## 8. Feature Envy

> **Definition:** Class A frequently accesses the data and methods of class B more than its own — class A "envies" class B.

### MailboxController — 75+ references to `mailboxDashBoardController`

```dart
// Field declaration (line 114) — service locator
final mailboxDashBoardController = Get.find<MailboxDashBoardController>();

// In _registerObxStreamListener() — subscribes to 3 observables on dashboard
ever(mailboxDashBoardController.accountId, ...)
ever(mailboxDashBoardController.routerParameters, ...)
ever(mailboxDashBoardController.mailboxUIAction, ...)
ever(mailboxDashBoardController.viewState, ...)  // 110+ lines if-else

// In business methods — reads/writes dashboard state
mailboxDashBoardController.setMapDefaultMailboxIdByRole(...)  // write
mailboxDashBoardController.setMapMailboxById(...)             // write
mailboxDashBoardController.outboxMailbox = ...               // write
mailboxDashBoardController.searchController.isSearchEmailRunning  // read nested
mailboxDashBoardController.dashboardRoute                    // read
mailboxDashBoardController.selectedMailbox.value             // read
```

### Feature Envy Diagram — Coupling Web

```
                  ┌───────────────────────────────────┐
                  │     MailboxDashBoardController     │
                  │   🔴 God Object · 3,508 lines      │
                  └──────────────┬────────────────────┘
       ┌─────────────────────────┼──────────────────────────┐
       │                         │                          │
 Get.find() 75+ refs       Get.find() 20+ refs       Get.find() ×5
 ever(mailboxUIAction)     delegate email actions     reads shared state
 ever(viewState) 110 lines
       │                         │                          │
       ▼                         ▼                          ▼
 MailboxController       SingleEmailController      ThreadDetailController
  (1,602 lines)            (1,630 lines)               (312 lines)

       │                         │
 ever(dashBoardAction)    mutates dashboard.emailList!
 Heavy Feature Envy        20+ method delegation
       │                         │
       ▼                         ▼
 ThreadController         SearchEmailController
  (1,625 lines)             (1,198 lines)
```

### ThreadController — Heavy Feature Envy

```dart
// _getAllEmailSuccess() calls methods on the dashboard controller
mailboxDashBoardController.updateRefreshAllEmailState()
mailboxDashBoardController.updateEmailList(newListEmail)  // mutates another controller's state!
mailboxDashBoardController.listEmailSelected
mailboxDashBoardController.isSelectionEnabled()

// _loadMoreEmails() reads state from dashboard
mailboxDashBoardController.emailsInCurrentMailbox
mailboxDashBoardController.filterMessageOption.value
mailboxDashBoardController.selectedMailbox.value
```

### SingleEmailController — Delegation chain

```dart
// 20+ references to mailboxDashBoardController
mailboxDashBoardController.openComposer(...)    // open composer
mailboxDashBoardController.moveToMailbox(...)   // delegate email action
mailboxDashBoardController.downloadAttachment(...)
mailboxDashBoardController.listIdentities       // read identity state

// Also depends on ThreadDetailController
_threadDetailController?.closeThreadDetailAction()
_threadDetailController?.emailIdsPresentation[emailId]
_threadDetailController?.cacheEmailLoaded()
```
