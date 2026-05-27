---
name: ios-dev
description: Swift/iOS code style and formatting conventions — one type per file, explicit types everywhere, strict member declaration ordering, .init() syntax, full Swift concurrency with no Combine, Sendable models, English code with French user-facing strings. Use whenever writing, editing, or reviewing Swift code in any iOS or macOS project.
license: MIT
---

# iOS Code Style

Personal Swift conventions. Every iOS/macOS project, any architecture. Project structure (Coordinator → ViewModel → Store → View, file tree, scaffolding) → use **ios-project** skill.

Apply every Swift file you write or edit. Not suggestions.

## 1. One type per file

Exactly one `class` / `struct` / `enum` / `protocol` per file. Filename match top-level type (`POIStore.swift` hold `POIStore`).

**Only exception:** `private extension` nesting helper types of main type may share file.

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

Always annotate type, even when compiler infer.

```swift
let number: Int = 0
let pois: [POI] = []
let store: POIStore = .init()
```

Prefer `.init()` on right side when type explicit on left. No repeat type name.

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

Members in fixed order. Within each numbered section, order by access: `public` → internal (no keyword) → `private`, separate access subgroups with single blank line.

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

- **Full Swift concurrency. No Combine.** Use `async`/`await`, `AsyncSequence`/`AsyncStream`, `Observations` / `withObservationTracking` for reactive propagation. Never `CurrentValueSubject`, `PassthroughSubject`, `.sink`, `AnyCancellable`, completion handlers.
- **`Sendable`** on all domain models. `actor` / `@MainActor` isolation for shared mutable state.
- **English for code** (identifiers, comments, log messages). **French for user-facing strings** — keep in `Localizable.strings` even when app French-only at launch, for forward compat.
- **SwiftUI previews** for every reusable component and screen.