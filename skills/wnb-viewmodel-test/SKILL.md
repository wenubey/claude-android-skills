---
name: wnb-viewmodel-test
description: Use this skill when writing, reviewing, or refactoring a unit test for an Android ViewModel that follows the UDF contract (StateFlow<XxxState> + sealed XxxAction + onAction). Enforces the canonical test skeleton — @get:Rule MainDispatcherRule, TestDispatcherProvider(mainDispatcherRule.testDispatcher), factory function newViewModel(fake: FakeXxxRepository = FakeXxxRepository()), runTest { advanceUntilIdle() }, Truth assertions, Turbine for Flow/SharedFlow emissions, no Thread.sleep, no mocking of repository interfaces (use fakes). Triggers on "viewmodel test", "ViewModel unit test", "MainDispatcherRule", "TestDispatcherProvider", "runTest", "advanceUntilIdle", "Turbine", "fake repository", "test SharedFlow", "test one-shot event", "StandardTestDispatcher", "test coroutines", "Truth assertThat".
---

# Android ViewModel — unit-test skeleton

Every ViewModel that follows the UDF contract ships with a matching unit test in this shape. Same imports, same rule ordering, same factory pattern. If a rule below blocks the task, stop and surface the conflict — don't silently break the contract.

Pairs with `[[wnb-viewmodel-udf]]`.

## Non-negotiables

1. **`@get:Rule val mainDispatcherRule = MainDispatcherRule()`** at the top of every test class. Replaces `Dispatchers.Main` with a `StandardTestDispatcher` for the duration of each test.
2. **`private val dispatcherProvider = TestDispatcherProvider(mainDispatcherRule.testDispatcher)`** — reuses the rule's dispatcher so all IO work runs on the same virtual clock as the main dispatcher. Never construct a second `TestDispatcher`.
3. **Factory function** `newViewModel(...)` at the top of the class. Every dependency has a default `FakeXxxRepository()`. Individual tests override only what they need. Keeps each test independent, no shared mutable state.
4. **`@OptIn(ExperimentalCoroutinesApi::class)`** on the class (or per-test) — `runTest`, `advanceUntilIdle`, `TestDispatcher` require it.
5. **`runTest { ... }`** wraps every coroutine test. `advanceUntilIdle()` after any `launch { }` in the SUT to drain the queue. Use `runCurrent()` when you need a single step and want to inspect intermediate state.
6. **`Truth` assertions** (`assertThat(x).isEqualTo(y)`), not JUnit `assertEquals`. Clearer failure messages, chained matchers.
7. **Fakes, never mocks**, for repository interfaces. Fakes live in `app/src/test/java/.../testing/fakes/` (or the test source set your project uses). A mocked interface is a lie — a fake exercises the real contract.
8. **No `Thread.sleep`.** Use `advanceUntilIdle()`, `advanceTimeBy(ms)`, or Turbine's `awaitItem()`.
9. **Turbine for `Flow` / `SharedFlow` emissions.** Snapshot with `state.value` is fine when asserting a single final state, but any sequence-of-emissions assertion (loading → loaded, one-shot events) uses `state.test { ... }` / `navigationEvent.test { ... }`.
10. **Backtick test names** describe the behavior, not the method under test: `` `unauthenticated user routes to SignUp and marks initialized`() ``. Given–When–Then is fine but keep it terse.

## Canonical example

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class FooViewModelTest {

    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    private val dispatcherProvider = TestDispatcherProvider(mainDispatcherRule.testDispatcher)

    private fun newViewModel(
        repo: FakeFooRepository = FakeFooRepository(),
    ) = FooViewModel(repo, dispatcherProvider)

    @Test
    fun `initial state is loading, no foos, no error`() = runTest {
        val vm = newViewModel()

        // Observe the first emission before letting coroutines run
        assertThat(vm.state.value.isLoading).isTrue()
        assertThat(vm.state.value.foos).isEmpty()
        assertThat(vm.state.value.error).isNull()
    }

    @Test
    fun `observeFoos emits foos and clears loading`() = runTest {
        val repo = FakeFooRepository(foos = listOf(Foo("1"), Foo("2")))
        val vm = newViewModel(repo = repo)

        advanceUntilIdle()

        assertThat(vm.state.value.foos).hasSize(2)
        assertThat(vm.state.value.isLoading).isFalse()
        assertThat(vm.state.value.error).isNull()
    }

    @Test
    fun `refresh failure surfaces error and clears loading`() = runTest {
        val repo = FakeFooRepository(refreshResult = Result.failure(RuntimeException("nope")))
        val vm = newViewModel(repo = repo)
        advanceUntilIdle()

        vm.onAction(FooAction.Refresh)
        advanceUntilIdle()

        assertThat(vm.state.value.error).isEqualTo("nope")
        assertThat(vm.state.value.isLoading).isFalse()
    }

    @Test
    fun `Select action updates selected state`() = runTest {
        val foo = Foo("1")
        val vm = newViewModel(repo = FakeFooRepository(foos = listOf(foo)))
        advanceUntilIdle()

        vm.onAction(FooAction.Select(foo))

        assertThat(vm.state.value.selected).isEqualTo(foo)
    }

    @Test
    fun `NavigateToDetail emits a one-shot navigation event`() = runTest {
        val vm = newViewModel()
        advanceUntilIdle()

        vm.navigationEvent.test {
            vm.onAction(FooAction.NavigateToDetail("foo-1"))
            assertThat(awaitItem()).isEqualTo("foo-1")
            cancelAndConsumeRemainingEvents()
        }
    }

    @Test
    fun `state emits loading then loaded when observing repo`() = runTest {
        val repo = FakeFooRepository(foos = listOf(Foo("1")))

        newViewModel(repo = repo).state.test {
            assertThat(awaitItem().isLoading).isTrue()          // initial
            advanceUntilIdle()
            val settled = expectMostRecentItem()
            assertThat(settled.isLoading).isFalse()
            assertThat(settled.foos).hasSize(1)
            cancelAndConsumeRemainingEvents()
        }
    }
}
```

## Common mistakes

- **Constructing a new `UnconfinedTestDispatcher()` inside the test** → the SUT and the test now use different schedulers; `advanceUntilIdle()` won't drive the SUT. Always reuse `mainDispatcherRule.testDispatcher`.
- **`Thread.sleep(500)` to wait for a coroutine** → flaky on CI, doesn't advance the virtual clock. Use `advanceUntilIdle()` or `advanceTimeBy(ms)`.
- **Mocking `FooRepository` interface** → a mock lies about the contract. Write a `FakeFooRepository` that implements the real interface with in-memory state.
- **Reusing a single ViewModel instance across tests as a class field** → cross-test bleed. Always construct via `newViewModel(...)` inside the test.
- **Asserting `SharedFlow` with `state.value`** → `SharedFlow` has no `.value`. Use Turbine (`event.test { assertThat(awaitItem())... }`).
- **Calling `runBlocking` instead of `runTest`** → skips the test-dispatcher machinery, hits real time. Always `runTest`.
- **Forgetting `advanceUntilIdle()` after `onAction`** → assertion runs before the launched coroutine mutates state; test fails intermittently.
- **Asserting on `_state` (private backing field)** via reflection → fragile and tells you nothing the public `state` doesn't. Assert on `state.value` or via Turbine.
- **One giant test asserting five unrelated things** → when it fails, you don't know which behavior broke. One test = one behavior.
- **Silent `advanceUntilIdle()` inside a `state.test { }` block** — Turbine already collects; you rarely need to advance again unless the SUT is doing timed work.

## When to move a test to `:domain`

If the assertion is about **rules** (discount math, coupon eligibility, cart total) and the ViewModel is just plumbing, extract to a use case in the domain module and test the use case with pure JUnit — no dispatcher, no Turbine, no fakes. Faster to write, faster to run, catches the real bug.

## Related skills

- `[[wnb-viewmodel-udf]]` — the ViewModel shape this test targets.
- `[[wnb-compose-ui-test]]` — the UI-side test that verifies the state renders correctly.
