---
name: paragraph-blog-proxy
description: Serve a company's Paragraph blog at a path on its own website, like example.com/blog, through a reverse proxy on Cloudflare Workers, Vercel, Next.js, or Netlify. Paragraph keeps hosting the blog. Trigger when the user wants their Paragraph blog on their own domain under a path, asks to proxy /blog to Paragraph, or asks how to move their blog off a paragraph.com address or a subdomain.
license: MIT
metadata:
  author: paragraph-com
  version: "1.0"
allowed-tools: Bash(curl:*) Read Write Edit
---

# Serve a Paragraph blog at example.com/blog

Paragraph keeps hosting the blog. The company's website forwards one path (usually `/blog`) to Paragraph, so readers and search engines see the articles on the company's own domain. Paragraph renders every link, canonical tag, sitemap, and RSS link with the company's address.

## What you need from the user

- **Handle**: the publication's paragraph.com address, like `paragraph.com/@acme`. Use it exactly as written, lowercase, with the `@`.
- **Site and path**: the domain and path the blog should live at, like `https://acme.com/blog`.
- **Host**: where the website is deployed. Detect it from the repository: `wrangler.toml` or `wrangler.jsonc` (Cloudflare), `vercel.json` (Vercel), `next.config.*` (Next.js), `netlify.toml` (Netlify). Ask if it's unclear.

## Step 1: Point Paragraph at the site

The user does this in Paragraph (it needs a paid plan):

1. Open **Connectors → Channels**, then the **Paragraph-hosted website** page from the **Website** card.
2. In the **Custom domain or path** card, enter the blog's address, like `acme.com/blog`, and choose **Add**.
3. Add the TXT record Paragraph shows at the domain's DNS provider, then refresh the verification status.

Paragraph serves the blog through the proxy only after this check passes. Until the proxy is live, paragraph.com keeps showing the blog.

## Step 2: Add the proxy

Replace `@acme` with the handle and `/blog` with the path. Forward both the path itself and everything under it.

### Cloudflare Workers

Route the Worker on `acme.com/blog` and `acme.com/blog/*`.

```js
const HANDLE = "@acme"
const BLOG_PATH = "/blog"

export default {
  async fetch(request) {
    const url = new URL(request.url)
    const rest = url.pathname.slice(BLOG_PATH.length)
    const target = `https://paragraph.com/${HANDLE}${rest}${url.search}`
    const proxied = new Request(target, request)
    proxied.headers.set("X-Forwarded-Host", url.host)
    proxied.headers.delete("Cookie")
    proxied.headers.delete("Authorization")
    return fetch(proxied, { redirect: "manual" })
  },
}
```

### Vercel

In `vercel.json` (Vercel sends `X-Forwarded-Host` on its own):

```json
{
  "rewrites": [
    { "source": "/blog", "destination": "https://paragraph.com/@acme" },
    {
      "source": "/blog/:path*",
      "destination": "https://paragraph.com/@acme/:path*"
    }
  ]
}
```

### Next.js

In `next.config.js` (Next.js sends `X-Forwarded-Host` on its own, on Vercel or self-hosted):

```js
module.exports = {
  async rewrites() {
    return [
      { source: "/blog", destination: "https://paragraph.com/@acme" },
      {
        source: "/blog/:path*",
        destination: "https://paragraph.com/@acme/:path*",
      },
    ]
  },
}
```

If the config already has `rewrites`, add these two entries to it.

### Netlify

In `netlify.toml` (the `headers` line only works here, not in `_redirects`):

```toml
[[redirects]]
  from = "/blog"
  to = "https://paragraph.com/@acme"
  status = 200
  force = true
  headers = { X-Forwarded-Host = "acme.com" }

[[redirects]]
  from = "/blog/*"
  to = "https://paragraph.com/@acme/:splat"
  status = 200
  force = true
  headers = { X-Forwarded-Host = "acme.com" }
```

### Other hosts

Any reverse proxy works when it forwards `<path>` and `<path>/*` to `https://paragraph.com/<handle>` and `https://paragraph.com/<handle>/*`, keeps the query string, sends `X-Forwarded-Host` with the site's domain, and passes redirects through to the browser instead of following them.

## Step 3: Check it

After deploying, verify with `curl`:

```bash
curl -sI https://acme.com/blog | head -1          # 200
curl -s https://acme.com/blog | grep -o '<link rel="canonical"[^>]*>'
```

The canonical tag should name `https://acme.com/blog`, not paragraph.com. If it names paragraph.com or the page redirects to paragraph.com, the proxy isn't sending `X-Forwarded-Host`, or the address in Step 1 doesn't match the site (check the domain, `www.`, and path).

Add the blog's sitemap, `https://acme.com/blog/sitemap.xml`, to the site's `robots.txt` or submit it in Google Search Console.

## Notes

- Articles, the subscribe page, RSS (`/blog/rss.xml`), and images all load through the proxy.
- `paragraph.com/@acme` redirects readers to the blog on the company site.
- To stop, remove the proxy, then remove the path in the **Custom domain or path** card on the **Paragraph-hosted website** page.
