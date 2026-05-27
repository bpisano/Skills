# Architecture reference

Four layers, strict separation. Each layer only talks to the one directly below it.

## Coordinator

A `View` struct that owns navigation and composes children. It creates view models / stores and injects them. It contains **no UI of its own** beyond wiring.

- **Root coordinator**: generic over its view model, switches on an `AppState` enum to pick the active flow.
- **Feature coordinator**: hosts a `NavigationStack(path:)` driven by a `Router`, declares `navigationDestination`s, presents sheets, and builds child views — handing each child its callbacks (`.onSelect`, `.onFinish`) and translating them into navigation.

## ViewModel

Three files per screen:

1. `protocol FooViewModel` — `@MainActor`. Declares the state the view reads and the intents it can call.
2. `final class UIFooViewModel: FooViewModel` — `@Observable @MainActor`. The real implementation; depends on stores/repositories.
3. `final class MockFooViewModel: FooViewModel` — `@Observable`. Static state for previews, plus `static func mock(...)` exposed through an extension on the protocol so previews write `.mock(...)`.

The **View is generic over the protocol**, so previews inject the mock and production injects the `UI` implementation. No business logic lives in the view model beyond orchestrating store calls and mapping results into view state (`DataState`).

## Store

`@MainActor @Observable final class FooStore`. Holds state and business logic; the only layer that touches repositories / the network / persistence.

- **Shared store**: `static let shared` singleton (e.g. app context, a global POI cache).
- **Per-instance store**: created by a coordinator for a specific entity and passed down with `.environment(...)`.

Reactive propagation to view models uses `Observations` / `AsyncStream` — **never Combine**. See `store-reactivity.md`.

## View

Generic over its `ViewModel`. UI only — no navigation, no business logic. It reads `viewModel` state and calls `viewModel` intents. It communicates upward to its coordinator through chained callback modifiers that return `Self`:

```swift
MapView(viewModel: viewModel)
    .onSelectPOI { poi in router.navigate(to: .poiDetail(poi)) }
```

## Router

`@MainActor @Observable final class Router` holding a `NavigationPath` (or a typed `[Route]` for simple feature routers) and a nested `enum Route: Hashable`. Exposes `navigate(to:)`. Injected into the navigation subtree via `.environment(...)` and read with `@Environment(Router.self)`.

## Wiring conventions

- View model injected via `init(viewModel: @autoclosure @escaping () -> ViewModel)` stored in `@State private var viewModel`.
- Stores / routers read with `@Environment(FooStore.self)`.
- One `#Preview` per view, using `.mock(...)`.
- `Sendable` on domain models; explicit types + `.init()` everywhere (see the `ios-dev` skill).
