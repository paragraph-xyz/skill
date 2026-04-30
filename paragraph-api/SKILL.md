---
name: paragraph-api
description: Use the Paragraph REST API or TypeScript SDK to manage posts, publications, subscribers, and coins on paragraph.com. No installation required — just HTTP requests or `npm install @paragraph-com/sdk`. Trigger when the user asks to integrate with, build on, or call the Paragraph API.
license: MIT
metadata:
  author: paragraph-com
  version: "1.0"
allowed-tools: Bash(curl:*) Bash(npm:*) Bash(npx:*) Read Write WebFetch
---

# Paragraph API

REST API and TypeScript SDK for [Paragraph](https://paragraph.com) — a web3 publishing and newsletter platform. Manage posts, publications, subscribers, and coins without installing any CLI tools.

For CLI or MCP access, see the **paragraph-cli** skill instead.

## Choose your interface

| Interface | Install required | Best for |
|-----------|-----------------|----------|
| REST API | No | Any language, `curl`, quick scripts |
| TypeScript SDK | `npm install @paragraph-com/sdk` | TypeScript/JavaScript projects with type safety |

## Authentication

Generate an API key in your [publication settings](https://paragraph.com/settings/publication/#developer).

Many read endpoints are public — no key needed. Write endpoints (create, update, delete posts, manage subscribers) require authentication.

**REST API** — pass the key as a Bearer token:

```bash
curl https://public.api.paragraph.com/api/v1/me \
  -H "Authorization: Bearer <api-key>"
```

**TypeScript SDK:**

```typescript
import { ParagraphAPI } from "@paragraph-com/sdk";

// Public endpoints — no key needed
const api = new ParagraphAPI();

// Authenticated endpoints — pass your API key
const authedApi = new ParagraphAPI({ apiKey: "<api-key>" });
```

## Working agreement

- **Do not publish without explicit user approval.** Publishing sends content live and optionally emails subscribers.
- **Default to draft.** Created posts are drafts unless `status: "published"` is set explicitly.
- **Do not send custom emails without explicit user approval.** `api.emails.send` (and `POST /v1/emails/send`) delivers real email and can't be undone. Draft the subject and body first; use `dryRun: true` to preview filtering before a real send. On a `403`, surface "this publication isn't approved for custom email yet" and stop — do not retry.
- **Paginate with `cursor`.** Responses include `pagination.cursor` and `pagination.hasMore`.
- **Respect rate limits.** If you get a `429`, back off and retry. Avoid tight loops between paginated requests.
- **Check auth before writing.** Call `GET /v1/me` or `api.me.get()` to verify credentials are valid.

## TypeScript SDK

### Install

```bash
npm install @paragraph-com/sdk
```

### Posts

```typescript
import { ParagraphAPI } from "@paragraph-com/sdk";
const api = new ParagraphAPI({ apiKey: "<api-key>" });

// Create a draft
const newPost = await api.posts.create({
  title: "My Post",
  markdown: "# Hello World",
  subtitle: "A subtitle",
  categories: ["web3", "defi"],
});

// List your posts
const { items: posts, pagination } = await api.posts.list();
const { items: drafts } = await api.posts.list({ status: "draft" });

// Get a single post by ID
const postById = await api.posts.get({ id: "<post-id>" }).single();

// Get a single post by slugs
const postBySlugs = await api.posts.get({ publicationSlug: "my-blog", postSlug: "my-post" }).single();

// Get posts from a publication (paginated)
const { items: pubPosts, pagination: pubPagination } = await api.posts.get(
  { publicationId: "<publication-id>" },
  { limit: 10, includeContent: true }
);

// Update
await api.posts.update({ id: "<post-id>", title: "New Title", markdown: "Updated content" });
await api.posts.update({ slug: "my-post", status: "published" });

// Publish (set status to "published", optionally email subscribers)
await api.posts.update({ id: "<post-id>", status: "published", sendNewsletter: true });

// Schedule a post for future publication
const scheduled = await api.posts.create({
  title: "Launch Day",
  markdown: "# Launching tomorrow!",
  scheduledAt: Date.now() + 24 * 60 * 60 * 1000,
  sendNewsletter: true,
});
// scheduled.status === "scheduled"

// Schedule an existing draft
await api.posts.update({ id: "<post-id>", scheduledAt: Date.now() + 86400000 });

// Cancel a scheduled publication
await api.posts.update({ id: "<post-id>", scheduledAt: null });

// Backdate a post — publishedAt sticks across re-publishes
await api.posts.update({
  id: "<post-id>",
  publishedAt: new Date("2024-01-01T00:00:00Z").getTime(),
});

// Delete (by ID or by slug)
await api.posts.delete({ id: "<post-id>" });
await api.posts.delete({ slug: "my-post" });

// Send test newsletter email (draft posts only)
await api.posts.sendTestEmail({ id: "<post-id>" });

// Feed and tags
const { items: feed } = await api.feed.get();
const { items: tagged } = await api.posts.get({ tag: "web3" }, { limit: 10 });
```

### Publications

```typescript
const api = new ParagraphAPI();

const pubById = await api.publications.get({ id: "<publication-id>" }).single();
const pubBySlug = await api.publications.get({ slug: "@my-blog" }).single();
const pubByDomain = await api.publications.get({ domain: "blog.example.com" }).single();
```

Update settings on the publication that owns your API key. Only provided fields change.

```typescript
const authedApi = new ParagraphAPI({ apiKey: "<api-key>" });

// Name, summary, theme
await authedApi.publications.update("<publication-id>", {
  name: "My Blog",
  summary: "Notes on writing, building, and shipping.",
  themeColor: "purple-600",
  headerFont: "serif",
});

// Featured post — accepts "latest", "popular", "disabled", or a post ID
// (a specific ID must belong to this publication).
await authedApi.publications.update("<publication-id>", { featuredPost: "latest" });

// Pinned posts — replaces the existing list, capped at 50 IDs.
// Each ID must belong to this publication.
await authedApi.publications.update("<publication-id>", {
  pinnedPostIds: ["<post-id-1>", "<post-id-2>"],
});

// Email notifications — merged onto current settings; only the toggles you
// send are changed.
await authedApi.publications.update("<publication-id>", {
  emailNotifications: { newSubscriber: true, newComment: false },
});

// Comment visibility:
//   true          → disable all comments
//   false         → enable comments
//   "on-platform" → hide on-Paragraph comments, keep Farcaster comments
await authedApi.publications.update("<publication-id>", {
  disableComments: "on-platform",
});
```

### Subscribers

```typescript
const api = new ParagraphAPI({ apiKey: "<api-key>" });

// List
const { items: subscribers, pagination } = await api.subscribers.get({ limit: 50 });

// Count
const { count } = await api.subscribers.getCount({ id: "<publication-id>" });

// Add
await api.subscribers.create({ email: "user@example.com" });
await api.subscribers.create({ wallet: "0x1234..." });

// Remove (hard delete — irreversible). At least one of email or wallet
// must be provided; if you pass both, they must resolve to the same user.
await api.subscribers.remove({ email: "user@example.com" });
await api.subscribers.remove({ wallet: "0x1234..." });

// Import CSV
const file = new File([csvContent], "subscribers.csv", { type: "text/csv" });
await api.subscribers.importCsv({ file });
```

### Coins

```typescript
const api = new ParagraphAPI();

// Get coin info (by ID or contract address)
const coinById = await api.coins.get({ id: "<coin-id>" }).single();
const coinByContract = await api.coins.get({ contractAddress: "0x1234..." }).single();

// Popular coins
const { items: popular } = await api.coins.get({ sortBy: "popular" });

// Holders (by ID or contract address)
const holders = await api.coins.getHolders({ id: "<coin-id>" }, { limit: 50 });

// Price quote (amount in wei)
const quote = await api.coins.getQuote({ id: "<coin-id>" }, 1000000000000000000n);

// Buy and sell (requires a viem WalletClient and Account)
const txHash = await api.coins.buy({
  coin: { id: "<coin-id>" },
  client: walletClient,
  account,
  amount: 1000000000000000000n, // 1 ETH in wei
});
const sellHash = await api.coins.sell({
  coin: { id: "<coin-id>" },
  client: walletClient,
  account,
  amount: 1000000000000000000n, // coin amount in wei
});
```

### Search

```typescript
const api = new ParagraphAPI();

const posts = await api.search.posts("ethereum");
const blogs = await api.search.blogs("web3");
const coins = await api.search.coins("test");
```

### Users

```typescript
const api = new ParagraphAPI();

const userById = await api.users.get({ id: "<user-id>" }).single();
const userByWallet = await api.users.get({ wallet: "0x1234..." }).single();
```

### Me

```typescript
const api = new ParagraphAPI({ apiKey: "<api-key>" });
const myPublication = await api.me.get();
```

### Custom emails

Send a one-off markdown email from your publication to a specific recipient list you supply. Each recipient gets it individually with a mandatory unsubscribe footer (uses the same delivery pipeline as the newsletter).

**Use this for:**
- **Targeted segment sends** — "email everyone who opened my last post", "follow-up to my 50 most engaged subscribers", "reach out to this curated list of 20 readers". Build the segment via `api.analytics.query` or `api.subscribers.get`.
- **Self-notifications to the writer** — "email me when I hit 1,000 subscribers", "weekly analytics digest". Pair with `api.me.get()` if you need to look up the writer's email.
- **Outreach to non-subscriber addresses** — a CSV of conference contacts, a press list, an intro to friends-of-friends. Recipients can come from anywhere; they don't have to be in `api.subscribers.get()`. (Anyone who previously unsubscribed will still come back as `suppressed`.)
- **Re-engagement of inactive subscribers** — "email everyone who hasn't opened in 90 days." Identify the segment via `api.analytics.query` against `subscriber_engagement_scores` or `newsletter_metrics`.
- **Draft review to a few collaborators** — "send this draft pitch to my 3 co-authors for feedback." Use this when you need to email people other than the publication owner; `api.posts.sendTestEmail` only goes to the owner.

**Do NOT use this for newsletter blasts.** To email all subscribers with a post, set `sendNewsletter: true` on `api.posts.create` or `api.posts.update`. That's the newsletter pipeline; `api.emails.send` is for targeted lists you supply.

**Eligibility:** publications must be approved by Paragraph for custom email. Ineligible publications get a `403`.

**Per-recipient filtering** is server-side: malformed addresses come back as `invalid`, prior unsubscribes as `suppressed`, and queue failures as `scheduling_failed` (the only reason that's safe to retry).

**Caps:** up to 10,000 recipients per call.

**Always confirm before sending.** Email goes out for real and can't be undone — draft the subject and body first, then call `send`. Use `dryRun: true` to preview the accepted/skipped split without sending.

```typescript
const api = new ParagraphAPI({ apiKey: "<api-key>" });

// Send a markdown email to a list of recipients
const { accepted, skipped } = await api.emails.send({
  subject: "Hello from Paragraph",
  body: "# Welcome\n\nThanks for reading.",
  emails: ["reader@example.com", "another@example.com"],
});
console.log(`${accepted} queued, ${skipped.length} skipped`);
for (const s of skipped) {
  console.log(`${s.email}: ${s.reason}`);
}

// Dry run — check filtering without sending
const preview = await api.emails.send({
  subject: "Preview",
  body: "Body here",
  emails: ["reader@example.com"],
  dryRun: true,
});
```

### Auth (browser-based login)

For CLI or headless clients that need to authenticate without an API key:

```typescript
const api = new ParagraphAPI();

// Create an auth session — returns a URL the user opens in their browser
const session = await api.auth.createSession({ deviceName: "my-app" });
console.log("Open this URL to approve:", session.verificationUrl);

// Poll until the user approves (status: "pending" → "completed")
const status = await api.auth.getSession(session.sessionId);
if (status.apiKey) {
  // Use the returned API key for authenticated requests
  const authedApi = new ParagraphAPI({ apiKey: status.apiKey });
}

// Cancel a pending session
await api.auth.deleteSession(session.sessionId);
```

### Pagination

All list methods return `{ items, pagination }`. Paginate with the cursor:

```typescript
let cursor: string | undefined;
const allPosts = [];

do {
  const { items, pagination } = await api.posts.list({ cursor, limit: 50 });
  allPosts.push(...items);
  cursor = pagination.hasMore ? pagination.cursor : undefined;
} while (cursor);
```

## REST API

Base URL: `https://public.api.paragraph.com/api`

### Posts

```bash
# List your posts (requires auth)
curl https://public.api.paragraph.com/api/v1/posts?status=draft&limit=20 \
  -H "Authorization: Bearer <api-key>"

# Get post by ID
curl https://public.api.paragraph.com/api/v1/posts/<post-id>

# Get post by slug
curl https://public.api.paragraph.com/api/v1/posts/slug/<slug>

# Get posts from a publication
curl https://public.api.paragraph.com/api/v1/publications/<publication-id>/posts?limit=10&includeContent=true

# Get post by publication slug + post slug
curl https://public.api.paragraph.com/api/v1/publications/slug/<pub-slug>/posts/slug/<post-slug>

# Create a draft (requires auth)
curl -X POST https://public.api.paragraph.com/api/v1/posts \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"title": "My Post", "markdown": "# Hello World"}'

# Update (requires auth)
curl -X PUT https://public.api.paragraph.com/api/v1/posts/<post-id> \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"title": "Updated Title", "status": "published", "sendNewsletter": true}'

# Backdate a post (publishedAt sticks across re-publishes)
curl -X PUT https://public.api.paragraph.com/api/v1/posts/<post-id> \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"publishedAt": 1704067200000}'

# Update by slug (requires auth)
curl -X PUT https://public.api.paragraph.com/api/v1/posts/slug/<slug> \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"title": "Updated Title"}'

# Delete by ID (requires auth)
curl -X DELETE https://public.api.paragraph.com/api/v1/posts/<post-id> \
  -H "Authorization: Bearer <api-key>"

# Delete by slug (requires auth)
curl -X DELETE https://public.api.paragraph.com/api/v1/posts/slug/<slug> \
  -H "Authorization: Bearer <api-key>"

# Schedule a post for future publication (requires auth)
curl -X POST https://public.api.paragraph.com/api/v1/posts \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"title": "Launch Day", "markdown": "# Launching!", "scheduledAt": 1746090000000, "sendNewsletter": true}'

# Schedule an existing draft (requires auth)
curl -X PUT https://public.api.paragraph.com/api/v1/posts/<post-id> \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"scheduledAt": 1746090000000}'

# Cancel a scheduled publication (requires auth)
curl -X PUT https://public.api.paragraph.com/api/v1/posts/<post-id> \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"scheduledAt": null}'

# Send test newsletter email
curl -X POST https://public.api.paragraph.com/api/v1/posts/<post-id>/test-email \
  -H "Authorization: Bearer <api-key>"

# Feed
curl https://public.api.paragraph.com/api/v1/posts/feed?limit=10

# Posts by tag
curl https://public.api.paragraph.com/api/v1/posts/tag/web3?limit=20
```

### Publications

```bash
# By ID
curl https://public.api.paragraph.com/api/v1/publications/<publication-id>

# By slug
curl https://public.api.paragraph.com/api/v1/publications/slug/<slug>

# By custom domain
curl https://public.api.paragraph.com/api/v1/publications/domain/<domain>

# Subscriber count
curl https://public.api.paragraph.com/api/v1/publications/<publication-id>/subscribers/count

# Update settings (requires auth — publicationId must match the API key's publication).
# Only provided fields are changed. featuredPost accepts "latest" | "popular" |
# "disabled" | a post ID. pinnedPostIds replaces the existing list (max 50).
# emailNotifications is merged onto current settings.
curl -X PATCH https://public.api.paragraph.com/api/v1/publications/<publication-id> \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"name": "My Blog", "themeColor": "purple-600", "featuredPost": "latest"}'
```

### Subscribers

```bash
# List (requires auth)
curl https://public.api.paragraph.com/api/v1/subscribers?limit=100 \
  -H "Authorization: Bearer <api-key>"

# Add (requires auth)
curl -X POST https://public.api.paragraph.com/api/v1/subscribers \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"email": "user@example.com"}'

# Remove (requires auth — hard delete, irreversible). Pass email or wallet
# (or both — both must resolve to the same user).
curl -X DELETE https://public.api.paragraph.com/api/v1/subscribers \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"email": "user@example.com"}'

# Import CSV (requires auth)
curl -X POST https://public.api.paragraph.com/api/v1/subscribers/import \
  -H "Authorization: Bearer <api-key>" \
  -F "file=@subscribers.csv"
```

### Coins

```bash
# Get coin by ID
curl https://public.api.paragraph.com/api/v1/coins/<coin-id>

# Get coin by contract address
curl https://public.api.paragraph.com/api/v1/coins/contract/<address>

# Popular coins
curl https://public.api.paragraph.com/api/v1/coins/list/popular

# Holders (by ID or contract address)
curl https://public.api.paragraph.com/api/v1/coins/<coin-id>/holders?limit=50
curl https://public.api.paragraph.com/api/v1/coins/contract/<address>/holders?limit=50

# Price quote (by ID or contract address, amount in wei)
curl https://public.api.paragraph.com/api/v1/coins/quote/<coin-id>?amount=1000000000000000000
curl https://public.api.paragraph.com/api/v1/coins/quote/contract/<address>?amount=1000000000000000000

# Buy/sell args (for building onchain transactions)
curl https://public.api.paragraph.com/api/v1/coins/buy/<coin-id>?amount=1000000000000000000&walletAddress=0x...
curl https://public.api.paragraph.com/api/v1/coins/sell/<coin-id>?amount=1000000000000000000&walletAddress=0x...
```

### Search

```bash
curl https://public.api.paragraph.com/api/v1/discover/search?q=ethereum
curl https://public.api.paragraph.com/api/v1/discover/blogs/search?q=web3
curl https://public.api.paragraph.com/api/v1/discover/coins/search?q=test
```

### Users

```bash
curl https://public.api.paragraph.com/api/v1/users/<user-id>
curl https://public.api.paragraph.com/api/v1/users/wallet/<wallet-address>
```

### Me

```bash
curl https://public.api.paragraph.com/api/v1/me \
  -H "Authorization: Bearer <api-key>"
```

### Custom emails

Send a markdown email from your publication to a list of recipients. The publication must be approved by Paragraph; ineligible publications get a `403`. Body is rendered to HTML server-side. Up to 10,000 recipients per call. Always confirm with the user before sending — emails go out for real and can't be undone.

```bash
# Send (requires auth)
curl -X POST https://public.api.paragraph.com/api/v1/emails/send \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": "Hello from Paragraph",
    "body": "# Welcome\n\nThanks for reading.",
    "emails": ["reader@example.com", "another@example.com"]
  }'

# Dry run — preview accepted/skipped without sending
curl -X POST https://public.api.paragraph.com/api/v1/emails/send \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": "Preview",
    "body": "Body here",
    "emails": ["reader@example.com"],
    "dryRun": true
  }'
```

Response shape:

```json
{
  "accepted": 2,
  "skipped": [
    { "email": "bad@", "reason": "invalid" },
    { "email": "unsub@example.com", "reason": "suppressed" }
  ]
}
```

`reason` is one of `invalid` (malformed address), `suppressed` (recipient previously unsubscribed from this publication), or `scheduling_failed` (filtering passed but the delivery task could not be queued — the only reason that's safe to retry).

## JSON response shapes

Paginated list (note: the REST API and SDK wrap items under `items`; the CLI uses `data` instead):
```json
{
  "items": [{ "id": "...", "title": "..." }],
  "pagination": { "cursor": "abc123", "hasMore": true, "total": 42 }
}
```

Single item:
```json
{ "id": "...", "title": "...", "markdown": "..." }
```

Mutation:
```json
{ "success": true }
```

Error:
```json
{ "success": false, "msg": "Not found", "error": "Not found" }
```

HTTP status codes: `401` (unauthorized), `403` (forbidden), `404` (not found), `429` (rate limited), `500` (server error).

## Common patterns

### Create and publish in one flow (SDK)

```typescript
const api = new ParagraphAPI({ apiKey: "<api-key>" });
const post = await api.posts.create({ title: "My Post", markdown: "# Content" });
await api.posts.update({ id: post.id, status: "published", sendNewsletter: true });
```

### Create and publish in one flow (curl)

```bash
POST_ID=$(curl -s -X POST https://public.api.paragraph.com/api/v1/posts \
  -H "Authorization: Bearer $PARAGRAPH_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"title": "My Post", "markdown": "# Content"}' | jq -r '.id')

curl -X PUT "https://public.api.paragraph.com/api/v1/posts/$POST_ID" \
  -H "Authorization: Bearer $PARAGRAPH_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"status": "published", "sendNewsletter": true}'
```

### Export all posts as markdown (SDK)

```typescript
const api = new ParagraphAPI({ apiKey: "<api-key>" });
let cursor: string | undefined;

do {
  const { items, pagination } = await api.posts.list({
    cursor,
    limit: 50,
    includeContent: true,
  });
  for (const post of items) {
    fs.writeFileSync(`${post.slug}.md`, post.markdown);
  }
  cursor = pagination.hasMore ? pagination.cursor : undefined;
} while (cursor);
```

## Post fields reference

**Create / Update body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| title | string | Create only | Post title |
| markdown | string | Create only | Content in markdown |
| subtitle | string | No | Brief summary |
| imageUrl | string | No | Cover image URL |
| slug | string | No | Custom URL slug |
| postPreview | string | No | Preview text (meta description). Keep under 145 chars so it renders without truncation in Google, X, and Farcaster link previews |
| categories | string[] | No | Tags/categories |
| status | string | No | `published`, `draft`, or `archived` |
| sendNewsletter | boolean | No | Email all subscribers on publish. Default: false |
| scheduledAt | number\|null | No | Unix ms timestamp to schedule first-publish. `null` to cancel. Max 30 days out |
| publishedAt | number | No | Unix ms timestamp to set as the post's display publish date. Once set, sticks across re-publishes — useful for backdating imported content |

## Links

- [API reference & playground](https://paragraph.com/docs/api-reference/)
- [SDK on npm](https://www.npmjs.com/package/@paragraph-com/sdk)
- [SDK on GitHub](https://github.com/paragraph-xyz/paragraph-sdk-js)
- [OpenAPI spec](https://github.com/paragraph-xyz/paragraph-sdk-js/blob/main/openapi.json)
