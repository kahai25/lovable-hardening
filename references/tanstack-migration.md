# Migrating a Lovable Project to TanStack Start

Written after doing it on a live store with real orders. It worked, and almost nothing about it was two hours.

---

## FIRST: CHECK WHETHER THE PREMISE APPLIES TO YOU

The standard advice is "migrate for the new SEO." That advice assumes your project is a **client-rendered SPA serving an empty shell to crawlers**. That is the Lovable default, so it is true for most projects.

**It may not be true for yours.** Check before you spend anything:

```bash
curl -s https://yourdomain.com/ | wc -c
curl -s https://yourdomain.com/ | grep -o '<h1[^>]*>[^<]*' | head
curl -s https://yourdomain.com/some/deep/page | grep -c 'rel="canonical"'
```

If you get tens of kilobytes with real `<h1>` text and correct per-page canonicals, **you already prerender** and the SEO argument does not apply to you. On the source project, home was 54,590 bytes of complete HTML with a 7-entry hreflang set, and the "you need SSR for SEO" advice was simply wrong for that site.

**Google ranks the HTML it receives.** There is no ranking bonus for the rendering method.

## THE REAL REASONS TO MIGRATE

If you are not a shell SPA, these are what is left. Decide whether you actually need them:

1. **Real response headers.** A nonce CSP and `X-Frame-Options` are impossible on the static deployment. This is the strongest argument and it is security, not SEO.
2. **The stale-prerender bug class dies.** Any value that comes from a `useEffect` is `false`/empty at build time and freezes into the HTML. On the source project this served pre-drop copy for **four days after a product launch**.
3. **Dynamic pages without a hardcoded list.** Prerendering requires enumerating every product handle by hand.

**What you give up:** static files on a CDN. A static file cannot 5xx because an origin crashed. Measured TTFB went from a static file to **330 to 555 ms mean** at the edge, bimodal in a cache-hit-versus-cold-isolate pattern.

---

## THE TRAPS

### The migration converts routing and does nothing to metadata

Your head management will break. Expect it. Whether it breaks *loudly* depends on what you use:

- **`vite-react-ssg`'s `Head`** — deleted by the migration, so it breaks at compile time. Good.
- **`react-helmet-async`** — survives, runs client-side only, and **silently serves root defaults to every crawler** with a green build and a perfect-looking browser. This is the dangerous version.

### Head tags concatenate with no dedupe

Unlike react-helmet, TanStack does not deduplicate. **Anything your root emits and your pages also emit ships twice.**

**Rule: root owns zero `og:*` and zero `twitter:*`. Every leaf owns its own**, including the homepage. Root keeps only genuinely sitewide things: charset, viewport, CSP meta, preconnects, sitewide JSON-LD.

Move any page-specific JSON-LD (a `WebSite` schema, for instance) off the shell and onto the one page it belongs to.

### Module-level singletons become concurrency bugs

Anything mutable at module scope was safe under sequential static generation and is a **cross-request data leak** under per-request SSR.

The one that bit: a module-level i18n instance with `changeLanguage()` called **during render**. Two concurrent requests for different languages share and mutate the same object, so one visitor's language can leak into another's HTML **including their canonical URL**.

**Fix:** create a fresh instance per request, provide it through context, register any global adapter **only on the browser instance** so server instances never become the global default, and move mutation out of render into an effect.

**Prove it with genuine concurrency.** Sequential requests prove nothing. Fire N rounds of simultaneous requests for different states and assert no cross-contamination. 25 rounds x 6 languages = 150 responses is a reasonable bar.

### React 19 metadata hoisting is fragile by accident

If you keep a `PageMeta`-style component and rely on React 19 hoisting instead of route-level `head()`, it works — **but only because nothing in your app suspends.**

`PageMeta` typically sits below a `<Suspense>` boundary. One `React.lazy` or `useSuspenseQuery` moves that page's title, canonical and hreflang into a late chunk that a first-flush crawler never sees, **with no visible symptom in a browser.**

**Add a build guard** that fails on `React.lazy`, bare `lazy(`, `useSuspenseQuery`, `useSuspenseQueries` and `useSuspenseInfiniteQuery` under your routes and pages. It is a grep, so it is a tripwire not a proof, but it catches the direct cases.

### The dynamic-segment soft 404

`createFileRoute("/$lang")` with **no validation on the param** matches *any* first path segment. So `/literally-anything` matches, `/$lang/index.tsx` renders, and every junk URL returns **HTTP 200 with your homepage** and a canonical pointing at `/`. Unlimited indexable duplicates.

`notFoundComponent` is not broken in this scenario. **It is never reached**, because the URL genuinely matched a route.

**Fix:** validate the param in `beforeLoad` and `throw notFound()` when it is not a real value.

**And the trap inside the fix:** if you generate in-app links by prefixing the active language onto every path, real links like `/de/some-en-only-page` **depend on a strip-redirect**. A blanket 404 on the catch-all satisfies the spec and breaks navigation for every non-English visitor. Check whether the stripped path is a real route before deciding between redirect and 404.

### Your guards get wiped

The template overwrites `build`. Re-add the survivors. Any prerender-count guard is meaningless afterwards and needs replacing with an HTTP smoke test.

---

## THE ACCEPTANCE TEST

**This is the whole thing. Without it you cannot prove the migration was safe.**

Write a script that captures, per URL: HTTP status, final URL, `<html lang>`, `<title>`, description, canonical, **every** hreflang entry, robots, all `og:*` and `twitter:*`, JSON-LD block count, and byte length.

Two modes sharing one parser: filesystem walk, and **plain `fetch` with JavaScript never executed**. Store hrefs **byte for byte** — no normalising, no trimming.

**Capture the baseline over HTTP from the live site before you change anything.** After migration there is no `dist/` HTML tree to walk, and the pre-migration state cannot be recreated. This artifact is irreplaceable.

Then diff. **The bar: canonical, hreflang, title, description, htmlLang, robots and status all at ZERO differences across every URL.** Anything else is a regression.

On the source project the final diff was 480 differences that decomposed to exactly 5 fields across 96 URLs, every one intended.

**Watch for:** a jsdom-based diff normalises attribute case, so React 19 emitting `hrefLang` in camelCase does **not** surface in the diff even though the raw bytes differ. Check the bytes separately. It is benign to a conforming parser, but know about it.

---

## PRACTICAL

- **The framework migration is triggered from the Lovable UI, not over the API.** Do not let an agent hand-roll the framework swap; you end up with migrated code and the old deploy pipeline.
- **Stage it.** Connect GitHub and work on a branch, so you review a real diff and have a rollback that does not depend on chat history. A remix is a copy, not a branch, and has no supported merge-back.
- **Budget days.** Realistic sizing on a ~135-file project was **20 to 30 hours across 3 to 5 sessions**, with a 14-hour floor that was judged optimistic: head rewrite, route tree by hand, Tailwind v3 to v4 (silent visual regressions, no build failure), a strict-TypeScript wave, plus the i18n work.
- **Take screenshots before you start.** The Tailwind upgrade breaks things with a green build, and human eyes are the only detector.
- **The live site is protected throughout.** A red build produces no deployment and the old one keeps serving. Stalling mid-migration is not an outage.
