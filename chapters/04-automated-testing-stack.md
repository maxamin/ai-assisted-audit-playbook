# The Automated Testing Stack

A modern audit runs several engines in parallel during Phase 3. The agent's job is to configure them for the codebase, interpret their output and turn interesting results into hypotheses.

## The stack

| Layer | Tools | Agent task |
|---|---|---|
| Static | Slither, Aderyn, Semgrep rules | triage detectors; suppress known false positives with justification |
| Property fuzzing | Foundry invariants, Echidna, Medusa | draft handlers and invariants; tune bounds |
| Symbolic | Halmos, hevm, Kontrol | convert key invariants to symbolic tests |
| Formal | Certora (if budget allows) | draft CVL; explain counterexamples |
| Fork testing | Foundry `--fork-url`, Anvil | test integrations with real tokens/oracles |
| Mutation | Gambit, vertigo-rs | measure test suite strength |

## Static triage loop

```
slither . --json slither.json
→ agent groups by detector, opens each location,
  labels: true-positive / false-positive (reason) / needs-human
→ human reviews needs-human and samples 10% of the agent's FP labels
```

Sampling the agent's false-positive labels is important. If the sample shows errors, re-review the whole set.

## Running fuzzers well

- **Seed the corpus** with realistic sequences (deposit, then borrow, then price change).
- **Bound inputs** in handlers to realistic ranges so the fuzzer does not waste time on reverts.
- **Track revert rates**: above ~50% means handlers are badly bounded.
- **Run long**: overnight Medusa or Echidna campaigns on several cores find bugs that 10,000 Foundry runs miss.

## Mutation testing as a quality gate

Before trusting "all invariants pass", run mutation testing. If mutants that change `<=` to `<` in fee calculations survive, the suite does not check fees. Agents can read surviving mutants and propose the missing property, which is a nice self-improving loop.

## Choosing what to run under time pressure

Not every engagement can afford every layer. A rough guide:

| Engagement | Must run | Add if time allows |
|---|---|---|
| 1-week review, < 1,000 nSLOC | Slither, Aderyn, Foundry invariants on core accounting | Halmos on 2–3 key functions |
| 2–4 week review, DeFi core | above + Medusa or Echidna overnight + fork tests for integrations | Gambit mutation run, Certora on critical maths |
| Fix review | rerun all PoCs, invariants and static tools on the fix commit | diff-focused mutation run |

The agent's main value here is setup speed: writing handlers, `echidna.yaml` or `medusa.json`, fork test scaffolds and mutation scripts that would otherwise take a reviewer a day.

## Worked example: a Phase 3 sweep in one script

Not executed in this book's CI; the commands, code and expected outputs below are illustrative. Verify locally before relying on them.

The following runs the core layers on a Foundry project and leaves outputs in `audit/03-automated/`. Versions and flags should be pinned in your template repo.

```bash
#!/usr/bin/env bash
set -euo pipefail
OUT=audit/03-automated; mkdir -p "$OUT"

# 1. static
slither . --json "$OUT/slither.json" --exclude-dependencies || true
aderyn . -o "$OUT/aderyn.md"

# 2. invariants (Foundry)
forge test --match-path 'test/invariants/*' -vv | tee "$OUT/invariants.txt"

# 3. long fuzz campaign (Medusa), runs in background overnight
medusa init 2>/dev/null || true          # creates medusa.json on first run
timeout 8h medusa fuzz > "$OUT/medusa.txt" 2>&1 &

# 4. symbolic checks on functions named check_*
halmos --function check_ | tee "$OUT/halmos.txt"
```

Slither exits non-zero when it finds issues, hence the `|| true`. Medusa reads targets from `medusa.json`; set `fuzzing.targetContracts` to your handler or test contract.

**Agent triage of Slither output.** Load `slither.json`, group results by `check`, and for each result give the agent the code around `elements[0].source_mapping`. Ask for one of three labels with a one-line reason. Store labels in a CSV so that sampling is easy:

```python
import json, csv, random
res = json.load(open("audit/03-automated/slither.json"))["results"]["detectors"]
rows = [(d["check"], d["impact"], d["first_markdown_element"]) for d in res]
print(len(rows), "results;", len({r[0] for r in rows}), "detectors")
sample = random.sample(rows, k=max(1, len(rows)//10))   # 10% for human review
csv.writer(open("sample.csv", "w")).writerows(sample)
```

**Reading the outcome.** Expect most static results to be informational or false positives on mature code. A human reviews everything labelled "needs-human" plus the 10% sample. Any disagreement in the sample sends the whole set back for human review.

## Common pitfalls

- **Default fuzzer configs.** Without handlers and bounds, Echidna and Medusa call functions with random arguments that revert. Configure targets and check revert rates.
- **Running dependencies through Slither.** Use `--exclude-dependencies` or filters; otherwise results from OpenZeppelin and forge-std drown out project code.
- **Short campaigns.** A few minutes of fuzzing during the day is not a campaign. Schedule overnight runs early in the engagement.
- **Unpinned tool versions.** Detector sets and fuzzers change. Record versions in the report appendix.
- **Treating a clean run as assurance.** No tool output says "secure". It says "these checks found nothing", which the report must phrase carefully.

## Exercises

1. Set up Slither + Foundry invariants + Medusa in CI for a sample protocol.
2. Measure handler revert rates and fix the worst handler.
3. Run Gambit and write invariants for the three most important surviving mutants.

## Further reading

- Medusa: <https://github.com/crytic/medusa>
- Echidna tutorial: <https://secure-contracts.com/program-analysis/echidna/>
- Halmos: <https://github.com/a16z/halmos>
- Kontrol: <https://github.com/runtimeverification/kontrol>
- Gambit: <https://github.com/Certora/gambit>
