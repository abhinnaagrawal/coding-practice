# SpiceDB (and OpenFGA / Ory Keto)

## 30-second intuition

SpiceDB is an open-source, Go-based, Zanzibar-inspired permissions database from AuthZed: you define a **schema** (object types, relations, permissions with set-operator expressions), write **relationship tuples** as data, then call `CheckPermission` on every access and `LookupResources`/`LookupSubjects` to power filtered list views. OpenFGA (CNCF, backed by Auth0/Okta) and Ory Keto are the other production-grade implementations of the same Zanzibar model, differing mainly in schema DSL and consistency-token mechanics.

---

## What SpiceDB is

- Open-source database purpose-built for storing and evaluating fine-grained authorization relationships — a direct implementation of Google's Zanzibar paper.
- Written in Go; backs AuthZed's managed cloud service, but is fully self-hostable.
- gRPC + HTTP API surface (`CheckPermission`, `WriteRelationships`, `LookupResources`, `LookupSubjects`, `ExpandPermissionTree`, `Watch`).
- Actively maintained (releases in the v1.5x range as of 2026); CVE-2026-40091 was patched in v1.51.1 — **always run the latest patch release**, don't pin an old minor version for authorization-critical infra.

## Schema language

SpiceDB's DSL is the human-friendly authoring layer for Zanzibar namespace configs.

Core keywords:
- `definition <name> { ... }` — declares an object type
- `relation <name>: <type>[ | <type>]` — a direct relation to one or more subject types (subjects can be `user`, another `definition`, or `definition#relation` for usersets)
- `permission <name> = <expr>` — a *computed* set, using operators:
  - `+` union
  - `&` intersection
  - `-` exclusion
  - `->` arrow (tuple-to-userset "hop": follow a relation to a related object, then evaluate another relation/permission there)

```zed
definition user {}

definition group {
  relation member: user | group#member
}

definition folder {
  relation parent: folder
  relation owner:  user
  relation editor: user | group#member
  relation viewer: user | group#member

  permission view = viewer + editor + owner + parent->view
  permission edit = editor + owner + parent->edit
}

definition document {
  relation folder: folder
  relation owner:  user
  relation editor: user | group#member
  relation viewer: user | group#member

  permission view = viewer + editor + owner + folder->view
  permission edit = editor + owner + folder->edit
}
```

Note `folder->view`: this means "take this document's `folder` relation, land on the related `folder` object, then evaluate its `view` permission there" — this is how folder-level access cascades down to every document inside it, recursively, without app code.

## Relationships vs. schema

- **Schema** = the shape: what relations exist on each type, and how permissions are computed from them. Changed rarely, deployed like a migration.
- **Relationships (tuples)** = the data: concrete facts like `document:readme#viewer@user:alice`. Written constantly as your app's users share/unshare things.

```
document:readme#owner@user:alice
document:readme#viewer@user:bob
document:readme#folder@folder:eng-docs
folder:eng-docs#editor@group:eng-team#member
group:eng-team#member@user:carol
```

## CheckPermission API

```
CheckPermission(resource: document:readme, permission: view, subject: user:carol)
  -> PERMISSIONSHIP_HAS_PERMISSION | PERMISSIONSHIP_NO_PERMISSION
```

This is Zanzibar's `Check` — evaluated by recursively resolving the permission expression tree against stored relationships, following arrows into related objects and unsets into group membership, exactly as detailed in `zanzibar.md`. This is the call your app makes on **every protected access** — e.g. before serving a document, before allowing an edit.

## LookupResources / LookupSubjects (reverse-index queries)

- `LookupResources(subject: user:carol, permission: view, resource_type: document)` → stream of all document IDs Carol can view. Powers "show me my documents" list views.
- `LookupSubjects(resource: document:readme, permission: view)` → stream of all subjects who can view this document. Powers "who has access" UI, similar to Zanzibar's Expand but subject-enumerating rather than tree-shaped.

**Why these are the hardest queries in ReBAC**: `Check` is a targeted point query (one object, one subject) — bounded fan-out. `LookupResources` effectively has to evaluate Check-like logic against *every candidate resource* (or invert the relation graph to avoid a full scan), which is combinatorially worse, especially through arrows/unions across large group memberships. SpiceDB implements this with reverse-relationship indexing and internal dispatch, but it is inherently more expensive than a Check, and is the reason production systems often materialize a cache (e.g. a roaring-bitmap "visible resource set" — see `data-structures/roaring-bitmaps.md`) instead of calling LookupResources on every list-view request.

## Consistency models

SpiceDB exposes Zanzibar's consistency knob per-request:

| Consistency | Meaning | Cost |
|---|---|---|
| `minimize_latency` | Serve from the nearest/any available replica, staleness allowed | Cheapest, fastest |
| `at_least_as_fresh(zedtoken)` | Guarantee the read reflects at least the state as of the given ZedToken (a specific prior write) | Bounded — reintroduces the safety needed to avoid the "new enemy problem" without paying for full global consistency on every call |
| `fully_consistent` | Evaluate against the absolute latest global state | Most expensive/highest latency, use sparingly |

### ZedTokens

SpiceDB's implementation of Zanzibar's zookies. Every write (`WriteRelationships`) returns a ZedToken encoding a datastore-specific consistency marker (a timestamp or MVCC revision depending on backend). Callers persist/pass this token forward: "run this Check at least as fresh as the state right after I revoked Bob's access." This is the mechanism that closes the new-enemy problem in an SpiceDB deployment — see `zanzibar.md` for the underlying correctness argument.

## Datastore backends

| Backend | Recommended for |
|---|---|
| **CockroachDB** | Self-hosted, high-throughput and/or multi-region deployments — AuthZed's own managed service standardizes on CockroachDB |
| **Cloud Spanner** | Self-hosted GCP deployments wanting Spanner's native linearizability (closest to Zanzibar's original Spanner-based design; skips CockroachDB's transaction-overlap strategy since Spanner already guarantees linearizability) |
| **PostgreSQL** | Self-hosted single-region deployments — simplest operationally, well-tested, but weaker global-consistency guarantees than Cockroach/Spanner for multi-region ZedToken freshness |
| **MySQL** | Supported but *not recommended*; use only if Postgres isn't an option |
| **memdb** (in-memory) | Local dev / integration testing only |

Why Cockroach/Spanner are preferred over Postgres for "at least as fresh" guarantees: both provide externally-consistent, globally-orderable timestamps/MVCC across a distributed multi-region cluster, which is what lets a ZedToken minted in one region be honored correctly by a Check served in another region. Single-node/single-region Postgres doesn't have that distributed-consistency problem to begin with, so it's perfectly adequate for smaller or single-region deployments, but doesn't extend cleanly to a globally-distributed multi-region ZedToken freshness story the way Spanner/Cockroach do.

## Caching layer

SpiceDB caches Check results (and intermediate sub-check results in the dispatch tree) to avoid re-walking the same relationship graph on repeated calls. Cache invalidation is driven by watching relationship writes — conceptually the same **Watch API** stream from Zanzibar: as tuples change, SpiceDB's caches (and any downstream consumer's materialized views) get invalidated/updated rather than relying on TTL alone. This is also the integration point external systems use to keep their own bitmap/materialized caches fresh (see `data-structures/roaring-bitmaps.md`'s hybrid pattern).

## Performance characteristics

- Target: Check latency in the single-digit-to-low-tens-of-milliseconds range for p99 in well-tuned deployments, mirroring Zanzibar's own SLOs.
- **Deep relation graphs are the main latency risk** — the same recursive nested-group problem as Zanzibar: a permission that resolves through several arrow-hops and large/deeply-nested groups multiplies the number of sub-checks that must be dispatched and resolved.
- **Dispatch**: SpiceDB clusters split large Check/Lookup operations into sub-problems ("dispatch") distributed across cluster nodes, enabling parallel graph traversal — similar in spirit to Zanzibar's internal fan-out, and configurable via dispatch caching and concurrency limits.
- Schema design directly affects performance: prefer shallow, well-indexed relation chains over deeply nested arrow hops through huge groups where avoidable; use intersections/exclusions sparingly since they require evaluating and combining multiple subtrees rather than short-circuiting on the first match (as unions can).

## OpenFGA as the alternative

OpenFGA is a CNCF project (accepted 2022, promoted to CNCF **Incubating** in October 2025) backed primarily by Auth0/Okta. Same Zanzibar-derived relational model, different authoring surface:

```
model
  schema 1.1

type user

type group
  relations
    define member: [user, group#member]

type document
  relations
    define owner: [user]
    define editor: [user, group#member]
    define viewer: [user, group#member] or editor or owner
```

OpenFGA's "Configuration Language" ships in two forms: a JSON syntax that the API actually accepts (closest to the raw Zanzibar namespace-config shape) and a friendlier DSL (used in the Playground, CLI, and IDE extensions) that compiles down to that JSON.

### Comparison table

| | SpiceDB | OpenFGA | Ory Keto |
|---|---|---|---|
| Backer | AuthZed | CNCF / Auth0-Okta | Ory |
| Language | Go | Go | Go |
| Schema DSL | Zed schema language (`definition`/`permission`/`->`) | Configuration Language (DSL compiles to JSON `schema 1.1`) | Namespace configs (OPL, Ory Permission Language, close to Zanzibar's own config shape) |
| Governance | Company-led open source + managed cloud | CNCF Incubating project | Company-led open source + managed cloud (Ory Network) |
| Consistency token | ZedToken | (consistency handled per-store; less prominently surfaced as a client-facing token than SpiceDB) | supports Zanzibar-style consistency tuning |
| Notable datastore options | Postgres, CockroachDB, Spanner, MySQL, memdb | Postgres, MySQL, in-memory | Postgres, MySQL, CockroachDB, SQLite |
| Positioning | "Zanzibar done right," strong enterprise/managed offering | Vendor-neutral CNCF home, strong SDK/ecosystem breadth | First open-source Zanzibar implementation historically, Ory-ecosystem integrated (Kratos/Oathkeeper/Hydra) |

All three are legitimate production choices; pick based on ecosystem fit (already using Ory stack → Keto; want CNCF-neutral governance and broad SDKs → OpenFGA; want the most Zanzibar-paper-faithful schema semantics and strongest managed-service option → SpiceDB).

## Integration pattern

Typical application wiring:

1. **On resource creation**: write an `owner` relationship tuple for the creator.
2. **On sharing/permission change**: write/delete relationship tuples (`viewer`, `editor`, group membership) — capture the returned ZedToken if the caller needs to guarantee read-your-write freshness downstream.
3. **On every protected access** (view a doc, edit a doc, delete a doc): call `CheckPermission` with the appropriate consistency level — `at_least_as_fresh` for anything following a recent permission change in the same user flow, `minimize_latency` for high-QPS, less-sensitive checks.
4. **On list views** ("show me my documents"): call `LookupResources`, or — more commonly in performance-sensitive apps — maintain a materialized cache (bitmap or otherwise) of "resources visible to user X," refreshed via the Watch API, and only fall back to live LookupResources for cache misses or infrequent paths.

## Worked example: multi-tenant SaaS (org > team > project > document)

**Schema:**

```zed
definition user {}

definition organization {
  relation admin:  user
  relation member: user
  permission manage = admin
}

definition team {
  relation org:    organization
  relation lead:   user
  relation member: user
  permission manage = lead + org->manage
  permission view   = member + manage
}

definition project {
  relation team:   team
  relation owner:  user
  permission manage = owner + team->manage
  permission view   = team->view + manage
}

definition document {
  relation project: project
  relation editor:  user
  relation viewer:  user

  permission edit = editor + project->manage
  permission view = viewer + edit + project->view
}
```

**Relationship writes on setup:**

```
organization:acme#admin@user:root
team:platform#org@organization:acme
team:platform#member@user:dave
project:billing-svc#team@team:platform
document:design-doc#project@project:billing-svc
document:design-doc#viewer@user:erin      // erin added directly, outside the team
```

**Check call:** `CheckPermission(document:design-doc, view, user:dave)`

1. `view = viewer + edit + project->view`. `viewer` direct tuples: only `erin` — no match.
2. `edit = editor + project->manage`. No direct `editor` tuple for dave. Follow `project->manage`: resolve `document:design-doc#project` → `project:billing-svc`, recurse `CheckPermission(project:billing-svc, manage, user:dave)`.
   - `manage = owner + team->manage`. No `owner` tuple for dave. Follow `team->manage`: `project:billing-svc#team` → `team:platform`, recurse `CheckPermission(team:platform, manage, user:dave)`.
     - `manage = lead + org->manage`. No `lead` tuple for dave. `org->manage` → `organization:acme`, recurse `CheckPermission(organization:acme, manage, user:dave)` → `admin` tuples: only `root` — **no match**, this branch fails.
   - `team:platform`'s `manage` branch failed, but `team:platform`'s own `view = member + manage`: dave *is* a direct `member` — however this is `project->view` not `project->manage`; that recursion happens separately below.
3. Back up: `edit` branch overall fails for dave (no editor tuple, no project-manage path).
4. `project->view`: resolve `document:design-doc#project` → `project:billing-svc`, recurse `CheckPermission(project:billing-svc, view, user:dave)`.
   - `view = team->view + manage`. `team->view` → `team:platform`, recurse `CheckPermission(team:platform, view, user:dave)` → `view = member + manage`, dave has direct `member` tuple → **MATCH**.
5. Union succeeds through the `project->view` branch → `CheckPermission(document:design-doc, view, user:dave)` = **ALLOWED**.

This traces exactly the kind of multi-hop arrow chain (`document -> project -> team -> organization`) that makes schema design and dispatch efficiency matter in production: a naive implementation would re-walk the `team->manage->org->manage` dead-end branch on every call unless dispatch caching short-circuits it.

## Deep internals

- SpiceDB's dispatch layer decomposes a single `CheckPermission` into a tree of sub-dispatch requests that can be distributed and cached across cluster nodes — mirroring Zanzibar's internal fan-out architecture, but exposed as a tunable (dispatch concurrency limits, cluster dispatch caching) rather than a fully opaque internal detail.
- ZedToken freshness guarantees are only as strong as the datastore's own consistency model — this is *why* backend choice (Cockroach/Spanner vs. Postgres) is a first-class production decision, not an implementation detail: it directly determines what "at least as fresh as" can actually promise across regions.
- Intersection (`&`) and exclusion (`-`) operators cannot short-circuit the way union (`+`) can — they must evaluate all operand branches to produce a definitive answer, making schemas that overuse them in hot paths meaningfully more expensive than equivalent union-heavy schemas.
- Schema changes are versioned and validated against a compatibility checker in SpiceDB tooling (`zed`, CI schema-diff tools) because reinterpreting existing tuples under new permission logic is exactly as consequential as it is in raw Zanzibar.

## Key gotchas

- Defaulting every `CheckPermission` call to `minimize_latency` silently reopens the new-enemy problem for revocation-sensitive flows — deliberately choose consistency level per call-site, not globally.
- `LookupResources`/`LookupSubjects` do not scale like `CheckPermission` — treat them as expensive, and cache/materialize for list-view-heavy UIs rather than calling them per page load.
- Schemas with deep arrow chains through large groups are the single biggest performance risk — profile Check latency against realistic group sizes and nesting depth, not toy data.
- MySQL is explicitly the least-recommended datastore — don't default to it just because it's familiar.
- Keep SpiceDB on a current patched release; authorization infra is a high-value target for CVEs (e.g. CVE-2026-40091, fixed in v1.51.1).

## When to use / When NOT to use

- **Use SpiceDB/OpenFGA/Keto** when you need per-resource sharing with inheritance (folders/teams/orgs), a decoupled authorization service usable from multiple apps/services, or strong correctness guarantees around revocation.
- **Don't use them** as your *only* authorization layer if your access patterns are dominated by bulk list-filtering over millions of resources per request — pair with a materialized bitmap cache (see `data-structures/roaring-bitmaps.md`) rather than hammering LookupResources directly. Also reconsider if your model is simple global RBAC with no per-object sharing — the operational overhead of running a separate ReBAC service may not be worth it.
