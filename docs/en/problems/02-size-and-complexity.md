# Problems: Size & Complexity

> Issue group: **#1 Constructor Over-Injection · #2 Large Method · #3 Complex Method · #14 BaseController God Class**

---

## 1. Constructor Over-Injection

> **Definition:** Constructor receives too many dependencies — a direct symptom of SRP violation.

### MailboxDashboardController — 43 dependencies total

Constructor receives **29 interactors**:

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

Plus **14 deps from BaseController** (CachingManager, LanguageCacheManager, AuthorizationInterceptors x2, DynamicUrlInterceptors, DeleteCredentialInteractor, LogoutOidcInteractor, DeleteAuthorityOidcInteractor, AppToast, ImagePaths, ResponsiveUtils, Uuid, ToastManager, TwakeAppManager) = **43 total**.

**Bounded contexts crammed into 1 constructor:**
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
Get.find() in field declarations: 5
  (MailboxDashBoardController, SearchEmailController,
   NetworkConnectionController, ThreadDetailManager, DownloadManager)
```

### God Object Diagram — 43 dependencies of MailboxDashBoardController

```
 ┌──────────────────────────────────────────────────────────────────┐
 │          MailboxDashBoardController  (3,508 lines)               │
 │                    43 dependencies total                         │
 └────┬──────────┬──────────┬──────────┬──────────┬────────────────┘
      │          │          │          │          │
      ▼          ▼          ▼          ▼          ▼
 ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────────────┐
 │📧 Email │ │📤 Sending│ │🔐Session│ │💾 Cache │ │🔄 Recovery      │
 │ Actions │ │  Queue   │ │Identity │ │  (Web)  │ │   / Mailbox     │
 │8 inter. │ │5 inter.  │ │2 inter. │ │4 inter. │ │4 inter.         │
 └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────────────┘

 + Inherited from BaseController (14 additional deps):
 ┌──────────────────────────────────────────────────────────────────┐
 │🏛 CachingManager · LanguageCacheManager · AuthInterceptors ×2    │
 │   DynamicUrlInterceptors · DeleteCredential · LogoutOidc         │
 │   DeleteAuthorityOidc · AppToast · ImagePaths · ResponsiveUtils  │
 │   Uuid · ToastManager · TwakeAppManager                          │
 └──────────────────────────────────────────────────────────────────┘
```

---

## 2. Large Method

> **Definition:** Method > 30 lines — hard to read, test, and maintain.

| Method | File | Lines | Content |
|---|---|---|---|
| `handleSuccessViewState()` | MailboxDashboard | ~113 lines (453–566) | 31 if-else branches, 31 bounded contexts |
| `handleFailureViewState()` | MailboxDashboard | ~65 lines (568–619) | 22 if-else branches |
| `_registerObxStreamListener()` | ThreadController | ~85 lines (267–353) | 13 action types, 3-4 nesting levels |
| `_registerObxStreamListener()` | SingleEmail | ~108 lines (284–391) | 10 action types, 4 nesting levels |
| `handleEmailAction()` | SingleEmail | ~88 lines (874–962) | 20 switch cases |
| `handleMailboxAction()` | Mailbox | ~112 lines (1158–1270) | 18 switch cases |
| `pressEmailSelectionAction()` | Thread | ~60 lines (1271–1330) | 15 switch cases |
| `_registerObxStreamListener()` | Mailbox | ~159 lines (294–453) | nested `ever()` with 7+ action types |

### Concrete example — `handleSuccessViewState` (MailboxDashboard, lines 453–566)

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
  // ... 26 more branches — each branch is a completely different bounded context
}
```

**Impact:** Every time a new feature is added, this method must be modified — violates OCP. This method is acting as a **God Dispatcher** for the entire app.

---

## 3. Complex Method

> **Definition:** Cyclomatic complexity > 10 — too many branches, hard to trace logic.

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
    // 110+ lines of if-else chains checking each state type
  });
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

5 different concerns in 1 method: guard, transformation, logging, state update, UI command.

---

## 14. BaseController — God Class

> **Definition:** A base class that takes on too many responsibilities — every subcontroller inherits all of this "baggage".

**File:** `lib/features/base/base_controller.dart`
**~647 lines, 49 methods, 15+ fields, 7 responsibility domains**

### Fields — Importing from the data layer (Clean Architecture violation)

```dart
// BaseController directly imports and injects data-layer classes
final CachingManager cachingManager;
final LanguageCacheManager languageCacheManager;
final AuthorizationInterceptors authorizationInterceptors;
final AuthorizationInterceptors authorizationIsolateInterceptors;
final DynamicUrlInterceptors dynamicUrlInterceptors;
```

**Problem:** `presentation/base_controller.dart` imports the `data/` layer directly — a Clean Architecture violation. Every controller in the app inherits this violation.

### 7 Responsibility Domains in 1 class

| Domain | Methods |
|---|---|
| State management | `onData()`, `onError()`, `dispatchState()`, `clearState()` |
| Error handling | `handleErrorViewState()`, `handleUrgentException()`, `handleBadCredentialsException()`, `handleRefreshTokenFailedException()` |
| FCM / Push notifications | `injectFCMBindings()`, `injectWebSocket()` |
| Authentication | `logout()`, `logoutToSignInNewAccount()`, `_handleLogoutAction()` |
| Capability injection | `injectAutoCompleteBindings()`, `injectMdnBindings()`, `injectForwardBindings()`, `injectRuleFilterBindings()` |
| Performance monitoring | `startFpsMeter()`, `stopFpsMeter()` |
| Navigation | `goToLogin()`, `removeAllPageAndGoToLogin()`, `navigateToLoginPage()`, `navigateToTwakeWelcomePage()` |

**Impact:** Even the simplest controller in the app must drag along all 43 deps and these 7 domains.

### Diagram — BaseController pulling in the entire app

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

 Every controller inherits all of this baggage:
 ┌───────────────┐ ┌────────────────┐ ┌────────────────┐
 │MailboxDashboard│ │MailboxController│ │ThreadController│
 │43 deps total   │ │extends BaseCtrl │ │extends BaseCtrl│
 └───────────────┘ └────────────────┘ └────────────────┘
 ┌───────────────┐ ┌────────────────┐
 │SingleEmail    │ │SearchEmail     │
 │extends BaseCtrl│ │extends BaseCtrl│
 └───────────────┘ └────────────────┘
```
