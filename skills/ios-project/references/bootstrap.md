# Bootstrap a new project

Create these files to stand up an empty app on the architecture. Order: state primitives → app state → router → root store → root coordinator (+ VM trio) → app entry. Adjust the module/app name; `App` below is a placeholder.

## 1. DataState + DataStateView (`Core/State/`)

Local replacement for an external StateKit. Concurrency-native, `@Observable`.

```swift
// DataState.swift
@Observable
@MainActor
final class DataState<Value> {
    enum Phase {
        case loading
        case loaded(Value)
        case error(Error)
    }

    var phase: Phase

    init(_ phase: Phase = .loading) {
        self.phase = phase
    }
}

extension DataState {
    static var loading: DataState { .init(.loading) }

    static func loaded(_ value: Value) -> DataState { .init(.loaded(value)) }

    static func error(_ error: Error) -> DataState { .init(.error(error)) }
}
```

```swift
// DataStateView.swift
struct DataStateView<Value, Content: View, Loading: View, Failure: View>: View {
    private let state: DataState<Value>
    private let content: (Value) -> Content
    private let loading: () -> Loading
    private let failure: (Error) -> Failure

    init(
        _ state: DataState<Value>,
        @ViewBuilder content: @escaping (Value) -> Content,
        @ViewBuilder loading: @escaping () -> Loading,
        @ViewBuilder failure: @escaping (Error) -> Failure
    ) {
        self.state = state
        self.content = content
        self.loading = loading
        self.failure = failure
    }

    var body: some View {
        switch state.phase {
        case .loading:
            loading()
        case let .loaded(value):
            content(value)
        case let .error(error):
            failure(error)
        }
    }
}
```

## 2. AppState (`Core/AppState/`)

```swift
// AppState.swift
enum AppState: Hashable, Sendable {
    case authenticating
    case app
}
```

## 3. Router (`Core/Navigation/`)

```swift
// Router.swift
@MainActor
@Observable
final class Router {
    var path: NavigationPath = .init()

    func navigate(to route: Route) {
        path.append(route)
    }

    func pop() {
        guard !path.isEmpty else { return }
        path.removeLast()
    }
}

private extension Router {
    // declare the app's routes here as the project grows
}

extension Router {
    enum Route: Hashable {
        // case poiDetail(POI)
    }
}
```

## 4. Root store (`Core/Stores/AppContext/`)

```swift
// AppContextStore.swift
@MainActor
@Observable
final class AppContextStore {
    static let shared: AppContextStore = .init()

    private(set) var isAuthenticated: Bool = false

    private init() {}

    func start() async throws {
        // bootstrap session / environment here
        isAuthenticated = true
    }
}
```

## 5. Root coordinator + VM trio (`Coordinators/Root/`)

```swift
// RootCoordinatorViewModel.swift
@MainActor
protocol RootCoordinatorViewModel {
    var currentState: AppState { get }
    var error: Error? { get set }

    func start() async
}
```

```swift
// UIRootCoordinatorViewModel.swift
@Observable
@MainActor
final class UIRootCoordinatorViewModel: RootCoordinatorViewModel {
    private(set) var currentState: AppState = .authenticating
    var error: Error?

    private let appContextStore: AppContextStore = .shared

    func start() async {
        do {
            try await appContextStore.start()
            currentState = .app
        } catch {
            self.error = error
        }
    }
}
```

```swift
// MockRootCoordinatorViewModel.swift
@Observable
@MainActor
final class MockRootCoordinatorViewModel: RootCoordinatorViewModel {
    private(set) var currentState: AppState
    var error: Error?

    init(currentState: AppState) {
        self.currentState = currentState
    }

    func start() async {}
}

extension RootCoordinatorViewModel where Self == MockRootCoordinatorViewModel {
    static func mock(currentState: AppState = .app) -> Self {
        .init(currentState: currentState)
    }
}
```

```swift
// RootCoordinator.swift
struct RootCoordinator<ViewModel: RootCoordinatorViewModel>: View {
    @State private var viewModel: ViewModel

    init(viewModel: @autoclosure @escaping () -> ViewModel) {
        _viewModel = .init(wrappedValue: viewModel())
    }

    var body: some View {
        ZStack {
            switch viewModel.currentState {
            case .authenticating:
                authenticating
            case .app:
                app
            }
        }
        .task {
            await viewModel.start()
        }
    }

    @ViewBuilder
    private var authenticating: some View {
        ProgressView()
            .transition(.opacity.animation(.snappy))
    }

    @ViewBuilder
    private var app: some View {
        // MapCoordinator()  — first feature coordinator
        Text("App")
            .transition(.opacity.animation(.snappy))
    }
}

#Preview("Authenticating") {
    RootCoordinator(viewModel: .mock(currentState: .authenticating))
}

#Preview("App") {
    RootCoordinator(viewModel: .mock(currentState: .app))
}
```

## 6. App entry (`<App>App.swift`)

```swift
@main
struct App: SwiftUI.App {
    var body: some Scene {
        WindowGroup {
            rootCoordinator
        }
    }

    @ViewBuilder
    private var rootCoordinator: some View {
        let viewModel: UIRootCoordinatorViewModel = .init()
        RootCoordinator(viewModel: viewModel)
    }
}
```

After this, add the first feature with `feature-scaffold.md`.
