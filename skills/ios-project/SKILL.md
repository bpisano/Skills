---
name: ios-project
description: Bootstrap and scaffold iOS apps with Coordinator → ViewModel → Store → View architecture (file tree, RootCoordinator, AppState, Router, stores, DataState). Full Swift concurrency, no Combine. Use when creating new iOS project skeleton, adding new screen/feature, or organizing files into this architecture. Pure code style (formatting, types, ordering) → use ios-dev skill.
license: MIT
---

# iOS Project Architecture

Set up and extend iOS apps with strict four-layer architecture:

```
Coordinator (View struct: navigation + composition + child injection)
   → ViewModel (protocol FooViewModel + @Observable UIFooViewModel + MockFooViewModel; view is generic over the protocol)
      → Store (@MainActor @Observable; state + business logic; shared singleton or per-instance)
         → View (generic over its ViewModel; UI only; talks up via .onX callback modifiers)
```

**Always combine with `ios-dev` skill** for code style (one type per file, explicit types, member ordering, `.init()`).

**No Combine, anywhere.** Reactive store → view-model propagation use `Observations` / `AsyncStream`. See `references/store-reactivity.md`.

## Decide what to do

- **New project / empty app target** → bootstrap skeleton. Read `references/bootstrap.md`.
- **Add a screen or feature** → scaffold feature file set. Read `references/feature-scaffold.md`.
- **Reactive state from a store** (loading data, observing changes) → read `references/store-reactivity.md`.
- **Unsure how layers relate / naming** → read `references/architecture.md`.

Read relevant reference file before generating code — they hold full templates. Do not improvise patterns from memory.

## File tree

Features grouped by layer, then by feature name.

```
<App>/
├── <App>App.swift                      # @main → RootCoordinator(viewModel: UIRootCoordinatorViewModel())
├── Coordinators/
│   ├── Root/
│   │   ├── RootCoordinator.swift
│   │   └── ViewModels/{RootCoordinatorViewModel,UIRootCoordinatorViewModel,MockRootCoordinatorViewModel}.swift
│   └── <Feature>/
│       ├── <Feature>Coordinator.swift
│       └── Router/<Feature>Router.swift          # optional, only if the feature has internal navigation
├── Core/
│   ├── AppState/AppState.swift
│   ├── Navigation/Router.swift
│   ├── State/{DataState,DataStateView}.swift     # local StateKit replacement
│   └── Stores/<Domain>/<Domain>Store.swift
├── UI/
│   ├── Views/<Feature>/
│   │   ├── <Feature>View.swift
│   │   ├── ViewModels/{<Feature>ViewModel,UI<Feature>ViewModel,Mock<Feature>ViewModel}.swift
│   │   └── Components/
│   ├── Components/  ButtonStyles/  ViewModifiers/
└── Utils/Extensions/
```

If project use synchronized root group in Xcode (files on disk auto-join target), create files on disk. Otherwise add to target.

## Naming (non-negotiable)

| Layer | Pattern | Example |
|-------|---------|---------|
| Coordinator | `FooCoordinator` | `MapCoordinator` |
| ViewModel protocol | `FooViewModel` (`@MainActor`) | `MapViewModel` |
| ViewModel impl | `UIFooViewModel` (`@Observable`) | `UIMapViewModel` |
| ViewModel mock | `MockFooViewModel` + `static func mock` | `MockMapViewModel` |
| Store | `FooStore` (`@MainActor @Observable`) | `POIStore` |
| Router | `Router` / `FooRouter` + nested `enum Route` | `Router` |
| View | `FooView<ViewModel: FooViewModel>` | `MapView` |