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

## Confirmation discipline — applies to every step

Before every step — reads and writes alike — briefly state:
- what you are about to do, and
- why it is needed at this point in the process.

Then **stop and wait for explicit confirmation** before executing.

This applies to all tool calls without exception: reading the index, reading the vocabulary,
reading each source document, and writing the index. No step may be executed speculatively
or batched without prior approval for that specific step.

The two explicit confirmation gates in Phase 1 and Phase 3 are specific checkpoint summaries;
they do not replace this per-step rule.

---

## Identification — which entries need processing

Read `docs-index.json`. For each entry, apply these two checks — both require timezone-aware
UTC comparison (parse all timestamps as UTC before comparing):

- **Null path:** any of `type`, `perspective`, `state`, `summary` is `null`.
- **Stale path:** all four semantic fields are present **and** `touched` (parsed as UTC) is
  later than `enriched-at` (parsed as UTC).

An entry that fails both checks is up-to-date — skip it entirely.
An entry with `enriched-at: null` always qualifies (treat as infinitely old).

Report the count before starting: *"N entries need enrichment (X null, Y stale)."*

---

## Processing — four explicit phases

---

### Phase 1 · Load vocabulary and identify entries

1. Read `docs-index-vocabulary.md` to load the closed vocabularies.
2. Identify all entries using the rules above.
   **Implementation note:** Do NOT read `docs-index.json` in visual chunks.
   Load it in a single PowerShell call using `ConvertFrom-Json` and run the
   null/stale comparison in that same call. The file can exceed 50 KB — chunked
   `view` reads are slow, waste round-trips, and require manual token-burning
   analysis. One structured query is the correct tool for this mechanical task.
3. **Report** — print the identification result in this exact form:

   > I found N documents to enrich:
   > - `{repo}/{path}` (null / stale)
   > - …
   >
   > Shall I continue with analysing the diffs and updating the attributes?

4. **Confirmation gate** — stop here and wait for explicit approval.
   Do not read any source document until the human confirms.

---

### Phase 2 · Analyse entries (read and classify, no writes yet)

For each entry in the identified set, in order:

**Null-path entry:**

a. Print: `[X/N] Reading: {repo}/{path} (null-path)`
b. Read the source document at `{ROOT}\{repo}\{path}`.
c. Derive all four semantic fields:
   - `type` — single value from the `type` vocabulary
   - `perspective` — array of values from the `perspective` vocabulary (one or more)
   - `state` — single value from the `state` vocabulary
   - `summary` — one sentence (max 20 words) describing what the document covers
d. **Report** the derived values immediately:

   ```
   [X/N] {repo}/{path}
     type        : <value>
     perspective : [<values>]
     state       : <value>
     summary     : "<text>"
     enriched-at : <UTC timestamp>
   ```

**Stale-path entry:**

a. Print: `[X/N] Reading: {repo}/{path} (stale-path)`
b. Read the source document at `{ROOT}\{repo}\{path}`.
c. Check each existing semantic field for accuracy.
d. **Report** the review outcome immediately, showing old → new for any field that changes,
   and `(unchanged)` for fields that remain accurate:

   ```
   [X/N] {repo}/{path}
     type        : <old> → <new>  |  (unchanged)
     perspective : <old> → <new>  |  (unchanged)
     state       : <old> → <new>  |  (unchanged)
     summary     : <old> → <new>  |  (unchanged)
     enriched-at : <UTC timestamp>  (bumped regardless)
   ```

Continue until all N entries have been analysed. Do not write anything to `docs-index.json`
during this phase.

---

### Phase 3 · Pre-write confirmation

After all entries are analysed, print a consolidated change plan:

```
Analysis complete. Planned writes to docs-index.json:
  Entries with field changes  : A
  Entries with enriched-at bump only (no field changes): B
  Entries skipped (up-to-date): Z
  ─────────────────────────────
  Total entries to write      : A+B

Proceed with writing docs-index.json?
```

**Confirmation gate** — stop here and wait for explicit approval.
Do not touch `docs-index.json` until the human confirms.

---

### Phase 4 · Write and report

1. Write all processed entries back into `docs-index.json`.
   Preserve all mechanical fields and all unprocessed entries exactly as they are.
2. **Final report:**

   ```
   docs-index.json updated.
     Entries written with field changes   : A
     Entries written (enriched-at bump only): B
     Entries skipped                      : Z
   ```

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
