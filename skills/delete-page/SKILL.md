---
description: Delete a specific document (page) from the CustomGPT.ai agent's knowledge base by page ID.
triggers:
  - "delete page"
  - "delete document"
  - "remove page"
  - "remove document"
  - "delete page id"
  - "remove from index"
  - "delete from agent"
  - "remove from agent"
---

# delete-page

Permanently remove a document from the agent's knowledge base by its page ID. This cannot be undone — the file must be re-uploaded to restore it.

## Critical Rules
- ALWAYS confirm with the user before deleting — show the page ID and filename/URL
- ALWAYS read `.customgpt-meta.json` to get `agent_id`
- If no page ID is provided, run `/list-pages` first to find it
- A 200 response confirms deletion

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

## Step 3 — Get Page ID

If the user provided a page ID, use it.

If not, ask:
> "Which page ID should I delete? Run `/list-pages` to see all documents and their IDs."

---

## Step 4 — Confirm

Before deleting, confirm with the user:
> "About to delete page ID {PAGE_ID} from agent {AGENT_ID}. This cannot be undone. Continue? (yes/no)"

---

## Step 5 — Delete the Page

```bash
curl -s --request DELETE \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages/${PAGE_ID}" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

Read the response:
- HTTP 200 → success
- HTTP 404 → page not found — may already be deleted
- HTTP 401 → invalid API key
- HTTP 500 → server error

---

## Step 6 — Report

On success:
> "Page {PAGE_ID} deleted from agent {AGENT_ID}."
> "Run `/index-files` to re-upload the file, or `/reindex-file` to delete and re-upload in one step."

On failure:
> "Delete failed (HTTP {status}): {error message from response}"
