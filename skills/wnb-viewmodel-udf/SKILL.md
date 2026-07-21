---
name: wnb-viewmodel-udf
description: Use this skill when writing, reviewing, or refactoring an Android ViewModel that follows unidirectional data flow (UDF). Enforces a single StateFlow<XxxState>, a sealed XxxAction, one onAction() entry point, injected DispatcherProvider for IO, MutableSharedFlow only for one-shot navigation/toast events, Result.onSuccess/onFailure from repositories, Timber for logging. Applies to Kotlin + Jetpack Compose projects using MVVM + UDF. Triggers on "new viewmodel", "add viewmodel", "refactor viewmodel", "UDF", "UiState", "XxxState", "onAction", "sealed action", "StateFlow", "MutableStateFlow", "unidirectional data flow", "MVI-lite", "one-shot event", "SharedFlow event".
---

# Android ViewModel — UDF contract

This skill encodes a single ViewModel shape used across a Kotlin + Jetpack Compose codebase. Do not invent variants. If a rule blocks the task, stop and surface the conflict — don't silently break the contract.

## Non-negotiables

1. **One state class per ViewModel.** Named `XxxState`, immutable `data class`, mutated via `copy()`. Everything the UI renders is a field on this class — loading, error, list contents, dialog visibility, form input.
2. **One `StateFlow<XxxState>`.** Backing `MutableStateFlow` is private (`_state`); public is `val state: StateFlow<XxxState> = _state.asStateFlow()`.
3. **One `onAction(action: XxxAction)` entry point.** All UI events flow through it. `XxxAction` is a sealed interface (or sealed class) declared in the same file or feature package. No public `onXxx()` per-event methods.
4. **No `mutableStateOf` inside a ViewModel.** Compose-side only.
5. **No string keys for navigation.** Use type-safe route classes.
6. **IO always goes through an injected `DispatcherProvider`.** Never `Dispatchers.IO` directly — it can't be swapped in tests.
7. **Repositories return `Result<T>` or `Flow<T>`.**
   - `Result<T>` → `.onSuccess { _state.update { ... } }.onFailure { Timber.e(it, "..."); _state.update { it.copy(error = ...) } }`
   - `Flow<T>` → `.catch { Timber.e(it, "..."); _state.update { ... } }.collect { ... }`
8. **One-shot events** (navigation, snackbar, toast) go through `MutableSharedFlow<T>(extraBufferCapacity = 1)`, exposed as `SharedFlow<T>`. Never on the state — putting them there causes replay on config change / process death.
9. **Error logging:** `Timber.e(throwable, "ClassName: what failed")` — throwable is the first argument.
10. **Mutation is always atomic:** `_state.update { it.copy(...) }`. Never `_state.value = _state.value.copy(...)` (race window between read and write).

## Canonical example

```kotlin
class FooViewModel(
    private val fooRepository: FooRepository,
    private val dispatcherProvider: DispatcherProvider,
) : ViewModel() {

    private val ioDispatcher = dispatcherProvider.io()

    private val _state = MutableStateFlow(FooState())
    val state: StateFlow<FooState> = _state.asStateFlow()

    private val _navigationEvent = MutableSharedFlow<String>(extraBufferCapacity = 1)
    val navigationEvent: SharedFlow<String> = _navigationEvent.asSharedFlow()

    init {
        observeFoos()
    }

    private fun observeFoos() {
        viewModelScope.launch(ioDispatcher) {
            fooRepository.observeFoos()
                .catch { error ->
                    Timber.e(error, "FooViewModel: observeFoos failed")
                    _state.update { it.copy(isLoading = false, error = error.message) }
                }
                .collect { foos ->
                    _state.update { it.copy(foos = foos, isLoading = false) }
                }
        }
    }

    fun onAction(action: FooAction) {
        when (action) {
            is FooAction.Refresh       -> refresh()
            is FooAction.Select        -> _state.update { it.copy(selected = action.foo) }
            is FooAction.DismissError  -> _state.update { it.copy(error = null) }
            is FooAction.NavigateToDetail -> emitNavigation(action.fooId)
        }
    }

    private fun refresh() {
        viewModelScope.launch(ioDispatcher) {
            _state.update { it.copy(isLoading = true, error = null) }
            fooRepository.refresh()
                .onSuccess {
                    _state.update { it.copy(isLoading = false) }
                }
                .onFailure { error ->
                    Timber.e(error, "FooViewModel: refresh failed")
                    _state.update { it.copy(isLoading = false, error = error.message) }
                }
        }
    }

    private fun emitNavigation(fooId: String) {
        viewModelScope.launch { _navigationEvent.emit(fooId) }
    }
}

data class FooState(
    val foos: List<Foo> = emptyList(),
    val selected: Foo? = null,
    val isLoading: Boolean = true,
    val error: String? = null,
)

sealed interface FooAction {
    data object Refresh : FooAction
    data class Select(val foo: Foo) : FooAction
    data object DismissError : FooAction
    data class NavigateToDetail(val fooId: String) : FooAction
}
```

## Common mistakes

- **`_state.value = _state.value.copy(...)`** → use `_state.update { it.copy(...) }`. Why: atomic, no lost-write race under concurrent emissions.
- **Putting a navigation route on `XxxState` and observing it from Compose** → causes replay on rotation or when the screen re-collects. Use `SharedFlow` for one-shot events.
- **Injecting `Dispatchers.IO` directly** → not swappable in tests, so tests hit real threads and become flaky. Always inject `DispatcherProvider`.
- **Multiple public `onXxx()` methods instead of a single `onAction`** → each screen ends up different; contract drifts. Keep the single funnel.
- **Business logic inside a Composable** → move to the ViewModel, or extract to a UseCase in the domain layer for anything with rules (pricing, validation, eligibility).
- **Catching `Throwable` and swallowing it** → always `Timber.e(throwable, "ClassName: message")` before mutating state.
- **`init { }` doing IO on the main thread** → wrap in `viewModelScope.launch(ioDispatcher)`.
- **State with imperative fields like `showDialog: () -> Unit`** → state describes what the UI *is*, not what it *does*. Actions describe what the UI does. Use `data class` fields (`isDialogVisible: Boolean`) and `sealed interface` variants.

## When to extract to a UseCase

If the ViewModel is orchestrating **rules** (discount math, coupon eligibility, cart pricing, form validation) — extract to a use case in the domain module (`:domain/usecase/`) with its own pure JUnit test. The ViewModel then calls the use case and only maps its result to state. If the ViewModel is just observing → mapping → rendering, no use case needed.

## Related skills

- `[[wnb-viewmodel-test]]` — the matching unit-test skeleton for this ViewModel shape.
- `[[wnb-koin-feature-module]]` — how to wire this ViewModel into DI with a feature-scoped Koin module.
