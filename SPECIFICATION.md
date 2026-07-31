# Lyricsfile 1.0 Draft Specification

**Status:** Draft

This is an early draft and may change. In this document, **must** means a rule
is required, while **should** means it is recommended.

## 1. File Representation

A Lyricsfile should use the `.lyricsfile.yaml` extension.

The file must:

- use UTF-8 encoding;
- contain one YAML document;
- contain a mapping at the top level; and
- use only mappings, sequences, strings, integers, booleans, and null.

Files must not contain duplicate keys or custom YAML tags. Writers should also
avoid YAML anchors and aliases for now.

## 2. Top-Level Fields

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `version` | string | Yes | Lyricsfile format version. This draft uses `"1.0"`. |
| `metadata` | mapping | Yes | Information about the track and lyrics. |
| `lines` | sequence | No | Synchronized lyric lines. |
| `plain` | string | No | Unsynchronized lyrics with preserved line breaks. |
| `sections` | sequence | No | Song structure markers on their own timeline. |

A non-instrumental file should contain at least one of `lines` or `plain`.
An empty `lines` sequence is equivalent to having no synchronized lines.

## 3. Metadata

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `title` | string | Yes | Track title. |
| `artist` | string | Yes | Primary artist name. |
| `album` | string | No | Album name. |
| `duration_ms` | integer | No | Track duration in milliseconds. |
| `offset_ms` | integer | No | Global timing offset; exact application is unresolved. |
| `language` | string | No | Primary lyrics language as an ISO 639-1 code. |
| `instrumental` | boolean | No | Whether the track has no vocal lyrics. Defaults to `false`. |
| `vocalists` | sequence | No | Vocalists that lines can reference by id. |

`duration_ms` must not be negative.

The meaning of `offset_ms` is not settled yet. Writers should leave it out for
now unless the writer and reader already agree on how to use it.

When `instrumental` is `true`, `lines` and `plain` must be omitted or empty.

## 4. Synchronized Lines

Each item in `lines` must be a mapping with these fields:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `text` | string | Yes | Complete text of the line. |
| `start_ms` | integer | Yes | Start time in milliseconds. |
| `end_ms` | integer | No | End time in milliseconds. |
| `words` | sequence | No | Word- or segment-level synchronization. |
| `vocalist` | string | No | Id of a declared vocalist singing this line. |
| `role` | string | No | `lead` or `background`. Defaults to `lead`. |

`start_ms` must not be negative. When present, `end_ms` must be greater than or
equal to `start_ms`.

Writers should order lines by `start_ms`, from earliest to latest. Equal start
times and overlapping lines are allowed. Readers must not assume that one line
ends when another starts.

There is no standard rule for a missing `end_ms` yet. Readers may choose a
sensible display time, but should not save an invented end time back to the
file without user input.

## 5. Synchronized Words

Each item in `words` must be a mapping with these fields:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `text` | string | Yes | Text of the word or segment, including relevant whitespace. |
| `start_ms` | integer | Yes | Start time in milliseconds. |
| `end_ms` | integer | No | End time in milliseconds. |

Word `start_ms` values must not be negative. When present, a word's `end_ms`
must be greater than or equal to its `start_ms`.

Writers should order words by `start_ms`. Words may overlap, but writers should
only do this when the overlap is intentional.

Joining every `word.text` value should produce the line's `text`. Spaces needed
to rebuild the line should be included in word text. Languages that do not use
spaces do not need to add them.

When `words` is not empty, readers should use it for timed highlighting.
`line.text` remains the fallback for readers without word-sync support.

## 6. Plain Lyrics

`plain` is a separate unsynchronized version of the lyrics. It may include
section labels, blank lines, annotations, or other text not present in
`lines`.

Writers should use YAML's literal block style (`|`) for multiline lyrics.
Readers must still accept `plain` as a string written in another valid YAML
style.

Readers must not assume that `plain` and `lines` contain exactly the same text.

## 7. Song Sections

`sections` is an optional top-level sequence that describes song structure on
its own timeline. It does not change how `lines` are read, and readers that do
not support sections can ignore the field entirely.

Each item in `sections` must be a mapping with these fields:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `kind` | string | Yes | Section type. One of `intro`, `verse`, `pre-chorus`, `chorus`, `post-chorus`, `bridge`, `refrain`, `instrumental`, `outro`, or `other`. |
| `label` | string | No | Freeform display text, such as `Verse 2` or `Sax solo`. |
| `start_ms` | integer | Yes | Start time in milliseconds. |
| `end_ms` | integer | No | End time in milliseconds. |

`start_ms` must not be negative. When present, `end_ms` must be greater than
or equal to `start_ms`.

Readers must treat an unknown `kind` as `other`. This lets future revisions
add kinds without breaking existing readers. Renderers should show `label`
when present, and may otherwise show a default name for the `kind`.

Writers should order sections by `start_ms`. Sections should not overlap.

Instrumental passages inside a vocal track belong here as `instrumental`
sections rather than as entries in `lines`.

## 8. Vocalists And Roles

`metadata.vocalists` is an optional sequence that declares who sings. Lines
reference a vocalist by id using the line fields defined in section 4. Files
without vocalists remain valid, and readers that do not support vocalists can
ignore these fields.

Each item in `metadata.vocalists` must be a mapping with these fields:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `id` | string | Yes | Unique key that lines reference. |
| `name` | string | No | Display name. |
| `type` | string | No | `person`, `group`, or `other`. Defaults to `person`. |

Vocalist ids must be unique within a file.

A line's `vocalist` field takes a single id. When several vocalists sing the
same line together, declare a vocalist with type `group` and reference it.
When different parts are sung at the same time, use overlapping lines, each
with its own `vocalist`.

Background vocals are ordinary lines with `role: background`, usually
overlapping a lead line. Renderers may style them differently, for example
smaller or dimmed.

A `vocalist` value that does not match a declared id is a semantic error, not
a structural one. Readers should warn and render the line without
attribution.

## 9. Timing And Rendering

All timestamps are integer milliseconds from the start of the audio track.

Lines may overlap. A player may display and highlight several active lines at
the same time.

Rendering layout, animation, colors, and end-time inference are outside the
scope of this draft.

## 10. Version Handling

For this draft, `version` must be the exact string `"1.0"`.

Readers must not treat an unknown version as version 1.0. They may reject the
file or preserve it without displaying it.

How version 1.0 can grow without breaking readers is still being discussed.

## 11. Example

```yaml
version: '1.0'

metadata:
  title: 'Example Song'
  artist: 'Example Artist'
  language: 'en'
  instrumental: false

lines:
  - text: 'Hello world'
    start_ms: 1200
    end_ms: 2800
    words:
      - text: 'Hello '
        start_ms: 1200
        end_ms: 1900
      - text: 'world'
        start_ms: 1900
        end_ms: 2800

plain: |
  Hello world
```

Additional examples are available in [`examples/`](examples/).

## 12. Implementation Notes

- Parse YAML with safe loading enabled.
- Reject duplicate keys rather than choosing one value silently.
- Apply reasonable limits to document size, nesting depth, and alias expansion.
- Keep the original document when lossless editing is not possible.
- Report structural errors separately from semantic timing warnings.
