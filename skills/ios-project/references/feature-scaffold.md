# Scaffold a feature

Min file set add screen named `Foo`. Replace `Foo`, adapt state/intents. Make only files feature need (Store and Router optional).

```
Coordinators/Foo/
├── FooCoordinator.swift            # only if the feature owns navigation
└── FooRoute.swift                  # Route enum: cases + destination(for:) — only if it pushes screens
UI/Views/Foo/
├── FooView.swift
└── ViewModels/
    ├── FooViewModel.swift          # protocol
    ├── UIFooViewModel.swift        # @Observable impl
    └── MockFooViewModel.swift      # mock + static func mock
Core/Foo/                           # domain folder — only if the feature needs its own state/business logic
├── FooStore.swift
├── Foo.swift                       # domain model(s)
└── {Persistence,Requests,Wire}/    # co-locate this domain's persistence, network requests, DTOs
```

## ViewModel protocol

```swift
// FooViewModel.swift
@MainActor
protocol FooViewModel {
    var items: DataState<[FooItem]> { get }
    var searchText: String { get set }
    var errorAlert: Error? { get set }

    func load() async
    func refresh() async
}
```

## ViewModel implementation

Depend on store, map results into `DataState`, observe store reactively (see `store-reactivity.md` — no Combine).

```swift
// UIFooViewModel.swift
@Observable
@MainActor
final class UIFooViewModel: FooViewModel {
    let items: DataState<[FooItem]> = .loading
    var searchText: String = ""
    var errorAlert: Error?

    private let fooStore: FooStore = .shared

    func load() async {
        items.phase = .loading
        do {
            let fetched: [FooItem] = try await fooStore.load()
            items.phase = .loaded(fetched)
        } catch {
            items.phase = .error(error)
        }
        await observe()
    }

    func refresh() async {
        do {
            let fetched: [FooItem] = try await fooStore.load()
            items.phase = .loaded(fetched)
        } catch {
            errorAlert = error
        }
    }

    private func observe() async {
        let updates = Observations { self.fooStore.items }
        for await items in updates {
            self.items.phase = .loaded(items)
        }
    }
}
```

## ViewModel mock

```swift
// MockFooViewModel.swift
@Observable
@MainActor
final class MockFooViewModel: FooViewModel {
    let items: DataState<[FooItem]>
    var searchText: String = ""
    var errorAlert: Error?

    init(items: DataState<[FooItem]>) {
        self.items = items
    }

    func load() async {}

    func refresh() async {}
}

extension FooViewModel where Self == MockFooViewModel {
    static func mock(items: DataState<[FooItem]> = .loading) -> Self {
        .init(items: items)
    }
}
```

## View

Generic over protocol. UI only. Callbacks bubble up via chained modifiers returning `Self`.

```swift
// FooView.swift
struct FooView<ViewModel: FooViewModel>: View {
    private var onSelectItem: ((FooItem) -> Void)?

    @State private var viewModel: ViewModel

    init(viewModel: ViewModel) {
        _viewModel = .init(wrappedValue: viewModel)
    }

    var body: some View {
        DataStateView(viewModel.items) { items in
            List(items) { item in
                Button {
                    onSelectItem?(item)
                } label: {
                    Text(item.title)
                }
            }
        } loading: {
            ProgressView()
        } failure: { error in
            Text(error.localizedDescription)
        }
        .task {
            await viewModel.load()
        }
        .refreshable {
            await viewModel.refresh()
        }
    }
}

extension FooView {
    func onSelectItem(_ perform: @escaping (FooItem) -> Void) -> Self {
        var view: Self = self
        view.onSelectItem = perform
        return view
    }
}

#Preview("Loading") {
    FooView(viewModel: .mock())
}

#Preview("Loaded") {
    FooView(viewModel: .mock(items: .loaded([.mock(), .mock()])))
}
```

## Route (if the feature pushes screens)

Each feature owns a `Route` enum that maps its cases to destinations. No navigation `switch` in the coordinator — the route is the source of truth. See `architecture.md` for the `Route` protocol + generic `Router`.

```swift
// FooRoute.swift
enum FooRoute: Route {
    case detail(FooItem)

    static func destination(for route: FooRoute) -> some View {
        switch route {
        case let .detail(item):
            FooDetailView(item: item)
        }
    }
}
```

## Coordinator (if the feature owns navigation)

Owns the `Router`, binds it to the stack, registers each route type with `.navigationDestination(for:using:)`. Hands children navigation callbacks that call `router.navigate(to:)`.

```swift
// FooCoordinator.swift
struct FooCoordinator: View {
    @State private var router: Router = .init()

    var body: some View {
        NavigationStack(path: $router.path) {
            foo
                .navigationDestination(for: FooRoute.self, using: router)
        }
    }

    @ViewBuilder
    private var foo: some View {
        let viewModel: UIFooViewModel = .init()
        FooView(viewModel: viewModel)
            .onSelectItem { item in
                router.navigate(to: FooRoute.detail(item))
            }
    }
}

#Preview {
    FooCoordinator()
}
```

A descendant already inside the stack reads the router from the environment instead of taking a callback:

```swift
struct FooDetailView: View {
    let item: FooItem

    @Environment(Router.self) private var router

    var body: some View {
        Button("Voir plus") {
            router.navigate(to: FooRoute.related(item))
        }
    }
}
```

Store: see `store-reactivity.md`.