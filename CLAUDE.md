# CLAUDE.md

@AGENTS.md

## Stack
Astro + Tailwind v4 + TypeScript, static output, deployed on Netlify and Cloudflare Workers.
If both hosts are connected to the repo, one `git push` deploys twice.

## Where things live
- Pages: src/pages
- Styles and design tokens: src/styles/global.css
- Static assets: public
- Deploy config: netlify.toml, wrangler.jsonc

## Hard rules
- Tailwind v4 is CSS-first: tokens live in @theme in src/styles/global.css. Do not create tailwind.config.js.
- Use theme tokens, not arbitrary values like text-[#1a2b3c], unless no token exists.
- Components ship no JavaScript by default. Add client:* only when interaction needs it.
- Every page keeps a unique title, a meta description and one h1.
- Images go through astro:assets with alt text.
- Never read or edit .env files.
- Never push or deploy without asking.

## How to work
- At session start, read STATE.md if it exists.
- Before coding, restate the task: goal, constraints, done when.
- Done means ./check.sh prints GATE: PASS. Paste the output.
- Same error twice: stop and write what you tried to STATE.md.
