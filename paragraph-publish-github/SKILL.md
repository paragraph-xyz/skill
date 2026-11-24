---
name: paragraph-publish-github
description: Publish a Paragraph article to the company's own site when that site is built from a GitHub repository (Astro, Next.js, Hugo, Docusaurus, Jekyll, GitHub Pages). Adds the article as a Markdown or MDX file that matches the site's existing posts, then opens a pull request. Trigger when the user asks to publish, copy, or sync a Paragraph article or post to their own blog, site, or repo.
license: MIT
compatibility: Run it in a clone of the site's repository. Needs git, the GitHub CLI (`gh`) signed in with write access to the repository, and either the Paragraph MCP server or the Paragraph CLI.
metadata:
  author: paragraph-com
  version: "1.0"
allowed-tools: Bash(paragraph:*) Bash(git:*) Bash(gh:*) Bash(jq:*) Read Write Edit Glob Grep
---

# Publish a Paragraph article to your site through GitHub

Paragraph writes and stores the article. Your site's repository is where it gets published: this skill adds the article as a file next to your existing posts and opens a pull request. Merging the pull request publishes it, so your team reviews it the way it reviews any other change.

Paragraph can also do this on its own. With GitHub connected, ask Paragraph in chat to publish your articles to your repository, and every scheduled article, the Changelog's included, opens its pull request as it publishes. See [Publish to your site with GitHub](https://docs.paragraph.com/publishing/github).

## 1. Get the article

With the Paragraph MCP server (`claude mcp add paragraph --transport http https://mcp.paragraph.com/mcp`), find the article with `list-posts` and read it with `get-post`.

With the CLI (`npm install -g @paragraph-com/cli`, then `paragraph login`):

```bash
paragraph post list --json
paragraph post get --id <post-id> --json > article.json
```

Use these fields: `title`, `subtitle`, `slug`, `imageUrl` (the cover), `publishedAt`, `categories`, and `markdown` (the body, without the title).

If the user named the article, match it by title. If more than one matches, ask which one.

## 2. Learn how the site stores posts

Don't guess the format. Read the repository:

1. Find the content folder: look for existing `.md` or `.mdx` posts, for example `src/content/blog/`, `content/posts/`, `posts/`, `_posts/`, `blog/`, or `docs/blog/`. Check the framework's config (`astro.config.*`, `next.config.*`, `hugo.toml`, `docusaurus.config.*`, `_config.yml`) and any content schema (`src/content/config.ts`) if it's unclear.
2. Open two or three recent posts. Note:
   - The file extension (`.md` or `.mdx`) and file naming (`my-post.md`, `2026-11-17-my-post.md`, or a folder with `index.mdx`).
   - Every frontmatter field, its exact name and format: `date` or `pubDate`, ISO date or full timestamp, `tags` as a list, `draft: false`, `author`, `image` or `cover` or `heroImage`.
   - Required fields in the content schema, if there is one.
3. If the site has no posts yet, ask where posts should go before creating a folder.

## 3. Write the file

Build the frontmatter from the article, using the site's field names:

- Title from `title`, description from `subtitle` (or the first sentences of the article when it has none), date from `publishedAt` (today if it's not published), tags from `categories`, cover from `imageUrl`.
- Search engines and AI answers read the page's description, image and tags. Check the post layout or content schema for the fields it reads them from, and fill those.
- The byline is the article's own author, shaped the way the other posts write theirs. Never copy another person's name or author key. If the site looks authors up in a file (like `authors.yml` or `authors.json`), add the author there the way the other entries are written, or ask who to credit.
- If the other posts keep their covers in the repository, download the cover from `imageUrl` into the same place and point the field at it, so the article shows a cover like theirs.
- Fill other required fields the article doesn't have the way the other posts do, or ask.
- Never leave placeholder values.

If the blog page lists posts by hand (a data file like `src/data/posts.ts`, or links typed into the blog page) instead of building the list from the folder, add the article to that list the way the other entries are written, newest first.

Then the body, from `markdown`:

- Convert underlined headings (`Title` over a line of `===` or `---`) to `#` and `##` headings, and indented code blocks to fenced ones, so they match the rest of the site.
- Images stay on Paragraph's CDN and work as they are. Only download them into the repository if the site's other posts keep images locally.
- For `.mdx`, escape `{`, `}`, and `<` in prose (not in code) with a backslash, since MDX reads them as code. Keep `<u>` tags.

Name the file from `slug` with the site's naming pattern, and write it into the content folder.

If the site runs locally (`npm run dev` or the framework's build command), build it once to check the new post renders.

## 4. Open the pull request

```bash
git checkout -b paragraph/<slug>
git add <path> <list file, if you changed one>
git commit -m "Add <title>" -m "Published from Paragraph"
git push -u origin paragraph/<slug>
gh pr create --title "<title>" --body "Adds \"<title>\" from Paragraph. Merge it to publish the article."
```

Share the pull request link. The article isn't live until the pull request is merged and the site deploys.

## Updating a published article

To change an article that's already on the site, read the new version from Paragraph, update the same file, and open a new pull request. Don't create a second file for it.

## Links

- [Paragraph](https://paragraph.com)
- [MCP server](https://docs.paragraph.com/developers/mcp)
- [CLI](https://docs.paragraph.com/developers/cli)
