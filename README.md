# claude-android-skills

A collection of [Claude Code](https://claude.com/claude-code) **Skills** for building production Android apps with Kotlin, Jetpack Compose, and modern testing practices.

Each skill teaches Claude a specific pattern — how to write a ViewModel, how to structure a Compose UI test, when to reach for an animation — so it stops drifting and produces code that matches your project's conventions on the first try.

Extracted from real production Android code, then sanitized for reuse.

---

## What's in the box (v0.1)

| Skill | What it does |
|---|---|
| [`wnb-viewmodel-udf`](skills/wnb-viewmodel-udf/SKILL.md) | Enforces the UDF ViewModel contract — single `StateFlow<XxxState>`, sealed `XxxAction`, one `onAction()` entry point, `DispatcherProvider`-based IO. |
| [`wnb-viewmodel-test`](skills/wnb-viewmodel-test/SKILL.md) | The canonical ViewModel unit-test skeleton — `MainDispatcherRule`, `TestDispatcherProvider`, fake repositories, `runTest` + `advanceUntilIdle`, Turbine for `SharedFlow`. |
| [`wnb-compose-ui-test`](skills/wnb-compose-ui-test/SKILL.md) | Semantics-first Compose UI tests — `createComposeRule`, `onNodeWithTag`, no `Thread.sleep`, no hardcoded coordinates. |
| [`wnb-compose-animation`](skills/wnb-compose-animation/SKILL.md) | Decision-aware animation guidance — first asks *should* this UI animate, then picks the right primitive (`animate*AsState`, `AnimatedContent`, `updateTransition`, `Modifier.animate*`). |
| [`wnb-koin-feature-module`](skills/wnb-koin-feature-module/SKILL.md) | Feature-scoped Koin modules — one module per feature, `viewModel { }` bindings, injection over service locators. |

Plus an [`AGENTS.md` starter](templates/AGENTS.md.android-starter) for tool-agnostic Android AI guidance (works with Claude Code, Cursor, Aider, and any AGENTS.md-aware agent).

---

## Install

Add the repo as a git submodule inside any Android project:

```bash
cd my-android-project
git submodule add https://github.com/wenubey/claude-android-skills .claude/skills-shared
git submodule update --init --recursive
```

Claude Code auto-discovers skills under `.claude/skills-shared/skills/*/SKILL.md`. No extra config.

**To pull in updates:**
```bash
git submodule update --remote .claude/skills-shared
```

**To pin a specific version:**
```bash
cd .claude/skills-shared
git checkout v0.1.0
cd ../..
git add .claude/skills-shared && git commit -m "chore: pin skills to v0.1.0"
```

---

## How skills work (30-second version)

A **skill** is a folder with a `SKILL.md` inside. The file's YAML frontmatter tells Claude *when* to load it. The body tells Claude *what* to do.

```yaml
---
name: wnb-viewmodel-udf
description: Use this skill when writing or reviewing a ViewModel. Enforces StateFlow<XxxState>, sealed XxxAction, onAction() entry point. Triggers on "new viewmodel", "StateFlow", "onAction", "sealed action".
---

# Body: rules, canonical example, common mistakes
```

The **description** is the magnet — Claude only loads the skill when its description matches the current task. So descriptions must include the *exact words users type*.

See [`SKILL_TEMPLATE.md`](SKILL_TEMPLATE.md) for the full authoring format.

---

## License

MIT — see [`LICENSE`](LICENSE). Use these skills in any project, personal or commercial. Attribution appreciated but not required.

---

## Author

Built by [@wenubey](https://github.com/wenubey) while shipping a production Android e-commerce app. Follow along on [LinkedIn](https://www.linkedin.com/) for build-in-public updates.
