Title: Proposal: optional vocalists and line roles

## Motivation

`OPEN_QUESTIONS.md` asks how multiple vocalists and simultaneous vocal parts
should be identified. The format already supports *overlapping* lines
(`examples/overlapping-vocals.lyricsfile.yaml`), but there's no way to say
*who* sings a line, or that a line is a background vocal. Duet rendering
(left/right alignment), karaoke part selection, and conversion from richer
formats all need attribution.

In issue #1 you mentioned anchors/aliases could be useful "like referencing
which singer is on each line in a DRY way" — this proposal gets that DRY
reference structure with a plain field instead, so the spec can keep
discouraging anchors.

Everything here is optional: files without vocalists remain exactly as valid
as today, and readers that ignore the fields behave as today.

## Prior art

- **Apple Music TTML**: `ttm:agent` declares each person/group once with an id
  in the head; each line references one agent. Background vocals use
  `ttm:role="x-bg"`. This proposal adapts both ideas to flat YAML.
- **Walaoke-extension LRC**: `M:`/`F:`/`D:` gender tags — crude, but long-
  standing evidence that duet attribution is a real need.

## Proposed fields

`metadata.vocalists`, declared once:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `id` | string | Yes | Unique key that lines reference. |
| `name` | string | No | Display name. |
| `type` | string | No | `person` (default), `group`, or `other`. |

Line additions:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `vocalist` | string | No | Id of the vocalist singing this line. |
| `role` | string | No | `lead` (default) or `background`. |

Design choices, so the shape stays minimal:

1. **`vocalist` is a single id, not a list.** "Both sing this line" is a
   declared `group` vocalist (Apple's model). Simultaneous *different* parts
   are already representable as overlapping lines, each with its own
   `vocalist` — no list semantics needed.
2. **Identity, not layout.** Left/right duet alignment stays a renderer
   concern derived from vocalist identity.
3. **Background vocals are overlapping lines with `role: background`** —
   reusing the existing overlap model instead of TTML-style nested spans,
   which wouldn't fit the flat line array.
4. A `vocalist` value with no matching `id` is a semantic warning (line
   renders unattributed), not a structural error.

## Example

```yaml
metadata:
  title: 'Meet Me Halfway There'
  artist: 'Ada Vale & Rio March'
  vocalists:
    - id: ada
      name: 'Ada Vale'
      type: person
    - id: rio
      name: 'Rio March'
      type: person
    - id: both
      name: 'Ada Vale & Rio March'
      type: group

lines:
  - text: 'I took the early train alone'
    start_ms: 15000
    end_ms: 19000
    vocalist: ada
  - text: 'I waited by the station stone'
    start_ms: 19500
    end_ms: 23500
    vocalist: rio
  - text: 'Meet me halfway there'
    start_ms: 24000
    end_ms: 29000
    vocalist: both
  - text: '(halfway there)'
    start_ms: 27000
    end_ms: 29500
    vocalist: ada
    role: background
```

## Open questions

1. Is a single-id `vocalist` + `group` type the right call, or would you
   rather allow a list of ids per line?
2. Should `role` grow beyond `lead`/`background` (e.g. spoken)? I'd start
   minimal.
3. Should word-level `vocalist` overrides be possible, or is line granularity
   enough for 1.0?

This pairs with the sections proposal (https://github.com/tranxuanthang/lyricsfile/issues/2) — together they cover the
"richer structure" direction you mentioned in issue #1, and a section's lines
can each carry their own vocalist. Drafted spec text and a full example are on
the same branch:
https://github.com/prjoni99/lyricsfile/tree/proposal/richer-structure
(`SPECIFICATION.md` section 8, `examples/duet.lyricsfile.yaml`). Happy to
adjust or split this however is most useful.
