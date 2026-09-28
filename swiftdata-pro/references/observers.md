# Sectioned queries and store observation

**These APIs arrived in iOS 27 and the aligned 27 releases.** They only apply if the project targets those.


## Sectioned queries

`@Query` can section its results itself, which removes the usual pass that groups an array into a dictionary in the view body — work that ran on every `body` evaluation.

```swift
@Query(sort: \Trip.startDate, sectionBy: \.destination)
private var trips: [Trip]
```

The sections are reached through the underscore-prefixed property, which is the query itself:

```swift
ForEach(_trips.sections) { section in
    Section(section.id) {
        ForEach(section) { trip in
            TripListItem(trip: trip)
        }
    }
}
```

- The key path must lead to a `String`; `section.id` is that value.
- The wrapped value `trips` is still a flat `[Trip]`, so a count, a "no data" check, or any existing code reading it keeps working — adding `sectionBy:` to an existing query is not a breaking change at the call site.
- Flag `Dictionary(grouping:)` over a `@Query` result in a view, and any second query that exists only to produce section headers.


## Observing results outside a view: `ResultsObserver`

`@Query` only works inside a SwiftUI view. `ResultsObserver` is the supported way to hold a live, filtered, sorted, optionally sectioned set of results anywhere else — in an `@Observable` model, a controller, an actor — with the same primitives as `@Query`.

```swift
@Observable
@MainActor
final class MapCameraController {
    private let observer: ResultsObserver<Trip, Never>
    private var token: ObservationTracking.Token?

    init(modelContext: ModelContext) throws {
        observer = try ResultsObserver<Trip, Never>(modelContext: modelContext)
        token = withContinuousObservation(options: [.didSet]) { [weak self] in
            guard let self else { return }
            bounds = calculateBounds(trips: observer.results)
        }
    }
}
```

- The second generic parameter is the section identifier; use `Never`, or leave it off, when not sectioning.
- `results` is the live array, and the observer keeps watching the store for changes that affect it.
- The `ObservationTracking.Token` must be retained for as long as the observer is meant to keep firing. A discarded token is the commonest mistake here, and it fails silently — the data simply stops updating.
- This replaces the old pattern of a `FetchDescriptor` re-run on a timer, on `NotificationCenter` traffic, or after every write. Flag those.


## Observing the store's history: `HistoryObserver`

For work that has to react to *transactions* rather than to a set of results — pushing local changes to a server, reacting to writes made by an app extension or a widget — `HistoryObserver` reports when new persistent-history transactions exist.

```swift
observer = try HistoryObserver(authors: ["App"], modelContainer: modelContainer)
token = withContinuousObservation(options: .didSet) { [weak self] in
    _ = self?.observer.eventCounter
    self?.processChanges()
}
```

- `authors` filters by transaction author, which is how you avoid reacting to your own sync writes and looping. Filtering by model type is also possible.
- The only thing it publishes is `eventCounter`, an `Int` that increments when there is something new. Read the transactions themselves with `ModelContext.fetchHistory()`; the observer is a doorbell, not a payload.
- The `eventCounter` read inside the closure is not decorative — Observation only tracks what the closure actually touches, so dropping that line registers no dependency and the callback never fires again. Flag an observation closure that reads nothing observable.
- The default store supports history. A custom `DataStore` only appears here if it implements the history protocols itself.
