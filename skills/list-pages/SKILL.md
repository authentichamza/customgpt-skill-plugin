---
name: customgpt-ai-rag:list-pages
description: List all indexed documents in the CustomGPT.ai agent with their IDs, filenames or URLs, and crawl/index status. Paginates automatically.
argument-hint: "[optional: filter by status — ok | failed | queued]"
allowed-tools: Bash, Read
triggers:
  - "list pages"
  - "list indexed pages"
  - "list documents"
  - "list indexed documents"
  - "list indexed files"
  - "show pages"
  - "show indexed files"
  - "show indexed documents"
  - "show documents"
  - "what's indexed"
  - "what is indexed"
  - "what files are indexed"
  - "list all pages"
  - "get pages"
  - "show knowledge base"
  - "what's in the knowledge base"
  - "show all indexed"
---

# list-pages

List every document in the agent's knowledge base — their IDs, filenames or URLs, and crawl/index status. Useful for finding page IDs before running `/delete-page` or `/reindex-file`.

## Rules

- ALWAYS read `.customgpt-meta.json` to get `agent_id`
- Paginate until all documents are collected — use `data.pages.last_page`
- Claude reads and parses the JSON — do not use shell scripts or jq

---

## Step 1 — Resolve API Key

Follow the lookup procedure in `skills/_shared/api-key.md`. Store the result as `$API_KEY`.

---

## Step 2 — Read Meta File

Walk up from `$PWD` toward `/`, looking for `.customgpt-meta.json`. Extract `agent_id`.

If not found:
> "No agent found in this directory tree. Run `/create-agent` to set one up first."

---

## Step 3 — Check for Status Filter

If the user specified a status filter (e.g., "list failed pages", "list pages with status queued"), note the filter value. Valid values: `ok`, `failed`, `queued`, `limited`, `n/a`.

To apply the filter, append `&crawl_status={filter}` or `&index_status={filter}` to the API URL below.

---

## Step 4 — Fetch All Pages

Fetch page 1:

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages?page=1&limit=100&order=asc" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

From the response, read:
- `data.pages.data[]` — the array of document objects
- `data.pages.last_page` — total number of API pages
- `data.pages.total` — total document count

For each document, note: `id`, `filename`, `page_url`, `crawl_status`, `index_status`, `is_file`

If `last_page > 1`, repeat for pages 2 through `last_page`:

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages?page=${PAGE}&limit=100&order=asc" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

Collect all document objects across all pages.

---

## Step 5 — Display Results

Present a table. For `is_file: true` entries, show `filename` in the Name column; for URL entries, show `page_url`.

| ID | Type | Name | Crawl | Index |
|----|------|------|-------|-------|
| 12345 | file | skills/query-agent/SKILL.md | ok | ok |
| 12346 | url  | https://example.com/docs    | ok | queued |
| 12347 | file | README.md                   | ok | failed |

Call out documents that need attention:
- Flag rows where `crawl_status` or `index_status` is `failed` with a note: "Run `/reindex-file <filename>` to retry."
- Flag `queued` rows with: "Still processing — check again with `/check-status`."
- Flag `limited` rows with: "Plan limit reached for this document."

Show totals:
> Total: **{N}** document(s) — {ok_count} ready, {queued_count} queued, {failed_count} failed

Suggest next steps:
> Use `/delete-page <id>` to remove a document, `/reindex-file <filename>` to refresh a specific file, or `/check-page <name>` to look up a document by name.
