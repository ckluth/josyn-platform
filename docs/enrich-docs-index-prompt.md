# Enrich docs-index — AI enrichment prompt

**This is a standing prompt.** Load it in a Copilot CLI session to run the AI enrichment
pass on `docs-index.json`.

---

## What this prompt does

Reads `docs-index.json`, identifies entries that need semantic enrichment, classifies each
one by reading its source document, and writes the enriched fields back into the index.

---

## Vocabulary reference

All semantic fields must use values from:
`josyn-platform/architecture/docs-index-vocabulary.md`

Read that file before classifying any document. Do not invent values outside the vocabulary.

---

## Files

| File | Role |
|------|------|
| `C:\DevGit\josyn-platform\docs\docs-index.json` | The index to enrich |
| `C:\DevGit\josyn-platform\architecture\docs-index-vocabulary.md` | Closed vocabulary |

---

## Enrichment rule — economy first

Process an entry **only** if at least one of these conditions is true:

1. Any semantic field (`type`, `perspective`, `state`, `summary`) is `null`
2. `touched > enriched-at` (document changed since last enrichment)

Skip all other entries. Do not re-classify what has not changed.

---

## Instructions

1. Read `docs-index-vocabulary.md` to load the closed vocabularies.

2. Read `docs-index.json` and identify all entries that meet the enrichment rule above.
   Report the count before starting: *"N entries need enrichment."*

3. For each entry to enrich:
   a. Read the source document at `{ROOT}\{repo}\{path}`.
   b. Classify it:
      - `type` — single value from the `type` vocabulary
      - `perspective` — array of values from the `perspective` vocabulary (one or more)
      - `state` — single value from the `state` vocabulary
      - `summary` — one sentence (max 20 words) describing what the document covers
   c. Set `enriched-at` to the current UTC timestamp (ISO 8601).

4. Write the enriched entries back into `docs-index.json`.
   Preserve all mechanical fields and all other entries exactly as they are.
   Update only the semantic fields of the entries you processed.

5. Report a summary: how many entries were enriched, how many were skipped.

---

## Safety contract — read-only source documents

**Never modify any source document.**
Markdown files are read-only inputs. The only file that may be written is `docs-index.json`.
Any tool call that creates, edits, or deletes a `.md` file is a violation of this prompt.

---

## Notes

- `folder-context` is a hint, not a constraint. Use it to guide classification, but read
  the document — folder names can be misleading.
- A document in `thinking/` is almost always `state: thinking`, but verify.
- `perspective` is an array. Assign every perspective that genuinely applies. Minimum one.
- `summary` must be one sentence, no longer than 20 words. Plain English, no jargon.
- If a document is empty or stub-only, set `state: thinking` and `summary` to a
  description of what it appears intended to cover.
- ROOT is `C:\DevGit` (or the value resolved by `cfg-detect-root.cmd` for this machine).
