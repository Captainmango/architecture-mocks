# Auctionite - Architecture

C4 model of the Auctionite auction platform, drawn with Mermaid.

Technology commitments:

- **Database:** PostgreSQL - system of record (schema per [erd.md](erd.md)).
- **Cache:** Valkey - hot reads (current listing state, leading bid) and per-listing bid coordination.
- **API / Scheduler:** language- and framework-agnostic - any stack that can serve HTTP and talk to PostgreSQL and Valkey.

## System Context (C4 Level 1)

```mermaid
C4Context
    title System Context - Auctionite

    Person(user, "User", "Registered user. Creates items, lists them for auction, and bids on other users' listings.")

    System(auctionite, "Auctionite", "Auction platform. Items are sold through scheduled listings with a Draft / Active / Inactive / Void / Complete lifecycle. Every listing state transition propagates an event.")

    Rel(user, auctionite, "Manages account, items and listings; places bids", "HTTPS")
```

One system, one actor: a registered user both sells and bids. Scheduling and event propagation are internal concerns, shown at the lower levels.

## Containers (C4 Level 2)

```mermaid
C4Container
    title Container diagram - Auctionite

    Person(user, "User", "Registered user: seller and bidder.")

    System_Boundary(auctionite, "Auctionite") {
        Container(api, "API", "Any language / framework", "HTTP API. Accounts, items, listings, bids. Owns the listing state machine, enforces the bid rules, and propagates an event on every listing state transition.")
        Container(scheduler, "Scheduler", "Any language / framework", "Polls listing_schedules for rows whose activation/deactivation time is due and asks the API to perform those transitions.")
        ContainerDb(db, "Database", "PostgreSQL", "System of record: users, items, item_pictures, listings, listing_schedules, listing_state_transitions (doubles as the event log), bids.")
        ContainerDb(cache, "Cache", "Valkey", "Caches current listing state and leading bid. Per-listing coordination point for concurrent bids.")
    }

    Rel(user, api, "Uses", "HTTPS/JSON")
    Rel(api, db, "Reads from and writes to", "SQL")
    Rel(api, cache, "Reads from and writes to", "RESP")
    Rel(scheduler, db, "Polls due schedules", "SQL")
    Rel(scheduler, api, "Triggers due activations / deactivations", "HTTPS")
```

Design notes:

- **All state transitions go through the API.** The Scheduler never writes transitions itself; it only detects due schedules and calls the API, so the state machine and event dispatch live in exactly one place.
- **The transition rows are the events.** Per the ERD, `listing_state_transitions` doubles as the domain event log - no separate event store or broker is committed to.

## Components (C4 Level 3 - inside the API)

```mermaid
C4Component
    title Component diagram - Auctionite API

    Person(user, "User", "Registered user: seller and bidder.")
    Container(scheduler, "Scheduler", "Any language / framework", "Triggers due listing activations / deactivations.")
    ContainerDb(db, "Database", "PostgreSQL", "System of record.")
    ContainerDb(cache, "Cache", "Valkey", "Listing state / leading bid cache and bid coordination.")

    Container_Boundary(api, "API") {
        Component(http, "HTTP Endpoints", "Any web framework", "Routing, authentication, request validation.")
        Component(auth, "Auth and Accounts", "Module", "Account creation and login; password hashing; session / token issuing.")
        Component(items, "Item Catalog", "Module", "Items and item pictures. Editable only while the listing status allows (Draft, Inactive); locked while Active, Void or Complete.")
        Component(listings, "Listing Service", "Module", "Listing CRUD and lifecycle. Appends one row per change to listing_state_transitions; soft-deleting a row retracts it, reverting to the previous state.")
        Component(sm, "Listing State Machine", "Module", "Draft to Active / Inactive / Void; Active to Inactive / Void / Complete; Inactive to Active / Void / Complete; Void and Complete are terminal. Current state = latest undeleted transition row.")
        Component(sched, "Schedule Service", "Module", "Activation / deactivation times for a listing (listing_schedules).")
        Component(bids, "Bid Service", "Module", "Places bids. Rejects bids on non-Active statuses, on the user's own listing, and from the current leading bidder. Serializes concurrent bids on the same listing; different listings never influence each other. Highest amount wins; earliest bid breaks ties.")
        Component(events, "Event Dispatcher", "Module", "Propagates an event on every listing state transition: created, activated, deactivated, deleted, completed.")
        Component(repo, "Repository", "Module", "Data access for all aggregates. Integrity is enforced here in code; the schema defines indexes only.")
        Component(store, "Cache Store", "Module", "Caches current listing state and leading bid; holds the per-listing lock that serializes concurrent bids.")
    }

    Rel(user, http, "Calls", "HTTPS/JSON")
    Rel(scheduler, http, "Triggers due transitions", "HTTPS")

    Rel(http, auth, "Signup / login")
    Rel(http, items, "Item and picture CRUD")
    Rel(http, listings, "Listing CRUD / transitions")
    Rel(http, sched, "Manage schedule")
    Rel(http, bids, "Place bid")

    Rel(listings, sm, "Validates transitions; derives current state")
    Rel(bids, sm, "Checks listing status")
    Rel(listings, events, "Emits transition events")

    Rel(auth, repo, "Persists")
    Rel(items, repo, "Persists")
    Rel(listings, repo, "Persists")
    Rel(sched, repo, "Persists")
    Rel(bids, repo, "Persists")

    Rel(bids, store, "Leading-bid lookup; per-listing lock")
    Rel(listings, store, "Caches / invalidates current state")

    Rel(repo, db, "SQL")
    Rel(store, cache, "RESP")
```

### How the spec maps to the components

- **Bid concurrency** ("two users bidding on the same listing at the same time"): the Bid Service takes a per-listing lock in Valkey, then writes the bid; contention on different listings never interacts.
- **Bid rules** (status, own-listing, already-leading): enforced by the Bid Service against the State Machine and the leading bid (`idx_bids_listing_leading` - highest amount, earliest bid breaks ties).
- **Scheduled activation/deactivation**: the Scheduler polls `listing_schedules` (indexed on `activates_at` / `deactivates_at`) and calls the API, which runs the normal transition path - state machine validation plus event dispatch.
- **Events**: every transition the Listing Service appends is pushed to the Event Dispatcher; the transition table itself is the durable event log.
