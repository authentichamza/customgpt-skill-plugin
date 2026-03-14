---
name: customgpt-ai-rag:check-page
description: Look up whether a specific file or URL was successfully crawled and indexed in the CustomGPT.ai agent. Returns status, page ID, and diagnostic guidance.
argument-hint: "[filename or URL to look up]"
allowed-tools: Bash, Read
triggers:
  - "check page"
  - "find page"
  - "lookup page"
  - "page status"
  - "was indexed"
  - "was this indexed"
  - "was crawled"
  - "check if indexed"
  - "check if crawled"
  - "did it index"
  - "is this file indexed"
  - "is this indexed"
  - "was this file crawled"
  - "find this file in the index"
  - "look up this file"
  - "check this file"
---

# check-page

Find a specific file or URL in the agent's knowledge base and report its crawl and index status. Returns the page ID so you can reference it in `/delete-page` or `/reindex-file`.

## Rules

- ALWAYS read `.customgpt-meta.json` to get `agent_id`
- Match against BOTH `filename` (uploaded files) AND `page_url` (web sources)
- Use case-insensitive substring matching — the user may give a partial name or path segment
- Paginate until a match is found or all pages are checked

---

## Step 1 — Resolve API Key

Follow the lookup procedure in `skills/_shared/api-key.md`. Store the result as `$API_KEY`.

---

## Step 2 — Read Meta File

Walk up from `$PWD` toward `/`, looking for `.customgpt-meta.json`. Extract `agent_id`.

If not found:
> "No agent found in this directory tree. Run `/create-agent` to set one up first."

---

## Step 3 — Get Search Term

If the user provided a filename, path segment, or URL with the command, use it.

Otherwise ask:
> "Which file or URL do you want to look up? You can provide a partial name."

---

## Step 4 — Search by Paginating

Fetch pages 100 at a time, checking each document's `filename` and `page_url` for a case-insensitive substring match against the search term. Stop as soon as a match is found.

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages?page=${PAGE}&limit=100&order=asc" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

For each entry in `data.pages.data[]`, check `filename` and `page_url` (both lowercased) against the lowercased search term.

Continue through pages up to `data.pages.last_page`. If more than 500 entries have been scanned with no match, warn the user: "Scanned 500+ documents with no match. Try a more specific search term."

---

## Step 5 — Report

**If found:**

> **Found:** `{filename or page_url}`
>
> | Field | Value |
> |---|---|
> | Page ID | {id} |
> | Type | {is_file ? "uploaded file" : "URL"} |
> | Crawl status | {crawl_status} |
> | Index status | {index_status} |
> | Created | {created_at} |

**Status guidance:**

| Crawl | Index | Meaning |
|-------|-------|---------|
| `queued` | any | Not yet crawled — indexing is in progress |
| `ok` | `queued` | Crawled successfully, waiting to be indexed |
| `ok` | `ok` | Fully indexed and available for queries |
| `limited` | any | Plan limit reached — content was partially processed |
| `failed` | any | Crawl failed — see below |
| any | `failed` | Indexing failed — see below |

If either status is `failed`:
> "This document failed to process. Try `/reindex-file {filename}` to delete and re-upload it. If it continues to fail, check that the file format is supported and the file is not corrupted."

If status is `limited`:
> "Your plan limit was reached while processing this document. Upgrade your CustomGPT.ai plan at https://app.customgpt.ai/billing to process more content."

**If not found:**

> "No document matching '{search_term}' found in the knowledge base."
> "If you uploaded this file recently, it may still be processing — run `/check-status` to see overall progress, or `/list-pages` to browse all indexed documents."
> "To add this file, run `/index-files {filename}`."
