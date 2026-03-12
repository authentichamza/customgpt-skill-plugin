---
description: Change which folder is indexed for an existing CustomGPT.ai agent. Deletes all current pages and re-indexes from the new folder.
triggers:
  - "update folder"
  - "update folders"
  - "change folder"
  - "change indexed folder"
  - "switch folder"
  - "index different folder"
  - "change source folder"
  - "update source"
---

# update-folders

Change which folder is indexed for an existing agent. Deletes all current pages and re-indexes the new folder.

## Critical Rules
- ALWAYS read `.customgpt-meta.json` before doing anything
- ALWAYS delete ALL existing pages before uploading the new folder
- Paginate the fetch loop — read `data.pages.last_page` to know when to stop
- Update `.customgpt-meta.json` with the new folder after success

---

## Step 1 — Get API Key

Check in order:
1. Env var `$CUSTOMGPT_API_KEY`
2. `.env` file — walk up from `$PWD` to `/` looking for a file containing `CUSTOMGPT_API_KEY=`
3. Read `~/.claude/customgpt-config.json` and extract the `apiKey` field

Use the first non-empty value. If none found, ask the user.

---

## Step 2 — Read Existing Meta

Walk up from `$PWD` to find `.customgpt-meta.json`. Extract `agent_id` and `indexed_folder`.

If not found:
> "No agent found. Run `/create-agent` first."

---

## Step 3 — Ask for New Folder

> "Which folder should I index now? Press Enter to keep the current folder: {current_folder}"

If no input, keep the existing folder. Confirm the folder exists on disk.

---

## Step 4 — Fetch All Page IDs

Fetch page 1:

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages?page=1&limit=100&order=asc" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

Read `data.pages.data[]` and collect all `id` values. Read `data.pages.last_page` — if greater than 1, repeat for each remaining page:

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages?page=${PAGE}&limit=100&order=asc" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

Collect all IDs across all pages before starting deletion.

---

## Step 5 — Delete Each Existing Page

For each page ID collected in Step 4:

```bash
curl -s --request DELETE \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages/${PAGE_ID}" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

HTTP 200 = deleted. Report progress as you go.

---

## Step 6 — Collect Eligible Files from New Folder

```bash
find "$NEW_FOLDER" -type f \
  -not -path "*/.git/*" \
  -not -path "*/node_modules/*" \
  -not -path "*/__pycache__/*" \
  -not -path "*/.next/*" \
  -not -path "*/dist/*" \
  -not -path "*/build/*" \
  -not -path "*/.cache/*" \
  -not -path "*/vendor/*" \
  -not -path "*/coverage/*" \
  -not -path "*/.venv/*" \
  -not -path "*/venv/*" \
  \( \
    -name "*.js" -o -name "*.ts" -o -name "*.tsx" -o -name "*.jsx" \
    -o -name "*.py" -o -name "*.go" -o -name "*.rb" -o -name "*.java" \
    -o -name "*.cs" -o -name "*.cpp" -o -name "*.c" -o -name "*.h" \
    -o -name "*.rs" -o -name "*.swift" -o -name "*.kt" -o -name "*.php" \
    -o -name "*.sh" -o -name "*.bash" -o -name "*.zsh" \
    -o -name "*.html" -o -name "*.css" -o -name "*.scss" \
    -o -name "*.sql" -o -name "*.graphql" \
    -o -name "*.json" -o -name "*.yaml" -o -name "*.yml" -o -name "*.toml" \
    -o -name "*.xml" -o -name "*.ini" -o -name "*.cfg" -o -name "*.conf" \
    -o -name "*.md" -o -name "*.mdx" -o -name "*.rst" -o -name "*.txt" \
    -o -name "*.csv" -o -name "*.pdf" -o -name "*.docx" \
  \)
```

Tell the user the total file count before uploading.

---

## Step 7 — Upload Each File

For each file, compute `REL` as its path relative to `$NEW_FOLDER`, then:

```bash
curl -s --request POST \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/sources" \
  --header "Authorization: Bearer ${API_KEY}" \
  --form "file=@${ABSOLUTE_PATH};filename=${REL}"
```

HTTP 200 or 201 = success. Report each result: `OK: {REL}` or `FAILED [{status}]: {REL}`.

---

## Step 8 — Update Meta File

Write updated `.customgpt-meta.json` to `$NEW_FOLDER`:

```bash
cat > "${NEW_FOLDER}/.customgpt-meta.json" << EOF
{
  "agent_id": ${AGENT_ID},
  "agent_name": "$(basename $NEW_FOLDER)",
  "indexed_folder": "${NEW_FOLDER}",
  "created_at": "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
}
EOF
```

If the folder changed, remove the old `.customgpt-meta.json` from the previous location.

---

## Step 9 — Report

> "Done. Agent {agent_id} is now indexed from {new_folder}.
> - Deleted: {D} old pages
> - Uploaded: {U} / {total} files, Failed: {F}"
