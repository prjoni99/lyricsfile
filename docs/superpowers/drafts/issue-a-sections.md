Title: Proposal: optional top-level `sections` for song structure

## Motivation

`OPEN_QUESTIONS.md` asks whether `lines` should support non-vocal events such
as section markers. I'd like to propose an answer that keeps `lines` purely
vocal: an optional top-level `sections` sequence on its own timeline.

Today song structure only exists as text conventions inside `plain`
(`[Verse 1]`, `[Chorus]`). That works for unsynced display, but the structure
is invisible to synced rendering: a player can't show "instrumental break"
during a solo, can't build a section-based scrubber, and converters from
formats that carry structure (Apple Music TTML's `itunes:songPart`) have to
throw that data away.

Because `sections` is a separate optional field, a file without it is
byte-for-byte a valid 1.0 file, and a reader that ignores it behaves exactly
as today. Simple files stay simple.

## Prior art

- **Apple Music TTML** marks up song parts with `itunes:songPart`
  (Verse, Chorus, Bridge, Instrumental, ...) by nesting lines inside `div`
  elements. Nesting doesn't fit Lyricsfile's flat `lines` array — and flat is
  worth keeping — so this proposal uses a parallel timeline instead.
- **Plain-text/LRC convention**: bracketed `[Verse 1]` labels. Human-readable
  but unstructured and unsynced.

## Proposed fields

Each item in `sections`:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `kind` | string | Yes | One of `intro`, `verse`, `pre-chorus`, `chorus`, `post-chorus`, `bridge`, `refrain`, `instrumental`, `outro`, `other`. |
| `label` | string | No | Freeform display text ("Verse 2", "Sax solo"). |
| `start_ms` | integer | Yes | Same timing rules as lines. |
| `end_ms` | integer | No | When present, `end_ms >= start_ms`. |

Rules:

- Readers treat an unknown `kind` as `other`, so the enum can grow without
  breaking readers. Renderers show `label` if present, else a default for
  `kind`.
- Sections should be ordered by `start_ms` and should not overlap.
- Structural errors (wrong types) reject the file; semantic issues (unknown
  `kind`, overlap) degrade gracefully with warnings — matching the spec's
  existing structural/semantic split.

## Example

```yaml
sections:
  - kind: intro
    start_ms: 0
    end_ms: 12000
  - kind: verse
    label: 'Verse 1'
    start_ms: 12000
    end_ms: 40000
  - kind: chorus
    start_ms: 40000
    end_ms: 62000
  - kind: instrumental
    label: 'Guitar solo'
    start_ms: 62000
    end_ms: 90000
  - kind: outro
    start_ms: 90000
    end_ms: 200000
```

## Open questions

1. Should `kind` be a closed enum with `other` as the escape hatch (proposed),
   or fully open strings with a recommended set?
2. Should overlapping sections be structurally invalid instead of a warning?
3. Should section times be bounded by `metadata.duration_ms`?

I've drafted ready-to-merge spec text and a full example file on a branch, in
case any of this is useful:
https://github.com/prjoni99/lyricsfile/tree/proposal/richer-structure
(`SPECIFICATION.md` section 7, `examples/sections.lyricsfile.yaml`). Happy to
open a PR, rework the design, or drop parts of it based on this discussion.
