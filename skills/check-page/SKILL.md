---
description: Check whether a specific file or URL was successfully crawled and indexed in a CustomGPT.ai agent by scanning all pages.
triggers:
  - "check page"
  - "was indexed"
  - "was crawled"
  - "find page"
  - "page status"
  - "check if indexed"
  - "check if crawled"
  - "did it index"
  - "is this file indexed"
  - "was this file crawled"
  - "find this file"
  - "lookup page"
---

# check-page

Find a specific file or URL in a CustomGPT.ai agent's page list and report its crawl and index status.

## Critical Rules
- ALWAYS read `.customgpt-meta.json` to get `agent_id`
- Match against BOTH `filename` (uploaded files) AND `page_url` (web sources)
- Use case-insensitive substring matching — the user may give a partial name
- Paginate until a match is found or there are no more pages

---

## Step 1 — Get API Key

Check in order:
1. Env var `$CUSTOMGPT_API_KEY`
2. `.env` file — walk up from `$PWD` to `/` looking for a file containing `CUSTOMGPT_API_KEY=`
3. Read `~/.claude/customgpt-config.json` and extract the `apiKey` field

Use the first non-empty value. If none found, ask the user.

---

## Step 2 — Read Meta File

Walk up from `$PWD` to find `.customgpt-meta.json`. Extract `agent_id`.

If not found:
> "No agent found in this directory tree. Run `/create-agent` first."

---

## Step 3 — Get Search Term

If the user provided a filename or URL, use it. Otherwise ask:
> "Which file or URL do you want to look up?"

---

## Step 4 — Paginate and Search

Fetch pages 100 at a time, checking each entry's `filename` and `page_url` for a case-insensitive match against the search term. Stop when a match is found or `next_page_url` is null.

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages?page=${PAGE}&per_page=100" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

Warn the user if more than 500 entries have been scanned with no match.

---

## Step 5 — Report

**If found:**

> **Page found:** {filename or page_url}
>
> | Field | Value |
> |---|---|
> | ID | {id} |
> | Crawl status | {crawl_status} |
> | Index status | {index_status} |
> | Created | {created_at} |

Status interpretations:
- `crawl_status: queued` → not yet crawled
- `crawl_status: crawled` + `index_status: queued` → crawled but not yet indexed
- `crawl_status: crawled` + `index_status: indexed` → fully indexed
- either `failed` → suggest `/refresh-agent`

**If not found:**
> "No page matching '{search}' found. The file may not have been uploaded yet — run `/index-files` to add it."
