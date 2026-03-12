---
description: List all indexed documents (pages) in the CustomGPT.ai agent with their IDs, filenames, URLs, and crawl/index status.
triggers:
  - "list pages"
  - "list indexed pages"
  - "list documents"
  - "list indexed documents"
  - "show pages"
  - "show indexed files"
  - "show documents"
  - "what's indexed"
  - "what is indexed"
  - "what files are indexed"
  - "list all pages"
  - "get pages"
---

# list-pages

List all documents in the agent's knowledge base. Each document has an `id`, `filename` (for uploaded files), `page_url`, `crawl_status`, and `index_status`. Paginate until all documents are collected.

## Critical Rules
- ALWAYS read `.customgpt-meta.json` to get `agent_id`
- Paginate — keep fetching until `next_page_url` is null or the returned count is less than the limit
- Claude reads and parses the JSON response — do NOT use shell scripts or python

---

## Step 1 — Get API Key

Check in order:
1. Env var `$CUSTOMGPT_API_KEY`
2. `.env` file — walk up from `$PWD` to `/` looking for a file containing `CUSTOMGPT_API_KEY=`
3. Read `~/.claude/customgpt-config.json` and extract the `apiKey` field

Use the first non-empty value as `$API_KEY`. If none found, ask the user.

---

## Step 2 — Read Meta File

Walk up from `$PWD` to find `.customgpt-meta.json`. Extract `agent_id`.

If not found:
> "No agent found in this directory tree. Run `/create-agent` first."

---

## Step 3 — Fetch First Page

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages?page=1&limit=100&order=asc" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

From the response read:
- `data.pages.data[]` — the array of document objects
- `data.pages.last_page` — total number of pages
- `data.pages.total` — total document count

For each document in `data.pages.data[]`, note: `id`, `filename`, `page_url`, `crawl_status`, `index_status`, `is_file`

---

## Step 4 — Fetch Remaining Pages (if any)

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

Present a table:

| ID | Type | Filename / URL | Crawl | Index |
|----|------|----------------|-------|-------|
| 12345 | file | skills/index-files/SKILL.md | ok | ok |
| 12346 | url  | https://example.com/page    | ok | queued |

- For `is_file: true` entries, show `filename` in the Filename/URL column
- For URL entries, show `page_url`
- Highlight rows where `crawl_status` or `index_status` is `failed` or `queued`
- Show total: "Total: {N} document(s)"

> "Use `/delete-page <id>` to remove a document, or `/reindex-file <filename>` to delete and re-upload a specific file."
