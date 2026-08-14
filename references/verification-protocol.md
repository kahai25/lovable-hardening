# Verification Protocol

Prompt patterns that get real evidence out of an AI build agent, and the checks that catch it when they do not.

The agent's **conclusions are usually right. Its evidence is usually incomplete.** Design around that rather than around distrust.

---

## THE FOUR PHRASES THAT WORK

**1. Name what you already know.**

> "I have already read `src/i18n.ts` and `PageMeta.tsx` myself. I am not asking you to summarise. I want raw output."

Removes the option of a confident recap and forces new information.

**2. Pre-empt the collapse.**

> "Paste the complete output. Do not write 'identical' in place of results you did not read. If it is genuinely 4,000 lines, give me the aggregate, every line representing a difference or failure, and the first and last rounds."

Without this you get `(rounds 2-24 identical)`. With it you usually get the real thing, and when you do not, at least you know to ask again.

**3. Forbid dropping findings, explicitly.**

> "Report everything you find, with your assessment attached. You may write 'I found X and I believe it is benign, here is why.' You may not silently omit it because you concluded it was fine."

This is worth including in every audit prompt. An agent once discovered a real detail, reasoned about it privately, decided it was harmless, and **deleted it from its report**. It was harmless. That is not the point.

**4. Demand the artifact, not the claim.**

> "Show me the full diff, not a summary. Paste the rendered `<head>` in your reply text, not in a tool call."

"Pasted above in the tool output" is an assertion. If it is not in the reply, treat it as unverified.

---

## STRUCTURAL RULES

**One job per message.** Multi-part jobs are where items silently vanish. Two documented cases: one deletion of three skipped, one config entry dropped mid-edit. Both found only by reading the diff.

**Explicit do-not-touch lists, naming files.** And add: *"if my instructions conflict, stop and ask rather than reconciling."* It will silently reconcile otherwise.

**Say what must not happen.** "Do not deploy" at the top and bottom of the message. Then verify it did not.

**Ask it to state limitations.** *"If any part of this cannot be done cleanly, stop and tell me rather than working around it."* This produces genuinely useful disclosures: which artifact numbers came from, which tool refused to start, which proof is weaker than it looks.

---

## WHAT TO CHECK AFTER, EVERY TIME

| Check | How |
|---|---|
| Did it change only what you asked? | `get_diff` on the commit. Read it |
| Did it change files outside scope? | Look for `package.json`, config, lockfiles riding along |
| Did it deploy? | Check deployment history, not its word |
| Did it skip anything? | Compare the diff against your numbered list |
| Is the claim reproducible? | Re-run the decisive check yourself where you can |

**Watch for unrequested changes riding along.** A backend-only job once carried a build-config version bump in the same commit. Probably the platform updating its own package rather than the agent choosing to — but it turned an edge-function-only change into something that rebuilds the frontend, which was not the scoped risk.

---

## VERIFY COMPLETIONS, NOT FAILURES

The documented failure direction is **success reported for work not fully done**:

- Two messages reported success and **never committed**.
- One job reported success while **silently skipping one of three deletions**.
- One dropped a config entry mid-edit without mentioning it.

All caught only by reading the diff.

**A report of "I could not do that" is the safer kind** — it is self-limiting and it invites you to look. A report of "done" is the one that needs evidence.

**Before re-sending anything, check the current state of the file or the live page.** Re-running work that already shipped causes its own damage.

---

## GIVE CREDIT WHERE IT IS DUE

This is not adversarial, and treating it that way makes the output worse.

The same agent that collapsed its output four times also:

- Volunteered that two of its own test methodologies had been wrong, and explained exactly how
- Led a report with a defect it had found rather than burying it under good news
- Refused to claim a deployment-ID match it could not actually make
- Caught a bug **it had introduced**, in its own probe run, and reported it rather than quietly patching
- Pushed back on a specification that would have satisfied the letter of the request and broken navigation for every non-English visitor

**Push on evidence, not on trust.** When it tells you a spec is wrong, listen — that is the highest-value thing it does.
