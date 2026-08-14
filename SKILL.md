---
name: lovable-hardening
description: Platform-specific security, deployment and verification practice for apps built on Lovable with Lovable Cloud (Supabase). Use when working on a Lovable project and the user asks to deploy, publish, verify a change went live, audit security, add or remove an edge function, wire build guards, investigate why preview and the live site differ, migrate to TanStack Start, or check whether the Lovable agent actually did what it reported. Also use before any launch on Lovable and after any edge-function change.
---

# Lovable Hardening

Everything a Lovable project needs that is not obvious from the platform, learned by running a live store on it.

Assumes Lovable + Lovable Cloud (Supabase underneath) + a custom domain. Much of it applies to any managed AI build platform.

---

## 🔴 THE FIVE THAT COST REAL TIME

Read these before anything else. Each one produced a live incident or a near miss.

### 1. Source deletion is not undeployment

Removing an edge function from the repo and from `config.toml` **does not stop it serving.**

> A public, unauthenticated endpoint holding an admin API token, echoing raw upstream errors to any caller, was deleted from source. It kept answering. It was found only because someone probed the live URL after the commit.

**Rule: after removing any edge function, `curl` it and require a 404.** If it answers, undeploy it explicitly.

```bash
curl -s -o /dev/null -w "%{http_code}\n" "https://<project>.supabase.co/functions/v1/<name>"
```

### 2. The preview does not run your build

The dev preview does not run the production build, the prerender step, or any guard script in your `build` chain.

**So a broken guard makes the preview look perfect while every publish silently fails, and the live site keeps serving the last good deployment.**

> A route-count guard drifted out of sync after a routing change. Publishes appeared to schedule and nothing shipped. The custom domain sat frozen on an old build for an hour while the preview showed the new one.

**Rule: before trusting a publish, have the agent run the full production build and report the exit code.** Then verify the live domain changed.

### 3. Lovable does not serve custom response headers

`public/_headers`, `netlify.toml`, `vercel.json` and friends are conventions Lovable **never implemented**. A `_headers` file in your repo enforces nothing.

**Worse: it gets served as a public file.** A `_headers` file was live at `/_headers` as a 200 download, publishing an intended-but-unenforced CSP and Permissions-Policy to anyone who guessed the path.

**Consequences:**
- No real CSP, no `X-Frame-Options`, no `frame-ancestors` on a static Lovable deployment.
- `frame-ancestors` in a `<meta>` tag is **ignored by the spec**. Only a response header counts.
- A JS frame-buster is defence in depth only; a sandboxed iframe defeats it.

**Options:** put your own CDN in front (Lovable documents a connect-time proxy option), or migrate to a server-rendered setup where you control the response. **Weigh the DNS risk honestly** — if your domain carries email records, moving nameservers to get headers can silently break order email, which is worse than the thing you are fixing.

### 4. Treat Lovable as auto-deploying

Frontend edits have shipped despite an explicit instruction not to publish.

**Rule: never rely on "do not publish" as a safety mechanism.** If a change must not go live, do not make it yet. Use a branch via GitHub sync.

### 5. Do not verify a deploy with `x-deployment-id`

That header is not a stable contract. On one project it changed from a UUID to a 64-character hash during a framework migration, and then **disappeared entirely** on the next deployment.

A polling loop keyed on "the ID changed" is satisfied by a *format* change and will report a false success.

**Rule: verify with a content signal.** Poll until a known-new asset appears, or until a URL's behaviour changes in a way only the new build produces.

---

## VERIFYING THE AGENT

The Lovable agent's **conclusions are usually right. Its evidence is usually incomplete.** Both halves matter.

Observed repeatedly on one project:

- Raw output it was explicitly asked to paste in full, collapsed to `(rounds 2-24 identical)` — 138 of 150 results asserted rather than shown. Four separate occasions.
- A finding **deleted from a report entirely** after the agent privately decided it was benign.
- A job reporting success while **silently skipping one of three deletions**.
- A config entry **dropped mid-edit** without mention.
- A **reported "stopped"** for work that had actually shipped.

**Practice that works:**

1. **Ask for raw output, and reject "identical", "as above", or a summary table where you asked for results.**
2. **Read the diff.** Never accept a summary of a diff. `get_diff` on the commit.
3. **Give explicit do-not-touch lists naming files**, and require it to stop rather than reconcile conflicting instructions. It will silently reconcile otherwise.
4. **Tell it: report every finding with your assessment attached. You may say "I think this is benign." You may not omit it.**
5. **Verify completions, not failures.** A report of "I could not" is safer than a report of "done".
6. **Before re-sending work, check the current state of the file or the live page.** Re-sending something already shipped causes its own damage.

Credit where due: the same agent volunteered two failed test methodologies, led a report with a defect rather than burying it, and refused to claim a deployment match it could not make. **Push on evidence, not on trust.**

---

## BUILD GUARDS THAT DO NOT ROT

Guards are the highest-leverage thing you can add to a Lovable project, and the easiest to get wrong.

**Split them by network dependency.**

| Kind | Where | Why |
|---|---|---|
| **Offline** — key parity, route coverage, asset checks, sitemap generation, static greps | In `prebuild`, failing the build | Deterministic, sub-second, cannot be taken out by a third party. A failure means the artifact is wrong |
| **Network** — live price drift, live API shape | **Out of the build**, run manually or on a schedule | A third-party blip must never be able to freeze a hotfix |

> The hour-long deploy freeze came from a guard in the deploy path. **The network dependency was the problem, not the concept of a guard.**

**Two more rules:**

- **A guard that can warn-pass on zero inputs must fail on zero inputs.** A price guard sat at "0 of 26 verified" for weeks because every product was draft and the API returned not-found for all of them. It printed green the whole time.
- **Any hardcoded list inside a guard will drift.** Derive it from the filesystem instead — `import.meta.glob` over your route files, a directory read, anything that cannot disagree with reality. If a list must be duplicated, add a bidirectional assertion so a missing entry fails, not just an extra one.

---

## LOVABLE CLOUD (SUPABASE) NOTES

- **New tables inherit default public-schema grants.** `anon` gets SELECT, `authenticated` gets INSERT/UPDATE/DELETE, without you granting anything. RLS blocks it, but the second layer is gone. **Re-run a grants audit after every migration.**
- **`TRUNCATE` ignores RLS.** Check raw grants, not just policies.
- **Edge functions default to a platform `verify_jwt` setting.** Pin it explicitly per function; do not inherit.
- **CORS allowlists will lock out your own preview.** If you tighten CORS to your custom domain, `id-preview--*.lovable.app` stops working. Add it deliberately or accept that preview testing breaks.
- **Session limiting, idle timeout and concurrent-session caps are Pro-gated**, and may be unreachable at all if the Supabase project is Lovable-owned rather than yours. **Check who owns the project before planning around those settings.**
- **httpOnly cookie storage is not achievable** for a client-side SPA with no server. Tokens live in `localStorage`. That is a decision to make consciously, because it changes the severity of any XSS finding.
- **A database query tool against the production database is the single most useful verification channel available.** Use it to confirm state rather than trusting an agent's report. It costs no credits.

---

## MIGRATING TO TANSTACK START

If you are considering it, read `references/tanstack-migration.md`. The short version:

- **Check first whether you are actually a shell SPA.** The "migrate for SEO" argument assumes crawlers get an empty div. If your project already prerenders, that argument does not apply to you, and you can prove it in ten seconds with view-source.
- **The migration converts routing. It does nothing to metadata.**
- **Head tags concatenate with no dedupe**, unlike react-helmet. Anything your root emits *and* your pages emit ships twice.
- **Capture an SEO baseline over HTTP from the live site before you start.** After migration it cannot be recreated.
- Budget days, not hours.

---

## THE PRE-SHIP GATE

Before any merge or publish, the agent answers four questions **in writing**. An "I don't know" on any of them blocks the ship until it is a real answer. This is how you keep AI velocity without shipping AI-speed bugs.

1. **Security** — is every mutation behind server-side auth? New tables have RLS *and* correct grants? No secrets in `VITE_*` or the bundle? Every user-supplied ID checked for ownership?
2. **Efficiency** — any N+1 query, unbounded `SELECT *`, or client-side loop that should be a server-side aggregate?
3. **Regressions** — what existing routes and functions does this touch, what covers them today, and did any shared component change signature?
4. **Tests** — what proves this works, and what would prove it broken? If there is no testable surface, write down how it was verified manually.

The gate matters most when the feature went *fast*. "I built it in 20 minutes" is the trigger, not the reassurance.

---

## DEPLOY AND VERIFY

The loop that works:

1. Have the agent run the **full production build** and report the exit code, not just "done".
2. **Read the diff.**
3. Publish.
4. **Poll for a content signal**, not a deployment ID. Edge propagation has ranged from seconds to twenty minutes on the same project.
5. **Verify on the live custom domain**, never on preview. Cache-bust every check.
6. If something looks wrong, **re-check before concluding** — a single failed verification during propagation is not evidence of a failed deploy.

---

## REFERENCES

| File | Use |
|---|---|
| `references/tanstack-migration.md` | Whether to migrate, and the traps if you do |
| `references/verification-protocol.md` | Prompt patterns that get real evidence out of the agent |
