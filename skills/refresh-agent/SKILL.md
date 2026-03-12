# refresh-agent

Delete all files from the agent and re-sync from the same indexed folder. Use this after making changes to your codebase.

## Critical Rules
- ALWAYS read `.customgpt-meta.json` to get `agent_id` and `indexed_folder`
- Delete ALL pages before re-uploading — do not skip deletion
- Paginate the delete loop — there may be more than 100 pages
- `touch` the meta file after upload to reset the freshness timestamp

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

Priority: env var → `.env` file → saved config. Use the first non-empty value found as `$API_KEY`. If missing, ask the user for it and save:
```bash
mkdir -p ~/.claude && echo '{"apiKey":"KEY_HERE"}' > ~/.claude/customgpt-config.json
```

---

## Step 2 — Read Meta File

Walk up from the current directory to find `.customgpt-meta.json`:
```bash
dir="$PWD"
while [ "$dir" != "/" ]; do
  [ -f "$dir/.customgpt-meta.json" ] && cat "$dir/.customgpt-meta.json" && break
  dir=$(dirname "$dir")
done
```

Extract `agent_id` and `indexed_folder`. If not found:
> "No agent found in this directory tree. Run `/create-agent` first."

Show the user what will be refreshed and ask for confirmation:
> "I will delete all pages from agent {agent_id} and re-index {indexed_folder}. Continue? (yes/no)"

---

## Step 3 — Check for Changed Files (Optional Info)

Show the user what has changed since the last index:
```bash
find "$INDEXED_FOLDER" -newer "$INDEXED_FOLDER/.customgpt-meta.json" -type f \
  -not -path "*/.git/*" -not -path "*/node_modules/*" -not -path "*/__pycache__/*" \
  -not -name ".DS_Store" -not -name "*.pyc"
```

This is informational only — refresh always re-syncs everything.

---

## Step 4 — Delete ALL Existing Pages

Run directly in the current shell so `$API_KEY` and `$AGENT_ID` expand without `sed`:
```bash
DELETED=0
PAGE=1

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

## Step 5 — Re-Upload Files

**Collect files:**
```bash
find "$INDEXED_FOLDER" -type f \
  -not -path "*/.git/*" \
  -not -path "*/node_modules/*" \
  -not -path "*/__pycache__/*" \
  -not -path "*/.next/*" \
  -not -path "*/dist/*" \
  -not -path "*/build/*" \
  -not -path "*/.cache/*" \
  -not -path "*/vendor/*" \
  -not -path "*/coverage/*" \
  -not -name "*.pyc" -not -name "*.pyo" -not -name "*.class" \
  -not -name "*.exe" -not -name "*.bin" -not -name "*.zip" \
  -not -name "*.tar" -not -name "*.gz" \
  -not -name "*.png" -not -name "*.jpg" -not -name "*.jpeg" \
  -not -name "*.gif" -not -name "*.ico" -not -name "*.webp" \
  -not -name "*.svg" -not -name "*.mp3" -not -name "*.mp4" \
  -not -name "*.mov" -not -name "*.woff" -not -name "*.woff2" \
  -not -name "*.ttf" -not -name "*.lock" \
  -not -name ".DS_Store" -not -name ".env" -not -name ".env.*" \
  > /tmp/customgpt-files.txt

wc -l < /tmp/customgpt-files.txt
```

**Upload** (inline — no `sed` substitution needed):
```bash
UPLOADED=0
FAILED=0

while IFS= read -r FILE; do
  REL="${FILE#${INDEXED_FOLDER}/}"
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

## Step 6 — Reset Freshness Timestamp

```bash
touch "$INDEXED_FOLDER/.customgpt-meta.json"
```

---

## Step 7 — Report

> "Agent {agent_id} refreshed.
> Folder: {indexed_folder}
> Uploaded: {N} files.
>
> Use `/query-agent` to search the updated index."
