# How to write a skill

Copy this file to `skills/wnb-<your-name>/SKILL.md` and fill it in.

Every skill in this repo follows the same shape. Do not deviate — the shape is what makes them predictable and useful.

---

## The template

```markdown
---
name: wnb-<kebab-case-name>
description: Use this skill when <TASK>. Enforces <THE RULE / PATTERN>. Triggers on "<phrase user types>", "<other phrase>", "<class name>", "<API name>".
---

# <Skill title>

<One-paragraph statement of the pattern this skill enforces and why.>

## Non-negotiables

1. **Rule.** Short imperative. Reason if not obvious.
2. **Rule.** …
3. **Rule.** …

## Canonical example

<A complete, compilable Kotlin snippet showing the pattern. Not a fragment.>

```kotlin
// full code here
```

## Common mistakes

- **`badPattern()`** → use `goodPattern()`. Why: <one sentence>.
- **Doing X in Composable** → move to ViewModel. Why: <one sentence>.

## Related skills

- `[[wnb-other-skill]]` — <when to also load it>
```

---

## Writing rules

### The `description` field is the whole game

Claude only loads a skill when the description matches the current task. Write it with **the exact words users type** — including class names, API names, error messages, and common phrasings.

**Bad:**
> Helps write ViewModels the right way.

**Good:**
> Use this skill when writing or reviewing a ViewModel. Enforces `StateFlow<XxxState>`, sealed `XxxAction`, single `onAction()` entry point. Triggers on "new viewmodel", "add viewmodel", "StateFlow", "onAction", "sealed action", "viewmodel test", "MainDispatcherRule".

Aim for 3–5 sentences. Include:
- The trigger situation ("when writing a ViewModel")
- The rule enforced ("single StateFlow, sealed Action")
- Concrete API names Claude/users will search for
- A trailing sentence starting with `Triggers on "..."`, packed with quoted phrases

### Non-negotiables must be numbered and short

Not paragraphs. Numbered rules. If a rule needs a "why," give one sentence.

### The canonical example must compile

Not a snippet with `...` in the middle. A full, working example — even if the surrounding types are illustrative. Someone should be able to paste it, rename `Foo`, and have a starting point.

### Common mistakes must be real

Don't invent anti-patterns. Every entry should be something you actually caught yourself or Claude doing wrong in real code. If you can't think of any, the skill isn't ready.

### One skill = one pattern

If you're tempted to add a second unrelated pattern, that's a second skill. Skills are cheap. Compose them via `[[related-skill]]` links.

### Keep it under ~250 lines

If the body is longer, something's wrong — either the pattern is too broad (split it), or you're teaching Android instead of encoding a decision. Skills are not tutorials.

---

## Extraction workflow

The skills in this repo are extracted from real production code, not invented. To add a new one:

1. Pick a pattern from your codebase you write repeatedly.
2. Grep all instances (`find . -name "*ViewModel.kt"` etc.).
3. Note what's consistent across files and what varies.
4. Draft the skill using this template.
5. **Sanitize** — replace project names, package paths, domain classes with generic `Foo`/`Bar`.
6. Verify the canonical example compiles in a scratch Android project (or is obviously syntactically valid).
7. Open a PR.

---

## Naming

- `wnb-` prefix on every skill (project brand — see repo README).
- Kebab-case.
- Structure: `wnb-<subject>-<qualifier>` — e.g. `wnb-viewmodel-udf`, `wnb-viewmodel-test`, `wnb-compose-animation`.
- Subject before qualifier so related skills sort together.
