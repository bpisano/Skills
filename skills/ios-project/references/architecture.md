# Architecture reference

Four layers, strict separation. Each layer talk only to one directly below.

## Coordinator

`View` struct. Own navigation, compose children. Create view models / stores, inject them. **No UI of own** beyond wiring.

- **Root coordinator**: generic over view model. Switch on `AppState` enum to pick active flow.
- **Feature coordinator**: host `NavigationStack(path:)` driven by `Router`. Declare `navigationDestination`s, present sheets, build child views — hand each child its callbacks (`.onSelect`, `.onFinish`), translate into navigation.

## ViewModel

Three files per screen:

1. `protocol FooViewModel` — `@MainActor`. Declare state view reads + intents it can call.
2. `final class UIFooViewModel: FooViewModel` — `@Observable @MainActor`. Real impl. Depend on stores/repositories.
3. `final class MockFooViewModel: FooViewModel` — `@Observable`. Static state for previews, plus `static func mock(...)` exposed through protocol extension so previews write `.mock(...)`.

**View generic over protocol** — previews inject mock, production inject `UI` impl. No business logic in view model beyond orchestrating store calls + mapping results into view state (`DataState`).

## Store

`@MainActor @Observable final class FooStore`. Hold state + business logic. Only layer that touch repositories / network / persistence.

- **Shared store**: `static let shared` singleton (e.g. app context, global POI cache).
- **Per-instance store**: created by coordinator for specific entity, passed down with `.environment(...)`.

Reactive propagation to view models use `Observations` / `AsyncStream` — **never Combine**. See `store-reactivity.md`.

## View

Generic over `ViewModel`. UI only — no navigation, no business logic. Read `viewModel` state, call `viewModel` intents. Talk upward to coordinator through chained callback modifiers returning `Self`:

```swift
MapView(viewModel: viewModel)
    .onSelectPOI { poi in router.navigate(to: .poiDetail(poi)) }
```

## Router

`@MainActor @Observable final class Router` holding `NavigationPath` (or typed `[Route]` for simple feature routers) + nested `enum Route: Hashable`. Expose `navigate(to:)`. Inject into navigation subtree via `.environment(...)`, read with `@Environment(Router.self)`.

## Wiring conventions

- View model injected via `init(viewModel: @autoclosure @escaping () -> ViewModel)` stored in `@State private var viewModel`.
- Stores / routers read with `@Environment(FooStore.self)`.
- One `#Preview` per view, using `.mock(...)`.
- `Sendable` on domain models. Explicit types + `.init()` everywhere (see `ios-dev` skill).