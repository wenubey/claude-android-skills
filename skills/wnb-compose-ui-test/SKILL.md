---
name: wnb-compose-ui-test
description: Use this skill when writing, reviewing, or refactoring a Jetpack Compose UI test on Android. Enforces the canonical skeleton — @get:Rule createComposeRule(), an inline DispatcherProvider using Dispatchers.Unconfined so state propagates synchronously to the tree, a renderScreen(...) factory that constructs the ViewModel + sets content and returns the VM for assertion, semantics-first finders (onNodeWithText, onNodeWithTag, onNodeWithContentDescription, hasTestTag), Truth assertions on state, no Thread.sleep (waitForIdle / waitUntil instead), no hardcoded coordinates, no mocking of repositories (use fakes). Applies to instrumentation-style tests under app/src/androidTest. Triggers on "compose ui test", "createComposeRule", "androidTest", "instrumentation test", "onNodeWithText", "onNodeWithTag", "performClick", "performTextInput", "waitForIdle", "setContent", "compose test dispatcher", "test Screen composable".
---

# Jetpack Compose UI test — canonical skeleton

Every screen-level Compose test in this project has the same shape. Same rule, same dispatcher trick, same factory pattern, same finder discipline. Do not invent variants.

Pairs with `[[wnb-viewmodel-udf]]` and `[[wnb-viewmodel-test]]` — a screen's ViewModel unit test verifies the state machine; this UI test verifies the state renders correctly and one critical interaction works.

## Non-negotiables

1. **`@get:Rule val composeTestRule = createComposeRule()`** — one rule per test class. Never `createAndroidComposeRule<Activity>()` unless you specifically need the host Activity (rare).
2. **Inline `DispatcherProvider` with `Dispatchers.Unconfined`** for all three (`main`, `io`, `default`). This makes VM state updates propagate to the tree synchronously — no missed emissions, no `advanceUntilIdle`. Only exception: tests that specifically need to inspect intermediate loading state can use a `StandardTestDispatcher` + `MainDispatcherRule` combo; the default is Unconfined.
3. **`renderScreen(...)` factory** at the top of every test class. Constructs the VM with defaultable fake repositories, calls `composeTestRule.setContent { XxxScreen(viewModel = vm, …) }`, and **returns the VM** so tests can assert against `vm.state.value` directly.
4. **Semantics-first finders**, in this preference order:
   - `onNodeWithTag("submitButton")` — best for interactive elements (add `Modifier.testTag(...)` in the composable).
   - `onNodeWithContentDescription("Cart icon")` — for icons/images; also earns you a11y coverage.
   - `onNodeWithText("Sign In")` — for user-visible text. Fine for static labels; brittle for i18n strings and dynamic content.
   - `onNode(hasTestTag("...") and hasText("..."))` — for compound matches.
   - Never coordinates (`onNodeAt(x, y)`), never index-based sibling access.
5. **Assertion split.** UI assertions on nodes: `assertIsDisplayed()`, `assertIsEnabled()`, `assertTextEquals(...)`, `assertIsToggleable()`. State assertions on the ViewModel via Truth: `assertThat(vm.state.value.email).isEqualTo(...)`.
6. **`composeTestRule.waitForIdle()` after every interaction** that triggers a coroutine or recomposition. Never `Thread.sleep`.
7. **No mocking of repository interfaces.** Use `TestXxxRepository` fakes with in-memory state, mirroring the unit-test convention.
8. **One test = one behavior.** Render, one interaction (or none), one assertion cluster. If you need two flows, write two tests.

## Canonical example

```kotlin
class FooScreenTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    private val dispatcherProvider = object : DispatcherProvider {
        override fun main(): CoroutineDispatcher = Dispatchers.Unconfined
        override fun io(): CoroutineDispatcher = Dispatchers.Unconfined
        override fun default(): CoroutineDispatcher = Dispatchers.Unconfined
    }

    private fun renderScreen(
        repo: TestFooRepository = TestFooRepository(),
        onNavigateToDetail: (String) -> Unit = {},
    ): FooViewModel {
        val vm = FooViewModel(repo, dispatcherProvider)
        composeTestRule.setContent {
            FooScreen(
                viewModel = vm,
                onNavigateToDetail = onNavigateToDetail,
            )
        }
        return vm
    }

    @Test
    fun renders_title_and_empty_state_when_no_foos() {
        renderScreen(repo = TestFooRepository(foos = emptyList()))

        composeTestRule.onNodeWithText("Foos").assertIsDisplayed()
        composeTestRule.onNodeWithText("No foos yet").assertIsDisplayed()
    }

    @Test
    fun renders_list_of_foos_when_repo_returns_data() {
        renderScreen(repo = TestFooRepository(foos = listOf(Foo("1", "Alpha"), Foo("2", "Beta"))))

        composeTestRule.onNodeWithText("Alpha").assertIsDisplayed()
        composeTestRule.onNodeWithText("Beta").assertIsDisplayed()
    }

    @Test
    fun clicking_refresh_button_reloads_and_shows_new_items() {
        val repo = TestFooRepository(foos = listOf(Foo("1", "Alpha")))
        renderScreen(repo = repo)
        composeTestRule.onNodeWithText("Alpha").assertIsDisplayed()

        repo.setFoos(listOf(Foo("2", "Beta")))
        composeTestRule.onNodeWithTag("refreshButton").performClick()
        composeTestRule.waitForIdle()

        composeTestRule.onNodeWithText("Beta").assertIsDisplayed()
    }

    @Test
    fun clicking_a_row_invokes_navigation_callback_with_id() {
        var navigatedTo: String? = null
        renderScreen(
            repo = TestFooRepository(foos = listOf(Foo("42", "Alpha"))),
            onNavigateToDetail = { navigatedTo = it },
        )

        composeTestRule.onNodeWithText("Alpha").performClick()
        composeTestRule.waitForIdle()

        assertThat(navigatedTo).isEqualTo("42")
    }

    @Test
    fun error_state_shows_retry_and_dismisses_on_click() {
        val vm = renderScreen(repo = TestFooRepository(refreshResult = Result.failure(RuntimeException("nope"))))
        vm.onAction(FooAction.Refresh)
        composeTestRule.waitForIdle()

        composeTestRule.onNodeWithText("nope").assertIsDisplayed()
        composeTestRule.onNodeWithTag("dismissErrorButton").performClick()
        composeTestRule.waitForIdle()

        assertThat(vm.state.value.error).isNull()
    }
}
```

## Common mistakes

- **`Thread.sleep(500)` to wait for recomposition** → flaky on CI. Use `composeTestRule.waitForIdle()` or `composeTestRule.waitUntil(2_000) { predicate }`.
- **Using `Dispatchers.Main` in `DispatcherProvider`** → hangs the test because there is no real main looper in the Compose test runner. Use `Dispatchers.Unconfined`.
- **`onNodeWithText("Sign In")` for a dynamic i18n string** → breaks when copy changes. Prefer `onNodeWithTag("signInButton")`.
- **Hardcoded coordinates** (`performTouchInput { down(Offset(200f, 400f)) }`) → device-specific, brittle. Address by semantics.
- **Asserting only on the UI (`assertIsDisplayed`) but never on VM state** → misses regressions where the render is right but the state is wrong. Assert both when a behavior spans both.
- **`onNodeWithText` on merged text (`"Email$errorSuffix"`)** → won't match; text is merged from separate `Text` composables. Use `hasText(..., substring = true)` or a testTag.
- **Constructing the ViewModel in a `@Before`/`init` block** → shared across tests, cross-test bleed. Always construct via `renderScreen(...)` inside the test.
- **Mocking the repository interface** → same reasoning as unit tests. Use a `TestFooRepository` fake.
- **`setContent { }` called twice in one test** → `IllegalStateException`. `renderScreen` guarantees single call.
- **Reading `vm.state.value` before `waitForIdle()`** after an interaction → may read stale state. Always wait first.
- **Using `createAndroidComposeRule<MainActivity>()` for a standalone screen** → boots the whole app, slow, brittle. Use `createComposeRule()` unless the SUT explicitly reads from the host `Activity`.

## Preview screenshots vs UI tests

If the goal is "does this screen render for every state" — that's a screenshot / preview job (Paparazzi, Roborazzi, or the on-device HotSwan preview catalog), not a Compose UI test. Reserve Compose UI tests for **behavior** (state + interaction), not visual regression.

## Related skills

- `[[wnb-viewmodel-udf]]` — the ViewModel shape the SUT depends on.
- `[[wnb-viewmodel-test]]` — the unit-test companion. Split what each covers: unit test = state machine; UI test = renders + one critical interaction.
- `[[wnb-compose-animation]]` — animation authoring rules; testing animated UI has extra care around `mainClock.advanceTimeBy`.
