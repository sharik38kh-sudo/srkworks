<!--
TODO for the maintainer / AI assistant updating this file:
This README was written WITHOUT access to the repository's files. Before publishing, verify every
item marked [VERIFY] or written as a YOUR-... placeholder against the real repo:
  - the Cloudflare domain (replace YOUR-CLOUDFLARE-DOMAIN everywhere)
  - the Vercel domain(s) (replace YOUR-PROJECT.vercel.app / YOUR-VERCEL-DOMAIN)
  - the actual contents of vercel.json (the example below is a reference, not a copy)
  - the file/folder list in "Repository Structure"
  - any package.json scripts (none are assumed here)
Delete this comment once everything is verified.
-->

# SRKWorks — Vercel Redirect Layer

This repository is a **redirect-only deployment** for SRKWorks. It is hosted on [Vercel](https://vercel.com) and forwards every incoming request to the matching URL on the main SRKWorks website, which is hosted on **Cloudflare Pages**.

> **The Cloudflare Pages site is the source of truth.** All website content, pages, and indexed URLs live there. This repository contains no website content of its own.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Deployment Architecture](#deployment-architecture)
3. [How Redirects Work](#how-redirects-work)
4. [Repository Structure](#repository-structure)
5. [Local Development Setup](#local-development-setup)
6. [Deploying to Vercel](#deploying-to-vercel)
7. [Configuring and Testing `vercel.json`](#configuring-and-testing-verceljson)
8. [SEO Considerations](#seo-considerations)
9. [Troubleshooting](#troubleshooting)
10. [Maintenance Checklist](#maintenance-checklist)

---

## Project Overview

| Item | Value |
|---|---|
| Project name | SRKWorks |
| Purpose | Forward Vercel traffic to the main Cloudflare Pages website |
| Main website (source of truth) | `https://YOUR-CLOUDFLARE-DOMAIN` |
| Vercel deployment | `https://YOUR-PROJECT.vercel.app` (and any custom domain attached to it) |
| Redirect type | Permanent (HTTP 308) unless configured otherwise |

### What this repository does

- Homepage visits redirect to the Cloudflare homepage.
- Product pages, store pages, service pages, and every other path redirect to the **same path** on Cloudflare.
- URL paths and query parameters are preserved wherever possible.

### What this repository does not do

- It does not host or render any website content.
- It does not generate its own `sitemap.xml` or `robots.txt` for search engines to index (see [SEO Considerations](#seo-considerations)).
- It does not replace or modify the Cloudflare Pages website.

---

## Deployment Architecture

```text
┌──────────┐    push    ┌──────────┐   request   ┌────────────────────┐
│  GitHub  │ ─────────► │  Vercel  │ ──────────► │  Cloudflare Pages  │
│  (repo)  │   deploy   │ redirect │  HTTP 308   │  (main website)    │
└──────────┘            └──────────┘             └────────────────────┘
```

1. **GitHub** stores this repository, including `vercel.json`.
2. **Vercel** builds and deploys the repository whenever changes are pushed.
3. A visitor opens a Vercel URL. Vercel answers with a redirect response and a `Location` header pointing to Cloudflare.
4. The browser follows the redirect, and **Cloudflare Pages** serves the real page.

Because the redirect happens before any file is served, the visitor's address bar ends up on the Cloudflare URL.

---

## How Redirects Work

Vercel reads the `redirects` array in `vercel.json` and evaluates rules **in order from top to bottom**. The first matching rule wins.

### Request flow

```text
Visitor → https://YOUR-PROJECT.vercel.app/products/example?color=blue
        → Vercel responds: 308, Location: https://YOUR-CLOUDFLARE-DOMAIN/products/example?color=blue
        → Browser requests the Cloudflare URL
        → Cloudflare Pages serves the page
```

### Examples

| Visitor opens (Vercel) | Redirected to (Cloudflare) |
|---|---|
| `https://YOUR-PROJECT.vercel.app/` | `https://YOUR-CLOUDFLARE-DOMAIN/` |
| `https://YOUR-PROJECT.vercel.app/products/example-product` | `https://YOUR-CLOUDFLARE-DOMAIN/products/example-product` |
| `https://YOUR-PROJECT.vercel.app/store/example-store` | `https://YOUR-CLOUDFLARE-DOMAIN/store/example-store` |
| `https://YOUR-PROJECT.vercel.app/services/example-service` | `https://YOUR-CLOUDFLARE-DOMAIN/services/example-service` |
| `https://YOUR-PROJECT.vercel.app/search?q=test` | `https://YOUR-CLOUDFLARE-DOMAIN/search?q=test` |

> The paths above (`/products/`, `/store/`, `/services/`) are illustrative. Use the real paths from your Cloudflare site. **[VERIFY]**

### Reference configuration

The following shows the standard pattern for a "forward everything" setup. **Compare it with the actual `vercel.json` in this repository; do not assume they are identical. [VERIFY]**

```json
{
  "redirects": [
    {
      "source": "/:path*",
      "destination": "https://YOUR-CLOUDFLARE-DOMAIN/:path*",
      "permanent": true
    }
  ]
}
```

What each field means:

| Field | Meaning |
|---|---|
| `source` | The incoming Vercel path pattern. `/:path*` matches `/` and every nested path. |
| `destination` | Where to send the visitor. `:path*` re-inserts the matched path. |
| `permanent` | `true` sends HTTP 308 (permanent). `false` sends HTTP 307 (temporary). |

> Use `"permanent": false` while testing, so browsers and search engines do not cache the redirect. Switch to `true` once everything works.

### Special cases go first

If a specific path must behave differently (for example, an old URL that maps to a new one), add that rule **above** the catch-all rule:

```json
{
  "redirects": [
    {
      "source": "/old-products/:slug",
      "destination": "https://YOUR-CLOUDFLARE-DOMAIN/products/:slug",
      "permanent": true
    },
    {
      "source": "/:path*",
      "destination": "https://YOUR-CLOUDFLARE-DOMAIN/:path*",
      "permanent": true
    }
  ]
}
```

If the catch-all rule is placed first, the specific rule will never run.

---

## Repository Structure

```text
.
├── .gitignore      # Git ignore rules for Vercel and system artifacts
├── readme.md       # Documentation and operational guide
└── vercel.json     # Redirect configuration (core of this repository)
```

---

## Local Development Setup

No build step is required for a redirect-only project. You only need Git, and optionally Node.js to test locally.

### 1. Clone the repository

```bash
git clone https://github.com/YOUR-GITHUB-USERNAME/YOUR-REPO-NAME.git
cd YOUR-REPO-NAME
```

### 2. (Optional) Run Vercel locally

This requires [Node.js](https://nodejs.org) and a free Vercel account.

```bash
# Install the Vercel CLI (one time)
npm install --global vercel

# Log in
vercel login

# Link this folder to your Vercel project (follow the prompts)
vercel link

# Start the local dev server
vercel dev
```

Then open `http://localhost:3000/products/example` in a browser or test with `curl` (see [Testing redirects](#testing-redirects)).

> If you do not want to install anything locally, you can skip this step and test on a Vercel **Preview Deployment** instead (see below).

---

## Deploying to Vercel

### First-time setup

1. Push this repository to GitHub.
2. Sign in to [vercel.com](https://vercel.com) and click **Add New → Project**.
3. Import this GitHub repository.
4. Configure the project:

   | Setting | Suggested value |
   |---|---|
   | Framework Preset | `Other` |
   | Build Command | *(leave empty)* |
   | Output Directory | *(leave empty / default)* |
   | Install Command | *(leave empty)* |

   > These values assume a repository that contains only `vercel.json`. **[VERIFY]** If the repository has a `package.json` with build scripts, adjust accordingly.

5. Click **Deploy**.

### Ongoing deployments

| Action | Result |
|---|---|
| Push to the production branch (usually `main`) | New **Production** deployment |
| Push to another branch or open a pull request | New **Preview** deployment with its own URL |

### Adding a custom domain (optional)

1. In Vercel, open **Project → Settings → Domains**.
2. Add your domain and follow the DNS instructions Vercel shows.

> **Important:** Do not attach the same hostname to both Vercel and Cloudflare Pages. The Vercel domain and the Cloudflare destination must be **different hostnames**, otherwise you create a redirect loop.

---

## Configuring and Testing `vercel.json`

### Configuration checklist

- [ ] `destination` uses `https://` and the exact canonical hostname of the Cloudflare site (include `www.` only if the Cloudflare site uses it).
- [ ] The `destination` hostname is **not** the same as the Vercel hostname.
- [ ] Specific rules come **before** the catch-all rule.
- [ ] The file is valid JSON (no trailing commas, no comments).

Validate the JSON syntax:

```bash
# Using Node.js
node -e "JSON.parse(require('fs').readFileSync('vercel.json','utf8')); console.log('vercel.json is valid JSON')"
```

### Testing redirects

Use `curl -I` to see the response headers without following the redirect.

```bash
# Homepage
curl -I https://YOUR-PROJECT.vercel.app/

# A product page
curl -I https://YOUR-PROJECT.vercel.app/products/example-product

# A URL with query parameters (quote the URL so the shell does not break it)
curl -I "https://YOUR-PROJECT.vercel.app/search?q=test&page=2"
```

A correct response looks like this:

```text
HTTP/2 308
location: https://YOUR-CLOUDFLARE-DOMAIN/products/example-product
```

To follow the redirect and confirm the final page loads:

```bash
curl -IL https://YOUR-PROJECT.vercel.app/products/example-product
```

The last response should be `HTTP/2 200` from the Cloudflare domain.

### Test table

| Test | Command | Expected result |
|---|---|---|
| Homepage | `curl -I https://YOUR-PROJECT.vercel.app/` | `308` → `https://YOUR-CLOUDFLARE-DOMAIN/` |
| Nested path | `curl -I https://YOUR-PROJECT.vercel.app/products/example` | `308` → same path on Cloudflare |
| Query string | `curl -I "https://YOUR-PROJECT.vercel.app/?a=1&b=2"` | `Location` contains `?a=1&b=2` |
| Final page | `curl -IL https://YOUR-PROJECT.vercel.app/` | Ends with `200` |
| Loop check | `curl -IL --max-redirs 5 https://YOUR-PROJECT.vercel.app/` | Finishes before 5 redirects |

You can also test in a browser: open the Vercel URL and confirm the address bar changes to the Cloudflare URL.

---

## SEO Considerations

The goal is for search engines to index **only the Cloudflare site** and treat the Vercel URL as a forwarding address.

### Canonical URLs

- Every page on the Cloudflare site should include a canonical tag pointing to its own Cloudflare URL:

  ```html
  <link rel="canonical" href="https://YOUR-CLOUDFLARE-DOMAIN/products/example-product" />
  ```

- This repository does not serve HTML pages, so canonical tags are managed in the Cloudflare site's repository.

### `sitemap.xml`

- Keep `sitemap.xml` on the Cloudflare site only: `https://YOUR-CLOUDFLARE-DOMAIN/sitemap.xml`.
- All URLs inside it must use the Cloudflare domain, never the Vercel domain.
- This repository does not need its own sitemap. Adding one would risk telling Google to index the redirecting URLs.

### `robots.txt`

- Keep `robots.txt` on the Cloudflare site: `https://YOUR-CLOUDFLARE-DOMAIN/robots.txt`.
- Reference the Cloudflare sitemap inside it:

  ```text
  Sitemap: https://YOUR-CLOUDFLARE-DOMAIN/sitemap.xml
  ```

- Do not block the Vercel domain in a way that stops crawlers from reading the redirect. Crawlers need to follow the redirect to discover the final URL.

### Google indexing

1. Add the **Cloudflare domain** as a property in [Google Search Console](https://search.google.com/search-console) and submit its sitemap there.
2. Use the **URL Inspection** tool on Cloudflare URLs to check indexing status.
3. Use permanent redirects (`308`/`301`) on Vercel once testing is finished. Google treats these as a signal that the destination is the preferred URL.
4. Avoid redirect chains. A Vercel URL should reach the final Cloudflare page in **one** hop. Point `destination` at the exact canonical host (for example, the correct `www` or non-`www` version), so Cloudflare does not add a second redirect.
5. Vercel preview deployments are intended for testing, not for indexing. Do not link to preview URLs publicly.

### Redirect vs. proxy (rewrite)

This project uses **redirects**, so the visitor lands on the Cloudflare URL. A *rewrite* would keep the Vercel URL in the address bar while showing Cloudflare content, which creates duplicate-content risk and is **not** what this repository does.

---

## Troubleshooting

### Quick diagnosis table

| Symptom | Likely cause | Fix |
|---|---|---|
| Browser says "too many redirects" | Vercel and Cloudflare hostnames are the same, or Cloudflare redirects back to Vercel | See [Redirect loops](#redirect-loops) |
| 404 on a Vercel URL | `vercel.json` is missing, invalid, or not deployed | See [404 errors](#404-errors) |
| 404 on the Cloudflare URL after redirect | That path does not exist on the Cloudflare site | Open the same path directly on Cloudflare |
| Product page lands on the wrong page | A broader rule matches before the specific one, or the path differs on Cloudflare | See [Incorrect product URLs](#incorrect-product-urls) |
| Query parameters missing | Destination URL was written with a fixed query string, or a different rule matched | Test with `curl -I` and review the matching rule |
| Old redirect still happens after a fix | Browser cached a permanent redirect | Test in a private window, or use `"permanent": false` while debugging |

### Broken redirects

1. Confirm the latest commit is deployed: **Vercel → Project → Deployments**, and check that the status is **Ready**.
2. Check the **Production** deployment, not just a preview.
3. Validate `vercel.json` syntax (see [Configuration checklist](#configuration-checklist)).
4. Run `curl -I` against the failing URL and read the `Location` header.
5. Compare the `Location` value with the URL you expected.

### 404 errors

1. Confirm `vercel.json` is in the **root** of the repository (not in a subfolder), unless the project's Root Directory setting says otherwise. **[VERIFY]**
2. Check the build logs in Vercel for configuration errors.
3. Confirm the deployed branch is the branch containing your latest `vercel.json`.
4. If the 404 happens **after** the redirect, the page does not exist on Cloudflare. Open `https://YOUR-CLOUDFLARE-DOMAIN/the/same/path` directly to confirm.

### Redirect loops

1. Run:

   ```bash
   curl -IL --max-redirs 10 https://YOUR-PROJECT.vercel.app/
   ```

2. If the same URLs repeat, check the following:
   - The `destination` hostname in `vercel.json` is **not** a Vercel hostname.
   - The domain in `destination` is not attached to the Vercel project (**Settings → Domains**).
   - Cloudflare (Redirect Rules, Page Rules, `_redirects`, or DNS) is not forwarding visitors back to the Vercel domain.
   - The `www` / non-`www` redirect on Cloudflare agrees with the host used in `destination`.
3. Fix the conflicting rule and redeploy.

### Incorrect product URLs

1. Open the product page directly on Cloudflare and copy its exact path.
2. Test the Vercel equivalent with `curl -I` and compare the two paths.
3. Check rule order in `vercel.json`: **specific rules first, catch-all last**.
4. Look for path differences between the two sites:
   - Trailing slash (`/products/item` vs `/products/item/`)
   - Upper/lower case in the slug
   - Special or encoded characters in the slug
5. If a product path on Cloudflare was renamed, add an explicit rule for the old path above the catch-all rule.
6. Redeploy and re-test.

---

## Maintenance Checklist

When the Cloudflare site changes (new sections, renamed paths, new domain):

- [ ] Update `destination` in `vercel.json` if the Cloudflare hostname changes.
- [ ] Add explicit redirect rules for any renamed or moved paths.
- [ ] Re-run the [redirect tests](#testing-redirects).
- [ ] Confirm `sitemap.xml` and `robots.txt` on Cloudflare still reference Cloudflare URLs only.
- [ ] Check Google Search Console for crawl or indexing errors.

---

## License

Add license information here if applicable. **[VERIFY]**