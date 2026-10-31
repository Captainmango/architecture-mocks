# Auctionite - ERD

Auctionite is a basic auctioning application. Users own items, items are sold
through listings, and users place bids on listings. A listing's schedule is
kept in a separate table, and its current state is derived from its
state-transition rows.

## Entities

| Table                       | Purpose                                                                 |
| --------------------------- | ----------------------------------------------------------------------- |
| `users`                     | A registered user who can log in, sell items, and place bids.           |
| `items`                     | A thing a user wants to sell, with its pictures.                        |
| `item_pictures`             | A picture belonging to an item.                                         |
| `listings`                  | An item offered for auction.                                            |
| `listing_schedules`         | The activation/deactivation times for a listing.                        |
| `listing_state_transitions` | One row per listing state transition; doubles as the emitted events.    |
| `bids`                      | An amount offered by a user on a listing.                               |

## Relationships

```
users 1───* items 1───* item_pictures
  │         │
  │         └──1───* listings 1───* listing_schedules
  │                    │
  │                    ├──1───* listing_state_transitions
  │                    │
  └────1───* bids *────┘
```

- A **user** owns many **items**; an item belongs to one user.
- An **item** has many **item_pictures**; a picture belongs to one item.
- An **item** can be listed many times over its life (e.g. re-listed after a
  void); a **listing** references exactly one item.
- A **listing** keeps its activation/deactivation times in
  **listing_schedules**, decoupled from the listing itself.
- A **listing** has many **listing_state_transitions**; the most recent
  undeleted transition row is the listing's current state.
- A **listing** receives many **bids**; a bid belongs to one listing.
- A **user** places many **bids**; a bid belongs to one user.

## Listing state

Transition rules:

- `DRAFT` -> `ACTIVE`, `INACTIVE`, or `VOID`
- `ACTIVE` -> `INACTIVE`, `VOID`, or `COMPLETE`
- `INACTIVE` -> `ACTIVE`, `VOID`, or `COMPLETE`
- `VOID` and `COMPLETE` are terminal.

State is not stored on `listings`. Every transition appends a row to
`listing_state_transitions`; a listing's current state is the `status` of its
most recent row where `deleted_at IS NULL`. Soft-deleting a transition
(setting `deleted_at`) retracts it, reverting the listing to the previous
undeleted transition.

These rows also double as the domain events from the spec: the first row
(`DRAFT`) is "listing created", a transition into `ACTIVE` is "listing
activated", into `INACTIVE` "listing deactivated", into `COMPLETE` "listing
complete", and so on.

## DDL

> Constraints (primary keys, foreign keys, uniqueness, checks) are intentionally
> omitted; integrity is enforced by the application. Only indexes are defined.

```sql
CREATE TABLE users (
    id            BIGINT GENERATED ALWAYS AS IDENTITY,
    email         VARCHAR(320),
    password_hash TEXT,
    created_at    TIMESTAMP
);

-- Look up a user at login.
CREATE INDEX idx_users_email ON users (email);


CREATE TABLE items (
    id          BIGINT GENERATED ALWAYS AS IDENTITY,
    user_id     BIGINT,
    name        VARCHAR(200),
    description TEXT,
    created_at  TIMESTAMP
);

-- List an owner's items.
CREATE INDEX idx_items_user_id ON items (user_id);


CREATE TABLE item_pictures (
    id         BIGINT GENERATED ALWAYS AS IDENTITY,
    item_id    BIGINT,
    url        TEXT,
    sort_order INTEGER,
    created_at TIMESTAMP
);

-- Fetch all pictures for an item, in display order.
CREATE INDEX idx_item_pictures_item_id ON item_pictures (item_id, sort_order);


CREATE TABLE listings (
    id             BIGINT GENERATED ALWAYS AS IDENTITY,
    item_id        BIGINT,
    starting_price NUMERIC(12, 2),
    created_at     TIMESTAMP
);

-- Find the listing(s) for an item.
CREATE INDEX idx_listings_item_id ON listings (item_id);


CREATE TABLE listing_schedules (
    id             BIGINT GENERATED ALWAYS AS IDENTITY,
    listing_id     BIGINT,
    activates_at   TIMESTAMP,
    deactivates_at TIMESTAMP,
    created_at     TIMESTAMP
);

-- Find the schedule for a listing.
CREATE INDEX idx_listing_schedules_listing_id ON listing_schedules (listing_id);

-- Find schedules due for activation/deactivation.
CREATE INDEX idx_listing_schedules_activates_at ON listing_schedules (activates_at);
CREATE INDEX idx_listing_schedules_deactivates_at ON listing_schedules (deactivates_at);


CREATE TABLE listing_state_transitions (
    id         BIGINT GENERATED ALWAYS AS IDENTITY,
    listing_id BIGINT,
    status     VARCHAR(16),         -- DRAFT, ACTIVE, INACTIVE, VOID, COMPLETE
    created_at TIMESTAMP,
    deleted_at TIMESTAMP            -- soft delete; NULL means the transition stands
);

-- Resolve a listing's current state: most recent row WHERE deleted_at IS NULL.
CREATE INDEX idx_listing_state_transitions_listing_id
    ON listing_state_transitions (listing_id, created_at DESC);


CREATE TABLE bids (
    id         BIGINT GENERATED ALWAYS AS IDENTITY,
    listing_id BIGINT,
    user_id    BIGINT,
    amount     NUMERIC(12, 2),
    created_at TIMESTAMP
);

-- Fetch all bids for a listing.
CREATE INDEX idx_bids_listing_id ON bids (listing_id);

-- Fetch all bids placed by a user.
CREATE INDEX idx_bids_user_id ON bids (user_id);

-- Determine the leading bid for a listing (highest amount; earliest bid
-- breaks ties when two users bid the same amount).
CREATE INDEX idx_bids_listing_leading ON bids (listing_id, amount DESC, created_at ASC);
```
