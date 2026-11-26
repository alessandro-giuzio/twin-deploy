# Measurements

All times are UTC. Recorded on 2026-11-26.

## URLs

| Host | URL |
|---|---|
| Netlify | https://twin-deploy.netlify.app/ |
| Cloudflare Workers | https://twin-deploy.giuzio.workers.dev/ |

Both are meant to rebuild on every push to `main`. On the first test push, only
Netlify did (see section 3).

## 1. First deploy

| | Netlify | Cloudflare Workers |
|---|---|---|
| Commit | `98d8b72` | `main`, then at `98d8b72` (no hash seen in the dashboard) |
| Build start | 12:56:14 | 12:59:58 |
| Live | 12:56:26 (published) | 13:01:06 (triggers deployed) |
| Build end | 12:56:25 | 13:01:11 (after cache upload) |
| **Dashboard build time** | **11 s** | **1 min 13 s** |
| Node / npm | 24.22.3 / 11.19.1 | 24.22.1 / 10.9.2 |
| Packages installed | 205 | 212 |
| Dependency cache | Cold: "Failed to fetch cache" | Restore step logged, no hit or miss shown |

Cloudflare phases:

| Initializing | Cloning | Installing | Building | Deploying |
|---|---|---|---|---|
| 26 s | 1 s | 8 s | 2 s | 36 s |

- Wrangler was not a project dependency, so `npx wrangler deploy` installed
  `wrangler@4.168.0` during the deploy phase. That step took about 19 s
  (13:00:36.6 to 13:00:55.6).
- Cloudflare uploaded 4 assets in 1.15 s. Netlify reported 2 new files uploaded.
- Wrangler warned that `workers_dev` and `preview_urls` are not set in
  `wrangler.jsonc`, so both are enabled by default.
- Cloudflare also lists two dashboard deployments (12:59:51 and 12:59:53) made
  before the first Git build. They are not counted here.

Sources: Netlify deploy `6b082c6d935620f518d74300` (API and deploy log),
Cloudflare build `ecce7755` (build log).

## 2. Default 404 behaviour (before `404.astro`)

Measured at 13:07 with `curl` from one location, sending browser-like `Accept`
and `Sec-Fetch-Mode: navigate` headers. Two requests per path. Paths:
`/no-such-page`, `/no-such-page/`, `/_astro/missing.css`.

| | Netlify | Cloudflare Workers |
|---|---|---|
| Status | 404 | 404 |
| Body | Netlify's "Page not found" HTML, in English | Empty |
| Size | 3984 bytes (1629 compressed) | 0 bytes |
| `content-type` | `text/html; charset=utf-8` | None |
| `cache-control` | `public,max-age=0,must-revalidate` | None |
| Cache header | `cache-status`: `fwd=miss` on both requests | None |
| Time to first byte (6 requests) | 0.26 to 0.39 s | 0.15 to 0.21 s |

The result was the same for all three paths on each host. The timings are a
small sample and not a performance comparison.

## 3. Warm-cache deploy, first attempt

Commit `eefcd32` (one line added to the page, plus these notes). Push started
13:08:15.

| | Netlify | Cloudflare Workers |
|---|---|---|
| Result | Deployed | No build started |
| Build start | 13:08:19 | n/a |
| Published | 13:08:28 | n/a |
| New content seen live | 13:08:30 | Not seen; old page still served at 13:14:24 |
| Dashboard build time | 8 s (first deploy: 11 s) | n/a |
| Dependency cache | "Building with cache" | n/a |
| npm install | "up to date" in 636 ms (first deploy: 205 packages in 3 s) | n/a |
| Files uploaded | 1 | n/a |

Cloudflare's build settings showed: "This project is disconnected from your Git
account. This may cause deployments to fail." When or why it disconnected is not
known. The first Cloudflare build is labelled "Manually deployed", so it does not
show that a push had ever triggered one.

Source: Netlify deploy `6b082f4245c2837a7bc36068`.
