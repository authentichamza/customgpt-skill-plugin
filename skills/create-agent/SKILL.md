# create-agent

Create a CustomGPT.ai agent for a local folder and index its files into it.

## Critical Rules
- NEVER skip the API key check
- ALWAYS save `.customgpt-meta.json` before reporting success
- NEVER index `.git/`, `node_modules/`, `__pycache__/`, binary files, or `.env` files
- Run the upload loop via Bash — do NOT upload files one by one yourself

---

## Step 1 — Get API Key

Check in order:
1. Env var `$CUSTOMGPT_API_KEY`
2. `.env` file — walk up from `$PWD` to `/` looking for a file containing `CUSTOMGPT_API_KEY=`
3. Read `~/.claude/customgpt-config.json` and extract the `apiKey` field

Use the first non-empty value. If none found, ask:
> "Please provide your CustomGPT.ai API key. You can find it at https://app.customgpt.ai/profile#api-keys"

Once you have the key, save it:
```bash
echo '{"apiKey":"KEY_HERE"}' > ~/.claude/customgpt-config.json
```

---

## Step 2 — Get Folder to Index

Ask the user which folder to index, or use `$PWD` if they don't specify. Confirm it exists.

---

## Step 3 — Create the Agent

```bash
curl -s -X POST "https://app.customgpt.ai/api/v1/projects" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "project_name=$(basename $FOLDER)&is_chat_active=1"
```

Extract `data.id` as `agent_id`. If missing, show the error and stop.

---

## Step 4 — Collect and Upload Files

Find eligible files (exclude `.git/`, `node_modules/`, `__pycache__/`, binaries, `.env`). Tell the user the count before uploading.

```bash
UPLOADED=0; FAILED=0; TOTAL=$(wc -l < /tmp/customgpt-files.txt)
while IFS= read -r FILE; do
  REL="${FILE#${FOLDER}/}"
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
echo "Done — uploaded: $UPLOADED / $TOTAL, failed: $FAILED"
```

---

## Step 5 — Save Meta File

Write `.customgpt-meta.json` to the indexed folder:
```json
{
  "agent_id": <id>,
  "agent_name": "<folder basename>",
  "indexed_folder": "<absolute path>",
  "created_at": "<ISO timestamp>"
}
```

---

## Step 6 — Report

> "Agent created. ID: {agent_id} | Folder: {folder} | Files uploaded: {N}
> Use `/query-agent` to search it or `/refresh-agent` to re-sync after changes."
