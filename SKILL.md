---
name: notion-mcp-playbook
description: "Use before calling any notion-* MCP tool, or when editing Notion pages with CJK content. Covers undocumented quirks: DDL parser limits, write-format ≠ read-format, timeout handling, code-point failures in update_content."
---

# Notion MCP implementation guide

This skill captures Notion MCP tool quirks that cause silent wrong behaviour or confusing errors. Each section describes the quirk, why it happens, and the working pattern.

This skill covers only Notion MCP tool mechanics. It does not prescribe how to organize your content (page structure, indexing, naming) — those are yours to decide.

## Quick Reference

| Intent | Tool & key constraint | Section |
|---|---|---|
| Create database | `notion-create-database`; no `(` `)` in option names; STATUS → SELECT | §1 |
| Add database rows | `notion-create-pages`; parent = `data_source_id`; multi-select = JSON string | §3, §6.1 |
| Inspect database schema | `notion-fetch` on `collection://` URL | §6.1 |
| Small text edit on a page | `notion-update-page` `update_content`; `old_str` copied verbatim from fetch | §7.3 |
| Large rewrite of a page | `notion-update-page` `replace_content`; carry child tags | §7, §7.2 |
| Update properties only | `notion-update-page` `update_properties`; `content_updates = []` | §6.3 |
| Enumerate all rows in a database | NOT search; multiple targeted `data_source_url` searches | §4.2 |
| Inline link to another page | Markdown link syntax, NOT `<page url="...">` | §2.1 |

For error-driven lookup (you have a symptom), see §9 Error triage. For tool-by-tool last checks before calling, see §10 pre-flight.

## 1. DDL parser limits (creating databases)

Notion's `notion-create-database` accepts SQL DDL via the `schema` parameter. The parser has two known limits.

### 1.1 Option names cannot contain `(` or `)`

The DDL parser uses `(` to mark the start of a type's option list. An option string containing `(` makes the parser think the option list ended early, producing `Expected column name in double quotes, got "("`.

**Workaround**: replace `()` with `[]` or a dash.

```
BAD:  MULTI_SELECT('Status (active)':green, ...)
GOOD: MULTI_SELECT('Status [active]':green, ...)
GOOD: MULTI_SELECT('Status - active':green, ...)
```

This applies to both `SELECT` and `MULTI_SELECT`. Test the names with `(` before submitting — the error message points at a position in the DDL string, not at the offending option, so it is not a reliable locator.

### 1.2 STATUS does not accept inline options

`STATUS('opt':color, ...)` fails. The DDL only accepts `STATUS` (no option list). This means you cannot create a Status property with three pre-defined options like `unknown / in_progress / done` and their To-do/In progress/Complete group assignments via DDL.

**Two workarounds**:

- **Use `SELECT` instead.** Same UI affordance for filtering and grouping; loses the To-do/In progress/Complete swim-lane semantics. Fine for most checklists.
- **Create empty `STATUS`, then add options via UI.** Notion will auto-create options as rows are written, but they all land in the To-do group with random colours. The user has to drag them into the right group. Annoying for batch creation.

Default to `SELECT` unless the user explicitly needs Status property features (the To-do/In progress/Complete group behaviour and the visual progress UI).

## 2. Write-format ≠ read-format

Many things `notion-fetch` returns in markdown are **rendered representations**, not the literal markdown that was written to the page. Writing those representations back as input does not round-trip.

### 2.1 `<page url="..."></page>` is read-only

When `notion-fetch` shows a page's content, child pages appear as `<page url="https://www.notion.so/{id}">Title</page>`. This is Notion's render of the parent-child relationship.

**Writing** `<page url="..."></page>` into a page's content via `notion-update-page` does not place an inline link at that location. Notion parses the tag, treats it as a child page reference, and re-orders it to the top of the page as a child page block. The original literal text is gone — a subsequent fetch shows the rendered child reference at the top, not where you wrote it.

**Working pattern for inline page links inside a content section**: use markdown link syntax.

```
- [Page Title](https://www.notion.so/{page_id})
```

This stays as inline text where you wrote it.

### 2.2 Angle-bracket tags from a fetch are render-output

Treat anything in angle brackets that came from a fetch as render-output unless explicitly documented as write-input. The two that actually get misused as write input: `<ancestor-path>` (page hierarchy, computed from parent relationships) and `<data-source url="collection://...">` (data-source binding, set at create time — not settable by writing the tag).

### 2.3 Other read-format quirks

- `<details>` toggle inner content is prefixed with `\t` in fetch output. This is a storage artifact, not a content change — don't misread it as drift from your last write.
- Code blocks without an explicit language are auto-detected by Notion (observed: detected as javascript). For plain text content, write ` ```text ` or ` ```plain ` explicitly to avoid wrong syntax highlighting.

## 3. Multi-select write payload

Multi-select properties are written as a **JSON array string**, not as a literal array.

```
GOOD: properties: { "Tags": "[\"Option A\", \"Option B, with comma\"]" }
BAD:  properties: { "Tags": ["Option A", "Option B"] }              // literal array; connector expects a string
BAD:  properties: { "Tags": "Option A, Option B" }                  // comma-joined; breaks on option names containing commas
```

Notion parses the string as JSON. This format survives option names containing commas, brackets, or other characters that would break a comma-joined string. After write, `notion-fetch` returns the property as a real array `["Option A", "Option B, with comma"]` — read-format ≠ write-format strikes again.

If the option name does not exist in the schema, the write fails. Always cross-check option names against the schema (fetch the data source first if unsure).

## 4. Search and listing limits

### 4.1 Page size cap, and no usable pagination

`notion-search` caps `page_size` at 25. The non-obvious part: the tool schema exposes **no pagination parameter** — no cursor, offset, or page token. Whether the backend returns a continuation token in practice is unverified. Treat search as un-paginatable; widen coverage with multiple targeted queries instead (see §4.2).

### 4.2 Search is for finding, not listing

Search ranks by semantic relevance. It is **not** the right tool to "list every row in a database". For full enumeration:

- `notion-fetch` on a `collection://` URL returns the schema and SQLite definition but not the rows.
- This connector does **not** expose a `query_data_sources` tool, even though other tool descriptions reference it. Do not attempt to call it. The only row-reaching mechanism is `notion-search` with `data_source_url` set to the `collection://` URL — still relevance-ranked, still capped at 25.
- Run multiple targeted `data_source_url`-scoped searches with different queries to cover the row space. Full enumeration of a large database is not guaranteed.

## 5. Parent-child relationships

### 5.1 Child page blocks are auto-rendered, not maintained by you

When a child page exists under a parent, Notion automatically renders a child page block at the top of the parent. You did not write it — Notion did. You cannot remove it via `notion-update-page` content edits because it is not part of the page's markdown content; it is rendered from the parent-child structural relationship.

**Implications**:

- To remove the child page block, delete (or move) the child page itself.
- The child page block appears in fetch output as `<page url="...">Title</page>` — see §2.1.
- This means parent pages always show their children at the top, even if you maintain a separate manual index lower in the page. The auto-block and the manual index can coexist.

### 5.2 Database row pages cannot be moved out of the database

A database row page's parent is the data source. Moving it out via `notion-move-pages` would detach it from the database. Don't attempt this when "cleaning up" workspace clutter — the rows are not clutter, they are data.

## 6. Update workflow patterns

### 6.1 Always fetch schema before writing rows

After `notion-create-database` returns, the response shows the schema you submitted. But before writing rows, fetch the data source once and read the actual stored option names. Reasons:

- Confirms option names were created correctly (no silent normalization)
- Gets the data-source ID in the form Notion expects (`collection://...`)
- Surfaces any schema details the create response truncated

### 6.2 For batch updates, dry-run locally first

When updating more than ~5 rows:

1. Build a local mapping `{page_id: new_property_values}`.
2. Print the mapping for the user to review before any API call.
3. Update the rows one at a time after confirmation.

This catches data errors before they hit the API and creates an audit trail of what was changed.

### 6.3 update_properties only touches what you send

`notion-update-page` with `command: "update_properties"` only modifies the properties you pass. Omitted properties are unchanged. This is safe for partial updates — no need to fetch-modify-write.

The `content_updates` parameter is required by the schema even when you're not changing content. Pass `[]`.

```
GOOD: { "command": "update_properties", "properties": {...}, "content_updates": [] }
BAD:  { "command": "update_properties", "properties": {...} }       // content_updates omitted; schema rejects
```

### 6.4 Write verification loop

Required after any of: batch writes (>5 rows), `replace_content`, `update_content` after timeout.

1. Fetch a representative sample:
   - Batch writes: 2-3 random rows
   - `replace_content`: the whole page
   - `update_content` after timeout: just the affected section
2. Compare against intended state. Note: the `as of <timestamp>` in fetch output is a snapshot — in rare cases a fetch immediately after a write shows stale data.
3. If verification fails: wait briefly and refetch once more before retrying the write (rules out snapshot staleness).
4. **Stop after one retry cycle** unless the error pattern changes between attempts. Repeated identical failures mean the write is genuinely broken (schema mismatch, code-point issue, permission), not stale snapshots. Re-reading docs or asking the user is more productive than further retries.

## 7. Content commands (`update_content` / `replace_content`)

Picking the wrong command from the start is the most common cause of wasted retries in this connector. Decide before writing:

| Situation | Use | Reason |
|---|---|---|
| Local edit, page stays mostly intact | `update_content` | Targeted, low risk |
| Large rewrite (~30%+ of page changes) | `replace_content` | Simpler backend; more reliable on large payloads |
| Bulk cleanup across multiple non-adjacent sections | `replace_content` | Stacked `update_content` chains compound the code-point risk in §7.3 |
| `replace_content` on a parent page with children | `replace_content` + carry child tags into `new_str` | See §7.2 |

`update_content` is for local, well-targeted edits with an unambiguous `old_str`. Chaining 7+ `update_content` calls to delete sections that would fit in one `replace_content` rewrite is a smell — each call has overhead and adds one more chance for a Han variant code-point mismatch (§7.3) anywhere in the chain.

### 7.1 Timeout ≠ failure

Large-payload writes (especially `update_content`) often return `notionhq_client_request_timeout` while having actually succeeded — the timeout is at the API gateway, not the Notion backend.

**Rule**: after any timeout, fetch first to verify the actual state. Only retry if fetch confirms the write did not land. Retrying without verifying causes duplicate writes.

### 7.2 `replace_content` and child pages

Child pages/databases are structural (§5.1), not part of the fetched markdown, so they are easy to forget on a parent page. `replace_content` errors rather than silently dropping them. To keep them, include the child `<page url="...">` / `<database url="...">` tags in `new_str`; to drop them, set `allow_deleting_content`. Before a `replace_content` on a parent page, fetch first and carry every child tag into `new_str` verbatim.

### 7.3 `update_content` old_str must be copied verbatim from fetch

The connector docs say to fetch first to get the snippets. The non-obvious failure mode: the LLM tends to retype from conversation memory or its own draft rather than copy verbatim from the fetch result. With Chinese content this fails silently because visually identical Han variants have different code points:

- 鏈 (U+93C8) vs 鍊 (U+930A) vs 鍗 (U+9357)

These produce `validation_error: No matches found`. When that error appears, **do not** start debugging the semantics or syntax of old_str — the root cause is almost always a code-point mismatch. Refetch, copy verbatim, and try again. If still failing, switch to `replace_content` (observing the §7.2 child-page caveat).

## 8. Multi-stage operations: keep the user oriented

Building a database from scratch involves many turns: parent page, child page, schema, options, rows, index. Errors in mid-stage (DDL parser failure, option-name conflict) require backtracking.

**Pattern**: at the start of each new stage, give the user a one-line summary of what completed in the previous stage and what the next stage will do. This costs almost nothing and prevents the user from losing the thread when they're reviewing a session days later.

## 9. Error triage

When a Notion MCP call fails or behaves unexpectedly, check this table first before re-reading docs or guessing.

| Symptom | Most likely cause | First action |
|---|---|---|
| `Expected column name in double quotes, got "("` | Option name contains `(` | §1.1 — replace `()` with `[]` or `-` |
| STATUS create rejects inline options | DDL parser does not accept options for STATUS | §1.2 — use SELECT or empty STATUS |
| `validation_error: No matches found` on `update_content` | Han variant code-point mismatch in `old_str` | §7.3 — refetch and copy verbatim |
| `notionhq_client_request_timeout` | Gateway timeout; write may have succeeded | §7.1 — fetch before retrying |
| Multi-select write rejected | Option name not in schema, or literal array instead of JSON string | §3 |
| Properties unchanged after `update_properties` | `content_updates` missing `[]`, or property name typo | §6.3 |
| Child page block reappears at top of page after edit | Auto-rendered from parent-child relationship | §5.1 — not removable via content edit |
| `<page url="...">` written inline ends up at top of page | Notion re-parses tag as child page reference | §2.1 — use markdown link syntax |

## 10. Quick-reference: pre-flight checks before each tool call

Before calling, verify:

| Tool | Check |
|---|---|
| notion-create-database | No `(` `)` in option names. STATUS used without options or replaced by SELECT. |
| notion-create-pages | parent is `data_source_id` (not database_id) for database rows. Multi-select values are JSON array strings. |
| notion-update-page | content_updates passed as `[]` when not editing content. Property names match schema exactly. Content commands: large rewrite → `replace_content` (carry child `<page>`/`<database>` tags into new_str if the page has children, §7.2); small local edit → `update_content`. After timeout, fetch before retrying. |
| notion-update-data-source | Same DDL limits as create-database. |
| notion-fetch | `collection://` prefix for data sources, raw UUID for pages. |
| notion-search | page_size ≤ 25. Don't expect to enumerate full databases this way. |