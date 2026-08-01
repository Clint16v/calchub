# External app.js rollout — fixing "Google chose different canonical than user"

## What changed, and why
Every one of your 198 pre-rendered pages previously embedded the entire ~880KB app bundle inline — byte-for-byte identical across all of them. Only a small fraction of each page's actual bytes were unique content. Google's duplicate-content detection likely saw that overwhelming shared-boilerplate ratio and started overriding your correct per-page canonical tags, picking its own (wrong) canonical instead — this is the "Duplicate, Google chose different canonical than user" issue in Search Console.

**The fix:** the shared app code now lives in exactly one file, `app.js`, referenced by every page via `<script src="/app.js"></script>` instead of being duplicated inline. Each page is now ~100-130KB instead of ~880KB, and almost entirely unique content.

**Bonus, not just an SEO fix:** browsers cache `app.js` once and reuse it across every page a visitor loads — genuinely faster for real users too, not just something aimed at a crawler.

## What's in this zip
- `index.html` — now ~99KB (was ~869KB), references `/app.js` externally
- `app.js` — the extracted shared application code (~770KB)
- `calc/`, `article/`, `cat/` — all 198 pre-rendered pages, regenerated fresh from the current codebase (includes the image/og:image work from last time — nothing was lost or reverted)
- `vercel.json` — updated to exclude `/app.js` and `/images/` from the SPA rewrite rule (critical — without this, requests for `app.js` could get accidentally served as HTML instead of JavaScript), plus caching headers
- `images/`, `icons/`, `manifest.json`, `robots.txt`, `sitemap.xml` — included for completeness, unchanged from before

## Deploy steps
1. Replace your entire repo contents with what's in this zip (this is a full, clean replacement — safer than trying to merge given how many files changed)
2. Commit and push

## How to verify it worked
1. Visit `calchub.dpdns.org/app.js` directly — you should see raw JavaScript code, **not** HTML. This is the single most important check; if this ever shows HTML instead, something is broken and needs immediate attention.
2. Spot-check a few pages (View Page Source) — confirm they're much smaller than before and reference `<script src="/app.js">` near the bottom
3. Confirm the site still works normally — calculators still calculate, charts still render, navigation works
4. In Search Console, use URL Inspection → Request Indexing on a handful of the previously-flagged pages, then check back in 1-2 weeks to see if the "chose different canonical" count is dropping

## Validation performed before this was handed off
- Full syntax check on the extracted `app.js`
- Rendered pages via a real local HTTP server (not just passing HTML as a string) so `app.js` was fetched exactly the way a real browser fetches it in production
- Confirmed all 198 pages render correctly with accurate content, titles, and canonical URLs
- Confirmed all prior work survived intact: 153 FAQs, 153 How It Works, 153 Common Mistakes, 152 MathSolver entries, 40 charts, 76 article links, 7 images with correct og:image tags
- Explicitly tested that `/app.js` serves as JavaScript and not as the rewritten SPA HTML — this was the one failure mode that could break the entire site, so it got tested directly rather than assumed
- Tested the homepage and the SPA fallback path (for any route without a pre-rendered file) still work correctly with the new external script

## One thing to know
`app.js` is served with a 1-hour cache (`max-age=3600`). If you deploy a change to the app in the future, visitors might see the old cached version for up to an hour. This is a reasonable default for now — if it becomes annoying, the standard fix is adding a content hash to the filename (e.g. `app.abc123.js`) so each deploy gets a fresh, uniquely-cacheable filename. Not needed today, just worth knowing for later.
