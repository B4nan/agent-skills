---
name: triage-advisory
description: Triage a GitHub security advisory (GHSA) reported against a repo you maintain — verify the claim empirically, re-derive an honest severity, correct the advisory fields, and draft a reply to the reporter. Use whenever the user mentions a GHSA id or security advisory URL, asks whether a security report is valid or overstated, says a severity looks too high, asks whether something deserves a CVE, or wants help answering a vulnerability reporter. Also use for a bulk drop of several advisories from one reporter, where cross-report chaining and consistent severity matter.
argument-hint: <GHSA id or advisory URL>
---

# Triage a Security Advisory

Reports arrive pre-framed by whoever filed them. Some are real, most are overstated, a few are not vulnerabilities at all — and the overstated ones still tend to contain a genuine bug worth fixing. The job is to separate the defect from the framing.

Two failure modes to avoid, in both directions. Accepting the reporter's severity because a security label makes people cautious. And dismissing the whole thing because the framing is inflated, which loses the real bug hiding inside it. Verify first, then judge.

## 1. Read the advisory from the API

```bash
gh api /repos/{owner}/{repo}/security-advisories/{ghsa-id}
```

Note `severity`, `cvss.vector_string`, `cwes`, `state`, `credits`, and `vulnerabilities[].{package, vulnerable_version_range, patched_versions}` alongside the description. Keep the reporter's original vector — you will be arguing with it specifically, axis by axis, and that is far more persuasive than asserting a different number.

If the user dropped several advisories at once, list them all (`gh api /repos/{owner}/{repo}/security-advisories`) and read every one before triaging any. Step 6 depends on it.

## 2. Reproduce it yourself

Never accept the PoC as evidence. Reproduce the mechanism directly: extract the suspect function standalone and run it, or drive the real code path against a real fixture or dependency. Report the **literal observed output**. If you find yourself writing what you expect the code to produce, stop and run it.

This cuts both ways — it is equally how you confirm a real bug and how you catch one that no longer exists. Check the version too: a report against `<= 1.2.3` may describe something already fixed.

## 3. Read the whole function, not the reported payload

Reporters find one payload. You have the source. Read the entire function and every one of its call sites.

This reliably finds more than the report. In one case a three-line escaping helper had two further holes the reporter never reached — one branch left an interpolation sequence unescaped, and another produced a syntax error for *any* input containing a particular character — plus a nearby call site that skipped the helper entirely, which was strictly worse than the reported issue. All three shipped in the same fix.

Finding what they missed is also the strongest possible position from which to dispute their severity. It demonstrates you read the code rather than the write-up.

## 4. Check whether the test suite already encodes the bug

Grep the committed tests and snapshots for the broken output. Twice in one session a snapshot had been asserting uncompilable generated code for years.

When you find this, it tells you two things: the bug is real, and it is mundane rather than exotic. A defect the test suite has been happily asserting is a correctness problem that happens to have a security framing — which is usually the honest way to describe it and to fix it.

## 5. Establish the minimum privilege empirically

This is normally the crux, and it is where instinct is least reliable. Do not reason about the required privilege — test it.

A worked example of why. The maintainer's first instinct was "anyone who can do that already has enough access to cause worse damage directly, so there is no vector here." Reasonable, and false. Provisioning an account with *only* the single permission the attack needs — and nothing else — showed it could plant the payload while being denied the broader access the argument assumed it implied. That dismissal would have been refuted in about thirty seconds by a reporter who tried it.

The same test also showed that no ordinary, correctly-provisioned account could reach the code at all, which is what actually sank the reported severity. Empirical beats intuitive in both directions, which is the point: run the test before you rely on the claim in public.

## 6. Trace every input path before accepting "attacker-controlled"

Enumerate exhaustively where the tainted value can come from: config keys, environment-variable mappings and their whitelists, public API callers, HTTP/admin surfaces, CI variables. Name the file and line for each route you rule out.

This is what turns a "High — arbitrary code execution" into not a vulnerability. In one case the only supplier of the input was the developer typing it on their own command line — someone who could already execute arbitrary code without going through the reported path. No privilege boundary was crossed, so there was nothing to escalate.

Watch for the public-API caveat: if a library function *could* be called with untrusted input by a downstream application, that is a misuse contract for that application, not an exposure in your project. Say so explicitly rather than ignoring it; it is the reporter's most likely comeback.

## 7. Look for chains across multiple reports

When several advisories land together, check whether any two compose. Reporters filing in bulk usually miss this because they write each one in isolation.

In one bulk drop, a path-traversal report controlled the *destination path* of a generated file and an escaping report controlled its *contents* — both reachable from a single attacker-supplied value, together writing attacker-chosen executable code over a file the application already loaded, with no user interaction. That combination was worse than either report alone.

If you find a chain, decide what it means for your ratings before you publish them. A chain that is materially worse than either component invites the obvious question of why both are rated low — have the answer ready (usually the trust-model argument in step 8, which the chain does not weaken).

## 8. Re-derive severity, and ask the trust-model question

Go axis by axis against their vector. Recurring inflation patterns:

- `PR:L` where the attack actually requires DDL rights or object ownership
- `AC:L` where it requires a developer to point a dev-time CLI at hostile input
- `UI:N` where a human has to run something for the payload to fire
- `S:C` where no privilege domain is actually crossed
- `C:H` beside `I:N/A:N` when the same defect also destroys or corrupts data. Impact zeroed on the axes that are genuinely worse is a strong sign the vector was fitted to the write-up rather than measured — say so.
- severity anchored to a narrative ("the third library in a row with this bug class") rather than to the artifact

**The gate question, ask it first: can untrusted data reach the defect through a public entry point at runtime** — the documented, exported functions, called the way a pass-through application calls them? A library's contract is its public surface, so a sink reachable *only* by the library's own internal construction, or by calling an unexported helper directly, is not a public vector. But **the type system is not the boundary, so "it wouldn't type-check" is not a dismissal.** Untrusted input routinely enters a typed API as `any`: `JSON.parse(body)` returns `any`, and `any` assigns to any declared parameter type with no cast anyone has to write. An application that forwards a parsed-JSON `where` / options / data object into a public call hands the attacker every runtime-reachable field, including ones the declared type would reject as a literal. So reproduce the *pass-through* path (step 2): assign the parsed-JSON object to the public parameter type and call the public function, exactly as such an app would. If the payload reaches the sink that way, it is reachable, whatever the types say. Forwarding `JSON.parse(body)` straight into a public call as its query, options, or data argument is the realistic shape, not a hand-written cast past the types.

**Reachability includes *who* can reach it, not only whether the value can flow to the sink.** A vulnerability requires the tainted value to be controllable by a party the application does not already trust with code execution — its end users, or an external actor across a network, tenant, or account boundary. Inputs that only the application's own developers write are not a vulnerability boundary, however dangerous the sink they reach: the party who sets that value already controls the surrounding code, so there is no privilege to escalate. Definition and configuration surfaces are the usual trap — schema and model definitions, route and handler tables, build and framework config, plugin registration, any structure the developer authors for the framework to compile or wire up. A payload arriving through one of those is a developer injecting code into their own program, which is not an exposure. Ask it concretely: in a normal deployment, can anyone other than the developer set this value? When the honest answer is no, it is a hardening bug at most — fix it as ordinary correctness (step 10) and close the report.

Have the answer to the reporter's likely escape hatch ready: *"a dynamic or multi-tenant app might build this structure from user input."* A fast falsifier is to check whether that same value, in that same hypothetical app, also flows into a second sink the framework already treats as trusted — an identifier, a file path, a generated column, a compiled name. If it does, any app exposing the value to users is already broken through that more direct path, which confirms the value was never a user-facing input in the first place: the premise defeats itself rather than establishing a vector. Say that explicitly.

Once reachability holds — and holds for an untrusted party — the verdict turns on **effect**, not on how exotic the shape looks:

- *Reachable and inert* — a throwaway copy whose `[[Prototype]]` is reassigned and then discarded, or a catchable error on malformed input. Dismiss on impact. Say it is reachable and say why it is harmless; do not reach for "internal method" or "needs a cast", which are weaker lines a reporter will knock down.
- *Reachable and consequential* — a write to shared global state (`Object.prototype` itself, a module singleton), a real authorization or filter bypass that is not already granted by the documented API. This is a vulnerability. Rate it honestly; do not talk yourself out of it because the trigger looked like a typing mistake.

The tell that separates the two in the prototype-pollution family: reassigning *a throwaway object's* prototype (`copy.__proto__ = x`) is inert; writing *through* the prototype onto the shared object (`obj.__proto__.foo = x`, i.e. `Object.prototype.foo = x`) corrupts every object in the process and is real. Check which one the code actually does — run it and probe a fresh `{}` afterwards.

Only dismiss on reachability when the value genuinely cannot arrive through any public parameter an application populates from untrusted data. When it can, and the effect is real, hardening the code and rating the severity are the same conclusion, not opposites.

Then the question that decides most of these: **is the scenario inside what the tool ever claimed to defend?** Plenty of tooling is designed to execute, generate, or trust its input — build scripts, code generators, plugin loaders, anything that runs on a developer machine or in CI. Feeding such a tool input an untrusted party controls is outside its threat model however severe the payload gets. `npm install` against a hostile lockfile is the familiar version of this.

Prefer this argument. Privilege-based arguments can be falsified by a reporter with a terminal, as in step 5. The trust-model argument cannot, because it is about what the tool is for.

Be honest when a boundary genuinely is crossed. Something that turns write access to one system into code execution on a developer machine or CI runner is a real escalation even when the preconditions are narrow, and conceding it costs nothing while denying it costs the whole argument.

## 9. Decide on a CVE

Default to no for low-severity or dev-time-only issues, and explain why to the user rather than just asserting it: a CVE propagates to NVD and every corporate scanner, generating alerts users cannot act on beyond upgrading; it is very hard to walk back once issued; and it spends credibility needed for something that genuinely warrants one. Publishing the GHSA already drives Dependabot, so the CVE mostly buys reach and permanence.

## 10. Fix it on its own merits

Frame the fix as ordinary correctness, because that is usually what it is. Keep every trace of the security context out of branch names, commit messages, test names, code comments, and the PR description — no GHSA id, no CVE, no "vulnerability", "RCE" or "injection", no reference to the reporter. Describe the actual defect instead: "unsanitised name produces an uncompilable class identifier".

Check the branch you are *already on* before committing. A session opened for advisory work often starts on a branch whose name gives the whole thing away, and pushing it defeats every other precaution here.

**Do not adopt the reporter's recommended patch unverified.** They optimise for closing their payload, which is routinely broader than the defect: a blanket guard where only one key actually misbehaves silently discards data the code handled correctly before. Find the narrowest change that closes the mechanism, and let the existing suite arbitrate — an over-broad fix usually breaks a test that documents the behaviour you were about to destroy, and that test is the argument for narrowing.

Decide private-fork versus public **before the first push**, since pull requests cannot be deleted.

If a project skill for fixing bugs exists (`/fix`, or `/polish` for finishing a change), use it rather than reimplementing test-first-then-PR here.

## 11. Update every advisory field

**Put the proposed changes in front of the user before you PATCH anything.** A short before/after table of every field you intend to touch, plus the severity reasoning axis by axis. This is their advisory and the reporter sees the result, so they need to review the content, not authorise a black box. Never ask "shall I update the advisory?" without that table already written — a confirmation request with nothing to confirm is worse than just proceeding, because it reads as caution while telling them nothing.

Partial updates are the standard failure. Walk the whole list:

- `summary` — **the one everyone forgets.** Lowering severity while the title still reads "Authorization Bypass" or "Arbitrary Code Execution" leaves the two contradicting each other, and the title is what appears in listings, Dependabot alerts and downstream mirrors. Rewrite it to describe the defect.
- `severity`
- `cvss_vector_string` — clear it or replace it. A vector left in place **drives the displayed severity**, so it will override the severity you just set. Omitting it lets your chosen severity stand, which is often what you want when an honest vector still computes higher than the severity you can defend.
- `cwe_ids` — reclassify when the framing changed (CWE-863 authorization bypass → CWE-670 always-incorrect control flow, say)
- `vulnerable_version_range` and `patched_versions` — do not promise a patched version for a fix that has not landed; re-check the merge state rather than assuming
- the description — rewrite it to **stand alone**. It is published to people who never saw the report, so it states the defect, the real preconditions, and what the attacker does and does not control. Corrections to the reporter's claims go in the reply, not here. A description carrying "the impact is smaller than reported" or "what the report missed" reads as half of a conversation the reader cannot see, and it is the most common thing to get wrong at this step. Grep your draft for "report" before sending it.

```bash
gh api --method PATCH /repos/{owner}/{repo}/security-advisories/{ghsa-id} --input payload.json
```

Verify by re-fetching. Do not trust the PATCH response you did not read.

**Gotchas worth knowing before you start:**

- **There is no API for advisory comments.** No REST endpoint, and GraphQL has no `RepositoryAdvisory` type at all — only the read-only global `SecurityAdvisory`, which has no comments field. Replies have to be posted by hand. **Do not drive a browser to post or edit them** — it is slow, error-prone (the comment box shares a hidden mirror with the description field, kebab menus target the wrong node), and the user can paste the text themselves in under three minutes. Always hand the reply back as a **direct advisory link plus the comment body in a copy-paste fenced block** (see step 12 for the exact shape). Never imply you posted it, and never claim to have read a reply you cannot fetch.
- **The severity enum has no "not a vulnerability" value.** Setting `low` on a non-vulnerability is self-contradictory. Use `state: closed` and say why in the reply.
- **`triage` → `draft` is the maintainer accepting the submission**, not a side effect of your edit; `submission.accepted` flips at the same time. Before reporting any field change as something you caused, check whether your payload even contained that field.
- Publishing even a `low` advisory fires Dependabot for every dependent. When the fix ships in a normal patch release anyway, raise "publish or just release" as a deliberate choice rather than defaulting either way.

## 12. Draft the reply

Blunt and factual. Publishing exposes the advisory *content* — summary, description, severity, credits. The comment thread with the reporter stays private, so you are writing for them, not for a future audience. Do not invoke publication as a reason to soften, restructure, or hedge the reply.

Structure that works: concede what reproduced, correct the vector axis by axis with the evidence from steps 5 and 6, name what they missed from step 3, then state what you are fixing and that it lands on its own merits.

Two things to get right. Build the argument only on premises that are expensive to falsify — the cheap-to-falsify kind hands the reporter the thread, as in step 5. And keep the criticism factual rather than sharpened; the facts do more damage than adjectives.

If the user's project has a text-humanising skill, run the draft through it before handing it over.

**Delivery format — always.** The user posts and edits these by hand, so make it copy-paste ready. For each advisory, give its **direct URL** (`https://github.com/{owner}/{repo}/security/advisories/{ghsa-id}`) as a markdown link, then the comment body in its own ```` ```text ```` fenced block — advisory comments are full of inline `` `code` `` (identifiers, `__proto__`, CVSS strings), and a fenced block keeps them intact for a clean copy. One link + one block per advisory. When you are editing a comment the user already posted rather than writing a fresh one, say "paste over your existing comment on" and give the same link + block. Do not bury the text in prose or a table; the block is the deliverable. On a bulk drop, list every advisory this way, and call out which ones need **no** change (comment already accurate) so the user doesn't hunt for a diff that isn't there.

## Report back

Close with:

- **What was verified** — the literal reproduction output, and which claims held versus failed
- **Honest severity** — with the axis-by-axis reasoning, and the CVE recommendation
- **Anything the report missed** — extra defects, and any cross-report chain
- **Advisory fields changed** — before and after, confirmed by re-fetching
- **The drafted reply(ies)** — in the step-12 delivery format: per advisory, a direct link plus the comment in a copy-paste fenced block, with any no-change-needed ones flagged
- **What needs the user's own hands** — pasting the comment (link + block provided), publishing, closing, and any fix still unwritten

Flag disagreements plainly. If reasoning that is actually going into the reply or the advisory is falsifiable, say so before it ships — that is far more useful than agreeing. Scope this to text headed for the reporter: an opinion the user voices to you in conversation is not draft copy, and arguing with it as though it were is noise.
