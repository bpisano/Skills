# Scaffold a feature

Minimum file set to add a screen named `Foo`. Replace `Foo` and adapt state/intents. Create only the files the feature actually needs (Store and Router are optional).

```
Coordinators/Foo/FooCoordinator.swift           # only if the feature owns navigation
UI/Views/Foo/
├── FooView.swift
└── ViewModels/
    ├── FooViewModel.swift          # protocol
    ├── UIFooViewModel.swift        # @Observable impl
    └── MockFooViewModel.swift      # mock + static func mock
Core/Stores/Foo/FooStore.swift      # only if it needs its own state/business logic
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

Depends on a store, maps results into `DataState`, observes the store reactively (see `store-reactivity.md` — no Combine).

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

Generic over the protocol. UI only. Callbacks bubble up via chained modifiers returning `Self`.

```swift
// FooView.swift
struct FooView<ViewModel: FooViewModel>: View {
    private var onSelectItem: ((FooItem) -> Void)?

    @State private var viewModel: ViewModel

    init(viewModel: @autoclosure @escaping () -> ViewModel) {
        _viewModel = .init(wrappedValue: viewModel())
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

## Coordinator (if the feature owns navigation)

```swift
// FooCoordinator.swift
struct FooCoordinator: View {
    @State private var router: Router = .init()

    var body: some View {
        NavigationStack(path: $router.path) {
            foo
                .navigationDestination(for: Router.Route.self) { route in
                    Group {
                        switch route {
                        case let .detail(item):
                            detail(for: item)
                        }
                    }
                    .environment(router)
                }
        }
    }

    @ViewBuilder
    private var foo: some View {
        let viewModel: UIFooViewModel = .init()
        FooView(viewModel: viewModel)
            .onSelectItem { item in
                router.navigate(to: .detail(item))
            }
    }

    @ViewBuilder
    private func detail(for item: FooItem) -> some View {
        // child view
        Text(item.title)
    }
}

#Preview {
    FooCoordinator()
}
```

For the store, see `store-reactivity.md`.
