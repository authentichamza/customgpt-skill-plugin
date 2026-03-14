---
name: customgpt-ai-rag:reindex-file
description: Delete a specific file from the CustomGPT.ai agent and re-upload it from disk. Use when a single file has changed and you need to refresh it without a full re-sync.
argument-hint: "[filename or path to refresh]"
allowed-tools: Bash, Read
triggers:
  - "reindex file"
  - "reindex this file"
  - "reindex specific file"
  - "re-index file"
  - "re-index this file"
  - "refresh file"
  - "refresh this file"
  - "refresh specific file"
  - "re-upload file"
  - "re-upload this file"
  - "update file in agent"
  - "update indexed file"
  - "update this file in the index"
  - "sync this file"
  - "sync file"
---

# reindex-file

Find a specific file in the agent's knowledge base, delete it, then re-upload the current version from disk. The surgical alternative to `/refresh-agent` — use this when only one or a few files have changed.

## Rules

- ALWAYS read `.customgpt-meta.json` to get `agent_id` and `indexed_folder`
- For uploaded files: delete then re-upload (the `/reindex` API endpoint only works for URL-based documents)
- For URL-based documents: use the `/reindex` API endpoint directly
- Match files by case-insensitive substring match against `filename` in the pages list
- Confirm before deleting — the delete is irreversible

---

## Step 1 — Resolve API Key

Follow the lookup procedure in `skills/_shared/api-key.md`. Store the result as `$API_KEY`.

---

## Step 2 — Read Meta File

Walk up from `$PWD` toward `/`, looking for `.customgpt-meta.json`. Extract `agent_id`, `indexed_folder`. Store the full path as `$META_FILE_PATH`.

If not found:
> "No agent found in this directory tree. Run `/create-agent` to set one up first."

---

## Step 3 — Resolve the Target File

If the user provided a filename or path, use it. Otherwise ask:
> "Which file do you want to reindex? Provide a filename or relative path."

Resolve to an absolute path:

```bash
realpath "$TARGET" 2>/dev/null
```

If that fails, search under `indexed_folder`:

```bash
find "$indexed_folder" -name "$(basename $TARGET)" -type f
```

Use the first match. If no match is found on disk, tell the user and stop.

---

## Step 4 — Find the Page ID in the Agent

Search the agent's page list for a document whose `filename` contains the target filename as a case-insensitive substring.

Fetch pages until a match is found or all pages are checked:

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages?page=${PAGE}&limit=100&order=asc" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

Match each entry's `filename` (case-insensitive, substring) against `$(basename $TARGET)`.

**If no match found after all pages:**
> "'{filename}' is not currently indexed. Uploading it now as a new document..."
> Skip directly to Step 6.

**If matched:** note the `id` as `$PAGE_ID`, the `filename`, and `is_file`. For URL-based documents (`is_file: false`), go to Step 5b.

---

## Step 5a — Delete the Existing File Page

Confirm with the user:
> "Found '`{filename}`' as page ID {PAGE_ID}. I will delete it and re-upload `{target}` from disk. Continue? (yes/no)"

```bash
curl -s --request DELETE \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages/${PAGE_ID}" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

On HTTP 200, proceed to Step 6. On failure, show the error response and stop — do not attempt re-upload.

---

## Step 5b — Reindex URL-Based Document (alternative path)

For URL-based documents (`is_file: false`), use the reindex endpoint instead of delete+upload:

```bash
curl -s --request POST \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages/${PAGE_ID}/reindex" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

Read the response. `data.updated: true` = reindex triggered successfully. Tell the user:
> "Reindex triggered for URL document (page ID {PAGE_ID}). CustomGPT.ai will re-crawl the URL. Run `/check-page` in a moment to verify the new index status."

Then skip to Step 7 (update freshness) and Step 8 (report).

---

## Step 6 — Re-Upload the File

Compute `REL` as the file path relative to `indexed_folder`:

```bash
curl -s --request POST \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/sources" \
  --header "Authorization: Bearer ${API_KEY}" \
  --form "file=@${ABSOLUTE_FILE_PATH};filename=${REL}"
```

HTTP 200 or 201 = success. On failure, report the error with the HTTP status and full response body.

---

## Step 7 — Update Freshness Timestamp

```bash
touch "$META_FILE_PATH"
```

---

## Step 8 — Report

> **Reindexed `{filename}`**
>
> - Deleted old page ID: {PAGE_ID}
> - Uploaded new copy: HTTP {status}
>
> Run `/check-page {filename}` to verify the new crawl and index status once processing completes.
