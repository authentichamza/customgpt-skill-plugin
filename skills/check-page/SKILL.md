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

Find a specific file or URL in a CustomGPT.ai agent's page list and report its crawl and index status. Paginates through all pages (100 per request) until a match is found or all pages are exhausted.

## Critical Rules
- ALWAYS read `.customgpt-meta.json` to get `agent_id`
- Match against BOTH `filename` (for uploaded files) AND `page_url` (for web sources)
- Use case-insensitive substring matching — the user may give a partial name
- Paginate until a match is found or `next_page_url` is null — do NOT assume it fits on page 1
- Show `crawl_status` AND `index_status` — both matter

---

## Step 1 — Get API Key

```bash
# 1. Already-exported env var
echo "${CUSTOMGPT_API_KEY:-}"

# 2. .env file in current or parent directories
dir="$PWD"
while [ "$dir" != "/" ]; do
  if [ -f "$dir/.env" ]; then
    grep -E '^export\s+CUSTOMGPT_API_KEY=|^CUSTOMGPT_API_KEY=' "$dir/.env" \
      | sed 's/^export\s*//' | sed 's/CUSTOMGPT_API_KEY=//' | tr -d '"'"'" | head -1
    break
  fi
  dir=$(dirname "$dir")
done

# 3. Saved config file
cat ~/.claude/customgpt-config.json 2>/dev/null
```

Use the first non-empty value as `$API_KEY`. If missing, ask the user for it.

---

## Step 2 — Read Meta File

```bash
dir="$PWD"
while [ "$dir" != "/" ]; do
  [ -f "$dir/.customgpt-meta.json" ] && cat "$dir/.customgpt-meta.json" && break
  dir=$(dirname "$dir")
done
```

Extract `agent_id`. If not found:
> "No agent found in this directory tree. Run `/create-agent` first."

---

## Step 3 — Get Search Term

If the user provided a filename or URL with the skill invocation, use it directly as `$SEARCH`.
Otherwise ask:
> "Which file or URL do you want to look up? Enter a filename (e.g. `SKILL.md`) or a partial URL."

Lowercase `$SEARCH` for case-insensitive matching.

---

## Step 4 — Paginate and Search

Fetch pages 100 at a time. For each page, check whether any entry's `filename` or `page_url` contains `$SEARCH` (case-insensitive). Stop as soon as a match is found or there are no more pages.

**Important:** The pages API does not support search — every page must be fetched and scanned locally. With large agents this may require many requests. Warn the user if more than 5 pages (500 entries) have been scanned with no match yet.

```bash
SEARCH="$SEARCH"   # set from Step 3
PAGE=1
FOUND=""

while true; do
  RESP=$(curl -s --request GET \
    --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages?page=${PAGE}&per_page=100" \
    --header "Authorization: Bearer ${API_KEY}" \
    --header "accept: application/json")

  # Extract filename and page_url fields with their surrounding context using line-by-line grep
  # Check for a match (case-insensitive)
  MATCH=$(echo "$RESP" | python3 -c "
import sys, json, os
data = json.load(sys.stdin)
search = os.environ.get('SEARCH','').lower()
pages = data.get('data', {}).get('pages', {}).get('data', [])
for p in pages:
    filename = (p.get('filename') or '').lower()
    url = (p.get('page_url') or '').lower()
    if search in filename or search in url:
        print(json.dumps(p))
        break
" SEARCH="$SEARCH" 2>/dev/null)

  if [ -n "$MATCH" ]; then
    FOUND="$MATCH"
    break
  fi

  # Check if there is a next page
  NEXT=$(echo "$RESP" | python3 -c "
import sys, json
data = json.load(sys.stdin)
print(data.get('data', {}).get('pages', {}).get('next_page_url') or '')
" 2>/dev/null)

  [ -z "$NEXT" ] && break

  PAGE=$((PAGE + 1))

  # Warn after 5 pages scanned with no match
  if [ "$PAGE" -eq 6 ]; then
    echo "Still searching... scanned 500 entries so far."
  fi
done

echo "$FOUND"
```

---

## Step 5 — Report

**If a match was found**, present the details:

> **Page found:** {filename or page_url}
>
> | Field | Value |
> |---|---|
> | ID | {id} |
> | Filename | {filename} |
> | URL | {page_url} |
> | Crawl status | {crawl_status} |
> | Index status | {index_status} |
> | Is file | {is_file} |
> | File size | {filesize} bytes |
> | Created | {created_at} |
> | Updated | {updated_at} |

Interpret the statuses for the user:

- `crawl_status: queued` → "Not yet crawled — indexing may still be in progress."
- `crawl_status: crawled` + `index_status: queued` → "Crawled but not yet indexed."
- `crawl_status: crawled` + `index_status: indexed` → "Fully crawled and indexed."
- `crawl_status: failed` → "Crawl failed — try `/refresh-agent` to retry."
- `index_status: failed` → "Indexing failed — try `/refresh-agent` to retry."

**If no match was found:**

> "No page matching '{search}' was found in agent {agent_id}.
>
> - The file may not have been uploaded yet — run `/index-files` to add it.
> - Or the filename may differ — try a shorter search term."
