# demining-actor — the cljc actor boundary for Humanitarian Mine Action

`cloud-itonami/demining-actor` holds the **actor identity and the pure `.cljc` planning
boundary** for a Humanitarian Mine Action (HMA) agent: who this actor is
(`actor-manifest.jsonld`, `.well-known/did.json`) and, given a set of gate attestations,
**what records it would write** (`src/demining/murakumo.cljc`).

It plans. It does not execute. `cell-plan` returns a `:blocked` or `:ready` value
containing `:mst/put-record` effect *descriptions*; nothing in this repository opens a
socket, holds a key, or writes to a PDS. That separation is the point — the decision to
publish a hazardous-area record is reviewable as a value before anything leaves the process.

## Boundary against the nearest repository

| Repository | What it owns |
|---|---|
| **`cloud-itonami/demining`** | The design corpus and edge facade — IMAS/UNSPSC/CPC crosswalks, legal instruments across 31 jurisdictions, crawl seeds, the deployed appview |
| **`cloud-itonami/demining-actor`** (here) | The actor's DID composition, its manifest, and the `.cljc` cell-plan boundary derived from it |

If you are looking for the standards, the treaty layer, or the deployed surface, it is in
`cloud-itonami/demining`, not here. If you are looking for *what this actor is allowed to
emit and under which attestations*, it is here.

## Scope boundary — read this before anything else

Manufacture, stockpile, transfer, or deployment of anti-personnel mines is **out of scope
and not implemented**. The prohibition is legal, not stylistic: the Anti-Personnel Mine Ban
Convention (Ottawa, 1997), the Convention on Cluster Munitions, CCW Protocol V, and
対人地雷の製造の禁止及び所持の規制等に関する法律 (平成10年法律第115号).
`actor-manifest.jsonld` carries the exclusion in its `description` and enforces it in
`pipelineGate.validateScope`.

**Sensitivity tiering is a safety control, not a privacy preference.** Publishing the
coordinates of an *uncleared* hazardous area endangers civilians (IMAS 05.10). Uncleared
SHA/CHA polygons and victim PII are Tier 3 and never enter the public record; a polygon is
demoted to Tier 1 only on a Land Release decision. The full invariant list is in
`CLAUDE.md` and is not restated here — one copy, so the two cannot drift.

## What is actually in this repository

| Path | What it is |
|---|---|
| `actor-manifest.jsonld` | Actor definition: 8-way path-based Multi-DID composition, 5 capabilities, 3 pipelines, `pipelineGate.validateScope`, the convo system prompt |
| `.well-known/did.json` | `did:web:etzhayyim.com:actor:demining` — PDS and appview service endpoints |
| `src/demining/murakumo.cljc` | The planning boundary: `cell-specs`, `missing-gates`, `records-for`, `cell-plan`, `all-cell-plans` |
| `test/demining/murakumo_test.cljc` | 9 contract tests / 57 assertions. Introspects `cell-specs` rather than hardcoding cell names, so it holds as the manifest changes |
| `CLAUDE.md` | The five actor invariants, and the DID → IMAS-series role table |
| `storage-profile.edn` | Declares the local-agent kagi-chunked storage profile |

## The shape of the boundary

`cell-specs` names one planning cell per manifest pipeline. Every cell carries the same
seven `common-gates`. `cell-plan` is total on that map:

- **any gate unattested** → `{:status :blocked :effects []}` and the missing gates named.
  It does not partially proceed.
- **all seven attested** → `{:status :ready :records [...] :effects [...]}` with one
  `:mst/put-record` per declared collection.
- **unknown cell** → `ex-info`, not `nil`. A typo in a cell name cannot silently plan nothing.

`gate-value` accepts attestations as either a map or a set, and keyed by either keyword or
string, because the attestation carrier differs by caller. That tolerance is deliberate and
is pinned by test, so narrowing it is a visible change rather than a silent one.

## Running it

Both runtimes are verified and produce identical results; the `nbb` path needs no JVM.
See **[`docs/operator-quickstart.md`](docs/operator-quickstart.md)** — it is four commands,
each one shown with the output it actually produced and with the way it fails.

## Naming

`demining` = 地雷除去 — the *counter* side of explosive ordnance. This repository exists to
help find and remove mines and to teach people to avoid them. The `-actor` suffix marks the
role face: this is the actor boundary, distinct from the `demining` subject repository above.

## Licence and notice

See `NOTICE`.
