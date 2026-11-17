---
name: paragraph-api
description: Use the Paragraph REST API or TypeScript SDK to manage posts, drafted content (X posts, LinkedIn posts, newsletters, X articles), the publishing schedule, agent runs, chat, publications, subscribers, and analytics on paragraph.com. No installation required — just HTTP requests or `npm install @paragraph-com/sdk`. Trigger when the user asks to integrate with, build on, or call the Paragraph API.
license: MIT
metadata:
  author: paragraph-com
  version: "1.1"
allowed-tools: Bash(curl:*) Bash(npm:*) Bash(npx:*) Read Write WebFetch
---

# Paragraph API

REST API and TypeScript SDK for [Paragraph](https://paragraph.com), which turns a company's work into articles, newsletters, and social posts. Manage posts, drafted content, the schedule, agent runs, publications, and subscribers without installing any CLI tools.

For CLI or MCP access, see the **paragraph-cli** skill instead.

## Choose your interface

| Interface | Install required | Best for |
|-----------|-----------------|----------|
| REST API | No | Any language, `curl`, quick scripts |
| TypeScript SDK | `npm install @paragraph-com/sdk` | TypeScript/JavaScript projects with type safety |

## Authentication

Create an API key in the app at [app.paragraph.com](https://app.paragraph.com) under **Settings → API keys**. Each key belongs to one publication.

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
- **Drafting never sends.** Creating or editing with `api.content` (and `/v1/content`) only writes to the publication's library, and agent runs only write drafts. Say so rather than implying a piece went out.
- **Do not publish, approve, or launch without explicit user approval.** `api.content.publish`, the `api.schedule` approvals, and `api.agents.launchRun` count the same as approving in the app, and the pieces go out for real.
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
  categories: ["launch", "product"],
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

// Set or replace the cover image — the URL is fetched, re-hosted on Paragraph's CDN, and a placeholder is generated
await api.posts.update({
  id: "<post-id>",
  imageUrl: "https://example.com/cover.jpg",
});

// Remove the existing cover image
await api.posts.update({ id: "<post-id>", clearImage: true });

// Delete (by ID or by slug)
await api.posts.delete({ id: "<post-id>" });
await api.posts.delete({ slug: "my-post" });

// Send test newsletter email (draft posts only)
await api.posts.sendTestEmail({ id: "<post-id>" });

// Feed and tags
const { items: feed } = await api.feed.get();
const { items: tagged } = await api.posts.get({ tag: "product" }, { limit: 10 });
```

### Content

The publication's short-form library: X posts and threads, LinkedIn posts, one-off emails, and X articles. Creating or editing a piece never posts, emails, or schedules it; see [Publish and approve](#publish-and-approve) for that. Long-form Paragraph posts are a different resource; use `api.posts` for those.

Kinds are `tweet`, `linkedin`, `newsletter`, and `x_article`. `body` takes the shape its kind uses, validated the same way the Paragraph app validates it. X and LinkedIn attachments still need the app. Newsletters accept either Markdown `body` or a complete rich `bodyJson` document, never both. Read a saved rich newsletter before editing and preserve its authoritative `bodyJson`; converting it to Markdown can lose formatting and media. The JSON document is an object, unlike the stringified `bodyJson` on the separate posts API. Creating or editing a draft does not approve delivery.

```typescript
const api = new ParagraphAPI({ apiKey: "<api-key>" });

// Draft an X thread — one entry per tweet, each under 280 characters
const thread = await api.content.create({
  kind: "tweet",
  title: "Thread on writing in public",
  body: {
    tweets: [
      "Writing in public changes what you write.",
      "Here's what changed for me.",
    ],
  },
});

// Draft a single X post
await api.content.create({
  kind: "tweet",
  title: "Launch note",
  body: { text: "We shipped it." },
});

// Draft a LinkedIn post
await api.content.create({
  kind: "linkedin",
  title: "Launch note",
  body: { text: "We shipped it." },
});

// Draft a one-off email
await api.content.create({
  kind: "newsletter",
  title: "October update",
  body: {
    subject: "What we shipped in October",
    preheader: "Three things worth your time",
    body: "Hi everyone,",
  },
});

// Draft an X Article — body is CommonMark markdown
await api.content.create({
  kind: "x_article",
  title: "Why we rebuilt the editor",
  body: {
    title: "Why we rebuilt the editor",
    body: "## The old editor\n\nIt fought us.",
    canonicalUrl: "https://example.com/posts/why-we-rebuilt-the-editor",
  },
});

// List the library (newest activity first, bodies not included)
const { items, pagination } = await api.content.list();
const { items: drafts } = await api.content.list({ kind: "tweet", status: "draft" });

// Read one piece, with its body — do this before editing
const piece = await api.content.get({ id: "<content-id>" });

// Edit — body replaces the artifact entirely, so send the whole thing
await api.content.update({ id: "<content-id>", body: { text: "Rewritten, and shorter." } });

// Rename — always allowed, even when a queued send has locked the words
await api.content.update({ id: "<content-id>", title: "Launch note, second pass" });

// Archive and restore
await api.content.archive({ id: "<content-id>" });
await api.content.restore({ id: "<content-id>" });
```

`status` on `list` defaults to `all`, which is everything except the pieces you've archived. A piece that was delivered stays listed whether or not it was archived afterwards.

`lockedReason` on a piece explains why it can't be edited right now, or is `null` when it can — a queued or in-flight send locks the words, because they go out exactly as written. Cancel the schedule in the app first.

### Rich newsletter documents

```ts
const document = {
  type: "doc" as const,
  content: [{ type: "paragraph", content: [{ type: "text", text: "Keep this exact wording.", marks: [{ type: "bold" }] }] }],
};
await api.content.create({
  kind: "newsletter",
  title: "Product update",
  body: { subject: "What we shipped", bodyJson: document },
});
```

The REST request uses the same object under `body.bodyJson`:

```bash
curl -X POST https://public.api.paragraph.com/api/v1/content \
  -H "Authorization: Bearer $PARAGRAPH_API_KEY" \
  -H "Content-Type: application/json" \
  --data '{"kind":"newsletter","title":"Product update","body":{"subject":"What we shipped","bodyJson":{"type":"doc","content":[{"type":"paragraph","content":[{"type":"text","text":"Keep this exact wording.","marks":[{"type":"bold"}]}]}]}}}'
```

On update, send the complete saved document and subject/preheader, not a partial node or a Markdown rewrite. The server validates supported editor nodes and safe destinations; invalid documents and queued-body edits are refused.

### Content groups

A group is one identity for a post and everything made out of it — the post, the X thread drawn from it, the LinkedIn version, the newsletter. It's what the team sees as a single stacked row under Content in the Paragraph app.

**When repurposing a post, group what you draft with it.** Seed the group from the post, then pass the id as `bucketId` on every piece derived from it. Without it the team gets unrelated drafts that don't know about each other.

```typescript
const api = new ParagraphAPI({ apiKey: "<api-key>" });

// Get or create the post's group. Safe to repeat: the same id comes back.
const { bucketId } = await api.buckets.createForPost({ postId: "<post-id>" });

// Read it first, so you don't remake something that already exists
const group = await api.buckets.get({ id: bucketId });
const channels = new Set(group.members.map((m) => m.channel));

if (!channels.has("tweet")) {
  await api.content.create({
    kind: "tweet",
    title: "Thread on writing in public",
    body: { tweets: ["Writing in public changes what you write."] },
    bucketId,
  });
}

// Group a draft made before the group existed
await api.content.update({ id: "<content-id>", bucketId });

// Find a post's group without creating one, or list every group
const existing = await api.buckets.forPost({ postId: "<post-id>" }); // { bucketId: string | null }
const { items } = await api.buckets.list({ limit: 20 });
```

Every piece reports its group as `bucketId` on reads and writes, so you can tell what's grouped without guessing.

Each member's `kind` says where to read it: `post` with `api.posts.get({ id })`, `content` with `api.content.get({ id })`. A `kind` of `other` is a member Paragraph groups but the API can't fetch.

A piece already grouped with a different post is refused with a 409 rather than moved — tell the user to ungroup it in the app. Taking a piece out of a group is done in the app, not through the API.

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

// Remove by email (hard delete — irreversible).
await api.subscribers.remove({ email: "user@example.com" });

// Import CSV
const file = new File([csvContent], "subscribers.csv", { type: "text/csv" });
await api.subscribers.importCsv({ file });
```

### Publish and approve

Publishing or approving with an API key counts the same as approving in the app and runs the same checks. Only do it when the user asks.

```typescript
const api = new ParagraphAPI({ apiKey: "<api-key>" });

// Post an X post, X article, or LinkedIn post now, or schedule an X or LinkedIn post up to 30 days out
await api.content.publish({ id: "<content-id>" });
await api.content.publish({ id: "<content-id>", publishAt: "2026-12-01T16:00:00Z" });

// The Calendar, from 30 days ago to 30 days ahead
const { items: entries } = await api.schedule.list({ status: "needs_approval" });

// Approve one at its proposed time, or send it now (also retries a failed send)
const [entry] = entries;
await api.schedule.approve({
  id: entry.id,
  entryRevision: entry.entryRevision,
  contentRevision: entry.contentRevision ?? undefined,
});
await api.schedule.approve({ id: "<entry-id>", publishNow: true });

// Approve everything waiting that isn't on hold, or only the entries you name
const { approved, failed } = await api.schedule.approveAll();

// Move, hold, or take one off the Calendar (the draft stays in the library)
await api.schedule.reschedule({ id: "<entry-id>", publishAt: "2026-12-02T16:00:00Z" });
await api.schedule.hold({ id: "<entry-id>", reason: "Waiting on legal" });
await api.schedule.unschedule({ id: "<entry-id>" });

// Post an X or LinkedIn entry from another connected account. It needs approval again
const { accounts } = await api.schedule.accounts();
await api.schedule.changeAccount({ id: "<entry-id>", accountId: accounts[0].accountId });
```

Pass `entryRevision` and `contentRevision` from the entry you showed the user, so the approval is refused if it changed after they reviewed it. A rescheduled entry, or one moved to another account, needs approval again.

### Agents

Agents, like the Changelog, turn a source into drafts: an article, social posts, and more. A run only writes drafts; nothing goes out until someone approves it.

```typescript
const api = new ParagraphAPI({ apiKey: "<api-key>" });

// What can run, with each agent's field and deliverable keys
const { agents } = await api.agents.list();
const changelog = await api.agents.get({ key: "changelog" });

// Set it up. Only what you send changes, and it changes what the team sees in the app
await api.agents.configure({
  key: "changelog",
  repository: "acme/widget",
  values: { audience: "Developers evaluating the SDK" },
  deliverables: ["article", "x_post", "linkedin_post"],
});

// Start a run. Reusing an idempotencyKey returns the same run instead of starting another
const { runId } = await api.agents.startRun({ key: "changelog", idempotencyKey: "release-2.4.0" });

// Poll every 30 seconds or so; most runs finish in a few minutes
const run = await api.agents.getRun({ key: "changelog", runId });
const { runs } = await api.agents.listRuns({ key: "changelog", limit: 5 });

// Once the user approves, launch the run's pieces together, the same as Launch in the app
await api.agents.launchRun({ key: "changelog", runId, start: "launch_now", timeZone: "America/New_York" });
```

`status` is `queued` or `running`, then `succeeded`, `completed_with_warnings`, `failed`, `blocked`, or `canceled`. `start` on a launch is `suggested` (the default, each piece at its suggested time), `now`, `all_now`, `launch`, or `launch_now`.

### Chat

Ask Paragraph to do anything it does in the app's chat, such as making a cover or changing the brand. Chat can't publish, schedule, or send.

```typescript
const api = new ParagraphAPI({ apiKey: "<api-key>" });

let turn = await api.chat.send({ message: "Make a cover for my latest post" });
while (turn.status === "working") {
  turn = await api.chat.get({ sessionId: turn.sessionId });
}
console.log(turn.reply, turn.files, turn.url);

// Continue the same chat, or ask about a specific post
await api.chat.send({ sessionId: turn.sessionId, message: "Make it darker" });
await api.chat.send({ postId: "<post-id>", message: "Is this ready to publish?" });
```

`status` is `working` while Paragraph is still answering, `done`, or `failed` (send the message again). A `409` means the last message in that chat is still being answered.

### Analytics

Read-only SQL over the publication's analytics, scoped to the publication that owns the key. Don't add publication filters yourself.

```typescript
const api = new ParagraphAPI({ apiKey: "<api-key>" });

const schema = await api.analytics.schema();
const { rows } = await api.analytics.query({
  sql: "SELECT title, total_views, open_rate FROM post_analytics_summary ORDER BY total_views DESC LIMIT 5",
});
```

Prefer the pre-aggregated views (`post_analytics_summary`, `subscriber_engagement_scores`, `blog_subscriber_counts`). SELECT and WITH only, no semicolons, at most 10,000 rows, and a 30-second timeout.

Search and AI visibility isn't in the SQL schema. `api.analytics.searchAi()` returns visits from search engines and AI assistants (ChatGPT, Perplexity, ...), AI crawler hits, and the pages crawlers fetch most. Crawlers are only tracked on a live custom website (`crawlerTrackingAvailable`).

```typescript
const report = await api.analytics.searchAi({ period: "weekly" }); // daily (default), weekly, monthly, all
console.log(report.summary.aiVisits, report.summary.crawlerHits, report.crawlers);
```

`api.me.get()` returns the plan and `usage`: credits used this billing cycle against the limit, and emails sent against the plan's monthly allowance (`usage.emails` is omitted when the plan doesn't limit emails).

### Users

```typescript
const api = new ParagraphAPI();

const userById = await api.users.get({ id: "<user-id>" }).single();
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
- **Notifications to the team** — "email me when we hit 1,000 subscribers", "weekly analytics digest".
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

The same flow over REST:

```bash
curl -X POST https://public.api.paragraph.com/api/v1/api/auth/sessions \
  -H "Content-Type: application/json" \
  -d '{"deviceName": "my-app"}'
curl https://public.api.paragraph.com/api/v1/api/auth/sessions/<session-id>
curl -X DELETE https://public.api.paragraph.com/api/v1/api/auth/sessions/<session-id>
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

# Get posts from a publication
curl https://public.api.paragraph.com/api/v1/publications/<publication-id>/posts?limit=10&includeContent=true

# Get a publication's posts with one tag (for a tab or section on your site)
curl "https://public.api.paragraph.com/api/v1/publications/<publication-id>/posts?category=guides"

# Get post by publication slug + post slug
curl https://public.api.paragraph.com/api/v1/publications/slug/<pub-slug>/posts/slug/<post-slug>

# Get post by publication ID + post slug
curl https://public.api.paragraph.com/api/v1/publications/<publication-id>/posts/slug/<post-slug>

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

# Set/replace the cover image (URL is fetched + re-hosted on Paragraph's CDN)
curl -X PUT https://public.api.paragraph.com/api/v1/posts/<post-id> \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"imageUrl": "https://example.com/cover.jpg"}'

# Remove the existing cover image
curl -X PUT https://public.api.paragraph.com/api/v1/posts/<post-id> \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"clearImage": true}'

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

# Posts by tag, across every publication
curl "https://public.api.paragraph.com/api/v1/posts/tag/<tag>?limit=20"
```

### Content

Drafted X posts, LinkedIn posts, one-off emails, and X articles. Every content endpoint requires auth; the API key identifies the publication. Creating or editing a piece never sends it; see [Publish and approve](#publish-and-approve-1).

```bash
# Draft an X thread (requires auth)
curl -X POST https://public.api.paragraph.com/api/v1/content \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{
    "kind": "tweet",
    "title": "Thread on writing in public",
    "body": {"tweets": ["Writing in public changes what you write.", "Here'"'"'s what changed for me."]}
  }'

# Draft a LinkedIn post (requires auth)
curl -X POST https://public.api.paragraph.com/api/v1/content \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"kind": "linkedin", "title": "Launch note", "body": {"text": "We shipped it."}}'

# Draft a one-off email (requires auth)
curl -X POST https://public.api.paragraph.com/api/v1/content \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{
    "kind": "newsletter",
    "title": "October update",
    "body": {"subject": "What we shipped in October", "body": "Hi everyone,"}
  }'

# List the library — filter by kind and status (requires auth)
curl "https://public.api.paragraph.com/api/v1/content?kind=tweet&status=draft&limit=20" \
  -H "Authorization: Bearer <api-key>"

# Get one piece, with its body (requires auth)
curl https://public.api.paragraph.com/api/v1/content/<content-id> \
  -H "Authorization: Bearer <api-key>"

# Edit — body replaces the artifact entirely (requires auth)
curl -X PATCH https://public.api.paragraph.com/api/v1/content/<content-id> \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"body": {"text": "Rewritten, and shorter."}}'

# Rename (requires auth)
curl -X PATCH https://public.api.paragraph.com/api/v1/content/<content-id> \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"title": "Launch note, second pass"}'

# Archive and restore (requires auth)
curl -X POST https://public.api.paragraph.com/api/v1/content/<content-id>/archive \
  -H "Authorization: Bearer <api-key>"
curl -X POST https://public.api.paragraph.com/api/v1/content/<content-id>/restore \
  -H "Authorization: Bearer <api-key>"
```

`kind` is `tweet`, `linkedin`, `newsletter`, or `x_article`, and `body` takes the shape its kind uses:

| Kind | body fields |
|------|-------------|
| tweet | `text` for a single post, or `tweets` (array, one entry per tweet, each ≤ 280 characters) for a thread — never both |
| linkedin | `text` |
| newsletter | `subject`, either Markdown `body` or rich `bodyJson`, optional `preheader` |
| x_article | `title` (the headline X publishes), `body` as CommonMark markdown, optional `canonicalUrl` |

Sending `body.media` is rejected: media has to be uploaded to X or LinkedIn first, which the API can't do yet. Add images to the draft in the Paragraph app.

Unknown fields are rejected rather than ignored, so a misspelled key comes back as a 400 naming it.

### Content groups

A group is one identity for a post and everything made out of it. Seed it from the post, then pass the id as `bucketId` on everything drafted from that post, so the team sees one stacked row under Content instead of unrelated drafts.

```bash
# Get or create a post's group. Safe to repeat: the same id comes back. (requires auth)
curl -X POST https://public.api.paragraph.com/api/v1/posts/<post-id>/bucket \
  -H "Authorization: Bearer <api-key>"
# -> {"bucketId": "<bucket-id>"}

# Draft into it (requires auth)
curl -X POST https://public.api.paragraph.com/api/v1/content \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{
    "kind": "tweet",
    "title": "Thread on writing in public",
    "body": {"tweets": ["Writing in public changes what you write."]},
    "bucketId": "<bucket-id>"
  }'

# Group a draft made before the group existed (requires auth)
curl -X PATCH https://public.api.paragraph.com/api/v1/content/<content-id> \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"bucketId": "<bucket-id>"}'

# Read a group and everything already made from its post (requires auth)
curl https://public.api.paragraph.com/api/v1/buckets/<bucket-id> \
  -H "Authorization: Bearer <api-key>"

# Find a post's group without creating one (requires auth)
curl https://public.api.paragraph.com/api/v1/posts/<post-id>/bucket \
  -H "Authorization: Bearer <api-key>"
# -> {"bucketId": null} when nothing has been made from it yet

# List every group, most recently active first (requires auth)
curl "https://public.api.paragraph.com/api/v1/buckets?limit=20" \
  -H "Authorization: Bearer <api-key>"
```

Each member's `kind` says which endpoint reads it: `post` with `/v1/posts/{id}`, `content` with `/v1/content/{id}`. A `kind` of `other` is a member Paragraph groups but this API can't fetch.

A piece already grouped with a different post is refused with a 409 rather than moved. Taking a piece out of a group is done in the app.

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

# Remove by email (requires auth — hard delete, irreversible).
curl -X DELETE https://public.api.paragraph.com/api/v1/subscribers \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"email": "user@example.com"}'

# Import CSV (requires auth)
curl -X POST https://public.api.paragraph.com/api/v1/subscribers/import \
  -H "Authorization: Bearer <api-key>" \
  -F "file=@subscribers.csv"
```

### Publish and approve

Every call here counts the same as approving in the app. Only make one when the user asks.

```bash
# Post an X post, X article, or LinkedIn post now (requires auth)
curl -X POST https://public.api.paragraph.com/api/v1/content/<content-id>/publish \
  -H "Authorization: Bearer <api-key>"

# Schedule an X or LinkedIn post instead, up to 30 days out (requires auth)
curl -X POST https://public.api.paragraph.com/api/v1/content/<content-id>/publish \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"publishAt": "2026-12-01T16:00:00Z"}'

# The Calendar: needs_approval, scheduled, failed, or published (requires auth)
curl "https://public.api.paragraph.com/api/v1/schedule?status=needs_approval" \
  -H "Authorization: Bearer <api-key>"

# Approve one at its proposed time. Add "publishNow": true to send it now or retry a failed send (requires auth)
curl -X POST https://public.api.paragraph.com/api/v1/schedule/<entry-id>/approve \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"entryRevision": 3, "contentRevision": "<content-revision>"}'

# Approve everything waiting that isn't on hold. Pass "entries" to approve only some (requires auth)
curl -X POST https://public.api.paragraph.com/api/v1/schedule/approve \
  -H "Authorization: Bearer <api-key>"

# Move one to a new time. It needs approval again (requires auth)
curl -X POST https://public.api.paragraph.com/api/v1/schedule/<entry-id>/reschedule \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"publishAt": "2026-12-02T16:00:00Z"}'

# Pause one until it's approved by name (requires auth)
curl -X POST https://public.api.paragraph.com/api/v1/schedule/<entry-id>/hold \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"reason": "Waiting on legal"}'

# Take one off the Calendar without sending it. The draft stays (requires auth)
curl -X POST https://public.api.paragraph.com/api/v1/schedule/<entry-id>/unschedule \
  -H "Authorization: Bearer <api-key>"

# Post an X or LinkedIn entry from another connected account (requires auth)
curl https://public.api.paragraph.com/api/v1/schedule/accounts \
  -H "Authorization: Bearer <api-key>"
curl -X POST https://public.api.paragraph.com/api/v1/schedule/<entry-id>/account \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"accountId": "<account-id>"}'
```

`approve` returns the entry's new state. `approve` for several returns `approved` (entry ids) and `failed` (each with a `reason`), since one that can't go out doesn't hold back the rest.

### Agents
`<agent-key>` is the agent's `key` from the list, like `changelog`.

```bash
# What can run, with each agent's field and deliverable keys (requires auth)
curl https://public.api.paragraph.com/api/v1/agents \
  -H "Authorization: Bearer <api-key>"
curl https://public.api.paragraph.com/api/v1/agents/<agent-key> \
  -H "Authorization: Bearer <api-key>"

# Set it up. Only what you send changes; "repository" replaces its GitHub source (requires auth)
curl -X PUT https://public.api.paragraph.com/api/v1/agents/<agent-key> \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"repository": "acme/widget", "values": {"audience": "Developers evaluating the SDK"}, "deliverables": ["article", "x_post"]}'

# Start a run. It returns as soon as the work is queued (requires auth)
curl -X POST https://public.api.paragraph.com/api/v1/agents/<agent-key>/runs \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"idempotencyKey": "release-2.4.0"}'

# Check on it, or list recent runs (requires auth)
curl https://public.api.paragraph.com/api/v1/agents/<agent-key>/runs/<run-id> \
  -H "Authorization: Bearer <api-key>"
curl "https://public.api.paragraph.com/api/v1/agents/<agent-key>/runs?limit=5" \
  -H "Authorization: Bearer <api-key>"

# Launch its pieces together once the user approves (requires auth)
curl -X POST https://public.api.paragraph.com/api/v1/agents/<agent-key>/runs/<run-id>/launch \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"start": "launch_now", "timeZone": "America/New_York"}'
```

A launch with no `entries` approves every piece of the run that's waiting and isn't on hold. Pass `entries` (ids from `GET /v1/schedule`) to launch only some. `/v1/playbooks` is a deprecated alias of `/v1/agents`.

### Chat

```bash
# Ask, and get the reply (waits up to about 45 seconds) (requires auth)
curl -X POST https://public.api.paragraph.com/api/v1/chat \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"message": "Make a cover for my latest post"}'
# -> {"sessionId": "...", "status": "working" | "done" | "failed", "reply": "...", "files": [...], "url": "..."}

# While status is "working", check back for the rest of the reply (requires auth)
curl https://public.api.paragraph.com/api/v1/chat/<session-id> \
  -H "Authorization: Bearer <api-key>"
```

Send `sessionId` to continue a chat, and `postId` for a chat about one post.

### Analytics

```bash
# Tables and columns (requires auth)
curl https://public.api.paragraph.com/api/v1/analytics/schema \
  -H "Authorization: Bearer <api-key>"

# Run a read-only query (requires auth)
curl -X POST https://public.api.paragraph.com/api/v1/analytics/query \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"sql": "SELECT title, total_views FROM post_analytics_summary ORDER BY total_views DESC LIMIT 5"}'

# Search and AI visibility: search and AI-referred visits, AI crawler hits (requires auth)
curl "https://public.api.paragraph.com/api/v1/analytics/search-ai?period=weekly" \
  -H "Authorization: Bearer <api-key>"
```

### Users

```bash
curl https://public.api.paragraph.com/api/v1/users/<user-id>
```

### Me

```bash
# The key's publication, its plan, and credits and emails used this billing cycle
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
| imageUrl | string | No | Cover image URL. Fetched server-side and re-hosted on Paragraph's CDN; a blur placeholder is generated. Valid on both create and update |
| clearImage | boolean | No | Update only. Set to `true` to remove the existing cover image. Ignored if `imageUrl` is also provided |
| slug | string | No | Custom URL slug |
| postPreview | string | No | Preview text (meta description). Keep under 145 chars so it renders without truncation in Google, X, and Farcaster link previews |
| categories | string[] | No | Tags/categories |
| status | string | No | `published`, `draft`, or `archived` |
| sendNewsletter | boolean | No | Email all subscribers on publish. Default: false |
| scheduledAt | number\|null | No | Unix ms timestamp to schedule first-publish. `null` to cancel. Max 30 days out |
| publishedAt | number | No | Unix ms timestamp to set as the post's display publish date. Once set, sticks across re-publishes — useful for backdating imported content |

## Links

- [API reference](https://docs.paragraph.com/developers/api-reference)
- [SDK on npm](https://www.npmjs.com/package/@paragraph-com/sdk)
