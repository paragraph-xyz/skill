---
name: paragraph-cli
description: Use the Paragraph CLI to manage posts, publications, subscribers, and coins on paragraph.com. Trigger when the user asks to publish, create, update, or manage newsletter content on Paragraph.
license: MIT
compatibility: Requires Node.js 18+ and npm. Install the CLI with `npm install -g @paragraph-com/cli`.
metadata:
  author: paragraph-com
  version: "1.0"
allowed-tools: Bash(paragraph:*) Read
---

# Paragraph CLI

CLI for [Paragraph](https://paragraph.com) — a web3 publishing and newsletter platform. Use it to manage posts, publications, subscribers, and coins.

## MCP Server (recommended for MCP-compatible clients)

If your client supports [MCP](https://modelcontextprotocol.io), use the Paragraph MCP server instead of the CLI for a more integrated experience:

```bash
npx @paragraph-com/mcp
```

Setup: `claude mcp add paragraph -- npx @paragraph-com/mcp`

The MCP server exposes 25+ tools (posts, publications, subscribers, coins, search, feed, users) and shares authentication with the CLI. See [full docs](https://paragraph.com/docs/development/mcp).

## CLI Setup

Install globally:

```bash
npm install -g @paragraph-com/cli
```

Authenticate — login persists the key to `~/.paragraph/config.json`:

```bash
paragraph login --token <api-key>
echo "$PARAGRAPH_API_KEY" | paragraph login --with-token
```

Or skip login and pass the key per-command via env var:

```bash
PARAGRAPH_API_KEY=<api-key> paragraph post list --json
```

Verify: `paragraph whoami --json`

## Working agreement

- **Always use `--json`** for parseable output. Data goes to stdout, status/errors to stderr.
- **Always use `--yes`** on `delete` to skip confirmation prompts.
- **Use `--dry-run`** before `delete`, `publish`, and `archive` to preview what will happen.
- **Use flags, not just positional args.** Every identifier accepts `--id` so you can chain commands.
- **Pipe content via stdin** when creating or updating posts from files: `cat draft.md | paragraph post create --title "My Post"`
- **Paginate with `--limit` and `--cursor`.** The JSON response includes `pagination.cursor` and `pagination.hasMore`.
- **Do not use interactive login.** Use `--token` or `--with-token` for non-interactive auth.
- **Check auth before running commands.** Run `paragraph whoami --json` to verify credentials are valid.
- **Do not publish without explicit user approval.** Publishing sends content live and optionally emails subscribers.
- **Default to draft.** `post create` creates drafts. Only call `post publish` when the user asks.

## Commands

### Posts

```bash
# Create a draft
paragraph post create --title "My Post" --file ./post.md --tags "web3,defi" --json
paragraph post create --title "My Post" --text "# Hello World" --subtitle "A subtitle" --json
cat content.md | paragraph post create --title "My Post" --json

# List
paragraph post list --json
paragraph post list --status draft --limit 20 --json
paragraph post list --publication <slug-or-id> --json

# Get (by ID, URL, or @pub/slug)
paragraph post get --id <post-id> --json
paragraph post get @my-blog/post-slug --json

# Extract a single field
paragraph post get --id <post-id> --field markdown > post.md
paragraph post get --id <post-id> --field title

# Update
paragraph post update --id <id-or-slug> --title "New Title" --json
paragraph post update --id <id-or-slug> --text "Updated content" --subtitle "New subtitle" --json
paragraph post update --id <id-or-slug> --file ./updated.md --tags "new,tags" --json

# Publish
paragraph post publish --id <id-or-slug> --json
paragraph post publish --id <id-or-slug> --newsletter --json

# Revert to draft
paragraph post draft --id <id-or-slug> --json

# Archive
paragraph post archive --id <id-or-slug> --json

# Preview destructive actions
paragraph post delete --id <id-or-slug> --dry-run --json
paragraph post publish --id <id-or-slug> --dry-run --json

# Delete
paragraph post delete --id <id-or-slug> --yes --json

# Send test newsletter email
paragraph post test-email --id <post-id> --json

# Browse
paragraph post feed --limit 10 --json
paragraph post by-tag --tag web3 --limit 20 --json
```

### Publications

```bash
paragraph publication get --id <slug-or-id-or-domain> --json
```

### Search

```bash
paragraph search post --query "ethereum" --json
paragraph search blog --query "web3" --json
```

### Subscribers

```bash
paragraph subscriber list --limit 100 --json
paragraph subscriber count --publication <id> --json
paragraph subscriber add --email user@example.com --json
paragraph subscriber import --csv ./subscribers.csv --json
```

### Coins

```bash
paragraph coin get --id <id-or-address> --json
paragraph coin popular --limit 10 --json
paragraph coin search --query "ethereum" --json
paragraph coin holders --id <id-or-address> --limit 50 --json
paragraph coin quote --id <id-or-address> --amount <wei> --json
```

### Users

```bash
paragraph user get --id <user-id-or-wallet> --json
```

### Auth

```bash
paragraph login --token <api-key>
echo "<api-key>" | paragraph login --with-token
paragraph whoami --json
paragraph logout
```

## JSON response shapes

Paginated list:
```json
{
  "data": [{ "id": "...", "title": "..." }],
  "pagination": { "cursor": "abc123", "hasMore": true }
}
```

Single item:
```json
{ "id": "...", "title": "...", "markdown": "..." }
```

Mutation:
```json
{ "id": "...", "status": "published" }
```

Error (on stderr):
```json
{ "error": "Not found.", "code": "NOT_FOUND", "status": 404 }
```

Error codes: UNAUTHORIZED, FORBIDDEN, NOT_FOUND, RATE_LIMITED, SERVER_ERROR, REQUEST_FAILED, CLIENT_ERROR, UNKNOWN.

## Common patterns

### Create and publish in one flow
```bash
ID=$(paragraph post create --title "My Post" --file ./post.md --json | jq -r '.id')
paragraph post publish --id "$ID" --newsletter --json
```

### Export all posts as markdown
```bash
paragraph post list --limit 100 --json | jq -r '.data[].id' | while read id; do
  SLUG=$(paragraph post get --id "$id" --json | jq -r '.slug')
  paragraph post get --id "$id" --field markdown > "${SLUG}.md"
done
```

### Paginate through all subscribers
```bash
CURSOR=""
while true; do
  RESULT=$(paragraph subscriber list --limit 100 ${CURSOR:+--cursor "$CURSOR"} --json)
  echo "$RESULT" | jq '.data[]'
  CURSOR=$(echo "$RESULT" | jq -r '.pagination.cursor // empty')
  HAS_MORE=$(echo "$RESULT" | jq '.pagination.hasMore')
  [ "$HAS_MORE" = "true" ] || break
done
```

## Environment variables

| Variable | Purpose |
|----------|---------|
| PARAGRAPH_API_KEY | API key (skip login) |
| PARAGRAPH_API_URL | Custom API base URL |
| PARAGRAPH_NON_INTERACTIVE | Set to 1 to force CLI mode |
| CI | Set to true to force CLI mode |

## Troubleshooting

### Authentication errors

If commands fail with `UNAUTHORIZED`:

```bash
# Check if logged in
paragraph whoami --json

# Re-authenticate
paragraph login --token <api-key>
```

The CLI auto-clears stored credentials on 401. Re-login if credentials were revoked.

### Rate limiting

If you get `RATE_LIMITED`, wait and retry. The error includes a `429` status code. Avoid tight loops — add a delay between paginated requests.

### Command hangs

If a command appears to hang, it may be waiting for stdin. Ensure you're passing content via `--text`, `--file`, or piping to stdin. The CLI times out after 30 seconds if stdin is piped but no data arrives.

### CLI not found after install

```bash
npm install -g @paragraph-com/cli
# Verify
npx paragraph --version
```
