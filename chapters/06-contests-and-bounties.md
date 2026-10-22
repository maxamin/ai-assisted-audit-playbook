# Contests and Bug Bounties in the AI Era

Audit contests (Code4rena, Sherlock, Cantina, CodeHawks) and bounty platforms (Immunefi, HackenProof) turned auditing into a competitive market. AI has changed it in two visible ways.

## Effect 1: more noise

Platforms have seen floods of low-quality, AI-generated submissions: generic findings, wrong line references, and hallucinated impacts. Responses have included reputation gates, stricter validity rules, penalties for invalid submissions, and AI-based pre-filtering. If you use AI, **your submissions must look more rigorous than average, not less.** A runnable PoC is the clearest signal.

## Effect 2: faster coverage of known patterns

Low-hanging, pattern-matchable issues get found and duplicated quickly. Contest payouts usually split duplicates, so the value of generic findings drops. The remaining edge is in:

- deep protocol-specific logic,
- integration assumptions (tokens, oracles, other protocols),
- economic attacks needing simulation,
- new ecosystems (Move, Solana native, Cairo) where tools and models are weaker.

## A sustainable workflow for solo researchers

1. **Day 1:** read docs and code yourself. Run the agent inventory and static sweep in the background.
2. **Day 2+:** hypotheses → PoCs. Use the agent to write tests, not to choose targets.
3. **Before submitting:** run the agent as a *critic* against your own finding: "why would a judge mark this invalid?"
4. **After the contest:** compare your findings with the final report, and add the misses to your personal retrieval index.

## Bounties: extra rules

- Respect scope and the program's testing rules. Most forbid mainnet testing; use forks.
- Never touch user funds. A whitehat rescue is a special case with its own norms and legal risk (see the MEV book).
- Use the safe-harbour terms where programs offer them (e.g. the SEAL Whitehat Safe Harbor framework).

## What judges look for

Contest judging criteria differ by platform, but valid high-severity findings share traits: a clear root cause, a realistic attack path under the contest's stated assumptions, quantified impact, and a PoC that runs against the contest repository. Common reasons for invalidation include out-of-scope code, reliance on a trusted admin acting maliciously when the rules say admins are trusted, "user error" scenarios, and impacts that require unrealistic token behaviour the protocol explicitly does not support. Read the contest README's "known issues" and "trusted roles" sections before writing anything.

## Worked example: running a "judge critic" before submitting

Not executed in this book's CI; the commands, code and expected outputs below are illustrative. Verify locally before relying on them.

You have a draft finding and a PoC. The aim is to find reasons it would be invalidated, then fix or drop it.

**Step 1 – collect the rules.** Save the contest README (scope, known issues, trusted roles, supported tokens) and the platform's severity rules to `rules.md`.

**Step 2 – run the PoC in a clean checkout.**

```bash
git clone <contest-repo> clean && cd clean
cp ../my-poc/test/PoC_H01.t.sol test/
forge test --match-test test_H01 -vvv
```

Judges run your PoC against the contest repo. If it only passes with your local modifications, fix that first.

**Step 3 – critic prompt.**

```text
You are a contest judge applying these rules: <rules.md>.
Here is a submitted finding: <finding.md> and its PoC output: <forge output>.
List every reason this could be judged invalid, downgraded, or a duplicate of a
known issue. Quote the rule you rely on for each reason. If none apply, say none.
```

**Step 4 – resolve each objection.** For each reason, either change the finding (narrow the impact, add a precondition, show the admin is not involved) or accept it and downgrade or drop the submission. Check every quoted rule against `rules.md` yourself; models misquote.

**Step 5 – submit** with the PoC inline or linked as the platform requires.

What to expect (not a measured result): the critic will often flag a known-issues overlap or an admin-trust assumption on a meaningful share of drafts. Dropping those saves your reputation score on platforms that penalise invalid submissions.

## Bounty disclosure basics

For bounties, the process is different from contests. Report privately through the program's channel, include a fork-based PoC, and wait for triage. Do not publish details until the program agrees or its stated disclosure window passes. If a project has no bounty or security contact, look for a `SECURITY.md`, the security contact listed in verified contract metadata, or a coordinated-disclosure intermediary such as SEAL 911 for urgent issues.

## Common pitfalls

- **Submitting AI output with no PoC.** On most platforms this is now the fastest route to a low reputation.
- **Ignoring known issues.** Many "findings" are listed in the README or a previous audit.
- **Mainnet testing on bounties.** Almost all programs forbid it. Use a pinned local fork.
- **Racing duplicates with weak reports.** A rushed submission that is a duplicate of a better one may be judged lower or invalid.
- **Arguing severity with model-written text in escalations.** Judges respond to code and rules, not tone.

## Exercises

1. Take a finished contest. Before reading the report, spend 4 hours with your workflow. Then compute your recall.
2. Write a "judge critic" prompt and run it against three of your past invalid submissions.
3. Read a bounty program's rules and list what you may and may not do.

## Further reading

- Code4rena: <https://code4rena.com>; Sherlock: <https://www.sherlock.xyz>; Cantina: <https://cantina.xyz>
- Immunefi: <https://immunefi.com>
- SEAL Safe Harbor: <https://securityalliance.org>
