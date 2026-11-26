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

## 4. Pushes while Cloudflare was disconnected

| | `94742c7` | `c0fa4f0` |
|---|---|---|
| Change | Notes only | Home page and task contract |
| Netlify deploy created | 15:41:55 | 15:58:26 |
| Netlify published | 15:42:05 | 15:58:37 |
| Netlify dashboard build time | 9 s | 10 s |
| Cloudflare | No deployment | No deployment |

- Cloudflare incident "Cloudflare Dashboard and Cloudflare API service issues":
  posted 16:03:50, fix reported 16:39:33, still being monitored at 17:51.
- While it was open, the dashboard loaded only an empty frame and a Wrangler API
  request failed to authenticate. The site kept answering with status 200 from
  the one location tested.
- The incident does not explain the missing deploys on its own: the disconnected
  warning appeared before it (section 3) and was still shown at about 17:47.
- At about 17:51 the warning was gone, after the Git connection was fixed.
- At about 16:04 the HTML served by Netlify was 2646 bytes, against 2111 in a
  local build, and contained a "This site is hosted on Netlify" comment. The two
  copies may not be byte-identical.
