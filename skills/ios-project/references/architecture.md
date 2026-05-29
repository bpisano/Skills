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

**Callbacks are for navigation, not data.** A callback hands an event up to the coordinator that owns navigation (`.onSelectPOI`, `.onFinish`). State-mutating actions (delete, toggle, save, refresh) are **view-model intents** — the view calls `viewModel.delete(item)` directly, never a `.onDelete` callback routed to the coordinator. Because view models observe their store, every view bound to that store refreshes automatically; bouncing a mutation through the coordinator just to call the same store is needless indirection.

## Router

Navigation is **type-safe and route-driven**. Two pieces, both in `Core/Navigation/`.

**1. `Route` protocol** — every route enum owns the view it maps to, so destinations never live in a coordinator `switch`:

```swift
protocol Route: Hashable {
    associatedtype Destination: View

    @MainActor
    @ViewBuilder
    static func destination(for route: Self) -> Destination
}
```

**2. Generic `Router`** — one `@Observable` type reused by every flow. Holds a `NavigationPath`, pushes any `Hashable` route:

```swift
@MainActor
@Observable
final class Router {
    var path: NavigationPath = .init()

    func navigate(to route: some Hashable) {
        path.append(route)
    }

    func pop() {
        guard !path.isEmpty else { return }
        path.removeLast()
    }
}
```

A `View` helper binds a route type to its protocol-provided destination and forwards the router down:

```swift
extension View {
    func navigationDestination<R: Route>(for route: R.Type, using router: Router) -> some View {
        navigationDestination(for: R.self) { route in
            R.destination(for: route)
                .environment(router)
        }
    }
}
```

**Per-feature route enums** conform to `Route` and carry type-safe parameters; the same generic `Router` hosts them all on one stack:

```swift
enum ProfileRoute: Route {
    case personalInfo
    case invoice(Invoice.ID)

    static func destination(for route: ProfileRoute) -> some View {
        switch route {
        case .personalInfo:
            PersonalInfoView()
        case let .invoice(id):
            InvoiceView(id: id)
        }
    }
}
```

**Injection.** The coordinator owns the `Router` as `@State`, binds it to `NavigationStack(path:)`, registers each route type with `.navigationDestination(for:using:)`, and injects it via `.environment(router)`. Descendant views and view models reach it with `@Environment(Router.self)` and call `router.navigate(to: ProfileRoute.invoice(id))` — type-checked at compile time. **One `Router` per flow/tab**; switching tabs keeps each stack independent. An isolated modal flow (a sheet with its own steps) gets its own small router with a typed `[Route]` path and its own `destination` switch.

## Wiring conventions

- View model injected via `init(viewModel: ViewModel)` stored in `@State private var viewModel` (`_viewModel = .init(wrappedValue: viewModel)`). No closure/autoclosure — the coordinator already builds the instance, so wrapping it in a closure buys nothing.
- Stores / routers read with `@Environment(FooStore.self)`.
- One `#Preview` per view, using `.mock(...)`.
- `Sendable` on domain models. Explicit types + `.init()` everywhere (see `ios-dev` skill).