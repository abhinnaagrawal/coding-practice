# Google Zanzibar

## 30-second intuition

Zanzibar is Google's global authorization system: instead of storing permissions as rows in per-app ACL tables, it stores **relationship tuples** (`object#relation@subject`) in a planet-scale database and answers "can subject X do action Y on object Z" by **recursively walking a graph of relationships** — while guaranteeing that a permission revocation is never followed by a stale "yes" (the "new enemy" problem). Every modern ReBAC system (SpiceDB, OpenFGA, Ory Keto, Auth0 FGA) is an implementation of this 2019 paper.

---

## The problem Zanzibar solves

Pre-Zanzibar, Google had **one bespoke ACL system per product** — Drive, Calendar, Photos, YouTube, Maps all reinvented sharing semantics. Consequences:

- Inconsistent security models across products (a bug in one product's ACL logic doesn't fix others)
- Duplicated engineering effort for a problem that's the same at its core: "does subject S have relation R to object O"
- No shared low-latency, high-availability, planet-scale authorization primitive

Zanzibar (launched ~2018, publicly documented in a 2019 USENIX ATC paper) unifies this into one service. Scale at Google: **trillions of checks/day**, thousands of client applications (Drive, Calendar, YouTube, Cloud IAM), tens of trillions of relationship tuples, p99 latency in the low tens of milliseconds, and 99.999% availability over long windows.

---

## The relationship tuple model

The atomic unit of data is a **tuple**:

```
object # relation @ subject
```

Examples:

```
doc:readme#viewer@user:alice
doc:readme#editor@group:eng#member      <- subject is itself a userset (indirection)
group:eng#member@user:bob
folder:project#viewer@user:carol
doc:readme#parent@folder:project        <- structural relation, not a permission
```

- `object` = `type:id` (e.g. `doc:readme`)
- `relation` = a named relation defined in that type's namespace config (`viewer`, `editor`, `owner`, `member`, `parent`)
- `subject` = either a concrete principal (`user:alice`) **or a userset** — another object#relation pair (`group:eng#member`), meaning "anyone who has relation `member` on `group:eng`"

This last form — subject-as-userset — is what makes group membership, nested groups, and inherited folder permissions expressible as plain tuples instead of application logic.

## Namespace configs (the "schema")

Each object type declares its relations and how each relation is *computed* — this is the namespace config. Conceptually (Zanzibar's own syntax is protobuf-based; SpiceDB/OpenFGA give this a friendlier DSL — see `spicedb.md`):

```
namespace doc {
  relation owner:  user
  relation editor: user | group#member
  relation viewer: user | group#member
  relation parent: folder

  permission viewer = viewer + editor + owner + parent->viewer   // union rewrite
  permission editor = editor + owner
}

namespace folder {
  relation viewer: user | group#member
  permission viewer = viewer
}

namespace group {
  relation member: user | group#member   // groups can nest inside groups
}
```

## Userset rewrites: how permissions are computed

A **userset rewrite** is a set-operation expression over relations, evaluated at Check time. The operators:

| Operator | Meaning |
|---|---|
| `union` (`+`) | subject has permission if in *any* of the operand usersets — e.g. `editor` implies `viewer` |
| `intersection` (`&`) | subject must be in *all* operand usersets — e.g. "member of legal AND member of the doc's ACL" |
| `exclusion` (`-`) | subject must be in the first userset but *not* the second — e.g. "viewer minus banned" |
| `tuple-to-userset` (`->`, "arrow"/hop) | follow a *different* relation on a *related* object — e.g. `parent->viewer` means "look up this doc's `parent` folder, then check `viewer` on that folder" |

This is how "editor implies viewer" and "folder permissions cascade to documents inside it" are both expressed as data-driven graph traversal rather than hardcoded app logic.

## Nested/indirect relationships

Because a tuple's subject can itself be a userset (`group:eng#member`), and groups can contain other groups (`group:eng#member@group:backend#member`), relation resolution is **recursive graph traversal with arbitrary nesting depth**. Google's real-world requirement: support groups nested many levels deep (group-of-groups-of-groups), and resolve Check queries over that graph within tens of milliseconds.

```
user:alice ──member──> group:backend ──member──> group:eng ──editor──> doc:readme
```
`check(doc:readme, viewer, user:alice)` must walk: editor→viewer union, then editor relation, then group:eng#member, then group:backend#member, then find user:alice. All in one logical Check call.

## The Check API

```
check(object, relation, subject) -> ALLOWED | DENIED
```

Recursive resolution algorithm (conceptually):
1. Look up the namespace config for `object`'s type, find `relation`'s rewrite rule.
2. If it's a direct relation, scan stored tuples for `object#relation@subject` (or `subject` being a userset containing the target — recurse into that userset's membership).
3. If it's a union/intersection/exclusion, recursively evaluate each operand and combine.
4. If it's tuple-to-userset (`parent->viewer`), first resolve `object#parent` to get the related object(s), then recursively `check(related_object, viewer, subject)`.

This recursion is naturally parallelizable (fan out sub-checks concurrently) and cacheable per sub-call — Zanzibar's serving stack shards this work across many servers with request-scoped caching.

## The Expand API

```
expand(object, relation) -> tree of subjects/usersets
```

Instead of asking "can subject S access this," Expand answers "**who** (or what usersets) hold this relation" — it returns a tree representing the full union/intersection/exclusion structure, recursively expanded. Used for building "who has access" UI (e.g. a Drive sharing dialog), and as a building block for reverse-index queries.

---

## The "new enemy" problem and consistency

**The critical correctness bug ReBAC systems must avoid:** eventual consistency can leak data.

Scenario:
1. Alice removes Bob from doc D's ACL (revoke `viewer`).
2. Alice creates new content and shares it, believing Bob no longer has access.
3. If the authorization check for step 2 runs against a stale replica that hasn't yet seen step 1's write, Bob could be granted access to content Alice explicitly tried to hide from him — worse than a stale *read*, this is a stale **security decision**, and it can never be silently "corrected later" once Bob has seen the data.

This is the **new enemy problem**: revocations must be visible to all subsequent authorization checks that causally follow them, even under a globally distributed, eventually-consistent replicated store.

### Zookies (consistency tokens)

Zanzibar's fix: every write returns a **zookie** — an opaque token encoding a global timestamp from Google's **TrueTime**-derived timestamp oracle (built on Spanner). Clients pass the zookie from a prior write (or from reading related state) into subsequent Check calls as a **snapshot lower bound**: "evaluate this Check using a view of the tuple store that is *at least as fresh* as this token."

```
zookie = write(remove Bob from doc:D#viewer@user:bob)   // returns a fresh timestamp token
...
check(doc:D2, viewer, user:bob, at_least_as_fresh_as: zookie)  // guaranteed to see the revocation
```

This gives callers a knob: `full consistency` (use latest global timestamp — highest latency), `at-least-as-fresh-as(zookie)` (bounded staleness, cheaper), or `minimal latency` (any replica, staleness allowed — safe only for non-security-sensitive checks). This directly maps onto SpiceDB's `fully_consistent` / `at_least_as_fresh` (ZedToken) / `minimize_latency` consistency levels — see `spicedb.md`.

---

## Storage layer

- Tuples are stored in **Spanner** (Google's globally-distributed, externally-consistent SQL database), sharded by object ID for horizontal scale.
- Spanner's TrueTime gives Zanzibar globally-ordered, externally-consistent snapshots — the foundation zookies are built on.
- A caching layer sits in front of Spanner to absorb the read-heavy Check/Expand workload (trillions of reads/day vs. comparatively few writes).

### Leopard index

Some namespace graphs are pathologically expensive to resolve at Check time — e.g. a group with millions of nested members, or "is this the 8th level of nested group membership." The **Leopard index** precomputes and materializes the **transitive closure** of deeply-nested group memberships as a compact structure, so Check/Expand on those degenerate cases doesn't require live recursive graph walks. It's rebuilt incrementally as membership tuples change and is specifically targeted at the worst-case fan-out scenarios (huge groups, deep nesting) that would otherwise blow past latency SLOs.

### Watch API

```
watch(namespace, since_zookie) -> stream of tuple changes
```
A change-data-capture stream over relation tuple mutations. Used by downstream systems to invalidate their own caches or materialized views (e.g. a search index that needs to know "resource X is no longer visible to user Y" to update document-level security filters) without polling. This is the mechanism referenced in the roaring-bitmap hybrid pattern (see `data-structures/roaring-bitmaps.md`) for refreshing bitmap caches of "resources visible to user."

---

## Why ReBAC beats RBAC for complex sharing

Classic RBAC models permissions as `(user, role)` and `(role, permission)` — clean for org-chart-shaped access ("all Managers can approve expense reports") but a poor fit for **per-object, per-person, inheritance-based sharing**:

- "Share this specific doc with these 3 people and this team, and it should also inherit from the parent folder's permissions, and Editor should imply Viewer" — this is exactly Google Drive's model, and it doesn't reduce to a fixed role hierarchy because the *object graph itself* (folder → doc, doc → comment) carries permission structure, and grants are on individual objects, not just role assignments.
- RBAC roles are typically global or per-tenant; ReBAC relations are **per-resource-instance** and compose through the resource's own relationships (`parent`, `owner`, `member`) via the arrow operator.
- ReBAC subsumes RBAC (model roles as a `group` object with a `member` relation, and grant permissions to `group#member` as subject) but not vice versa.

---

## Worked example: Google-Drive-style doc sharing

**Namespace configs:**

```
namespace user {}

namespace group {
  relation member: user | group#member     // nested groups
}

namespace folder {
  relation owner:  user
  relation editor: user | group#member
  relation viewer: user | group#member
  relation parent: folder                  // folders can nest

  permission view_perm = viewer + editor + owner + parent->view_perm
  permission edit_perm = editor + owner + parent->edit_perm
}

namespace doc {
  relation owner:  user
  relation editor: user | group#member
  relation viewer: user | group#member
  relation parent: folder

  permission view_perm = viewer + editor + owner + parent->view_perm
  permission edit_perm = editor + owner + parent->edit_perm
}
```

**Tuples:**

```
group:eng#member@user:dave
folder:project#editor@group:eng#member       // eng team can edit anything in folder:project
doc:readme#parent@folder:project             // readme lives in that folder
doc:readme#viewer@user:alice                 // alice explicitly added as viewer
```

**Trace `check(doc:readme, view_perm, user:dave)`:**

1. Resolve `doc:readme`'s `view_perm` rewrite: `viewer + editor + owner + parent->view_perm`.
2. `viewer` direct tuples on `doc:readme`: only `user:alice` — no match for dave.
3. `editor` direct tuples on `doc:readme`: none.
4. `owner`: none.
5. `parent->view_perm`: resolve `doc:readme#parent` → `folder:project`. Recurse: `check(folder:project, view_perm, user:dave)`.
   - `folder:project`'s `view_perm` = `viewer + editor + owner + parent->view_perm`.
   - `editor` tuples on `folder:project`: `group:eng#member` (a userset, not a concrete user) → recurse into membership: `check(group:eng, member, user:dave)` → direct tuple `group:eng#member@user:dave` exists → **MATCH**.
6. Union short-circuits true → `check(doc:readme, view_perm, user:dave)` = **ALLOWED**.

This demonstrates the two core mechanisms in one call: **union rewrite** (view_perm falls back through several relations) and **tuple-to-userset hop + nested group resolution** (parent folder → group membership, two levels of indirection) — exactly the "arbitrarily nested" requirement Zanzibar is built to handle at low latency.

---

## Deep internals

- **Fan-out parallelism**: because Check recursion is a tree of independent sub-checks (union branches, tuple-to-userset hops), Zanzibar's serving nodes dispatch sub-checks concurrently to other shards/servers and combine results — this is why deep graphs don't linearly blow up latency, but do increase *fan-out* (number of parallel RPCs), which is the actual scaling bottleneck.
- **Read-heavy workload skew**: at Google scale, reads (Check/Expand) outnumber writes (tuple mutations) by many orders of magnitude, which is why so much engineering (caching, Leopard index) targets Check-path latency rather than write throughput.
- **Consistency vs. latency knob is per-request, not global**: callers choose `full`, `at-least-as-fresh`, or `minimal-latency` per Check call — critical security decisions (revocation-sensitive) use stronger consistency; high-QPS, less-sensitive checks (e.g. "should I render this button") can use weaker/cheaper consistency.
- **Namespace configs are versioned data, not code deploys** — updating what "editor implies viewer" means is a config change, not an app redeploy, which is part of why every product at Google could centralize on one authorization service without giving up per-product schema flexibility.
- Zanzibar intentionally does **not** support arbitrary attribute-based conditions (pure ABAC) in the original paper — it's relation/graph-first. Modern implementations (SpiceDB, OpenFGA) have since added **caveats/conditions** (attribute checks embedded in relations) as an extension beyond the original paper.

## Key gotchas

- **Consistency choice is a security decision, not just a performance knob** — defaulting everything to `minimize_latency`/eventual consistency silently reintroduces the new-enemy problem. Revocation-adjacent checks need `at_least_as_fresh` or full consistency.
- **Deep nesting is a real latency risk** in naive implementations — huge/deeply nested groups can cause Check fan-out to explode; this is precisely what the Leopard index exists to bound.
- **Namespace config changes affect all existing tuples' interpretation retroactively** — rewriting what a relation means is powerful but requires careful migration thinking (old tuples are reinterpreted under the new rules).
- **Zanzibar is a paper/internal system, not something you run** — for actual deployment, use an implementation (SpiceDB, OpenFGA, Ory Keto); see `spicedb.md`.

## When to use / When NOT to use

- **Use ReBAC (Zanzibar model)** when: sharing is per-object and per-person/group, permissions need to inherit through a resource hierarchy (folders, orgs, teams), or "who can see this" is a first-class product question (Drive/Docs/Notion/Figma-style collaboration).
- **Don't reach for it** when: your authorization is genuinely just a small set of global roles with no per-resource sharing (simple RBAC is simpler, cheaper, and easier to reason about) or when you need heavy attribute-based dynamic conditions as the *primary* model (pure ABAC / policy engines like OPA/Cedar may fit better, though SpiceDB/OpenFGA now support hybrid caveats).
