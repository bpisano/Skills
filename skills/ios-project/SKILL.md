---
name: ios-project
description: Bootstrap and scaffold iOS apps using the Coordinator → ViewModel → Store → View architecture (file tree, RootCoordinator, AppState, Router, stores, DataState) with full Swift concurrency and no Combine. Use when creating a new iOS project skeleton, adding a new screen/feature, or organizing files into this architecture. For pure code style (formatting, types, ordering) use the ios-dev skill.
license: MIT
---

# iOS Project Architecture

Sets up and extends iOS apps with a strict four-layer architecture:

```
Coordinator (View struct: navigation + composition + child injection)
   → ViewModel (protocol FooViewModel + @Observable UIFooViewModel + MockFooViewModel; view is generic over the protocol)
      → Store (@MainActor @Observable; state + business logic; shared singleton or per-instance)
         → View (generic over its ViewModel; UI only; talks up via .onX callback modifiers)
```

**Always combine this with the `ios-dev` skill** for code style (one type per file, explicit types, member ordering, `.init()`).

**No Combine, anywhere.** Reactive store → view-model propagation uses `Observations` / `AsyncStream`. See `references/store-reactivity.md`.

## Decide what to do

- **New project / empty app target** → bootstrap the skeleton. Read `references/bootstrap.md`.
- **Add a screen or feature** → scaffold the feature file set. Read `references/feature-scaffold.md`.
- **Reactive state from a store** (loading data, observing changes) → read `references/store-reactivity.md`.
- **Unsure how the layers relate / naming** → read `references/architecture.md`.

Read the relevant reference file before generating code — they hold the full templates. Do not improvise the patterns from memory.

## File tree

Features are grouped by layer, then by feature name.

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

If the project uses a synchronized root group in Xcode (files on disk auto-join the target), just create the files on disk. Otherwise add them to the target.

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
