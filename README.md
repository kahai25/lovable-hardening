# lovable-hardening

Platform-specific security, deployment and verification practice for apps built on Lovable with Lovable Cloud.

Written from running a live e-commerce store on Lovable through a launch, a security audit, and a framework migration. Everything here produced a real incident or a near miss.

## Who this is for

You are building something real on Lovable. It has users, maybe money. You have hit the point where "it works in preview" stopped being good enough.

## The five that cost real time

1. **Source deletion is not undeployment.** An unauthenticated endpoint holding an admin token was deleted from the repo and from config. **It kept serving.** Found only by probing the live URL.

2. **The preview does not run your build.** No production build, no prerender, no guard scripts. A broken guard means the preview looks perfect while every publish silently fails and the live site sits frozen on an old deployment.

3. **Lovable does not serve custom response headers.** `public/_headers` enforces nothing — and gets served as a **public 200 download**, publishing the security policy you thought you had to anyone who guesses the path.

4. **Treat Lovable as auto-deploying.** Frontend edits have shipped despite an explicit instruction not to publish.

5. **Do not verify a deploy with `x-deployment-id`.** It changed from a UUID to a hash during a migration, then disappeared entirely. A loop keyed on "the ID changed" reports false success. Use a content signal.

## Install

```
~/.claude/skills/lovable-hardening/
```

Or zip as `lovable-hardening.skill`.

Then ask: *"deploy this and verify it"*, *"audit my Lovable project"*, *"should I migrate to TanStack Start"*, *"did the agent actually do what it said"*.

## Contents

| File | What it is |
|---|---|
| `SKILL.md` | Platform traps, build guards, the pre-ship gate, Lovable Cloud notes, deploy loop |
| `references/tanstack-migration.md` | Whether to migrate, and every trap if you do |
| `references/verification-protocol.md` | Prompt patterns that get real evidence out of the agent |

## On working with the agent

Its **conclusions are usually right. Its evidence is usually incomplete.** Design around that, not around distrust.

Observed on one project: raw output collapsed to "(rounds 2-24 identical)" four separate times, a finding deleted from a report after the agent privately decided it was benign, a job reporting success while silently skipping one of three deletions.

Also observed on the same project: it volunteered two of its own failed test methodologies, led a report with a defect rather than burying it, caught a bug it had introduced in its own probe run and reported it, and pushed back on a specification that would have satisfied the letter of the request while breaking navigation for every non-English visitor.

**Push on evidence, not on trust.**

## Before you migrate to TanStack Start

Check whether the premise even applies to you:

```bash
curl -s https://yourdomain.com/ | wc -c
curl -s https://yourdomain.com/ | grep -o '<h1[^>]*>[^<]*' | head
```

If that returns tens of kilobytes with real heading text, **you already prerender**, and the "migrate for SEO" advice does not apply to your project. The real reasons to migrate are response headers, the stale-prerender bug class, and dynamic pages. Budget days, not hours.

## License

MIT.
