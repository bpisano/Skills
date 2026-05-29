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

Group by feature/domain, **not by file type**. Coordinators and UI views each get a per-feature folder. **Core groups each domain into its own self-contained folder** — store + models + networking + persistence co-located — keeping only cross-cutting primitives (state, app state, navigation, shared networking client) as standalone folders. Don't split a domain across top-level `Models/`, `Networking/`, `Persistence/` buckets.

```
<App>/
├── <App>App.swift                      # @main → RootCoordinator(viewModel: UIRootCoordinatorViewModel())
├── Coordinators/
│   ├── Root/
│   │   ├── RootCoordinator.swift
│   │   └── ViewModels/{RootCoordinatorViewModel,UIRootCoordinatorViewModel,MockRootCoordinatorViewModel}.swift
│   └── <Feature>/
│       ├── <Feature>Coordinator.swift
│       └── <Feature>Route.swift                  # Route enum (cases + destination) — only if the feature pushes screens
├── Core/
│   ├── AppState/AppState.swift                    # cross-cutting primitives stay standalone
│   ├── Navigation/{Route,Router}.swift            # generic Route protocol + reusable Router
│   ├── State/{DataState,DataStateView}.swift      # local StateKit replacement
│   ├── Networking/{Client,Middleware}.swift       # shared infra used by every domain
│   └── <Domain>/                                  # one folder per domain/service, self-contained
│       ├── <Domain>Store.swift
│       ├── <Domain>.swift                         # domain model(s)
│       ├── Persistence/                           # this domain's persistence
│       ├── Requests/                              # this domain's network requests
│       └── Wire/                                  # this domain's DTOs
├── UI/
│   ├── Views/<Feature>/
│   │   ├── <Feature>View.swift
│   │   ├── ViewModels/{<Feature>ViewModel,UI<Feature>ViewModel,Mock<Feature>ViewModel}.swift
│   │   └── Components/
│   ├── Components/  ButtonStyles/  ViewModifiers/
├── Resources/                                     # assets, localization, plists — kept out of source folders
│   ├── Assets.xcassets
│   └── Localizable.xcstrings
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
| Route | `FooRoute: Route` enum (owns `destination(for:)`) | `ProfileRoute` |
| Router | generic `Router` (`@MainActor @Observable`, `NavigationPath`) | `Router` |
| View | `FooView<ViewModel: FooViewModel>` | `MapView` |