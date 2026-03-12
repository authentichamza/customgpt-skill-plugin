---
description: Delete a specific file from the CustomGPT.ai agent and re-upload it from disk. Use when a single file has changed and needs to be refreshed without wiping the whole agent.
triggers:
  - "reindex file"
  - "reindex this file"
  - "reindex specific file"
  - "refresh file"
  - "refresh this file"
  - "refresh specific file"
  - "re-upload file"
  - "update file in agent"
  - "update indexed file"
  - "re-index file"
  - "re-index this file"
---

# reindex-file

Find a specific file in the agent's knowledge base, delete it, then re-upload it from disk. Useful for refreshing a single changed file without doing a full `/refresh-agent`.

## Critical Rules
- ALWAYS read `.customgpt-meta.json` to get `agent_id` and `indexed_folder`
- This skill chains three operations: find (list-pages), delete (delete-page), upload (index-files curl)
- Match the file by `filename` using case-insensitive substring match
- Confirm before deleting
- Only the `/reindex` API endpoint works for URL-based documents — for uploaded files, always delete then re-upload

---

## Step 1 — Get API Key

Check in order:
1. Env var `$CUSTOMGPT_API_KEY`
2. `.env` file — walk up from `$PWD` to `/` looking for a file containing `CUSTOMGPT_API_KEY=`
3. Read `~/.claude/customgpt-config.json` and extract the `apiKey` field

Use the first non-empty value as `$API_KEY`. If none found, ask the user.

---

## Step 2 — Read Meta File

Walk up from `$PWD` to find `.customgpt-meta.json`. Extract `agent_id` and `indexed_folder`.

If not found:
> "No agent found in this directory tree. Run `/create-agent` first."

---

## Step 3 — Resolve the Target File

If the user specified a file path or name, use it. Otherwise ask:
> "Which file do you want to reindex? Provide a filename or path."

Resolve to an absolute path:
```bash
realpath "$TARGET" 2>/dev/null || echo "not found"
```

Verify the file exists on disk. If not found, stop and tell the user.

---

## Step 4 — Find the Page ID

Search the agent's page list for a document whose `filename` matches the target file (case-insensitive substring match against the basename).

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages?page=1&limit=100&order=asc" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

Read `data.pages.data[]` and match against the target filename. If not found on page 1, repeat for subsequent pages using `data.pages.last_page`.

If no match is found after checking all pages:
> "File '{filename}' is not currently indexed. Uploading it now as a new document..."
> Skip to Step 6.

If matched, note the `id` (page ID) and `filename`.

---

## Step 5 — Delete the Existing Page

Confirm with the user:
> "Found '{filename}' as page ID {PAGE_ID}. I will delete it and re-upload from disk. Continue? (yes/no)"

```bash
curl -s --request DELETE \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages/${PAGE_ID}" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

Read the response. On HTTP 200, proceed. On failure, stop and report the error.

---

## Step 6 — Re-Upload the File

Compute the relative path from `indexed_folder` to use as the filename in the upload:

```bash
curl -s --request POST \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/sources" \
  --header "Authorization: Bearer ${API_KEY}" \
  --form "file=@${ABSOLUTE_FILE_PATH};filename=${RELATIVE_PATH}"
```

Where `RELATIVE_PATH` is the file path relative to `indexed_folder` (e.g. `skills/index-files/SKILL.md`).

Read the response:
- HTTP 200 or 201 → upload succeeded, note the new page ID from `data.id` if present
- Any other status → report the error

---

## Step 7 — Update Freshness Timestamp

```bash
touch "$META_FILE_PATH"
```

---

## Step 8 — Report

> "Reindexed '{filename}':
> - Deleted old page ID {PAGE_ID}
> - Uploaded new copy (HTTP {status})
>
> Run `/check-page {filename}` to verify the index status."
