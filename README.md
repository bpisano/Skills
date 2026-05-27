# dev-ios

Personal Claude Code plugin bundling iOS/Swift development conventions as two skills.

| Skill | Triggers | Does |
|-------|----------|------|
| **ios-dev** | Writing/editing/reviewing any Swift code | Code style: one type per file, explicit types, member ordering, `.init()`, no Combine, Sendable, English-code/French-strings. |
| **ios-project** | Setting up a project or adding a screen | Coordinator → ViewModel → Store → View architecture. Bootstraps a new project skeleton and scaffolds per-feature file sets. |

## Install

```
/plugin marketplace add https://github.com/bpisano/skills.git
/plugin install dev-ios@dev-ios
```
