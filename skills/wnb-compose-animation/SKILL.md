---
name: wnb-compose-animation
description: Use this skill when a Jetpack Compose UI is about to gain an animation, or when reviewing existing motion. Enforces the decision-aware gate — first ask "should this UI animate at all?"; only if yes, then pick the smallest API (AnimatedVisibility, animate*AsState, rememberTransition, AnimatedContent, Animatable). Requires every animation to earn its frame budget, carry a label, respect reduce-motion accessibility, and stay off scroll-adjacent recomposition paths. Triggers on "add animation", "animate", "fade in", "make it smooth", "smooth transition", "fancy transition", "AnimatedVisibility", "animate*AsState", "AnimatedContent", "rememberTransition", "Animatable", "Crossfade", "animateContentSize", "should this animate", "reduce motion".
---

# Jetpack Compose — decision-aware animation

Most animation bugs in Compose are not "wrong API" bugs — they are **"did not need to animate in the first place"** bugs. This skill's job is to make the first decision explicit before any code is written. Only after the "should we animate?" gate passes does API selection matter.

For deep API selection (which primitive to reach for), this skill is intentionally thin — see `[[compose-animations]]` (chrisbanes/skills) linked at the end. This skill focuses on the **gate**.

## Non-negotiables

1. **Every animation earns its frame budget.** If a reviewer cannot answer "what does this animation communicate?" in one sentence, remove it.
2. **The gate runs first, always.** Before typing `AnimatedVisibility` or `animate*AsState`, answer the three "should we animate?" questions below. If any answer is no, no motion.
3. **Every animation gets a `label`.** `animate*AsState(label = "fabWidth")`, `rememberTransition(label = "phase")`, `AnimatedContent(label = "profile-content")`. Enables the Compose Animation preview and tooling — no exceptions.
4. **Respect reduce-motion.** On Android, wrap animation opt-in on `LocalAccessibilityManager.current?.getRecommendedTimeoutMillis(...)` or the platform's `Settings.Global.TRANSITION_ANIMATION_SCALE`. If motion is disabled, jump to the end state directly.
5. **No animated value on the recomposition-hot path.** Reading a frame-updating `State` in a composable body causes recomposition every frame. For `Modifier.offset`, `Modifier.graphicsLayer`, and `Modifier.background`, use the block-lambda form and read the animated value inside — see `[[wnb-compose-ui-test]]` and Chris Banes' `compose-state-deferred-reads`.
6. **Do not fight Navigation Compose transitions.** If the animation is between destinations, use Nav's built-in transition APIs, not `AnimatedContent` layered on top.
7. **Test animated UI with `mainClock.advanceTimeBy(ms)`**, never `Thread.sleep`. Compose has virtual time in tests. See `[[wnb-compose-ui-test]]`.

## The gate — should this UI animate?

Answer all three. If any is **no**, do not animate.

### 1. Does the change carry meaning that motion clarifies?

Motion should encode one of:
- **Continuity** — the outgoing element represents the same concept as the incoming (loading → content, expand → collapse). The user should not think "a new thing appeared."
- **Hierarchy / spatial cues** — a container growing to reveal children, a sheet sliding in from an edge that implies "this is on top of what was there."
- **Feedback** — a tap ripple, a button press depression. The animation confirms the input landed.

If the change is none of those (a static label swap, an initial page render, an incoming push notification), **do not animate**.

### 2. Would a still-image transition feel broken?

Cover the UI transition with your hand and swap the before/after state instantly. If it feels fine — leave it instant. Animation is a fix for a specific perceived discontinuity, not a default polish layer.

### 3. Will the motion still work under adversity?

- **Reduce-motion enabled?** Skip the animation gracefully — don't render a frozen half-frame.
- **Slow device?** A 300ms `spring` on a `Modifier.offset` inside a scrolling list will jank. Move it off the hot path or drop it.
- **Repeated interaction?** If the user can trigger the same animation 5×/sec (rapid toggle), does the animation interrupt cleanly? Use `Animatable` for interruptible motion; `animate*AsState` handles simple retargeting but not multi-gesture handoff.

## If the gate passes: pick the smallest API

Quick-pick for the 80% case. For the full API selection tree, see `[[compose-animations]]` (chrisbanes/skills).

| Situation | API |
|---|---|
| Show/hide a subtree, leave composition after exit | `AnimatedVisibility` |
| One value glides to a new target from state | `animate*AsState` (`animateFloatAsState`, `animateDpAsState`, `animateColorAsState`, …) |
| Several values driven by one state, kept in sync | `rememberTransition` + `transition.animate*` |
| Swap between different composable trees in the same slot | `AnimatedContent` (add `contentKey` if state is a wrapper like `UiState`) |
| Gesture-driven or interruptible motion | `Animatable` |
| Layout size change (text wrap, chip expand) | `Modifier.animateContentSize()` |

## Common mistakes

- **Adding an animation because the screen "felt static"** without answering the gate — animation as decoration is drag. Ship it plain; add motion only if a user report says the transition is confusing.
- **Animating the initial state on first composition** — the first render should not fade in. That's not a state transition; that's an appearance. If the loading indicator is followed by content, the transition point is loading→content, not nothing→loading.
- **`animate*AsState` on alpha and expecting the child to stop laying out** — the child stays in composition and layout. Use `AnimatedVisibility` for enter/exit semantics.
- **Three parallel `animateDpAsState` calls that must stay in sync** — one `rememberTransition` binds them to a single state. Parallel calls drift.
- **`AnimatedContent(targetState = uiState)` where `uiState` is a sealed class with a `data` payload** — every payload change triggers an animation. Add `contentKey` mapped to the visual shape (`"loading"`, `"content"`, `"error"`).
- **Ignoring reduce-motion** — the accessibility setting exists. Users have vestibular sensitivity. Test with the system setting toggled.
- **Adding a `Crossfade` between destinations Navigation is already animating** — you get a double transition. Delete the `Crossfade`, keep Nav's transition.
- **Missing `label`** — the animation shows up as "Unlabelled" in the Compose Animation preview. Debugging suffers.

## Testing animated UI (one-liner)

Use `composeTestRule.mainClock.advanceTimeBy(500)` after the trigger, then assert on the final visual state. Do NOT try to intercept frames mid-flight — assert on the settled state. See `[[wnb-compose-ui-test]]` for the full test skeleton.

If the SUT uses `AnimatedContent`, remember: after `advanceTimeBy`, the enter/exit content may both be in the tree briefly — filter your finder with `hasParent(...)` or a testTag on the target content.

## Related skills

- `[[wnb-compose-ui-test]]` — how to test animated screens without flake.
- `[[wnb-viewmodel-udf]]` — the state-driven contract animations render against.
- **External:** [`chrisbanes/skills` → `compose-animations`](https://github.com/chrisbanes/skills/blob/main/skills/compose-animations/SKILL.md) — deep API selection (`AnimatedContent.contentKey`, `SeekableTransitionState`, `Animatable` gestures, performance-hot-path handling). If the gate passes and you need the full API tree, load it.

## Attribution

The API-selection philosophy ("pick the smallest API that matches the problem") is the framing from Chris Banes' `compose-animations` skill. This skill deliberately keeps the API section thin and refers to Chris's work for depth; the value added here is the upstream **decision gate** — "should this UI animate at all?" — which the referenced work does not enforce.
