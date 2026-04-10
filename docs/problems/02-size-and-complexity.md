# Problems: Kích thước & Độ phức tạp

> Nhóm vấn đề: **#1 Constructor Over-Injection · #2 Large Method · #3 Complex Method · #14 BaseController God Class**

---

## 1. Constructor Over-Injection

> **Định nghĩa:** Constructor nhận quá nhiều dependencies, là triệu chứng trực tiếp của SRP violation.

### MailboxDashboardController — 43 dependencies tổng cộng

Constructor nhận **29 interactors**:

```dart
MailboxDashBoardController(
  this._moveToMailboxInteractor,          // email action
  this._deleteEmailPermanentlyInteractor, // email action
  this._markAsMailboxReadInteractor,      // mailbox action
  this._getEmailCacheOnWebInteractor,     // cache
  this._getIdentityCacheOnWebInteractor,  // cache
  this._markAsEmailReadInteractor,        // email action
  this._markAsStarEmailInteractor,        // email action
  this._markAsMultipleEmailReadInteractor,
  this._markAsStarMultipleEmailInteractor,
  this._moveMultipleEmailToMailboxInteractor,
  this._emptyTrashFolderInteractor,       // mailbox action
  this._deleteMultipleEmailsPermanentlyInteractor,
  this._getEmailByIdInteractor,
  this._sendEmailInteractor,              // sending queue
  this._storeSendingEmailInteractor,      // sending queue
  this._updateSendingEmailInteractor,     // sending queue
  this._getAllSendingEmailInteractor,     // sending queue
  this._storeSessionInteractor,           // session
  this._emptySpamFolderInteractor,        // spam
  this._deleteSendingEmailInteractor,
  this._unsubscribeEmailInteractor,
  this._restoreDeletedMessageInteractor,  // email recovery
  this._getRestoredDeletedMessageInteractor,
  this._removeAllComposerCacheOnWebInteractor, // composer cache
  this._removeComposerCacheByIdOnWebInteractor,
  this._getAllIdentitiesInteractor,        // identity/session
  this.clearMailboxInteractor,
  this.storeEmailSortOrderInteractor,     // preferences
  this.getStoredEmailSortOrderInteractor,
);
```

Cộng thêm **14 deps từ BaseController** (CachingManager, LanguageCacheManager, AuthorizationInterceptors x2, DynamicUrlInterceptors, DeleteCredentialInteractor, LogoutOidcInteractor, DeleteAuthorityOidcInteractor, AppToast, ImagePaths, ResponsiveUtils, Uuid, ToastManager, TwakeAppManager) = **43 tổng**.

**Các bounded context bị nhét chung vào 1 constructor:**
- Email actions (move, delete, mark, star)
- Sending queue management
- Composer cache (web only)
- Session / identity
- Spam management
- Email recovery
- Mailbox operations
- User preferences

### ThreadDetailController — 15 dependencies

```
Constructor: 10 interactors
Get.find() trong field declarations: 5
  (MailboxDashBoardController, SearchEmailController,
   NetworkConnectionController, ThreadDetailManager, DownloadManager)
```

### Sơ đồ God Object — 43 dependencies của MailboxDashBoardController

```
 ┌──────────────────────────────────────────────────────────────────┐
 │          MailboxDashBoardController  (3,508 lines)               │
 │                    43 dependencies tổng cộng                     │
 └────┬──────────┬──────────┬──────────┬──────────┬────────────────┘
      │          │          │          │          │
      ▼          ▼          ▼          ▼          ▼
 ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────────────┐
 │📧 Email │ │📤 Sending│ │🔐Session│ │💾 Cache │ │🔄 Recovery      │
 │ Actions │ │  Queue   │ │Identity │ │  (Web)  │ │   / Mailbox     │
 │8 inter. │ │5 inter.  │ │2 inter. │ │4 inter. │ │4 inter.         │
 └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────────────┘

 + Kế thừa từ BaseController (14 deps thêm):
 ┌──────────────────────────────────────────────────────────────────┐
 │🏛 CachingManager · LanguageCacheManager · AuthInterceptors ×2    │
 │   DynamicUrlInterceptors · DeleteCredential · LogoutOidc         │
 │   DeleteAuthorityOidc · AppToast · ImagePaths · ResponsiveUtils  │
 │   Uuid · ToastManager · TwakeAppManager                          │
 └──────────────────────────────────────────────────────────────────┘
```

---

## 2. Large Method

> **Định nghĩa:** Method > 30 lines, khó đọc, khó test, khó maintain.

| Method | File | Lines | Nội dung |
|---|---|---|---|
| `handleSuccessViewState()` | MailboxDashboard | ~113 lines (453–566) | 31 if-else branches, 31 bounded contexts |
| `handleFailureViewState()` | MailboxDashboard | ~65 lines (568–619) | 22 if-else branches |
| `_registerObxStreamListener()` | ThreadController | ~85 lines (267–353) | 13 action types, 3-4 nesting levels |
| `_registerObxStreamListener()` | SingleEmail | ~108 lines (284–391) | 10 action types, 4 nesting levels |
| `handleEmailAction()` | SingleEmail | ~88 lines (874–962) | 20 switch cases |
| `handleMailboxAction()` | Mailbox | ~112 lines (1158–1270) | 18 switch cases |
| `pressEmailSelectionAction()` | Thread | ~60 lines (1271–1330) | 15 switch cases |
| `_registerObxStreamListener()` | Mailbox | ~159 lines (294–453) | nested `ever()` with 7+ action types |

### Ví dụ cụ thể — `handleSuccessViewState` (MailboxDashboard, lines 453–566)

```dart
void handleSuccessViewState(Success success) {
  super.handleSuccessViewState(success);
  if (success is GetAllMailboxSuccess) {
    _handleGetAllMailboxSuccess(success);
  } else if (success is MoveToMailboxSuccess) {
    _handleMoveToMailboxSuccess(success);
  } else if (success is DeleteEmailPermanentlySuccess) {
    _handleDeleteEmailPermanentlySuccess(success);
  } else if (success is MarkAsEmailReadSuccess) {
    _handleMarkAsEmailReadSuccess(success);
  } else if (success is SendEmailSuccess) {
    _handleSendEmailSuccess(success);
  }
  // ... 26 nhánh nữa — mỗi nhánh là 1 bounded context hoàn toàn khác nhau
}
```

**Impact:** Mỗi khi thêm tính năng mới phải sửa method này → vi phạm OCP. Method này đang là **God Dispatcher** cho toàn bộ app.

---

## 3. Complex Method

> **Định nghĩa:** Cyclomatic complexity > 10 — quá nhiều nhánh, khó trace logic.

### `_registerObxStreamListener()` — MailboxController (lines 294–453, CC ~15+)

```dart
void _registerObxStreamListener() {
  ever(mailboxDashBoardController.mailboxUIAction, (action) {  // observer 1
    if (action is SelectMailboxDefaultAction) { ... }
    else if (action is RefreshChangeMailboxAction) { ... }
    else if (action is OpenMailboxAction) {
      if (action.presentationMailbox.role == ...) { ... }  // nested condition
    }
    else if (action is SystemBackToInboxAction) { ... }
    else if (action is RefreshAllMailboxAction) { ... }
    else if (action is AutoCreateActionRequiredFolderMailboxAction) { ... }
    else if (action is AutoRemoveActionRequiredFolderMailboxAction) { ... }
  });

  ever(mailboxDashBoardController.viewState, (viewState) {  // observer 2
    // 110+ lines của if-else chains kiểm tra từng state type
  });
}
```

### `_handleDataFromNavigationRouter()` — MailboxController (lines 757–819, CC ~8)

```dart
switch(_navigationRouter!.dashboardType) {
  case DashboardType.search:
    if (_navigationRouter!.emailId != null) { ... }
    else if (_navigationRouter!.searchQuery?.value.isNotEmpty == true) { ... }
    else { ... }
    break;
  case DashboardType.normal:
    if (_navigationRouter!.labelId != null) { ... }
    else if (_navigationRouter!.mailboxId != null) {
      if (matchedMailboxNode != null) {
        if (_navigationRouter!.emailId != null) { ... }  // 4 levels deep
        else { ... }
      } else { ... }
    }
    else if (_navigationRouter!.emailId != null) { ... }
    else { ... }
    break;
}
```

### `_getAllEmailSuccess()` — ThreadController (lines 562–597)

```dart
void _getAllEmailSuccess(GetAllEmailSuccess success, {bool shouldJumpToFirstEmail = true}) {
  // Concern 1: null check / guard clause
  if (currentMailboxId != null && (isVirtualFolder || currentMailboxId != selectedMailboxId)) {
    return;
  }
  // Concern 2: data transformation
  final newListEmail = success.emailList.syncPresentationEmail(...);
  // Concern 3: logging
  logTrace('ThreadController::_getAllEmailSuccess():...');
  // Concern 4: state mutation (on another controller!)
  mailboxDashBoardController.updateEmailList(newListEmail);
  // Concern 5: UI scroll
  if (shouldJumpToFirstEmail && listEmailController.hasClients) {
    listEmailController.jumpTo(0);
  }
}
```

5 concerns khác nhau trong 1 method: guard, transformation, logging, state update, UI command.

---

## 14. BaseController — God Class

> **Định nghĩa:** Class cơ sở đảm nhận quá nhiều responsibilities, mọi controller con đều kế thừa toàn bộ "hành lý" này.

**File:** `lib/features/base/base_controller.dart`
**~647 lines, 49 methods, 15+ fields, 7 responsibility domains**

### Fields — Import từ data layer (vi phạm Clean Architecture)

```dart
// BaseController trực tiếp import và inject data-layer classes
final CachingManager cachingManager;
final LanguageCacheManager languageCacheManager;
final AuthorizationInterceptors authorizationInterceptors;
final AuthorizationInterceptors authorizationIsolateInterceptors;
final DynamicUrlInterceptors dynamicUrlInterceptors;
```

**Vấn đề:** `presentation/base_controller.dart` import thẳng `data/` layer — vi phạm Clean Architecture. Mọi controller trong app đều kế thừa sự vi phạm này.

### 7 Responsibility Domains trong 1 class

| Domain | Methods |
|---|---|
| State management | `onData()`, `onError()`, `dispatchState()`, `clearState()` |
| Error handling | `handleErrorViewState()`, `handleUrgentException()`, `handleBadCredentialsException()`, `handleRefreshTokenFailedException()` |
| FCM / Push notifications | `injectFCMBindings()`, `injectWebSocket()` |
| Authentication | `logout()`, `logoutToSignInNewAccount()`, `_handleLogoutAction()` |
| Capability injection | `injectAutoCompleteBindings()`, `injectMdnBindings()`, `injectForwardBindings()`, `injectRuleFilterBindings()` |
| Performance monitoring | `startFpsMeter()`, `stopFpsMeter()` |
| Navigation | `goToLogin()`, `removeAllPageAndGoToLogin()`, `navigateToLoginPage()`, `navigateToTwakeWelcomePage()` |

**Impact:** Mọi controller đơn giản nhất trong app cũng phải kéo theo 43 deps và 7 domains này.

### Sơ đồ — BaseController kéo theo toàn bộ app

```
              ┌──────────────────────────────────────────────┐
              │              BaseController                   │
              │   647 lines · 7 domains · import data layer  │
              └──┬──────────┬──────────┬──────────┬──────────┘
                 │          │          │          │
                 ▼          ▼          ▼          ▼
           State Mgmt  Error Handle  Auth/Logout  Navigation
           onData()    handleUrgent  logout()     goToLogin()
           dispatch    handleBadCred logoutToNew  navigateTo..

                 │          │          │
                 ▼          ▼          ▼
           FCM/WebSocket  Capability  Performance
           injectFCM      injectAuto  startFpsMeter
           injectWS       injectMdn   stopFpsMeter
                          injectFwd
                          injectRule

 Mọi controller đều kế thừa toàn bộ hành lý này:
 ┌───────────────┐ ┌────────────────┐ ┌────────────────┐
 │MailboxDashboard│ │MailboxController│ │ThreadController│
 │43 deps total   │ │extends BaseCtrl │ │extends BaseCtrl│
 └───────────────┘ └────────────────┘ └────────────────┘
 ┌───────────────┐ ┌────────────────┐
 │SingleEmail    │ │SearchEmail     │
 │extends BaseCtrl│ │extends BaseCtrl│
 └───────────────┘ └────────────────┘
```
