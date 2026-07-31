# Lyricsfile Richer-Structure Proposal Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Produce two example files, ready-to-merge spec text, and two upstream issue drafts proposing optional `sections` and vocalist/role fields for the Lyricsfile format, then post the issues after explicit user approval.

**Architecture:** Documentation-and-spec work in the `prjoni99/lyricsfile` fork on branch `proposal/richer-structure`. Examples are written first and validated with ad-hoc PyYAML scripts (they act as the tests for the spec text), then `SPECIFICATION.md` is extended to match, then the issue drafts package everything for upstream. Nothing is posted to GitHub until the user says go.

**Tech Stack:** YAML, Markdown, Python 3 + PyYAML (ad-hoc validation only — no tooling is committed), `git`, `gh` CLI.

## Global Constraints

- Working branch: `proposal/richer-structure`. Never commit to `main` (it tracks upstream `tranxuanthang/lyricsfile`).
- All example lyrics must be original/fictional (upstream CONTRIBUTING.md: no copyrighted lyrics). No PII anywhere.
- Example file style (matches existing `examples/`): UTF-8, `.lyricsfile.yaml` extension, single-quoted strings, 2-space indent, blank line between top-level keys.
- Spec prose follows `SPECIFICATION.md` conventions: "must" = required, "should" = recommended; tables with `Field | Type | Required | Description`.
- Field vocabulary (use these exact values everywhere): section `kind` ∈ `intro`, `verse`, `pre-chorus`, `chorus`, `post-chorus`, `bridge`, `refrain`, `instrumental`, `outro`, `other`; vocalist `type` ∈ `person`, `group`, `other` (default `person`); line `role` ∈ `lead`, `background` (default `lead`); line field is singular `vocalist` (single string id), metadata field is plural `vocalists`.
- No validators, converters, or schema files are committed to the repo (design non-goal). Validation scripts run as throwaway heredocs.
- Commits: conventional style, small and focused.
- HARD GATE: `gh issue create` against `tranxuanthang/lyricsfile` runs only after the user explicitly approves each issue (Task 7). Pushing the branch to the user's own fork (`prjoni99/lyricsfile`) is allowed in Task 6.

---

### Task 1: Sections example file

**Files:**
- Create: `examples/sections.lyricsfile.yaml`

**Interfaces:**
- Consumes: nothing.
- Produces: `examples/sections.lyricsfile.yaml` — quoted verbatim by Task 3 (spec consistency check) and Task 4 (issue draft A embeds its `sections` block).

- [ ] **Step 1: Confirm PyYAML is available**

Run: `python3 -c "import yaml; print(yaml.__version__)"`
Expected: a version number. If it fails with `ModuleNotFoundError`, run `python3 -m pip install --user pyyaml` and re-run the check.

- [ ] **Step 2: Write the example file**

Create `examples/sections.lyricsfile.yaml` with exactly this content:

```yaml
version: '1.0'

metadata:
  title: 'Paper Lanterns'
  artist: 'Example Artist'
  duration_ms: 200000
  language: 'en'

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

lines:
  - text: 'Paper lanterns on the water'
    start_ms: 12000
    end_ms: 16000
  - text: 'Drifting somewhere out of sight'
    start_ms: 16500
    end_ms: 20500
  - text: 'We let go of every light'
    start_ms: 40000
    end_ms: 45000

plain: |
  [Verse 1]
  Paper lanterns on the water
  Drifting somewhere out of sight

  [Chorus]
  We let go of every light
```

- [ ] **Step 3: Validate the file against the proposal's rules**

Run from the repo root:

```bash
python3 - <<'EOF'
import sys, yaml

path = 'examples/sections.lyricsfile.yaml'
doc = yaml.safe_load(open(path, encoding='utf-8'))
errors = []

KINDS = {'intro', 'verse', 'pre-chorus', 'chorus', 'post-chorus',
         'bridge', 'refrain', 'instrumental', 'outro', 'other'}

if not isinstance(doc, dict):
    errors.append('top level must be a mapping')
sections = doc.get('sections') or []
last_start = -1
last_end = 0
for i, s in enumerate(sections):
    if s.get('kind') not in KINDS:
        errors.append(f'section {i}: unknown kind {s.get("kind")!r}')
    if not isinstance(s.get('start_ms'), int) or s['start_ms'] < 0:
        errors.append(f'section {i}: start_ms must be a non-negative integer')
        continue
    if 'end_ms' in s and s['end_ms'] < s['start_ms']:
        errors.append(f'section {i}: end_ms before start_ms')
    if s['start_ms'] < last_start:
        errors.append(f'section {i}: not ordered by start_ms')
    if s['start_ms'] < last_end:
        errors.append(f'section {i}: overlaps previous section')
    last_start = s['start_ms']
    last_end = s.get('end_ms', s['start_ms'])
dur = (doc.get('metadata') or {}).get('duration_ms')
if dur is not None and sections and last_end > dur:
    errors.append('last section ends after duration_ms')

if errors:
    print('\n'.join(errors)); sys.exit(1)
print(f'PASS: {path} ({len(sections)} sections)')
EOF
```

Expected: `PASS: examples/sections.lyricsfile.yaml (5 sections)`

- [ ] **Step 4: Commit**

```bash
git add examples/sections.lyricsfile.yaml
git commit -m "feat(examples): add sections example"
```

---

### Task 2: Duet example file (vocalists + roles)

**Files:**
- Create: `examples/duet.lyricsfile.yaml`

**Interfaces:**
- Consumes: nothing.
- Produces: `examples/duet.lyricsfile.yaml` — quoted verbatim by Task 3 (spec consistency check) and Task 5 (issue draft B embeds its `vocalists`/`lines` blocks).

- [ ] **Step 1: Write the example file**

Create `examples/duet.lyricsfile.yaml` with exactly this content. Note the background line deliberately overlaps the group line — that pairing is the point of the example (it upgrades the existing `overlapping-vocals.lyricsfile.yaml`, which shows overlap but cannot say who sings):

```yaml
version: '1.0'

metadata:
  title: 'Meet Me Halfway There'
  artist: 'Ada Vale & Rio March'
  duration_ms: 185000
  language: 'en'
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

plain: |
  I took the early train alone
  I waited by the station stone
  Meet me halfway there
  (halfway there)
```

- [ ] **Step 2: Validate the file against the proposal's rules**

Run from the repo root:

```bash
python3 - <<'EOF'
import sys, yaml

path = 'examples/duet.lyricsfile.yaml'
doc = yaml.safe_load(open(path, encoding='utf-8'))
errors = []

TYPES = {'person', 'group', 'other'}
ROLES = {'lead', 'background'}

if not isinstance(doc, dict):
    errors.append('top level must be a mapping')
meta = doc.get('metadata') or {}
vocalists = meta.get('vocalists') or []
ids = [v.get('id') for v in vocalists]
if len(ids) != len(set(ids)):
    errors.append('vocalist ids are not unique')
for i, v in enumerate(vocalists):
    if not v.get('id'):
        errors.append(f'vocalist {i}: missing id')
    if 'type' in v and v['type'] not in TYPES:
        errors.append(f'vocalist {i}: unknown type {v["type"]!r}')
declared = set(ids)
for i, ln in enumerate(doc.get('lines') or []):
    if 'vocalist' in ln:
        if not isinstance(ln['vocalist'], str):
            errors.append(f'line {i}: vocalist must be a single string id')
        elif ln['vocalist'] not in declared:
            errors.append(f'line {i}: undeclared vocalist {ln["vocalist"]!r}')
    if 'role' in ln and ln['role'] not in ROLES:
        errors.append(f'line {i}: unknown role {ln["role"]!r}')

if errors:
    print('\n'.join(errors)); sys.exit(1)
print(f'PASS: {path} ({len(vocalists)} vocalists)')
EOF
```

Expected: `PASS: examples/duet.lyricsfile.yaml (3 vocalists)`

- [ ] **Step 3: Commit**

```bash
git add examples/duet.lyricsfile.yaml
git commit -m "feat(examples): add duet example with vocalists and roles"
```

---

### Task 3: Spec text additions to SPECIFICATION.md

**Files:**
- Modify: `SPECIFICATION.md` (tables in sections 2–4; insert two new sections after section 6; renumber sections 7–10 to 9–12)

**Interfaces:**
- Consumes: field vocabulary from Global Constraints; example files from Tasks 1–2 (consistency check).
- Produces: spec headings `## 7. Song Sections` and `## 8. Vocalists And Roles` — issue drafts in Tasks 4–5 tell the maintainer this drafted spec text exists on the branch.

- [ ] **Step 1: Add the `sections` row to the top-level fields table (section 2)**

In `SPECIFICATION.md`, find the table row:

```markdown
| `plain` | string | No | Unsynchronized lyrics with preserved line breaks. |
```

Insert directly after it:

```markdown
| `sections` | sequence | No | Song structure markers on their own timeline. |
```

- [ ] **Step 2: Add the `vocalists` row to the metadata table (section 3)**

Find the row:

```markdown
| `instrumental` | boolean | No | Whether the track has no vocal lyrics. Defaults to `false`. |
```

Insert directly after it:

```markdown
| `vocalists` | sequence | No | Vocalists that lines can reference by id. |
```

- [ ] **Step 3: Add `vocalist` and `role` rows to the line table (section 4)**

Find the row:

```markdown
| `words` | sequence | No | Word- or segment-level synchronization. |
```

Insert directly after it:

```markdown
| `vocalist` | string | No | Id of a declared vocalist singing this line. |
| `role` | string | No | `lead` or `background`. Defaults to `lead`. |
```

- [ ] **Step 4: Insert the two new spec sections after section 6**

Find the last paragraph of section 6:

```markdown
Readers must not assume that `plain` and `lines` contain exactly the same text.
```

Insert after it (before `## 7. Timing And Rendering`):

```markdown

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
```

- [ ] **Step 5: Renumber the four following section headings**

Make these four exact heading replacements:

- `## 7. Timing And Rendering` → `## 9. Timing And Rendering`
- `## 8. Version Handling` → `## 10. Version Handling`
- `## 9. Example` → `## 11. Example`
- `## 10. Implementation Notes` → `## 12. Implementation Notes`

- [ ] **Step 6: Verify heading order and numbering**

Run: `grep -n '^## ' SPECIFICATION.md`
Expected: sections numbered 1–12 in order, with `## 7. Song Sections` and `## 8. Vocalists And Roles` between `## 6. Plain Lyrics` and `## 9. Timing And Rendering`. No duplicate numbers.

- [ ] **Step 7: Cross-check spec vocabulary against the example files**

Run:

```bash
grep -o "kind: [a-z-]*" examples/sections.lyricsfile.yaml | sort -u
grep -o "type: [a-z]*" examples/duet.lyricsfile.yaml | sort -u
grep -o "role: [a-z]*" examples/duet.lyricsfile.yaml | sort -u
```

Expected: every printed value appears verbatim in the new spec tables (`intro`, `verse`, `chorus`, `instrumental`, `outro`; `person`, `group`; `background`).

- [ ] **Step 8: Commit**

```bash
git add SPECIFICATION.md
git commit -m "docs(spec): draft sections and vocalists additions"
```

---

### Task 4: Issue draft A — song sections

**Files:**
- Create: `docs/superpowers/drafts/issue-a-sections.md`

**Interfaces:**
- Consumes: `examples/sections.lyricsfile.yaml` (Task 1), spec heading `## 7. Song Sections` (Task 3).
- Produces: draft file whose line 1 is `Title: ...` and whose body starts at line 3 — Task 7 posts it with `gh` using exactly that split.

- [ ] **Step 1: Write the draft**

Create `docs/superpowers/drafts/issue-a-sections.md` with exactly this content:

````markdown
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
````

- [ ] **Step 2: Verify the Title/body split that Task 7 depends on**

Run: `head -1 docs/superpowers/drafts/issue-a-sections.md && sed -n '2p' docs/superpowers/drafts/issue-a-sections.md`
Expected: line 1 starts with `Title: `, line 2 is empty.

- [ ] **Step 3: Commit**

```bash
git add docs/superpowers/drafts/issue-a-sections.md
git commit -m "docs(drafts): add issue draft A (sections)"
```

---

### Task 5: Issue draft B — vocalists and roles

**Files:**
- Create: `docs/superpowers/drafts/issue-b-vocalists.md`

**Interfaces:**
- Consumes: `examples/duet.lyricsfile.yaml` (Task 2), spec heading `## 8. Vocalists And Roles` (Task 3).
- Produces: draft file with the same `Title:`/body layout; contains the placeholder token `ISSUE_A_URL` which Task 7 replaces with the real issue A URL before posting.

- [ ] **Step 1: Write the draft**

Create `docs/superpowers/drafts/issue-b-vocalists.md` with exactly this content:

````markdown
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

This pairs with the sections proposal (ISSUE_A_URL) — together they cover the
"richer structure" direction you mentioned in issue #1, and a section's lines
can each carry their own vocalist. Drafted spec text and a full example are on
the same branch:
https://github.com/prjoni99/lyricsfile/tree/proposal/richer-structure
(`SPECIFICATION.md` section 8, `examples/duet.lyricsfile.yaml`). Happy to
adjust or split this however is most useful.
````

- [ ] **Step 2: Verify the Title/body split and the placeholder token**

Run: `head -1 docs/superpowers/drafts/issue-b-vocalists.md && grep -c 'ISSUE_A_URL' docs/superpowers/drafts/issue-b-vocalists.md`
Expected: line 1 starts with `Title: `, and the count is `1`.

- [ ] **Step 3: Commit**

```bash
git add docs/superpowers/drafts/issue-b-vocalists.md
git commit -m "docs(drafts): add issue draft B (vocalists and roles)"
```

---

### Task 6: Full verification, push branch, user review gate

**Files:**
- No new files. Pushes `proposal/richer-structure` to `origin` (`prjoni99/lyricsfile`).

**Interfaces:**
- Consumes: everything from Tasks 1–5.
- Produces: pushed branch (the URLs inside both issue drafts resolve only after this push); user approval decision consumed by Task 7.

- [ ] **Step 1: Validate that every example file in the repo parses**

Run from the repo root:

```bash
python3 - <<'EOF'
import glob, sys, yaml
failed = False
for path in sorted(glob.glob('examples/*.lyricsfile.yaml')):
    try:
        doc = yaml.safe_load(open(path, encoding='utf-8'))
        assert isinstance(doc, dict) and doc.get('version') == '1.0'
        print(f'PASS: {path}')
    except Exception as e:
        print(f'FAIL: {path}: {e}'); failed = True
sys.exit(1 if failed else 0)
EOF
```

Expected: `PASS` for all 7 files (5 pre-existing + 2 new), exit 0.

- [ ] **Step 2: Confirm the branch has no changes outside the intended scope**

Run: `git diff --stat main...HEAD`
Expected: only `SPECIFICATION.md`, `examples/sections.lyricsfile.yaml`, `examples/duet.lyricsfile.yaml`, and files under `docs/superpowers/`. Nothing else.

- [ ] **Step 3: Push the branch to the user's fork**

```bash
git push -u origin proposal/richer-structure
```

Then verify the branch URL referenced by both drafts resolves:
`gh api repos/prjoni99/lyricsfile/branches/proposal/richer-structure --jq .name`
Expected: `proposal/richer-structure`

- [ ] **Step 4: STOP — present both drafts to the user**

Show the user the full text of `docs/superpowers/drafts/issue-a-sections.md`
and `docs/superpowers/drafts/issue-b-vocalists.md` and ask for explicit
approval to post. Do not proceed to Task 7 for an issue the user has not
approved. If the user requests edits, apply them, commit
(`docs(drafts): revise issue draft per review`), and re-present.

---

### Task 7: Post the issues upstream (HARD GATE: user-approved only)

**Files:**
- Modify: `docs/superpowers/drafts/issue-b-vocalists.md` (replace `ISSUE_A_URL` with the real URL)

**Interfaces:**
- Consumes: user approval from Task 6 Step 4; draft files with `Title:` line-1 / body-from-line-3 layout.
- Produces: two issues on `tranxuanthang/lyricsfile`, posted under the user's account.

- [ ] **Step 1: Post issue A (sections) — only if approved**

```bash
gh issue create -R tranxuanthang/lyricsfile \
  --title "$(head -1 docs/superpowers/drafts/issue-a-sections.md | sed 's/^Title: //')" \
  --body "$(tail -n +3 docs/superpowers/drafts/issue-a-sections.md)"
```

Expected: `gh` prints the new issue URL. Record it.

- [ ] **Step 2: Fill issue A's URL into draft B and commit**

Replace the single `ISSUE_A_URL` token in
`docs/superpowers/drafts/issue-b-vocalists.md` with the URL from Step 1.

Run: `grep -c 'ISSUE_A_URL' docs/superpowers/drafts/issue-b-vocalists.md`
Expected: `0`

```bash
git add docs/superpowers/drafts/issue-b-vocalists.md
git commit -m "docs(drafts): link issue A URL into draft B"
git push
```

- [ ] **Step 3: Post issue B (vocalists) — only if approved**

```bash
gh issue create -R tranxuanthang/lyricsfile \
  --title "$(head -1 docs/superpowers/drafts/issue-b-vocalists.md | sed 's/^Title: //')" \
  --body "$(tail -n +3 docs/superpowers/drafts/issue-b-vocalists.md)"
```

Expected: `gh` prints the new issue URL.

- [ ] **Step 4: Report back**

Tell the user both issue URLs and update the Tolaria note
`lyricsfile/lyricsfile-richer-structure-proposal.md` with the posted issue
links and date (append, don't rewrite conclusions).
