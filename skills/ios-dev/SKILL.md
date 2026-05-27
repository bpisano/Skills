---
name: ios-dev
description: Swift/iOS code style and formatting conventions — one type per file, explicit types everywhere, strict member declaration ordering, .init() syntax, full Swift concurrency with no Combine, Sendable models, English code with French user-facing strings. Use whenever writing, editing, or reviewing Swift code in any iOS or macOS project.
license: MIT
---

# iOS Code Style

Personal Swift conventions applied to every iOS/macOS project, regardless of architecture. For project structure (Coordinator → ViewModel → Store → View, file tree, scaffolding) use the **ios-project** skill instead.

Apply these on every Swift file you write or edit. They are not suggestions.

## 1. One type per file

Exactly one `class` / `struct` / `enum` / `protocol` per file. The filename matches the top-level type (`POIStore.swift` holds `POIStore`).

**Only exception:** a `private extension` that nests helper types belonging to the main type may live in the same file.

```swift
// POIMarker.swift
struct POIMarker: View { /* ... */ }

private extension POIMarker {
    enum Layout {                 // nested helper — allowed in the same file
        static let size: CGFloat = 56
    }
}
```

## 2. Explicit types on every declaration

Always annotate the type, even when the compiler could infer it.

```swift
let number: Int = 0
let pois: [POI] = []
let store: POIStore = .init()
```

Prefer `.init()` on the right-hand side when the type is already explicit on the left. Do not repeat the type name.

```swift
let store: POIStore = .init()        // ✅
let store = POIStore()               // ❌ type not explicit
let store: POIStore = POIStore()     // ❌ redundant
```

**Exception — optional binding.** `if let` / `guard let` take no explicit type:

```swift
guard let value else { return }                  // ✅ no type
if let coordinate { center(on: coordinate) }     // ✅ no type
```

## 3. Member declaration order

Members appear in this fixed order. Within each numbered section, order by access: `public` → internal (no keyword) → `private`, and separate those access subgroups with a single blank line.

1. **Static stored properties** — `public static let` · `static let` · `private static let`
2. **Instance stored properties** — `public let`/`var` · `let`/`var` · `private let`/`var`
3. **`init`** then **`deinit`**
4. **Static methods** — `public static func` · `static func` · `private static func`
5. **Instance methods** — `public func` · `func` · `private func`

Annotated example:

```swift
@MainActor
@Observable
final class POIStore {
    // 1 — static stored properties
    static let shared: POIStore = .init()

    private static let cacheLimit: Int = 500

    // 2 — instance stored properties
    private(set) var pois: [POI] = []

    private let repository: any POIRepository

    // 3 — init / deinit
    init(repository: any POIRepository) {
        self.repository = repository
    }

    // 4 — static methods
    static func live() -> POIStore {
        .init(repository: LivePOIRepository())
    }

    // 5 — instance methods
    func load() async throws {
        pois = try await repository.fetchAll()
    }

    private func broadcast() {
        // ...
    }
}
```

## 4. Language conventions

- **Full Swift concurrency. No Combine.** Use `async`/`await`, `AsyncSequence`/`AsyncStream`, and `Observations` / `withObservationTracking` for reactive propagation. Never `CurrentValueSubject`, `PassthroughSubject`, `.sink`, `AnyCancellable`, or completion handlers.
- **`Sendable`** on all domain models. `actor` / `@MainActor` isolation for shared mutable state.
- **English for code** (identifiers, comments, log messages). **French for user-facing strings** — keep them in `Localizable.strings` even when the app is French-only at launch, for forward compatibility.
- **SwiftUI previews** for every reusable component and screen.
