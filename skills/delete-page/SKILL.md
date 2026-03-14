---
name: customgpt-ai-rag:delete-page
description: Permanently remove a specific document (page) from the CustomGPT.ai agent's knowledge base by page ID. Requires confirmation. Use /list-pages to find page IDs.
argument-hint: "[page ID to delete — run /list-pages to find it]"
allowed-tools: Bash, Read
triggers:
  - "delete page"
  - "delete this page"
  - "delete document"
  - "remove page"
  - "remove document"
  - "remove from index"
  - "remove from agent"
  - "delete from agent"
  - "delete from knowledge base"
  - "remove from knowledge base"
  - "delete page id"
  - "remove page id"
---

# delete-page

Permanently remove a document from the agent's knowledge base by its page ID. The agent will no longer reference this content when answering questions. Cannot be undone — re-upload with `/index-files` to restore.

## Rules

- ALWAYS confirm before deleting — show the page ID and filename/URL to the user
- ALWAYS read `.customgpt-meta.json` to get `agent_id`
- If no page ID provided, look up the document first using the pages list

---

## Step 1 — Resolve API Key

Follow the lookup procedure in `skills/_shared/api-key.md`. Store the result as `$API_KEY`.

---

## Step 2 — Read Meta File

Walk up from `$PWD` toward `/`, looking for `.customgpt-meta.json`. Extract `agent_id`.

If not found:
> "No agent found in this directory tree. Run `/create-agent` to set one up first."

---

## Step 3 — Resolve the Page ID

**If the user provided a numeric page ID:** use it directly as `$PAGE_ID`.

**If the user provided a filename or URL instead of an ID:** search the pages list to find the matching document. Fetch pages until a match is found:

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages?page=${PAGE}&limit=100&order=asc" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

Match `filename` or `page_url` (case-insensitive substring). If found, note the `id`, `filename` or `page_url`.

If not found after checking all pages:
> "No document matching '{term}' found. Run `/list-pages` to browse all indexed documents."

**If no identifier provided at all:**
> "Please provide a page ID or filename. Run `/list-pages` to see all documents and their IDs."

---

## Step 4 — Confirm Deletion

Present the details and ask for confirmation:

> "About to permanently delete page **{PAGE_ID}** (`{filename or page_url}`) from agent **{agent_id}**.
> This cannot be undone. The file must be re-uploaded to restore it.
>
> Continue? (yes/no)"

If not confirmed: "Cancelled. Page {PAGE_ID} was not deleted."

---

## Step 5 — Delete the Page

```bash
curl -s --request DELETE \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages/${PAGE_ID}" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

Read the response:
- HTTP 200 → deleted successfully
- HTTP 404 → page not found — it may have already been deleted
- HTTP 401 → invalid API key
- HTTP 400 or 500 → server error; show the response body

---

## Step 6 — Report

On success:
> "Page **{PAGE_ID}** (`{filename or page_url}`) deleted from agent {agent_id}."
>
> "To restore it, run `/index-files {filename}`. To refresh a different version of the same file in one step, use `/reindex-file {filename}`."

On failure:
> "Delete failed (HTTP {status}): {error message from response body}"
