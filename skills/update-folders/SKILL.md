# update-folders

Change which folder is indexed for an existing agent. Deletes all current pages and re-indexes the new folder.

## Critical Rules
- ALWAYS read `.customgpt-meta.json` before doing anything
- ALWAYS delete ALL existing pages before uploading the new folder
- Paginate the pages list — there may be more than 100 pages
- Update `.customgpt-meta.json` with the new folder after success

---

## Step 1 — Get API Key

Check in order:
1. Env var `$CUSTOMGPT_API_KEY`
2. `.env` file — walk up from `$PWD` to `/` looking for a file containing `CUSTOMGPT_API_KEY=`
3. Read `~/.claude/customgpt-config.json` and extract the `apiKey` field

Use the first non-empty value. If none found, ask the user then save:
```bash
echo '{"apiKey":"KEY_HERE"}' > ~/.claude/customgpt-config.json
```

---

## Step 2 — Read Existing Meta

Walk up from `$PWD` to find `.customgpt-meta.json`. Extract `agent_id` and `indexed_folder`.

If not found:
> "No agent found. Run `/create-agent` first."

---

## Step 3 — Ask for New Folder

> "Which folder should I index now? Press Enter to keep the current folder: {current_folder}"

If no input, keep the existing folder. Confirm the folder exists.

---

## Step 4 — Delete ALL Existing Pages

```bash
DELETED=0; PAGE=1
while true; do
  RESP=$(curl -s \
    "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages?page=${PAGE}&per_page=100" \
    -H "Authorization: Bearer ${API_KEY}")
  IDS=$(echo "$RESP" | grep -o '"id":[0-9]*' | grep -o '[0-9]*')
  [ -z "$IDS" ] && break
  for ID in $IDS; do
    HTTP=$(curl -s -o /dev/null -w "%{http_code}" \
      -X DELETE "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages/${ID}" \
      -H "Authorization: Bearer ${API_KEY}")
    [ "$HTTP" = "200" ] && DELETED=$((DELETED + 1))
  done
  COUNT=$(echo "$IDS" | wc -l)
  [ "$COUNT" -lt 100 ] && break
  PAGE=$((PAGE + 1))
done
echo "Deleted $DELETED pages"
```

---

## Step 5 — Upload New Folder

Collect eligible files from `$NEW_FOLDER` (same exclusion rules as `index-files`). Write paths to `/tmp/customgpt-files.txt`.

```bash
UPLOADED=0; FAILED=0
while IFS= read -r FILE; do
  REL="${FILE#${NEW_FOLDER}/}"
  HTTP=$(curl -s -o /dev/null -w "%{http_code}" \
    -X POST "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/sources" \
    -H "Authorization: Bearer ${API_KEY}" \
    -F "file=@${FILE};filename=${REL}")
  if [ "$HTTP" = "200" ] || [ "$HTTP" = "201" ]; then
    UPLOADED=$((UPLOADED + 1))
  else
    FAILED=$((FAILED + 1))
    echo "FAILED [$HTTP]: $REL" >&2
  fi
done < /tmp/customgpt-files.txt
echo "Done — uploaded: $UPLOADED, failed: $FAILED"
```

---

## Step 6 — Update Meta File

Write updated `.customgpt-meta.json` to `$NEW_FOLDER` with the new `indexed_folder` value and current timestamp.

If the folder changed, remove the old `.customgpt-meta.json`.

---

## Step 7 — Report

> "Done. Agent {agent_id} is now indexed from {new_folder}. Uploaded: {N} files."
