# Richer Structure Proposal — Design

**Date:** 2026-07-30
**Status:** Approved design, pending implementation plan
**Target:** Upstream contribution to [tranxuanthang/lyricsfile](https://github.com/tranxuanthang/lyricsfile)

## Context

Lyricsfile is a draft 1.0 YAML lyrics format by the LRCGET/LRCLIB author. In
[issue #1](https://github.com/tranxuanthang/lyricsfile/issues/1) the maintainer
stated that richer structure — optional fields for song sections and vocalist
attribution, where "simple files stay simple" — is his next focus. This design
delivers a worked-out proposal for exactly that, contributed the way the
project asks: discussion-first, one issue per question, with examples.

Prior art grounding the design:

- **Apple Music TTML** ([AMLL docs](https://amll.dev/en/guides/lyric/ttml.html),
  [amll-ttml-db spec](https://github.com/amll-dev/amll-ttml-db/blob/main/instructions/ttml-specification-en.md)):
  `ttm:agent` (declared once, referenced per line), `itunes:songPart`
  (sections), `ttm:role="x-bg"` (background vocals). Production-proven
  concepts; we adapt them to lyricsfile's flat YAML idiom.
- **Chronograph chronie** (lyricsfile's inspiration): lines + words only — no
  sections or vocalists. This proposal goes beyond the ancestor format.
- **Enhanced LRC / Walaoke** `M:`/`F:`/`D:` gender tags: crude duet markers
  that demonstrate long-standing demand for vocalist attribution.

## Goals

1. Design `sections` and vocalist/role fields as optional, backward-compatible
   additions: a file without them is byte-for-byte today's format, and a
   reader that ignores them renders today's behavior.
2. Contribute the design upstream as two focused issues with worked examples,
   following the project's contribution norms.

## Non-Goals (deferred)

- Per-line language override, translations, romanization.
- JSON Schema, validators, converters, or any tooling.
- Announcements or cross-posting to LRCGET/LRCLIB communities.
- Taking positions on YAML-vs-JSON or the trailing-space debate.

## Feature 1: Song Sections

New optional top-level `sections` sequence — a parallel timeline, not nesting.
Nesting lines inside sections (Apple's approach) would restructure the format
and break existing readers; a parallel array costs nothing to ignore.

```yaml
sections:
  - kind: verse
    label: 'Verse 1'
    start_ms: 12000
    end_ms: 45000
  - kind: instrumental
    label: 'Guitar solo'
    start_ms: 45000
    end_ms: 62000
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `kind` | string | Yes | One of `intro`, `verse`, `pre-chorus`, `chorus`, `post-chorus`, `bridge`, `refrain`, `instrumental`, `outro`, `other`. |
| `label` | string | No | Freeform display text ("Verse 2", "Sax solo"). |
| `start_ms` | integer | Yes | Start time, same rules as line timing. |
| `end_ms` | integer | No | End time; when present, `end_ms >= start_ms`. |

Rules:

- Readers treat unknown `kind` values as `other` (lets the enum grow without
  breaking readers); renderers show `label` when present, else a default for
  `kind`.
- Sections should be ordered by `start_ms` and should not overlap.
- Also answers the maintainer's open question about non-vocal events
  (instrumental breaks live here, not as fake entries in `lines`) and lets
  TTML converters preserve `songPart` data.

## Feature 2: Vocalists & Roles

Vocalists declared once in `metadata.vocalists`, referenced by id per line —
the DRY reference structure the maintainer mused about achieving with YAML
anchors, delivered as a plain field instead (the spec discourages anchors).

```yaml
metadata:
  title: 'Duet Song'
  artist: 'A & B'
  vocalists:
    - id: a
      name: 'Artist A'
      type: person
    - id: b
      name: 'Artist B'
      type: person
    - id: both
      name: 'Artist A & Artist B'
      type: group

lines:
  - text: 'I was there first'
    start_ms: 12000
    vocalist: a
  - text: 'And now we sing together'
    start_ms: 18000
    vocalist: both
  - text: '(together, together)'
    start_ms: 19500
    role: background
    vocalist: a
```

Vocalist object:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `id` | string | Yes | Unique key referenced by lines. |
| `name` | string | No | Display name. |
| `type` | string | No | `person` (default), `group`, or `other`. |

Line additions:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `vocalist` | string | No | Id of the vocalist singing this line. |
| `role` | string | No | `lead` (default) or `background`. |

Design decisions:

1. **`line.vocalist` is a single string id, not a list.** "Both sing this
   line" is a declared `group` vocalist (Apple's model). Simultaneous
   *different* parts are already representable as overlapping lines, each with
   its own `vocalist` — no new list semantics.
2. **Identity, not layout.** Duet left/right alignment stays a renderer
   concern derived from vocalist identity.
3. **Background vocals are overlapping lines with `role: background`**,
   reusing the existing overlap model instead of TTML-style nested spans,
   which don't fit lyricsfile's flat shape.

## Validation & Error Handling

Matches the spec's existing split between structural errors and semantic
warnings:

- **Structural (reject):** wrong field types; duplicate vocalist `id`s;
  missing required fields.
- **Semantic (warn, degrade gracefully):** unknown `kind` → `other`;
  `vocalist` reference with no matching `id` → line renders unattributed;
  overlapping sections.
- Files with none of the new fields remain exactly as valid as today.

## Deliverables

All work on fork branch `proposal/richer-structure`
(`prjoni99/lyricsfile`), `main` stays clean tracking upstream:

1. **Examples:** `examples/sections.lyricsfile.yaml` and
   `examples/duet.lyricsfile.yaml` (vocalists + group + background overlap).
2. **Draft spec text:** ready-to-merge additions to `SPECIFICATION.md` on the
   branch — not PR'd until the maintainer signals interest.
3. **Two issue drafts** (kept as markdown in `docs/superpowers/drafts/`):
   - Issue A — song sections.
   - Issue B — vocalists & roles (references A where they interact, but
     stands alone).
   Each: motivation → prior art → proposed fields → worked example →
   validation rules → open questions inviting discussion.

## Verification

Before anything is posted publicly:

- All example files parse with a safe YAML loader.
- Examples satisfy the proposal's own validation rules.
- Spec text and examples cross-checked for consistency.
- **User reviews and explicitly approves each issue before it is posted** —
  issues go out under the user's GitHub account (`prjoni99`).

## Sequencing

Issue A (sections) is small and low-controversy — it goes first and builds
credibility. Issue B follows once A is posted (not gated on A's acceptance).
