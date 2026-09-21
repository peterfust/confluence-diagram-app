# confluence-diagram-app

A diagram editor for Confluence where humans draw and machines read. Semantics (Layer 1) and
presentation (Layer 2) are stored as two independent layers.

**The format is specified in `docs/diagram-format.md`.** Read it before touching
serialisation, the validator, or editor behaviour — it carries the invariants and the
reasoning behind them.

## Terminology — use precisely

- **app** = Confluence Cloud (Forge)
- **plugin** = Confluence Data Center / Server (Java plugin framework)
- **editor** = the product as a whole, independent of deployment

Atlassian renamed "plugin" to "app" around 2018, but both terms are live and denote different
codebases. Neither covers both at once, so neither may be used generically — when one term
has to span both, it is "editor". In prose covering both platforms, write "the Confluence app
(Cloud)" and "the Confluence plugin (Data Center)".

## When researching Atlassian documentation

Search for **"Forge app"** or **"Confluence Cloud app"**. Searching "Confluence plugin
development" returns Data Center and Server material, some of it a decade old. This is the
most common time sink on this project.

## Design rules that are easy to violate by accident

- **Layer 1 contains no number and no word about presentation.** The moment a coordinate, a
  pixel width or a colour value lands there, the separation is lost. Check this for every
  new feature.
- **No auto-layout.** Positions are authored and stored (D1). Do not add layout computation
  as a convenience — inserting a node must never move an existing one.
- **Colour carries no semantics** (D7). Appearance is derived from `tags` plus a theme.
- **Every editor gesture that means something to the viewer must produce semantics** (§6).
  A correctly drawn diagram that the machine misreads is the worst failure mode, because the
  picture looks right and nobody notices.
- **IDs are opaque and generated** — `n_4a2f`, never `slugify(label)` (D13). The obvious
  default is the wrong one here: a label-derived ID is a second copy of the label and drifts
  at the first rename. The editor mints every ID; never the author, never a model. Never
  parse an ID.
- **The textual projection is foundational, not a later add-on** (D3). Since IDs carry no
  meaning, the projection is what makes Layer 1 readable — it resolves IDs back to labels.

## Keeping the spec honest

If a change contradicts `docs/diagram-format.md`, update the spec **in the same commit**, not
afterwards. A spec that drifts from the code is the same disease this project exists to cure.
