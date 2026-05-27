# Store reactivity — no Combine

Stores are `@MainActor @Observable`. Reactive propagation to view models use **`Observations`** (Swift 6.2 / iOS 26) or explicit **`AsyncStream`**. Never `CurrentValueSubject`, `PassthroughSubject`, `.sink`, `AnyCancellable`, or `didSet`-driven publishers.

## The store: plain @Observable, no publishers

Store hold normal `var` state. It `@Observable`, so mutate state = observers notified. No manual broadcast.

```swift
// FooStore.swift
@MainActor
@Observable
final class FooStore {
    static let shared: FooStore = .init()

    private(set) var items: [FooItem] = []

    private let repository: any FooRepository

    init(repository: any FooRepository = LiveFooRepository()) {
        self.repository = repository
    }

    @discardableResult
    func load() async throws -> [FooItem] {
        let fetched: [FooItem] = try await repository.fetchAll()
        items = fetched
        return fetched
    }

    func add(_ item: FooItem) async throws {
        try await repository.create(item)
        items.insert(item, at: 0)
    }

    func delete(_ item: FooItem) async throws {
        try await repository.delete(item)
        items.removeAll { $0.id == item.id }
    }
}
```

Per-instance store: drop `static let shared`, take entity in `init`, coordinator pass down with `.environment(store)`.

## Primary: observe with `Observations`

In view model, observe store property as async sequence. Each change to tracked `@Observable` property emits new value. Launch from `.task`-driven intent.

```swift
private func observe() async {
    let updates = Observations { self.fooStore.items }
    for await items in updates {
        self.items.phase = .loaded(items)
    }
}
```

Assign `Observations { ... }` to `let` first — use directly as `for await ... in` source collides with loop trailing closure.

Loop ends when `.task` cancelled (view disappear). No tokens to retain, nothing to cancel manual.

## Alternative: vend an `AsyncStream`

Use only for **events** not modelled as observable state (one-shot signals: "session finished", "sync completed"), or pre-iOS-26 targets. Store own continuations and yield to them.

```swift
@MainActor
@Observable
final class SyncStore {
    static let shared: SyncStore = .init()

    private var continuations: [UUID: AsyncStream<SyncEvent>.Continuation] = [:]

    func events() -> AsyncStream<SyncEvent> {
        AsyncStream { continuation in
            let id: UUID = .init()
            continuations[id] = continuation
            continuation.onTermination = { [weak self] _ in
                Task { @MainActor in self?.continuations[id] = nil }
            }
        }
    }

    private func emit(_ event: SyncEvent) {
        for continuation in continuations.values {
            continuation.yield(event)
        }
    }
}
```

Consume in view model:

```swift
private func observeSync() async {
    for await event in syncStore.events() {
        handle(event)
    }
}
```

## Migrating Combine code

| Combine (remove) | Replacement |
|------------------|-------------|
| `var x { didSet { xPublisher.send(x) } }` + `CurrentValueSubject` | plain `var x` on an `@Observable` store |
| `store.xPublisher.sink { ... }` + `AnyCancellable` | `for await x in Observations { store.x }` |
| `PassthroughSubject` for events | `AsyncStream` vended by the store |
| `.receive(on: DispatchQueue.main)` | `@MainActor` on the store/view model |
| `.dropFirst()` | not needed — start the observation after the initial load |