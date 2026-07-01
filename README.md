# Android Claude Plugin

A [Claude Code](https://claude.ai/code) plugin with skills and commands for Android development following [Google's official architecture guidelines](https://developer.android.com/topic/architecture).

## Installation

```bash
claude plugin install https://github.com/pcha/android-claude-plugin
```

## Skills

Skills are invoked automatically by Claude based on context, or manually with `/skill-name`.

| Skill | Triggers automatically when... |
|---|---|
| `android-architecture-principles` | Discussing architecture design, separation of concerns, UDF, or SSOT |
| `android-layers` | Deciding where code belongs across UI / Domain / Data layers |
| `android-ui-layer` | Building UI with Jetpack Compose, collecting StateFlow, handling lifecycle |
| `android-viewmodel` | Creating ViewModels, exposing state with StateFlow, using `stateIn()` |
| `android-data-layer` | Implementing repositories, data sources, Room, offline-first patterns |
| `android-di-testing` | Setting up Hilt DI, writing ViewModel/repository tests, fakes vs mocks |

## Commands

Commands are invoked explicitly with `/command-name`.

| Command | Description |
|---|---|
| `/new-android` | Scaffold a new Android project from the official [architecture-templates](https://github.com/android/architecture-templates), with optional GitHub repo and release workflow |
| `/android-play-store-setup` | Guided setup for Play Store publishing secrets (keystore + Google Play service account) |

## Versioning

Releases follow [Conventional Commits](https://www.conventionalcommits.org/):

- `fix:` → patch bump
- `feat:` → minor bump
- `BREAKING CHANGE` → major bump

See [CHANGELOG.md](./CHANGELOG.md) for the full release history.
