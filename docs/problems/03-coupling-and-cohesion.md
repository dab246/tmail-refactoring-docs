# Problems: Liên kết & Gắn kết

> Nhóm vấn đề: **#4 Bumpy Road · #5 Deep Nested Complexity · #6 Low Cohesion · #8 Feature Envy**

---

## 4. Bumpy Road

> **Định nghĩa:** Method có 5+ sequential branches (if-else/switch), mỗi branch xử lý concern hoàn toàn khác nhau — đường đi "gồ ghề" qua nhiều logic không liên quan.

### MailboxDashboardController

| Method | Số branches | Lines |
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
  case EmailActionType.compose:          // concern: composer — hoàn toàn không liên quan
  // ... 7 cases nữa
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
  // ... 8 cases nữa — mỗi case là 1 feature độc lập
}
```

### SearchEmailController

```dart
// onDeleteSearchFilterAction() — 10 branches cho 10 filter types khác nhau
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

### Sơ đồ Bumpy Road — `handleSuccessViewState` Dashboard (31 branches)

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
 └──▶ ... 21 branches nữa (9 bounded contexts khác nhau)

 ⚠ Vấn đề: Mỗi khi thêm tính năng mới phải sửa method này
            → vi phạm Open/Closed Principle
```

---

## 5. Deep Nested Complexity

> **Định nghĩa:** Code có 4+ nesting levels liên tiếp — khó trace flow, khó test.

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
  // ... 9 action types nữa, mỗi loại có nesting riêng
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

> **Định nghĩa:** Methods trong một class không sử dụng fields của class đó — chúng không "thuộc về" class này.

### MailboxDashboardController — Wrapper methods không có state logic

```dart
// Lines 2005-2006: chỉ set 1 field — không cần là method
void clearFilterMessageOption() {
  filterMessageOption.value = FilterMessageOption.all;
}

// Lines 2437-2438: wrapper thuần cho 1 dispatch
void emptyTrashAction() {
  dispatchAction(EmptyTrashAction());
}

// Lines 2782-2785: không dùng bất kỳ field nào của class
void openDefaultMailbox() {
  dispatchRoute(DashboardRoutes.thread);
  dispatchMailboxUIAction(SelectMailboxDefaultAction());
}

// Lines 2938-2939
void selectAllEmailAction() {
  dispatchAction(SelectionAllEmailAction());
}
```

10+ methods tương tự — chỉ là thin wrappers cho `dispatchAction()`, không có cohesion thực sự.

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

### SingleEmailController

```dart
// Lines 1272-1273: pure delegation, no class state
void openNewTabAction(String link) {
  AppUtils.launchLink(link);
}

// Lines 469-471: pure utility
bool _isBelongToTeamMailboxes(PresentationMailbox? mailbox) {
  return mailbox != null && mailbox.isPersonal == false;
}
```

### MailboxController

```dart
// Lines 1317-1336: scroll utilities — không dùng bất kỳ business field nào
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

> **Định nghĩa:** Class A thường xuyên truy cập data và methods của class B nhiều hơn của chính mình — class A "ghen tị" với class B.

### MailboxController — 75+ references đến `mailboxDashBoardController`

```dart
// Field declaration (line 114) — service locator
final mailboxDashBoardController = Get.find<MailboxDashBoardController>();

// Trong _registerObxStreamListener() — subscribe 3 observables của dashboard
ever(mailboxDashBoardController.accountId, ...)
ever(mailboxDashBoardController.routerParameters, ...)
ever(mailboxDashBoardController.mailboxUIAction, ...)
ever(mailboxDashBoardController.viewState, ...)  // 110+ lines if-else

// Trong business methods — đọc/ghi state của dashboard
mailboxDashBoardController.setMapDefaultMailboxIdByRole(...)  // write
mailboxDashBoardController.setMapMailboxById(...)             // write
mailboxDashBoardController.outboxMailbox = ...               // write
mailboxDashBoardController.searchController.isSearchEmailRunning  // read nested
mailboxDashBoardController.dashboardRoute                    // read
mailboxDashBoardController.selectedMailbox.value             // read
```

**Nguyên nhân:** MailboxController thực chất là một sub-system của MailboxDashBoardController, nhưng được tổ chức như một controller độc lập.

### Sơ đồ Feature Envy — Coupling Web

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
 Feature Envy nặng        20+ method delegation
       │                         │
       ▼                         ▼
 ThreadController         SearchEmailController
  (1,625 lines)             (1,198 lines)
```

### ThreadController — Feature Envy nặng

```dart
// _getAllEmailSuccess() gọi methods trên dashboard controller
mailboxDashBoardController.updateRefreshAllEmailState()
mailboxDashBoardController.updateEmailList(newListEmail)  // mutate state của controller khác!
mailboxDashBoardController.listEmailSelected
mailboxDashBoardController.isSelectionEnabled()

// _loadMoreEmails() đọc state từ dashboard
mailboxDashBoardController.emailsInCurrentMailbox
mailboxDashBoardController.filterMessageOption.value
mailboxDashBoardController.selectedMailbox.value
```

### SingleEmailController — Delegation chain

```dart
// 20+ references đến mailboxDashBoardController
mailboxDashBoardController.openComposer(...)    // open composer
mailboxDashBoardController.moveToMailbox(...)   // delegate email action
mailboxDashBoardController.downloadAttachment(...)
mailboxDashBoardController.listIdentities       // read identity state

// Đồng thời cũng depend vào ThreadDetailController
_threadDetailController?.closeThreadDetailAction()
_threadDetailController?.emailIdsPresentation[emailId]
_threadDetailController?.cacheEmailLoaded()
```
