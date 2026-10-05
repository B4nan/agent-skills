---
name: triage-advisory
description: Triage a GitHub security advisory (GHSA) reported against a repo you maintain — verify the claim empirically, re-derive an honest severity, correct the advisory fields, and draft a reply to the reporter. Use whenever the user mentions a GHSA id or security advisory URL, asks whether a security report is valid or overstated, says a severity looks too high, asks whether something deserves a CVE, or wants help answering a vulnerability reporter. Also use for a bulk drop of several advisories from one reporter, where cross-report chaining and consistent severity matter.
argument-hint: <GHSA id or advisory URL>
---

# Triage a Security Advisory

Reports arrive pre-framed by whoever filed them. Some are real, most are overstated, a few are not vulnerabilities at all — and the overstated ones still tend to contain a genuine bug worth fixing. The job is to separate the defect from the framing.

Two failure modes to avoid, in both directions. Accepting the reporter's severity because a security label makes people cautious. And dismissing the whole thing because the framing is inflated, which loses the real bug hiding inside it. Verify first, then judge.

## Verdicts

Every advisory ends in exactly one of these. Steps 1–8 decide which; steps 9–12 carry it out. Name the verdict explicitly in the report back.

| Verdict | When | Advisory | CVE | Fix | Reply |
|---|---|---|---|---|---|
| **Vulnerability** | Untrusted party reaches the defect through a documented surface and the effect is real (step 8 gate passes) | Keep, correct every field (step 11), publish once the fix ships (for `low`, see the publish-or-release gotcha in step 11) | Consider (step 9) | Yes; decide private fork vs public first | Concede what holds, correct the vector axis by axis |
| **Already fixed** | Passes the step-8 gate on the affected version, but a released version no longer has it | Keep, set `patched_versions` to that release; publish vs close: step-11 gotcha | Same as above | None, unless a supported older line needs a backport | Point to the release |
| **Hardening bug** | Real defect, but it is reachable only internally, no untrusted party can set it, its effect is inert, or it is outside the tool's threat model | `state: closed` | No | Yes, as ordinary correctness in a normal release | State the gate that failed (surface, who sets it, inert effect, threat model) with its evidence; thank them for the bug |
| **Not a defect** | Nothing worth changing: the code behaves as intended or documented, or the effect is inert and the code needs no change | `state: closed` | No | No | Explain what actually happens and why |

The severity enum has no "not a vulnerability" value, so the hardening-bug and not-a-defect verdicts close with both `severity` and `cvss_vector_string` set to `null` rather than carry a `low`.

## Bug-class references

Specifics for common report classes live in `references/`. Read the matching file before step 3: it lists what to check while reading the function, how to tell inert from consequential (step 8), and what the narrowest fix looks like (step 10).

- `references/prototype-pollution.md`: `__proto__` / `constructor` keys, unsafe merges and copies
- `references/code-generation.md`: escaping and injection into generated source
- `references/denial-of-service.md`: unbounded recursion, ReDoS, resource exhaustion
- `references/path-traversal.md`: attacker-influenced file paths

## 1. Read the advisory, and what came before it

```bash
gh api /repos/{owner}/{repo}/security-advisories/{ghsa-id}
```

Note `severity`, both `cvss_severities.{cvss_v3,cvss_v4}.vector_string` and the legacy `cvss.vector_string` (deprecated but still returned; the reporter may have filed either version), `cwes`, `state`, `credits`, and `vulnerabilities[].{package, vulnerable_version_range, patched_versions}` alongside the description. Keep the reporter's original vector — you will be arguing with it specifically, axis by axis, and that is far more persuasive than asserting a different number.

Then list every advisory on the repo (`gh api --paginate /repos/{owner}/{repo}/security-advisories`), closed and published included. If you or a co-maintainer already ruled on the same bug class, reuse that verdict and its reasoning, or say explicitly why this one differs. Inconsistent rulings across advisories are the first thing a persistent reporter will quote back.

If the user dropped several advisories at once, read every one before triaging any. Step 7 depends on it.

## 2. Reproduce it yourself

Never accept the PoC as evidence. Reproduce the mechanism directly: extract the suspect function standalone and run it, or drive the real code path against a real fixture or dependency. Report the **literal observed output**. If you find yourself writing what you expect the code to produce, stop and run it.

This cuts both ways — it is equally how you confirm a real bug and how you catch one that no longer exists. Check the version too: a report against `<= 1.2.3` may describe something already fixed.

Read the report itself critically. Signs of a scanner-driven or LLM-written report: a CVSS vector that does not fit its own write-up, a PoC that calls unexported internals or builds inputs the public API never produces, no run against a released version, boilerplate impact paragraphs that could sit under any bug. Treat these as cues to verify harder, never as grounds to dismiss — templated reports still find real bugs, and "this reads like AI" is not an argument you can put in a reply.

## 3. Read the whole function, not the reported payload

Reporters find one payload. You have the source. Read the entire function and every one of its call sites.

The payload exercises one path; the function almost always has others the reporter never reached, and a sibling call site may bypass the function entirely — often strictly worse than what was reported. Fix all of them together.

Finding what they missed is also the strongest possible position from which to dispute their severity. It demonstrates you read the code rather than the write-up.

## 4. Check whether the test suite already encodes the bug

Grep the committed tests and snapshots for the broken output. A snapshot that has been asserting broken output for years is not unusual.

When you find this, it tells you two things: the bug is real, and it is mundane rather than exotic. A defect the test suite has been happily asserting is a correctness problem that happens to have a security framing — which is usually the honest way to describe it and to fix it.

## 5. Establish the minimum privilege empirically

This is normally the crux, and it is where instinct is least reliable. Do not reason about the required privilege — test it.

The tempting dismissal is "anyone who can do that already has enough access to cause worse damage directly, so there is no vector here." It is often false. Provision an account with *only* the single permission the attack needs, and nothing else: it may well plant the payload while being denied the broader access the argument assumed it implied. A reporter with a terminal refutes that dismissal in thirty seconds.

The same test can just as easily show that no ordinary, correctly provisioned account reaches the code at all — which is a far stronger argument against the reported severity. Empirical beats intuitive in both directions: run the test before you rely on the claim in public.

## 6. Trace every input path before accepting "attacker-controlled"

Enumerate exhaustively where the tainted value can come from: config keys, environment-variable mappings and their whitelists, public API callers, HTTP/admin surfaces, CI variables. Name the file and line for each route you rule out.

This is what turns a "High — arbitrary code execution" into not a vulnerability: when the only supplier of the input is the developer running a command on their own machine, no privilege boundary is crossed (gate 2 in step 8).

Watch for the public-API caveat: if a library function *could* be called with untrusted input by a downstream application, that is a misuse contract for that application, not an exposure in your project, unless gate 1's pass-through test (step 8) says otherwise. Say which; it is the reporter's most likely comeback.

## 7. Bulk drops: group by root cause, then look for chains

**Group first.** Reporters filing in bulk often submit the same mechanism several times — one unguarded pattern hit at different call sites, one missing check reached through different options. Cluster the advisories by mechanism before rating any of them. Each cluster is one investigation and one fix: grep for the pattern beyond the reported sites, since the reporter rarely found them all. Give every advisory in a cluster the same verdict and severity unless reachability or effect differs per site (one call site reachable from a public runtime option and another only from developer config, or one mutating a throwaway copy and another shared state); when it does, say which site and why in each reply. Cross-reference the siblings in each reply ("same root cause as <advisory>, fixed together").

**Then look for chains.** Check whether any two clusters compose. Reporters filing in bulk usually miss this because they write each one in isolation. The typical shape: one report controls *where* something is written, loaded, or executed, and another controls *what* — together they put attacker-chosen content somewhere the application trusts, which is worse than either alone.

If you find a chain, decide what it means for your ratings before you publish them. A chain that is materially worse than either component invites the obvious question of why both are rated low — have the answer ready (usually the trust-model argument in step 8, which the chain does not weaken).

## 8. Re-derive severity

### The gate

Answer these in order. The first "no" ends the vulnerability question: the verdict is **hardening bug**, or **not a defect** when there is nothing worth changing (see the verdict table).

1. **Documented surface**: can the reported value reach the defect through the documented surface? That means a public function called the way a pass-through application calls it, *or* a documented command (CLI, generator, CI step) run against a source the tool treats as data: records in a remote service, files users upload, database contents. A source the tool documents as code it will execute (a lockfile or build script an attacker changed in a PR, say) does not count, however the attacker got write access to it. A source the developer authored themselves (their own config, definitions, plugin list) is a gate-2 question.
2. **Untrusted party**: in a normal deployment, can anyone the application does not already trust with code execution (end users, other tenants, external accounts) set that value? Developers and the operators with host access who deploy it are trusted; an in-app role such as a tenant or customer admin is not, however the UI labels it — step 5 measures what that role can actually do. Answer from the evidence of steps 5 and 6, not by reasoning about it.
3. **Real effect**: is the outcome consequential (shared state corrupted, a real authorization or filter bypass the documented API does not already grant, code execution), rather than inert (a throwaway object, a catchable error on malformed input)?
4. **Threat model**: is the scenario inside what the tool ever claimed to defend?

All four "yes" → **vulnerability**, or **already fixed** when step 2 showed a released version no longer has it. One exception to "first no ends it", at gate 4 only: when gate 1 passed through a data source and an untrusted party's write becomes code execution on a developer machine or CI runner, the verdict is **vulnerability** even though the tool is dev-time only. Record the narrow preconditions in the worksheet, not as a dismissal:

- `AC:H`, or `AT:P` in 4.0, for the deployment precondition
- `UI:R` in 3.1; in 4.0, `UI:P` when the developer's action is routine (running the generator), `UI:A` only when they must do something specific
- `PR` as measured in step 5

Conceding it costs nothing; denying it costs the whole argument.

Whatever the verdict, answer the remaining gates too and fill in the worksheet at the end of this step. The first "no" fixes the verdict; the reply leads with the strongest failed gate (gate 4 when it also fails). On a closed verdict the worksheet is still the axis-by-axis rebuttal the reply needs.

### Gate 1: documented surface

A library's contract is its public surface, so a sink reachable *only* by the library's own internal construction, or by calling an unexported helper directly, is not a public vector. But **the type system is not the boundary, so "it wouldn't type-check" is not a dismissal.** Untrusted input routinely enters a typed API as `any`: `JSON.parse(body)` returns `any`, and `any` assigns to any declared parameter type with no cast anyone has to write. An application that forwards a parsed request body into a public options or data argument hands the attacker every runtime-reachable field, including ones the declared type would reject as a literal.

So reproduce the *pass-through* path (step 2): assign the parsed-JSON object to the public parameter type and call the public function, exactly as such an app would. If the payload reaches the sink that way, it is reachable, whatever the types say.

Frame the reply by why gate 1 failed. A "no" because the source is declared code *is* gate 4's threat-model argument: frame it that way, not as "not a documented surface". A developer-authored source belongs to gate 2: use its falsifier.

### Gate 2: who can set the value

Inputs that only the application's own developers write are not a vulnerability boundary, however dangerous the sink they reach: whoever sets that value already controls the surrounding code, so there is no privilege to escalate. Definition and configuration surfaces are the usual trap — route and handler tables, build and framework config, plugin registration, data-model and schema definitions, any structure the developer authors for the framework to compile or wire up. A payload arriving through one of those is a developer injecting code into their own program.

Have the answer to the reporter's likely escape hatch ready: *"a dynamic or multi-tenant app might build this structure from user input."* A fast falsifier is to check whether that same value, in that same hypothetical app, also flows into a second sink the framework already treats as trusted — an identifier, a file path, a compiled name. If it does, any app exposing the value to users is already broken through that more direct path, which confirms the value was never a user-facing input in the first place: the premise defeats itself rather than establishing a vector. Say that explicitly.

### Gate 3: effect

The verdict turns on **effect**, not on how exotic the shape looks. When the effect is inert, dismiss on impact: say it is reachable and why it is harmless. Do not reach for "internal method" or "needs a cast", which are weaker lines a reporter will knock down. When it is consequential, do not talk yourself out of it because the trigger looked like a typing mistake. The matching file in `references/` has the class-specific test for inert versus consequential. Fixing it does not lower the rating: hardening the code and rating the severity are the same conclusion, not opposites.

### Gate 4: threat model

**Is the scenario inside what the tool ever claimed to defend?** Tooling built to execute, generate, or trust its input (build scripts, code generators, plugin loaders) does not defend against hostile *code or configuration it is declared to execute*, however severe the payload — `npm install` against a hostile lockfile is the familiar version; declared data falls under gate 1 and the exception above. Prefer this argument: unlike a privilege-based one (step 5), a reporter cannot falsify it with a terminal, because it is about what the tool is for, not what an account can do.

### The worksheet

Fill this in for every advisory. It feeds the field table in step 11 and the reply in step 12, so write it once and reuse it. Use one row per base metric of the reporter's vector: AV, AC, PR, UI, S, C, I, A for CVSS 3.1; AV, AC, AT, PR, UI, VC, VI, VA, SC, SI, SA for CVSS 4.0.

| Axis | Reporter | Ours | Evidence |
|---|---|---|---|
| PR | L | H | step 5: a single-permission account was denied … |

Recurring inflation patterns:

- `PR:L` where the attack actually requires a role, permission, or ownership the attacker would not normally hold
- `AC:L` where it requires a developer to point a dev-time CLI at hostile input
- `UI:N` where a human has to run something for the payload to fire
- `S:C` (or non-zero subsequent-system impact in 4.0) where no privilege domain is actually crossed
- `AT:N` (4.0) where the attack depends on a deployment precondition the reporter assumes
- `C:H` beside `I:N/A:N` (`VC`/`VI`/`VA` in 4.0) when the same defect also destroys or corrupts data. Impact zeroed on the axes that are genuinely worse is a strong sign the vector was fitted to the write-up rather than measured — say so.
- severity anchored to a narrative ("the third library in a row with this bug class") rather than to the artifact

## 9. Decide on a CVE

Default to no for low-severity or dev-time-only issues, and explain why to the user rather than just asserting it: a CVE propagates to NVD and every corporate scanner, generating alerts users cannot act on beyond upgrading; it is very hard to walk back once issued; and it spends credibility needed for something that genuinely warrants one. Publishing the GHSA already drives Dependabot, so the CVE mostly buys reach and permanence.

## 10. Fix it on its own merits

Frame the fix as ordinary correctness, because that is usually what it is. Keep every trace of the security context out of branch names, commit messages, test names, code comments, and the PR description — no GHSA id, no CVE, no "vulnerability", "RCE" or "injection", no reference to the reporter. Describe the actual defect instead: "unescaped value breaks the generated output".

Check the branch you are *already on* before committing. A session opened for advisory work often starts on a branch whose name gives the whole thing away, and pushing it defeats every other precaution here.

**Do not adopt the reporter's recommended patch unverified.** They optimise for closing their payload, which is routinely broader than the defect: a blanket guard where only one key actually misbehaves silently discards data the code handled correctly before. Find the narrowest change that closes the mechanism, and let the existing suite arbitrate — an over-broad fix usually breaks a test that documents the behaviour you were about to destroy, and that test is the argument for narrowing.

Decide private-fork versus public **before the first push**, since pull requests cannot be deleted.

If a project skill for fixing bugs exists (`/fix`, or `/polish` for finishing a change), use it rather than reimplementing test-first-then-PR here.

## 11. Update every advisory field

**Put the proposed changes in front of the user before you PATCH anything.** A short before/after table of every field you intend to touch, plus the step-8 worksheet. This is their advisory and the reporter sees the result, so they need to review the content, not authorise a black box. Never ask "shall I update the advisory?" without that table already written — a confirmation request with nothing to confirm is worse than just proceeding, because it reads as caution while telling them nothing.

Partial updates are the standard failure. Walk the whole list:

- `summary` — **the one everyone forgets.** Lowering severity while the title still reads "Authorization Bypass" or "Arbitrary Code Execution" leaves the two contradicting each other, and the title is what appears in listings, Dependabot alerts and downstream mirrors. Rewrite it to describe the defect.
- `severity` / `cvss_vector_string` — the API accepts one or the other, not both. On a vulnerability or already-fixed verdict, send the worksheet's vector and let it compute the severity; when a narrow shape is hard to express in 3.1, use a CVSS 4.0 vector (`AT:P`, `UI:P` / `UI:A`), which GitHub accepts. Never publish a hand-picked `severity` in place of the vector you computed. On a hardening-bug or not-a-defect verdict send both as `null`. A field left out of the payload keeps its old value, so an old vector left in place keeps **driving the displayed severity**.
- `cwe_ids` — reclassify when the framing changed: a report filed as an authorization or injection weakness is often, on inspection, a plain logic or input-handling error, and the CWE should say so (e.g. CWE-20)
- `vulnerable_version_range` and `patched_versions` — do not promise a patched version for a fix that has not landed; re-check the merge state rather than assuming
- the description — rewrite it to **stand alone**. It is published to people who never saw the report, so it states the defect, the real preconditions, and what the attacker does and does not control. Corrections to the reporter's claims go in the reply, not here. A description carrying "the impact is smaller than reported" or "what the report missed" reads as half of a conversation the reader cannot see, and it is the most common thing to get wrong at this step. Grep your draft for "report" before sending it.

```bash
gh api --method PATCH /repos/{owner}/{repo}/security-advisories/{ghsa-id} --input payload.json
```

Verify by re-fetching. Do not trust the PATCH response you did not read.

**Gotchas worth knowing before you start:**

- **There is no API for advisory comments** (as of October 2026 — re-check if today is well past that date). No REST endpoint, and GraphQL has no `RepositoryAdvisory` type at all — only the read-only global `SecurityAdvisory`, which has no comments field. The advisory object exposes only a `comments` count: enough to see that a reply exists, not to read it. Replies have to be posted by hand. **Do not drive a browser to post or edit them** — it is slow and error-prone (the comment editor shares state with the description field, so edits can land in the published description), and the user can paste the text themselves in under three minutes. Hand the reply back in the step-12 delivery format. Never imply you posted it, and never claim to have read a reply you cannot fetch.
- **`triage` → `draft` is the maintainer accepting the submission**, not a side effect of your edit; `submission.accepted` flips at the same time. Before reporting any field change as something you caused, check whether your payload even contained that field.
- Publishing even a `low` advisory fires Dependabot for every dependent. When the fix ships in a normal patch release anyway, raise "publish or just release" as a deliberate choice rather than defaulting either way. The same choice applies to an already-fixed verdict: publishing alerts everyone still on the affected range, which is the point even when that range is out of support.

## 12. Draft the reply

Blunt and factual. Publishing exposes the advisory *content* — summary, description, severity, credits. The comment thread with the reporter stays private, so you are writing for them, not for a future audience. Do not invoke publication as a reason to soften, restructure, or hedge the reply.

Structure that works: concede what reproduced, correct the vector axis by axis from the step-8 worksheet with the evidence from steps 5 and 6, name what they missed from step 3, then state what you are fixing and that it lands on its own merits. For a step-7 cluster, name the sibling advisories that share the root cause.

Two things to get right. Build the argument only on premises that are expensive to falsify — the cheap-to-falsify kind hands the reporter the thread, as in step 5. And keep the criticism factual rather than sharpened; the facts do more damage than adjectives.

If the user's project has a text-humanising skill, run the draft through it before handing it over.

**Delivery format — always.** The user posts and edits these by hand, so make it copy-paste ready. For each advisory, give its **direct URL** (`https://github.com/{owner}/{repo}/security/advisories/{ghsa-id}`) as a markdown link, then the comment body in its own ```` ```text ```` fenced block — advisory comments are full of inline `` `code` `` (identifiers, `__proto__`, CVSS strings), and a fenced block keeps them intact for a clean copy. One link + one block per advisory. When you are editing a comment the user already posted rather than writing a fresh one, say "paste over your existing comment on" and give the same link + block. Do not bury the text in prose or a table; the block is the deliverable. On a bulk drop, list every advisory this way, and call out which ones need **no** change (comment already accurate) so the user doesn't hunt for a diff that isn't there.

## Report back

Close with:

- **Verdict** — one of the four, per advisory (and per cluster on a bulk drop)
- **What was verified** — the literal reproduction output, and which claims held versus failed
- **Honest severity** — the step-8 worksheet, the CVE recommendation, and which earlier advisory's ruling it follows or departs from (step 1)
- **Anything the report missed** — extra defects, root-cause clusters, and any cross-report chain
- **Advisory fields changed** — before and after, confirmed by re-fetching
- **The drafted reply(ies)** — in the step-12 delivery format, with any no-change-needed ones flagged
- **What needs the user's own hands** — pasting the comment, publishing, closing, and any fix still unwritten

Flag disagreements plainly. If reasoning that is actually going into the reply or the advisory is falsifiable, say so before it ships — that is far more useful than agreeing. Scope this to text headed for the reporter: an opinion the user voices to you in conversation is not draft copy, and arguing with it as though it were is noise.
