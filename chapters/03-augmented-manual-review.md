# Augmented Manual Review

Manual review remains the core of an audit. Used well, an assistant is a **fast, tireless junior who never gets to sign anything**.

## High-leverage questions to ask an assistant

- "Trace every path by which `totalShares` can change. List function, line and condition."
- "Which functions can be called between `_beforeTransfer` and `_afterTransfer`?"
- "Explain this assembly block in plain terms and list its memory assumptions."
- "Write the shortest Foundry test that calls `liquidate` on a position that is healthy by the oracle price."
- "What invariants would the author of this function believe hold? Which of them does the code not enforce?"

Questions that ask for **enumeration with citations** are reliable. Questions that ask "is this safe?" are not.

## Rubber-duck adversarial review

Explain your hypothesis to a model and ask it to **argue against it** with code citations. Either the counter-argument holds (you saved time) or it fails (your confidence rises). Do not let the model's tone sway you; check the cited lines.

## Diff review for fix rounds and upgrades

Give the model the diff plus the original finding and ask:
1. Does the change address the root cause or only the symptom?
2. What new paths does the change introduce?
3. Which existing tests should now fail, and which new test proves the fix?

## Notes discipline

Keep a running `notes.md` per contract: assumptions, questions for the team, and hypotheses with status. Paste relevant parts into the assistant's context instead of re-asking from scratch. It also makes the audit reproducible for the second reviewer.

## Worked example: verifying an enumeration answer

Not executed in this book's CI; the commands, code and expected outputs below are illustrative. Verify locally before relying on them.

Enumeration questions are reliable only if you check them. This is a repeatable way to do it for "every path that changes a state variable".

**Step 1 – ask.** "List every function, line and condition by which `totalShares` changes, including through internal calls and inherited contracts. Format: `Contract.function:line — condition`."

**Step 2 – get the ground truth from a tool.** Slither's `vars-and-auth` printer lists, per function, the state variables it writes. Filter for the variable:

```bash
slither . --print vars-and-auth 2>&1 | grep -B3 totalShares
```

For a grep-level cross-check of direct writes:

```bash
grep -rnE "totalShares\s*(\+|-|\*|/)?=" src/
```

**Step 3 – compare.** Every function in the tool output must appear in the model's list, and vice versa. Typical discrepancies:

| Discrepancy | Usual cause |
|---|---|
| Model misses a function | write happens in an internal helper or a parent contract |
| Model lists an extra function | it reads `totalShares` but does not write it |
| Line numbers off by a few | model counted from a different version of the file |

**Step 4 – record.** Put the verified list in `notes.md`. It is now a fact you can paste into later prompts.

**Step 5 – act.** For each writer, ask the worst-moment question from Chapter 2. Writers reachable by untrusted callers without a matching update to `totalAssets` are prime hypotheses.

## Working with long codebases

For codebases larger than a comfortable context window, do not paste everything. Give the assistant a **map** (file list with one-line purposes, the inheritance graph and the function summary) and let it request files. Many coding assistants can read files on demand. The map keeps answers grounded and makes it obvious when the model is talking about a file it has not read.

## Second-reviewer protocol

When two humans audit the same code, ask the assistant to compare their `notes.md` files and list hypotheses one reviewer has and the other does not. It is cheap and reliably surfaces blind spots. The humans still decide which gaps to chase.

## Anti-patterns

- Accepting a summary of a contract you did not read.
- Asking the model to "find all bugs" in 5,000 lines in one prompt.
- Copying model-written severity or impact text into a report unchanged.
- Letting the assistant run against mainnet RPCs with signing keys present.

## Common pitfalls

- **Unchecked citations.** A plausible line number that points to the wrong code misleads more than no citation. Spot-check every one you rely on.
- **Asking yes/no safety questions.** "Is this safe?" invites a confident yes. Ask for paths, conditions and counter-examples instead.
- **Stale context.** Pasting a file from yesterday's commit into today's question produces answers about code that no longer exists.
- **Anchoring on the model's framing.** If the assistant calls something a "reentrancy", you may stop looking for the accounting bug next to it. Form your own hypothesis first.
- **Mixing clients in one session.** Context from one engagement leaking into another is a confidentiality breach (see Chapter 7).

## Exercises

1. Pick a function you know well. Ask the assistant to enumerate paths that change one state variable. Check every citation.
2. Run an adversarial review of one of your past findings. Did the model find a valid objection?
3. Review a fix PR with the three diff questions.

## Further reading

- Trail of Bits, "How to prepare for a security review": <https://secure-contracts.com>
- Solodit checklist: <https://solodit.cyfrin.io/checklist>
- See **llm-smart-contract-vuln-research** Chapter 7 (agents with PoC gates).
