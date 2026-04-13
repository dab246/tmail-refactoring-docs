# Solution: SearchEmailController

**Problem:** 5 Mixins hiding SRP violations, Primitive Obsession (canSearchMore + SearchMoreState redundant), Bumpy Road (10-case filter switch), Excess Data Declarations (4 ScrollControllers + 4 FocusNodes), Feature Envy.

---

## Specific Problems

```
SearchEmailController (1,198 lines)
├── 5 Mixins: EmailActionController, DateRangePickerMixin, SearchLabelFilterModalMixin,
│             LabelSubMenuMixin, EmailMoreActionContextMenu
├── Primitive Obsession: canSearchMore (bool) redundant with SearchMoreState (enum)
├── Excess Data: 4 TextEditingController/ScrollController/FocusNode
├── onDeleteSearchFilterAction(): 10-case switch — unrelated filter types
├── handleSelectionEmailAction(): 8-case switch
├── Feature Envy: mailboxDashBoardController references
├── _initWorkerListener(): nested ever() callbacks
└── Debouncer pattern adds complexity, hard to test
```

---

## Solution 1 — Fix Primitive Obsession: Merge SearchState

```dart
// Before: 2 fields for the same concept
late SearchMoreState searchMoreState;  // enum
late bool canSearchMore;               // redundant boolean

// Current logic check:
// canSearchMore = true when searchMoreState == SearchMoreState.idle
// canSearchMore = false when searchMoreState == SearchMoreState.noMore

// After: remove canSearchMore, use computed getter
enum SearchLoadState {
  idle,
  searching,
  loadingMore,
  canLoadMore,
  allLoaded,
  error,
}

final searchLoadState = Rx<SearchLoadState>(SearchLoadState.idle);

bool get canSearchMore => searchLoadState.value == SearchLoadState.canLoadMore;
bool get isSearching => searchLoadState.value == SearchLoadState.searching;
bool get isLoadingMore => searchLoadState.value == SearchLoadState.loadingMore;
```

---

## Solution 2 — Fix 5 Mixins: Extract SearchFilterController

**Problem:** 5 mixins hide the fact that the controller is handling: search, email actions, date range, label filtering, context menu — 5 independent concerns.

```dart
// Before
class SearchEmailController extends BaseController
    with EmailActionController,
         DateRangePickerMixin,
         SearchLabelFilterModalMixin,
         LabelSubMenuMixin,
         EmailMoreActionContextMenu { ... }
```

**Extract `SearchFilterController`:**

```dart
// lib/features/search/email/presentation/controller/search_filter_controller.dart
class SearchFilterController extends GetxController {
  final AppEventBus _eventBus;

  // Date range filter state
  final Rxn<DateRangeFilter> selectedDateRange = Rxn();

  // Label filter state
  final RxList<LabelId> selectedLabels = <LabelId>[].obs;

  SearchFilterController(this._eventBus);

  // Fix Bumpy Road: onDeleteSearchFilterAction() 10 cases → delegate pattern
  void deleteFilter(SearchFilterType filterType) {
    switch (filterType) {
      case SearchFilterType.from:
      case SearchFilterType.to:
      case SearchFilterType.subject:
        _eventBus.dispatch(ContactFilterRemovedEvent(filterType));
      case SearchFilterType.hasKeyword:
      case SearchFilterType.notKeyword:
        _eventBus.dispatch(KeywordFilterRemovedEvent(filterType));
      case SearchFilterType.date:
        selectedDateRange.value = null;
        _eventBus.dispatch(DateFilterRemovedEvent());
      case SearchFilterType.mailbox:
        _eventBus.dispatch(MailboxFilterRemovedEvent());
      case SearchFilterType.label:
        selectedLabels.clear();
        _eventBus.dispatch(LabelFilterRemovedEvent());
      default:
        break;
    }
  }

  void applyDateRange(DateRangeFilter range) {
    selectedDateRange.value = range;
    _eventBus.dispatch(DateFilterAppliedEvent(range));
  }

  void toggleLabel(LabelId labelId) {
    if (selectedLabels.contains(labelId)) {
      selectedLabels.remove(labelId);
    } else {
      selectedLabels.add(labelId);
    }
  }
}
```

**SearchEmailController after extraction:**
```dart
// No more 5 mixins — just 1 focused controller
class SearchEmailController extends BaseController {
  final SearchFilterController _filterController;
  final AppEventBus _eventBus;

  // Search interactors only
  final QuickSearchEmailInteractor _quickSearch;
  final SearchEmailInteractor _searchEmail;
  final SearchMoreEmailInteractor _searchMore;
  final RefreshChangesSearchEmailInteractor _refreshSearch;
  final SaveRecentSearchInteractor _saveRecent;
  final GetAllRecentSearchLatestInteractor _getRecentSearches;

  SearchEmailController(
    this._filterController,
    this._eventBus,
    this._quickSearch,
    this._searchEmail,
    this._searchMore,
    this._refreshSearch,
    this._saveRecent,
    this._getRecentSearches,
    ISessionTokenRefresher tokenRefresher,
    ILanguageSettingService languageService,
    IAuthService authService,
  ) : super(tokenRefresher, languageService, authService, ...);
}
```

---

## Solution 3 — Fix Excess Data Declarations: UI State Object

```dart
// Before: 4 scattered UI controller fields
final textInputSearchController = TextEditingController();
final resultSearchScrollController = ScrollController();
final textInputSearchFocus = FocusNode();
final listSearchFilterScrollController = ScrollController();
FocusNode? keyboardFocusNode;

// After: grouped into SearchUIState
// lib/features/search/email/presentation/model/search_ui_state.dart
class SearchUIState {
  final textController = TextEditingController();
  final resultScrollController = ScrollController();
  final filterScrollController = ScrollController();
  final searchFocusNode = FocusNode();
  FocusNode? keyboardFocusNode;

  void dispose() {
    textController.dispose();
    resultScrollController.dispose();
    filterScrollController.dispose();
    searchFocusNode.dispose();
    keyboardFocusNode?.dispose();
  }
}

// SearchEmailController
class SearchEmailController extends BaseController {
  final SearchUIState _uiState = SearchUIState();

  // Exposed via getters for View access
  TextEditingController get textController => _uiState.textController;
  ScrollController get resultScrollController => _uiState.resultScrollController;
  FocusNode get searchFocusNode => _uiState.searchFocusNode;
}
```

---

## Solution 4 — Fix `_initWorkerListener()`: Debouncer + AppEventBus

```dart
// Before: Debouncer + nested ever() callbacks
void _initWorkerListener() {
  ever(mailboxDashBoardController.accountId, (id) {   // feature envy
    if (id != null) {
      _initDebounceSearch();
      // nested logic...
    }
  });
  ever(mailboxDashBoardController.searchQuery, (query) {  // feature envy
    if (query != null && query.isNotEmpty) { ... }
  });
}

void _initDebounceSearch() {
  _deBouncerTime = Debouncer<String>(
    const Duration(milliseconds: 500),
    initialValue: '',
    onChanged: (value) {
      if (value.trim().isNotEmpty) {
        _quickSearchEmails(value);
      }
    },
  );
  textInputSearchController.addListener(() {
    _deBouncerTime.value = textInputSearchController.text;
  });
}
```

```dart
// After: RxString + debounce via reactive stream
@override
void onInit() {
  super.onInit();
  _subscribeToSessionEvents();
  _setupSearchDebounce();
}

void _subscribeToSessionEvents() {
  _eventBus.stream
      .whereType<SessionInitializedEvent>()
      .listen(_onSessionInitialized)
      .addTo(compositeSubscription);
}

void _setupSearchDebounce() {
  // Use GetX debounce instead of custom Debouncer
  final searchText = ''.obs;
  _uiState.textController.addListener(() {
    searchText.value = _uiState.textController.text;
  });

  debounce(
    searchText,
    (value) {
      if (value.trim().isNotEmpty) {
        _quickSearchEmails(value.trim());
      }
    },
    time: const Duration(milliseconds: 500),
  );
}
```

---

## Solution 5 — Fix `handleSelectionEmailAction()` Bumpy Road

```dart
// Before: 8-case switch in SearchEmailController
switch (actionType) {
  case EmailActionType.markAsRead: _markAsRead(emails);
  case EmailActionType.markAsStarred: _markAsStar(emails);
  case EmailActionType.moveToMailbox: _moveToMailbox(emails);
  case EmailActionType.deletePermanently: _delete(emails);
  case EmailActionType.moveToSpam: _moveToSpam(emails);
  case EmailActionType.unSpam: _unSpam(emails);
  case EmailActionType.archiveMessage: _archive(emails);
  case EmailActionType.labelAs: _labelAs(emails);
}

// After: dispatch via EventBus — EmailActionController handles it
void handleSelectionEmailAction(EmailActionType actionType, List<PresentationEmail> emails) {
  _eventBus.dispatch(EmailBulkActionRequestedEvent(
    actionType: actionType,
    emails: emails,
    sourceContext: EmailActionSourceContext.searchResult,
  ));
}
```

---

## Fix StreamController + StreamSubscription Pair

```dart
// Before: 2 fields for 1 stream pair
StreamController<MailListShortcutActionViewEvent>? shortcutActionEventController;
StreamSubscription<MailListShortcutActionViewEvent>? shortcutActionEventSubscription;

// After: encapsulate or use AppEventBus
// Option A: Use AppEventBus for shortcut events
// _eventBus.stream.whereType<MailListShortcutActionViewEvent>().listen(...)

// Option B: If a local stream is needed, wrap in a helper
class _ShortcutEventStream {
  final _controller = StreamController<MailListShortcutActionViewEvent>.broadcast();
  Stream<MailListShortcutActionViewEvent> get stream => _controller.stream;
  void add(MailListShortcutActionViewEvent event) => _controller.add(event);
  void dispose() => _controller.close();
}

// SearchEmailController has only 1 field instead of 2
final _shortcutStream = _ShortcutEventStream();
```

---

## SearchEmailController after refactor

```dart
class SearchEmailController extends BaseController {
  // Dependencies
  final SearchFilterController _filterController;
  final AppEventBus _eventBus;
  final QuickSearchEmailInteractor _quickSearch;
  final SearchEmailInteractor _searchEmail;
  final SearchMoreEmailInteractor _searchMore;
  final RefreshChangesSearchEmailInteractor _refreshSearch;
  final SaveRecentSearchInteractor _saveRecent;
  final GetAllRecentSearchLatestInteractor _getRecentSearches;

  // UI state group
  final _uiState = SearchUIState();

  // Search state — fix primitive obsession
  final searchLoadState = Rx<SearchLoadState>(SearchLoadState.idle);
  final RxList<PresentationEmail> searchResults = <PresentationEmail>[].obs;
  final RxList<RecentSearch> recentSearches = <RecentSearch>[].obs;

  // Computed — no more canSearchMore field
  bool get canSearchMore =>
      searchLoadState.value == SearchLoadState.canLoadMore;

  // Gone: 5 mixins
  // Gone: Debouncer field
  // Gone: canSearchMore field (redundant with searchLoadState)
  // Gone: shortcutActionEventController + shortcutActionEventSubscription
}
```

---

## SearchBindings after refactor

```dart
class SearchEmailBindings extends Bindings {
  @override
  void dependencies() {
    Get.lazyPut(() => SearchFilterController(
      Get.find<AppEventBus>(),
    ));

    Get.lazyPut(() => SearchEmailController(
      Get.find<SearchFilterController>(),
      Get.find<AppEventBus>(),
      Get.find(), Get.find(), Get.find(),
      Get.find(), Get.find(), Get.find(),
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
| Lines | 1,198 | ~400 |
| Mixins | 5 | 0 |
| `onDeleteSearchFilterAction()` | 10-case switch | Delegate via EventBus |
| `handleSelectionEmailAction()` | 8-case switch | 1 line (dispatch) |
| Primitive obsession | canSearchMore + SearchMoreState | Single SearchLoadState |
| UI controller fields | 4-5 fields | 1 (SearchUIState group) |
| Debouncer | Custom Debouncer field | GetX built-in debounce() |
| Feature envy | mailboxDashBoardController refs | 0 (EventBus) |
| Controllers after split | 1 (+5 mixins) | 2 (Search + SearchFilter) |