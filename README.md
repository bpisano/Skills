# dev-ios

Personal Claude Code plugin bundling iOS/Swift development conventions as two skills.

| Skill | Triggers | Does |
|-------|----------|------|
| **ios-dev** | Writing/editing/reviewing any Swift code | Code style: one type per file, explicit types, member ordering, `.init()`, no Combine, Sendable, English-code/French-strings. |
| **ios-project** | Setting up a project or adding a screen | Coordinator → ViewModel → Store → View architecture. Bootstraps a new project skeleton and scaffolds per-feature file sets. |

## Install

This repo is its own marketplace.

```
/plugin marketplace add ~/Dev/Claude/Plugins/dev-ios
/plugin install dev-ios@dev-ios
```

To install on another machine, push to a git remote and use the git URL:

```
/plugin marketplace add <git-url>
/plugin install dev-ios@dev-ios
```

## Layout

```
dev-ios/
├── .claude-plugin/
│   ├── plugin.json          # plugin manifest (lists the two skills)
│   └── marketplace.json     # marketplace manifest (lists this plugin)
└── skills/
    ├── ios-dev/SKILL.md
    └── ios-project/
        ├── SKILL.md
        └── references/      # architecture notes + code templates, loaded on demand
```

## Updating

Edit the `SKILL.md` files, commit, push. On each machine run `/plugin marketplace update dev-ios`.
