# Diagram Format: Semantics and Presentation Separated

Concept and decision record for a Confluence diagram editor.
Implementation-neutral: applies to Cloud (Forge) and Data Center alike, independent of
frontend library and storage location.

Last updated: 2026-09-21 · Status: draft, validated by a working prototype

---

## 1. What this is

A diagram editor for Confluence where **humans draw** and **machines read**.
A diagram is stored in two independent layers:

| | Layer 1 — Semantics | Layer 2 — Presentation |
|---|---|---|
| Contains | nodes, relationships, containment, descriptions | coordinates, size, shape |
| Role | source of truth | purely derivative, may be missing |
| Numbers | never appear here | the only place they appear |

Positions are **stored, not computed**. There is no auto-layout. Inserting a new node does
not move any existing node.

### Why separate at all

1. **Searchability** — Layer 1 is plain text and can be indexed by Confluence. Diagram
   content is invisible to full-text search today, and that cannot be fixed structurally as
   long as semantics exist only as a byproduct of the graphics.
2. **Determinism for LLMs** — from a screenshot a model reads a guess; from Layer 1 it reads
   a fact. This matters as soon as someone builds on the result rather than skimming it.
3. **Reviewable change history** — Layer 1 diffs line by line, which surfaces in the
   Confluence page version history.
4. **Robustness** — machine-driven restructuring cannot damage the layout.

### Target audience

Not the quick sketcher (they will use a whiteboard), but people who must maintain the same
diagram for years: enterprise architecture, platform teams, anything under audit or
certification pressure. That is where diagram decay is expensive and structure is an
advantage rather than a burden.

---

## 2. The decision rule

> **Layer 1 contains everything that would still be true if someone rearranged the diagram
> completely.**

If a statement survives rearrangement → semantics.
If it disappears → presentation.

The boundary runs along **intent versus pixels**, not along "text versus graphics".
Containment and grouping look like layout but are semantics.

**Check for every new feature:** Layer 1 contains no number and no word about presentation.
The moment a coordinate, a pixel width or a colour value lands there, the separation is lost.

---

## 3. Layer 1 — Semantics

Persistence: JSON. Stored in the indexed page body.

```json
{
  "format": "diagram/1",
  "meta": {
    "title": "Order fulfilment",
    "kind": "architecture",
    "scope": "The ordering path only. Admin tooling and reporting are deliberately out of scope.",
    "updated": "2026-09-18"
  },
  "nodes": [
    { "id": "n_3f1", "label": "Frontend", "container": true },
    { "id": "n_a27", "label": "React SPA", "parent": "n_3f1", "desc": "Customer-facing ordering flow in the browser." },
    { "id": "n_5c8", "label": "Backend", "container": true },
    { "id": "n_b40", "label": "API Gateway", "parent": "n_5c8", "desc": "Terminates TLS, validates tokens, routes to services." },
    { "id": "n_92d", "label": "Order Service", "parent": "n_5c8", "desc": "Accepts orders and orchestrates payment and shipping." },
    { "id": "n_6e3", "label": "PostgreSQL", "desc": "Holds orders and line items, single source of truth." },
    { "id": "n_c1a", "label": "Payment Provider", "desc": "Third-party provider for card and invoice payment.", "tags": ["external"] }
  ],
  "edges": [
    { "id": "e_11b", "source": "n_a27", "target": "n_b40", "type": "flow", "label": "HTTPS" },
    { "id": "e_7d2", "source": "n_b40", "target": "n_92d", "type": "dependency" },
    { "id": "e_4a9", "source": "n_92d", "target": "n_6e3", "type": "flow", "direction": "both", "label": "reads/writes" },
    { "id": "e_f30", "source": "n_92d", "target": "n_c1a", "type": "flow", "label": "initiate payment", "tags": ["planned"] },
    { "id": "e_8c5", "source": "n_3f1", "target": "n_5c8", "type": "association" }
  ],
  "notes": [
    { "id": "t_2e6", "anchor": "n_c1a", "text": "Provider contract expires at the end of Q3." }
  ]
}
```

### ids

IDs are **opaque and generated**, never derived from the label: a short random suffix behind
a kind prefix — `n_` node, `e_` edge, `t_` note. They are meaningless on purpose. Everything
a reader needs is in `label` and `desc`, which the author maintains; an ID that also carried
the name would be a second copy of it and would drift on the first rename (D13).

The kind prefix duplicates what the object's position in the JSON already says, which is
tolerable because it is the one thing that **cannot** drift — a node never becomes an edge —
and it makes a broken reference diagnosable at a glance.

Never parse an ID. It is a handle, not a statement.

### meta

| Field | Required | Meaning |
|---|---|---|
| `title` | yes | diagram title |
| `kind` | yes | diagram type, e.g. `architecture`, `process`, `dataflow`. Determines how everything below it should be read |
| `scope` | recommended | one or two sentences: what this diagram shows **and what it deliberately omits**. Prevents the most common misreading, which is assuming completeness where a selection was drawn |
| `updated` | yes | ISO date of the last change |

### nodes

| Field | Required | Meaning |
|---|---|---|
| `id` | yes | opaque, generated, stable. Survives renaming and anchors all references. See **ids** above |
| `label` | yes | human-facing caption |
| `container` | no | permission to hold children. See invariant I2 |
| `parent` | no | container ID. Absent on root nodes |
| `desc` | recommended | one or two sentences on purpose. Carries the domain role |
| `tags` | no | semantic classes, e.g. `external`, `planned`, `deprecated` |

There is deliberately **no** `type` field. The domain role is expressed through `desc` and
`tags`; shape lives in Layer 2.

### edges

| Field | Required | Meaning |
|---|---|---|
| `id` | yes | opaque, generated, stable |
| `source`, `target` | yes | node IDs |
| `type` | yes | see below |
| `direction` | no | override. **Store only when it differs from the type's default** |
| `label` | no | caption |
| `tags` | no | as for nodes |

**Edge types.** Kind of relationship and direction are two independent axes; the kind brings
its usual direction with it, and `direction` exists only as an exception.

| `type` | Meaning | Default direction |
|---|---|---|
| `association` | neutral, "is related to" | undirected |
| `flow` | something moves: data, messages, documents, process steps | directed |
| `dependency` | A needs B, A calls B | directed |

`direction` accepts `to`, `none`, `both`. There is deliberately no `from`: an edge is reversed
by swapping `source` and `target`, so arrow direction and field order can never disagree.

Further types (`trigger`, `is_a`, `publishes`) are specific to a diagram `kind` and should
only be added once a `kind` is drawn more tightly.

### notes

| Field | Required | Meaning |
|---|---|---|
| `id` | yes | opaque, generated, stable |
| `anchor` | yes | ID of **exactly one** node or **one** edge |
| `text` | yes | content |

Exactly one anchor — allowing several would introduce a second kind of edge by the back door.

---

## 4. Layer 2 — Presentation

Persistence: JSON, stored separately from Layer 1 (for example as a property alongside the
page) so the separation is physical and Layer 1 can be indexed on its own.

```json
{
  "format": "diagram-layout/1",
  "nodes": {
    "n_3f1": { "x": 70,  "y": 90,  "w": 270, "h": 180 },
    "n_a27": { "x": 60,  "y": 75,  "w": 150, "h": 62, "shape": "rect" },
    "n_5c8": { "x": 440, "y": 70,  "w": 300, "h": 300 },
    "n_b40": { "x": 75,  "y": 60,  "w": 150, "h": 62, "shape": "rect" },
    "n_92d": { "x": 75,  "y": 190, "w": 150, "h": 62, "shape": "rect" },
    "n_6e3": { "x": 480, "y": 470, "w": 170, "h": 84, "shape": "ellipse" },
    "n_c1a": { "x": 860, "y": 190, "w": 180, "h": 62, "shape": "rect" }
  },
  "notes": {
    "t_2e6": { "x": 880, "y": 330, "w": 190, "h": 72 }
  }
}
```

- **Coordinates are relative to the parent node.** Children then move with their container
  automatically, and re-parenting requires exactly one conversion.
- **`shape`** accepts `rect` and `ellipse`. Containers carry no shape — how a container is
  drawn follows from `container: true` in Layer 1. Storing it here too would duplicate the
  information across the layer boundary.
- **Colour is not stored.** Appearance is derived from `tags` plus a theme
  (e.g. `planned` → dashed, `external` → muted).
- Layer 2 may be **incomplete**: a node without an entry gets an automatic position on load.
  Surplus entries, on the other hand, are errors (see I5).

---

## 5. Invariants

### Checked by the validator

I1–I6 are decidable against a single document state. The validator runs them on every load
and every change. The column says what happens when a violation is **found** — normally in
hand-edited JSON or after a bad merge, since the editor prevents all six during normal use.

| | Rule | When a violation is found |
|---|---|---|
| **I1** | Every `id` is unique within the diagram (nodes, edges and notes share one namespace) | repaired: the second occurrence is re-minted, its references follow |
| **I2** | Anything referenced as `parent` carries `container: true` | repaired: the referenced node gets `container: true` |
| **I3** | Every edge's `source` and `target` exist | the edge is discarded |
| **I4** | Every note's `anchor` exists | the note is discarded |
| **I5** | Every Layer 2 entry has a counterpart in Layer 1 | orphan, discarded |
| **I6** | `parent` relationships form no cycle | repaired: the `parent` closing the cycle is dropped, the node becomes a root |

Repair rather than rejection is the rule throughout: a page that refuses to open because of
one duplicate ID is a dead page. Every repair is reported to the author, never silent.

I1 in particular holds by construction, because the editor is the only thing that mints IDs
(§6) and checks each new one against the whole namespace. I3 and I4 never arise in the editor
either, which deletes an edge together with its endpoint and a note together with its anchor.

### Design statements

I7 and I8 read like invariants but are not checkable at a single point in time — I7 spans two
states, I8 is a definition. They constrain the editor and the reader, not the validator.

| | Statement |
|---|---|
| **I7** | IDs stay stable when the label is renamed |
| **I8** | `container: true` is a permission ("may hold children"), not a claim ("has children") |

**I7** is the most important rule in this document: once a note anchor, a model or a later
external reconciliation (§10) refers to an ID, renaming the label must not break that
reference. Opaque IDs (D13) make this nearly automatic — an ID that visibly means nothing
invites nobody to "correct" it along with the label.

---

## 6. Rules for the editor

The format only holds up if the editor cooperates. Core rule:

> **Every gesture that means something to the viewer must produce semantics.**

Otherwise you get the most dangerous case of all: a correctly drawn diagram that the machine
misreads — and nobody notices, because the picture looks right.

In practice:

- **Containment is a real parent relationship**, not overlap. Dragging a shape into a
  container sets `parent` in the model.
- **No free-floating text.** No loose text object; only a `label` on an object or a `note`
  with an anchor.
- **No purely visual frame.** A container is always a statement. Once both kinds of frame
  exist, nobody can tell which is which.
- **Edges attach to nodes**, not to coordinates.
- **Edge type is chosen while drawing**, via separate tools in the toolbar. A dropdown after
  every arrow means the default is what always ends up stored.
- **Ask for `desc` up front**, rather than hiding it in a sub-dialog. Without a `type` field,
  `desc` is the only field carrying the domain role. If it stays empty, Layer 1 is a list of
  names.
- **The editor mints every ID.** Never the author, never a model. A model proposing a change
  supplies label and relationship; identity is assigned here (§9.5). Duplicating a container
  re-mints the whole subtree **and** rewrites the edges internal to it — that is where
  implementations usually break.

This is deliberately more restrictive than draw.io — and that is exactly the difference:
draw.io lets you draw anything and therefore knows nothing.

---

## 7. Serialisation

Not `JSON.stringify(x, null, 2)`. Write a custom serialiser, because the Confluence version
history shows this very JSON to humans.

- **One object per line.** An added edge becomes an added line.
- **Stable key order**, fixed per object kind.
- **Omit empty fields.** The loader supplies the default for anything missing. For most
  fields that default is empty; for `direction` it is the default of the edge's `type`
  (see §3), not `none`.
- **Nodes in tree order**, children directly beneath their container. Array order carries no
  meaning, but it is free readability.
- **Arrays rather than maps** in Layer 1 so ordering stays stable. Layer 2 is pure lookup and
  may be a map.

Known side effect: re-parenting a node changes `parent` in Layer 1 **and** the coordinate
line in Layer 2. Unavoidable, but worth knowing before someone asks why a pure regrouping
touches two layers.

---

## 8. Decisions and their rationale

This section is the actual value of the document. The decisions are traceable; the reasoning
behind them is the first thing that gets lost otherwise.

| # | Decision | Rationale | Alternative, and why it was rejected |
|---|---|---|---|
| D1 | Positions are stored; no auto-layout | The human draws, so positions are authored. Stable incremental auto-layout is the hardest problem in this space and simply does not arise this way | The Mermaid model: layout is recomputed on every render, so one new node rearranges everything |
| D2 | JSON for persistence, no custom DSL | `JSON.parse` instead of a hand-written parser with error handling, line numbers and a migration path. Also a prerequisite for schema-validated LLM output (structured outputs / tool calls) | Custom DSL as the storage format: more compact, but needs a parser, and the Confluence editor makes broken input possible. YAML: type footguns (`no` → false), indentation-sensitive, an unquoted colon inside `desc` breaks the document |
| D3 | A textual projection as a view, not as storage — **part of the foundation, not a later addition** | Needs only a generator, no parser. Since D13 it carries a second job: it resolves IDs back to labels, so neither a human reading a diff nor a model answering a question has to dereference opaque handles. Also roughly half the tokens of JSON | Deferring it: tenable only while diagrams stay small enough to hand a model raw JSON, and it leaves edge and `parent` lines unreadable to humans |
| D4 | Mermaid is not the internal format | Mermaid's edge syntax encodes presentation (`-.->` means dotted), not meaning. There would be no way to express `dependency` or `planned`, and `desc`, `tags` and notes have nowhere to live | Mermaid **as an export** is worthwhile and acceptably lossy — a one-way street to GitHub, Markdown docs and other tools |
| D5 | The **projection's** arrow syntax is modelled on Mermaid: `->` = `direction: to`, `--` = `none`, `<->` = `both` | Direction is legible without explanation, for humans and models alike. This governs the projection (D3) only — storage keeps `type` and `direction` as fields, because an arrow glyph cannot carry the relationship kind (D4) | — |
| D6 | Shape (`rect`/`ellipse`) is purely visual, in Layer 2 | A deliberate decision against a `type` field. The cost: without `type`, validation against external sources is not possible — consistent with deferring reality reconciliation | `type` with domain roles (`service`, `datastore`, …): makes LLM output more reliable and validation possible. A candidate for later |
| D7 | Colour carries no semantics | In practice colour almost always means something (red = deprecated). Stored as a hex value, the machine can neither read nor set precisely the most important information | Solution: `tags` in Layer 1 plus a theme mapping tag → appearance |
| D8 | "Loose connection" is not an edge type | Dashed means "planned", "optional" or "uncertain" in practice — three different things. It belongs in `tags`: one mechanism instead of two | — |
| D9 | Flat node list with `parent` + `container: true` | Nesting would read more explicitly, but re-parenting a node turns into a vanishing block and an appearing block in the diff, rather than one changed line | Nested `children` arrays: structurally enforce acyclicity and the single-parent rule, and pay off from roughly four levels of hierarchy. Revisit once real diagrams get deeper |
| D10 | No `contains` array in addition to `parent` | Two fields for the same fact can drift apart, and a reader then cannot tell which one is right | Viable as a **replacement** for `parent`, but brings cycle checking and constant parent lookups in the renderer |
| D11 | `container: true` rather than inferring from existing children | Otherwise an empty container is semantically indistinguishable from an ordinary node | — |
| D12 | External reconciliation (a `ref` field) deferred | Deliberate reduction for the first pass | The strongest candidate for later, see section 10 |
| D13 | IDs are opaque and generated (`n_4a2f`), never derived from the label | A readable ID is a second copy of the label and drifts at the first rename — precisely what D10 rejects. It also makes I1 hold by construction, and removes the temptation to read meaning out of an ID, which would answer D6 through the back door. And it strengthens I7: an ID that visibly means nothing does not get "corrected" along with the label | Readable slugs (`order_service`): better raw-JSON diffs, one less dereferencing step for a model. But they need collision handling at mint time across a shared namespace, and both benefits return through the projection (D3) — which is why that moved forward |

**Rule of thumb following from D6 and D12:** any optional field the author does not fill in
as a side effect of drawing will stay empty in practice. A field that is usually empty is
worse than no field at all, because it cannot be trusted downstream.

---

## 9. Open questions

1. **Where Layer 1 and Layer 2 are stored** — page body plus property is the direction.
   Check size limits, behaviour across page versions and indexing against current platform
   documentation before building on it. The searchability argument depends on this.
2. **Editor library** — the prototype is hand-written. For the product, consider libraries
   offering absolute positioning, parent nodes and a free-form data field per node; check
   commercial licensing terms for each beforehand. Do not build yourself: snapping,
   orthogonal routing, multi-select, undo, copy/paste, keyboard handling.
3. **Orthogonal edge routing** — straight lines look restless in dense diagrams.
4. **Containers on resize** — must not shrink past their children.
5. **Write access by an LLM** — if at all, then through targeted operations (rename a node,
   add an edge), never as a rewrite of the whole document. As long as it only reads, merging,
   round-tripping and orphan handling do not arise at all.
6. **Measure rather than assume** — feed the same diagram to a model in different
   representations and ask the same questions ("what sits inside the backend", "what is
   planned", "what does X depend on"). Especially the questions that target `tags` and edge
   type.

---

## 10. Where this can go

The "an LLM can read my diagram" benefit erodes, because models interpret screenshots
increasingly well. What remains is determinism — and the real opportunity builds on it:

**Diagram decay.** Every sizeable Confluence contains architecture diagrams that have been
wrong for years, and nobody knows which ones. A machine-readable semantic layer can be
reconciled against reality: service catalogue, infrastructure state, repository structure,
other diagrams in the same space. "Diagram X names a service that no longer exists."
"Two pages contradict each other."

That needs two things deliberately deferred here: a `ref` field as external identity
(repository URL, catalogue key, ticket) and typed nodes (D6). Both are additive and break no
existing diagram.

---

## 11. Terminology

### Format concepts

- **Layer 1 / semantics** — what the diagram means. Source of truth.
- **Layer 2 / presentation** — how it looks. Derivative.
- **Containment** — nesting of nodes, semantic (not mere visual overlap).
- **Projection** — a text form generated from Layer 1 for display or for a model. Resolves
  opaque IDs back to labels and writes edges in the arrow syntax of D5. Never written back.
- **Orphan** — an entry in Layer 2 with no counterpart in Layer 1. Discarded.

### Platform vocabulary

Atlassian renamed "plugin" to "app" around 2018. The old term survives in the self-hosted
world, where the plugin framework, `atlassian-plugin.xml` and the plugin key are still
called that. Use the terms to distinguish the two codebases, consistently and on purpose:

| Term | Use for |
|---|---|
| **app** | Confluence Cloud (Forge). Also the Marketplace term for every listing, including its Data Center variants |
| **plugin** | Confluence Data Center / Server (plugin framework, Java) |
| **editor** | the product as a whole, independent of deployment — what this document specifies |

Neither *app* nor *plugin* covers both at once, so neither may be used generically. For the
thing itself, across both codebases, write **editor** or use the product name.

Three practical consequences:

- **When searching for documentation**, "Confluence plugin development" leads to Data Center
  and Server material, some of it very old. Search for "Forge app" or "Confluence Cloud app"
  instead. This is the most common early time sink.
- **In prose covering both platforms**, write "the Confluence app (Cloud)" and "the Confluence
  plugin (Data Center)" so it is always clear which codebase is meant.
- **When one term has to cover both**, reach for *editor*, not for whichever of the two feels
  more familiar. That slip is how the distinction erodes.

A single Marketplace listing can carry several deployment types, so both implementations
should share one product name.
