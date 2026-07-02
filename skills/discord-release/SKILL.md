---
description: Post the latest GitHub release to Discord via webhook with a Japanese summary
---

## Discord Release Notification

Post the latest release of the current repository (or a specified repo) to a Discord channel via webhook, with a Japanese summary of the changes.

### Prerequisites

- Environment variable `DISCORD_WEBHOOK_URL` must be set
- `gh` CLI must be authenticated

### Usage

- `/discord-release` — Post the latest release of the current repo
- `/discord-release receptron/mulmocast-cli` — Post the latest release of a specific repo

### Steps

1. **Check environment variable**:
   ```bash
   echo "${DISCORD_WEBHOOK_URL:?DISCORD_WEBHOOK_URL is not set}"
   ```
   If not set, **STOP** and tell the user to set `DISCORD_WEBHOOK_URL`.

2. **Get release info**:
   - If an argument is provided, use it as `owner/repo`
   - Otherwise, detect from the current git repo using `gh repo view --json nameWithOwner -q .nameWithOwner`
   ```bash
   gh release view --repo <owner/repo> --json tagName,name,url,body
   ```

3. **Generate Japanese summary**:
   - Read the release body (which may be in English or mixed)
   - Write a concise Japanese summary (2-4 bullet points) of the key changes
   - Format: each bullet starts with an appropriate emoji
   - Keep the summary under 500 characters
   - Example:
     ```
     🎬 音声が動画より長い場合、audio controls を動画上にオーバーレイ表示するように修正
     🔧 全コンポーネントの optional props に default 値を追加し lint warning を解消
     ```

4. **Format and post to Discord**:
   Use `curl` to POST a Discord webhook message with the Japanese summary as description.

   **IMPORTANT**:
   - Use `jq` to safely construct the JSON payload — NEVER build JSON by string concatenation.
   - Truncate description to 2000 characters (Discord embed limit).
   - Example using jq:
     ```bash
     TAG=$(gh release view --repo "$REPO" --json tagName -q .tagName)
     URL=$(gh release view --repo "$REPO" --json url -q .url)
     REPO_NAME=$(echo "$REPO" | cut -d/ -f2)

     # SUMMARY is the Japanese summary you generated
     JSON=$(jq -n \
       --arg title "$REPO_NAME $TAG" \
       --arg url "$URL" \
       --arg desc "$SUMMARY" \
       '{embeds: [{title: $title, url: $url, description: $desc, color: 5814783}]}')

     curl -s -o /dev/null -w "%{http_code}" -H "Content-Type: application/json" -d "$JSON" "$DISCORD_WEBHOOK_URL"
     ```

5. **Confirm success**: Check that curl returns HTTP 204 (Discord success). Show the posted summary to the user.

### Important Rules

- NEVER hardcode the webhook URL — always use `$DISCORD_WEBHOOK_URL`
- ALWAYS use `jq` to build JSON payloads safely
- ALWAYS write the summary in Japanese
- Truncate long descriptions to stay within Discord's 2000-char embed description limit
