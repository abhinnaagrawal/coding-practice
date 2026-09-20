# Docsify link rule — read before adding any new section

**Always write markdown links in this repo as absolute paths from the repo root (leading `/`), never relative.** This has broken twice already (Systems Engineering's sidebar entries, then its own README.md's internal links) — both times for the same reason.

## Why relative links break here

This site's `index.html` does not set docsify's `relativePath` option. Without it, docsify resolves **every** markdown link — both in `_sidebar.md` and inside page content — **relative to the site root**, not relative to the file that contains the link, and not relative to the current route either, inconsistently, depending on nesting depth.

The existing sections (`coding-practice/`, `python-tutorial/`, `numerical-methods/`, `ai-search-systems/`) never hit this because every file in them lives exactly **one level** below root — a relative link like `coding-practice/01-arrays-hashing.md` and a root-relative link resolve to the same place at that depth, so the bug was invisible.

`systems-engineering/` broke this the moment it introduced a **second level** of nesting (`systems-engineering/<category>/<doc>.md`). A relative-looking link written inside `systems-engineering/README.md` as `distributed-systems/consensus-raft-vs-paxos.md` silently resolved to `/distributed-systems/consensus-raft-vs-paxos.md` (root, missing the `systems-engineering/` prefix) instead of `/systems-engineering/distributed-systems/consensus-raft-vs-paxos.md` — a 404, in both `_sidebar.md` and the section's own `README.md`.

## The rule

Any link to a file **more than one directory below root**, in either `_sidebar.md` or a content page, must be absolute:

```markdown
✅ [Consensus: Raft vs Paxos](/systems-engineering/distributed-systems/consensus-raft-vs-paxos.md)
❌ [Consensus: Raft vs Paxos](distributed-systems/consensus-raft-vs-paxos.md)
❌ [Consensus: Raft vs Paxos](systems-engineering/distributed-systems/consensus-raft-vs-paxos.md)
```

Links exactly one level deep (the existing sections) work either way — but write those absolute too, for consistency, if you're touching that file anyway.

## If you add a new nested section

1. Every link in that section's own `README.md`: absolute, `/section-name/category/doc.md`.
2. Every link in `_sidebar.md` for that section: same, absolute.
3. Sanity check after pushing: open the deployed `#/section-name/README` page directly (not via the sidebar) and click one nested link — if it 404s, some link in that file is still relative.
4. `kb-doc-writer` (the skill that writes into `systems-engineering/`) already bakes this rule in as of the fix that produced this doc — if a future skill/section doesn't, add the same instruction.
