# Task contract: home page

## Goal

A visitor to [src/pages/index.astro](../src/pages/index.astro) understands the
experiment (one Astro site served from Netlify and Cloudflare Workers), can open
both copies, and sees the deploy measurements taken so far. The meta description
already promises real measurements; the page shows none yet.

## Constraints

- Change only `src/pages/index.astro`.
- Visible text is in Spanish.
- Every number comes from [notes/measurements.md](../notes/measurements.md).
  Nothing is estimated or filled in. A missing result is shown as missing.
- No client-side JavaScript, new dependencies or images.
- Style with Tailwind theme tokens. No `tailwind.config.js` and no arbitrary values.
- The `CLAUDE.md` hard rules apply.
- Out of scope: `404.astro`, pinning Wrangler, reconnecting Cloudflare, and the
  404 measurements.
- Do not commit or push without asking. A push to `main` can trigger deploys that
  are part of the measurements, so keep this change apart from the Wrangler pin.

## Done when

1. `./check.sh` ends with `GATE: PASS`, and the output is pasted.
2. `dist/index.html` has `lang="es"`, exactly one `<h1>`, a non-empty `<title>`
   and a non-empty meta description.
3. It links to `https://twin-deploy.netlify.app/` and
   `https://twin-deploy.giuzio.workers.dev/`.
4. A table with a `<caption>` and `<th scope>` headers shows the first deploy
   build time for each host, matching section 1 of the notes, with the date.
5. Every later deploy recorded in the notes appears too. A deploy that did not
   run, such as Cloudflare's in section 3, says so and shows no time.
6. Text next to the table says these are single runs, and that the two dashboard
   times are not a like-for-like performance comparison.
7. The page does not say that a push rebuilds both hosts, unless
   `notes/measurements.md` records a push-triggered Cloudflare build.
8. `dist/index.html` contains no `<script>` element.
9. At 375 px wide the page has no horizontal scroll. Evidence: a screenshot or a
   measured `scrollWidth` from `npm run preview`.
10. `git status` shows no changed files other than `src/pages/index.astro` and
    this contract.
