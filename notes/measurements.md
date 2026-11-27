# Measurements

All times are UTC. Recorded on 2026-11-26.

Where a Netlify build time is marked "API", it is the `deploy_time` field. It
matched the dashboard figure for the deploys in sections 1 and 3.

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
| Netlify build time (API) | 9 s | 10 s |
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
| Build time (Netlify: API, Cloudflare: dashboard) | 10 s | 41 s (first deploy: 1 min 13 s) |

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

## 7. First deploy with Wrangler pinned

Commit `8c60e00` adds `wrangler@4.168.0` as a dev dependency (and these notes up
to section 6). Push started 18:08:46.

| | Netlify | Cloudflare Workers |
|---|---|---|
| Build start | Deploy created 18:08:50 | 18:08:51 |
| Live | 18:09:05 (published) | 18:09:37 (triggers deployed) |
| Build time (Netlify: API, Cloudflare: dashboard) | 13 s | 50 s |

Cloudflare across its three builds:

| | Initializing | Cloning | Installing | Building | Deploying | Total |
|---|---|---|---|---|---|---|
| First, unpinned | 26 s | 1 s | 8 s | 2 s | 36 s | 1 min 13 s |
| Warm cache, unpinned | 14 s | 714 ms | 11 s | 2 s | 14 s | 41 s |
| First pinned | 29 s | 731 ms | 10 s | 2 s | 8 s | 50 s |

- The log showed both restore steps starting but, unlike the warm build, no
  "restored" message for either. The lockfile had changed, which is the likely
  reason for the dependency cache. Both caches were uploaded again at the end.
- `npm clean-install` added 285 packages in 10 s (before the pin: 212).
- Wrangler 4.168.0 started about 1.5 s after the deploy command, with no
  separate download reported.
- No assets needed uploading, because the page had not changed. This also
  shortens the deploy phase, so its drop to 8 s is not due to the pin alone.
- Initializing ranged from 14 to 29 s across the three builds, enough to hide
  smaller differences in the totals.
- Netlify took 13 s, against 8 to 11 s on its five earlier builds. Installing
  the new dev dependency is a likely reason; its log was not read.

Sources: Netlify deploy `6b0875b18a1bd8b619c825c2` (API), Cloudflare build
`2d54a082` (build log).

## 8. Warm-cache deploy with Wrangler pinned

Commit `7b3e70a` (home page and notes; dependencies unchanged). Push started
18:13:38.

| | Netlify | Cloudflare Workers |
|---|---|---|
| Build start | Deploy created 18:13:43 | 18:13:44 |
| Published or triggers deployed | 18:13:54 | 18:14:33 |
| New page seen live | 18:14:00 | 18:14:35 |
| Build time (Netlify: API, Cloudflare: dashboard) | 10 s | 52 s |

Cloudflare, warm cache, without and with the pin:

| | Initializing | Cloning | Installing | Building | Deploying | Total |
|---|---|---|---|---|---|---|
| Unpinned (`3eba4f1`) | 14 s | 714 ms | 11 s | 2 s | 14 s | 41 s |
| Pinned (`7b3e70a`) | 21 s | 1 s | 15 s | 2 s | 14 s | 52 s |

- Both caches were restored. `npm clean-install` took 10 s for 285 packages,
  against 4 s for 212 packages without the pin.
- The deploy phase took 14 s both times. Wrangler started about 1.6 s after the
  deploy command (unpinned warm build: 2.0 s).
- Each build uploaded 2 assets and reused 2.
- Of the 11 s difference in totals, 7 s is in initializing, before any project
  step runs. Installing accounts for 4 s.
- In these single runs, pinning did not make the warm build faster. It installs
  more packages, and the unpinned warm build did not report downloading Wrangler
  either. The pin fixes which version runs.

Source: Cloudflare build `51cb4a1b` (build log), Netlify deploy
`6b0876d6b5cb97e8f1436533` (API).

## 9. 404 behaviour with `404.astro`, no host configuration changed

Commit `91143cb`. Push started 18:22:31.

| Build | Netlify | Cloudflare Workers |
|---|---|---|
| New home page seen live | 18:22:51 | 18:23:26 |
| Build time (Netlify: API, Cloudflare: dashboard) | 10 s | 52 s |

Cloudflare phases: initializing 11 s, cloning 1 s, installing 25 s, building
4 s, deploying 11 s. Both caches were restored. `npm clean-install` took 21 s for
the same 285 packages that took 10 s in sections 7 and 8. Install time varies
enough between runs that the 4 s difference in section 8 cannot be put down to
the pin with confidence.

404 requests, measured at 18:23 in the same way as section 2:

| | Netlify | Cloudflare Workers |
|---|---|---|
| Status | 404 | 404 |
| Body | The custom page, in Spanish | Empty |
| Size | 1121 bytes (656 compressed), before: 3984 (1629) | 0 bytes, before: 0 |
| `content-type` | `text/html; charset=utf-8` | None |
| `cache-control` | `public,max-age=0,must-revalidate` | None |
| Cache header | `cache-status`: `fwd=miss` on every request | None |
| Time to first byte (6 requests) | 0.25 to 0.43 s | 0.14 to 0.17 s |
| Direct request to `/404` | 200, 1121 bytes | 200, 586 bytes |
| Direct request to `/404.html` | 200, 1121 bytes | 307 redirect to `/404` |

- The result was the same for all three missing paths on each host.
- Netlify used `404.html` with no configuration. Its 1121 bytes are the 586 bytes
  of the local file plus the comment and script described in section 6.
- Cloudflare uploaded the page (it is served at `/404`) but did not use it for
  missing paths. `wrangler.jsonc` does not set `assets.not_found_handling`.

Source: Cloudflare build `169c38a9` (build log), Netlify deploy
`6b0878eaf27d416b500c7266` (API).

## 10. 404 behaviour with Cloudflare configured

Commit `e939351` adds `"not_found_handling": "404-page"` under `assets` in
`wrangler.jsonc`. Push started 19:18:58.

| Build | Netlify | Cloudflare Workers |
|---|---|---|
| New page seen live | 19:19:19 | 19:19:48 |
| Build time (Netlify: API, Cloudflare: dashboard) | 12 s | 49 s |

Cloudflare phases: initializing 18 s, cloning 1 s, installing 13 s, building
2 s, deploying 14 s. Both caches were restored, and `npm clean-install` took 11 s.

404 requests, measured at 19:20 in the same way as section 2:

| | Netlify | Cloudflare Workers |
|---|---|---|
| Status | 404 | 404 |
| Body | The custom page | The custom page, identical to local `404.html` |
| Size | 1121 bytes (656 compressed) | 586 bytes (368 compressed) |
| `content-type` | `text/html; charset=utf-8` | `text/html` |
| `cache-control` | `public,max-age=0,must-revalidate` | `public, max-age=0, must-revalidate` |
| Cache header | `cache-status`: `fwd=miss` on every request | `cf-cache-status: HIT` on every request |
| Time to first byte (6 requests) | 0.30 to 0.40 s | 0.15 to 0.19 s |
| Direct request to `/404` | 200 | 200 |
| Direct request to `/404.html` | 200 | 307 redirect to `/404` |

The result was the same for all three missing paths on each host, and a request
to `/no-such-page` without browser headers also got the custom page.

Source: Cloudflare build `8058fee5` (build log), Netlify deploy
`6b08862587229df943a51527` (API).

## 404 summary

| Stage | Netlify | Cloudflare Workers |
|---|---|---|
| Default (section 2) | Netlify's English page, 3984 bytes | Empty body |
| `404.astro` added (section 9) | Custom page, no setting needed | Empty body |
| `not_found_handling` set (section 10) | Custom page | Custom page |

All three stages returned status 404 for missing paths. The timings come from one
location and a few requests, so they are not a performance comparison.
