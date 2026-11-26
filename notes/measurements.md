# Measurements

All times are UTC. Recorded on 2026-11-26.

## URLs

| Host | URL |
|---|---|
| Netlify | https://twin-deploy.netlify.app/ |
| Cloudflare Workers | https://twin-deploy.giuzio.workers.dev/ |

Both are meant to rebuild on every push to `main`. The first three test pushes
deployed on Netlify only (sections 3 and 4). Both deployed `3eba4f1`
(section 5).

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

## 5. Warm-cache deploy on both hosts

Commit `3eba4f1` (notes only). Push started 17:52:16. Wrangler still unpinned.

| | Netlify | Cloudflare Workers |
|---|---|---|
| Build start | Deploy created 17:52:21 | 17:52:21 |
| Live | 17:52:32 (published) | New page seen at 17:52:58 |
| Dashboard build time | 10 s | 41 s (first deploy: 1 min 13 s) |

Cloudflare compared with its first deploy:

| | Initializing | Cloning | Installing | Building | Deploying |
|---|---|---|---|---|---|
| First | 26 s | 1 s | 8 s | 2 s | 36 s |
| Warm cache | 14 s | 714 ms | 11 s | 2 s | 14 s |

- Both caches were restored. Installing includes about 5 s to restore the
  dependency cache, then `npm clean-install` for 212 packages in 4 s.
- The log did not report installing Wrangler. About 2 s passed between the
  deploy command and `wrangler 4.168.0` starting (first deploy: about 19 s for
  the install step). The likely reason is that the restored dependency cache
  held the earlier `npx` download. The log does not say so.
- Initializing was also 12 s shorter, so the cache is not the only difference.
- Cloudflare uploaded 2 assets and reused 2, in 1.27 s.
- The two hosts did different work: Cloudflare had not yet deployed the new home
  page, while Netlify already had.

Sources: Netlify deploy `6b0871d5588de84844c09f06`, Cloudflare build `3ed7f003`.

## 6. What each host serves

Checked at 17:53, after both had deployed `3eba4f1`.

| | Netlify | Cloudflare Workers |
|---|---|---|
| Home page size | 2646 bytes | 2111 bytes |
| Same as local `dist/index.html` | No | Yes, byte for byte |

Netlify adds two things to the HTML:

- A comment advertising Netlify, with a link tagged `utm_source=ai-legible`.
- `<script async src="/.netlify/scripts/hud?variant=public">`. The script was
  36372 bytes (11532 compressed). The build itself ships no JavaScript.
