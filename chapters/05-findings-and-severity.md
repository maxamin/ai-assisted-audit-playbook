# Writing Findings and Assigning Severity

A report is the product. Clients act on clear findings with honest severity. Unclear findings get ignored, and inflated severity costs you credibility.

## Finding template

```markdown
## [H-01] Reward debt updated after balance change lets users claim twice

**Severity:** High (impact: High, likelihood: Medium)
**Location:** `Staking.sol#L142-L160`

### Description
`stake()` increases `balances[user]` before calling `_updateReward(user)`, so the
reward for the new balance is computed for the whole elapsed period.

### Impact
Any staker can claim rewards for time they were not staked. On the current pool
parameters this drains the reward reserve in ~N days.

### Proof of concept
`test/poc/H01.t.sol` — `test_doubleClaim()` (fails before fix, passes after).

### Recommendation
Call `_updateReward(user)` before modifying `balances[user]`.
```

## Severity matrix

Most firms and contest platforms use an impact × likelihood matrix:

| | Likelihood High | Medium | Low |
|---|---|---|---|
| **Impact High** (loss of funds, protocol insolvency) | Critical/High | High | Medium |
| **Impact Medium** (temporary DoS, value leak with limits) | High | Medium | Low |
| **Impact Low** (best practice, minor griefing) | Medium | Low | Info |

Read the specific platform's rules: Code4rena, Sherlock, Cantina and Immunefi each have their own severity criteria, and they differ on admin-trust and external-protocol assumptions.

## Where AI helps in reporting

- Drafting descriptions from your notes and PoC.
- **Consistency checks**: same terminology, line references still valid at the final commit, every finding has a PoC or a justification for not having one.
- Detecting **duplicates** and root-cause groupings across findings.

## Where it must not decide

Severity. Models systematically over-rate severity and are swayed by wording. A human assigns it and states the reasoning.

## Writing for the reader who fixes it

The developer reading a finding needs four things in order: what is wrong, why it matters, how to see it, and what to change. Keep each to its section. Common improvements to model-drafted text:

- Replace adjectives ("critical flaw", "severe issue") with numbers or conditions ("any user can withdraw up to the full reward reserve").
- State preconditions explicitly ("requires the pool to be empty", "requires the admin to list a fee-on-transfer token").
- Name the root cause once, in the title, and keep it consistent across description and recommendation.
- Give the smallest correct fix, and mention alternatives only when they differ in trade-off.

## Worked example: a report consistency checker

Not executed in this book's CI; the commands, code and expected outputs below are illustrative. Verify locally before relying on them.

Before delivery, run mechanical checks on every finding file in `audit/05-findings/`. This catches the errors AI drafting tends to introduce.

```python
# check_findings.py
import re, pathlib, subprocess, sys

errors = []
for f in sorted(pathlib.Path("audit/05-findings").glob("*.md")):
    text = f.read_text()
    title = text.splitlines()[0]
    if not re.match(r"## \[(C|H|M|L|I)-\d{2}\] ", title):
        errors.append(f"{f.name}: bad title format")
    for path, a, b in re.findall(r"`([\w/\.]+\.sol)#L(\d+)(?:-L(\d+))?`", text):
        p = pathlib.Path(path)
        if not p.exists():
            errors.append(f"{f.name}: missing file {path}")
        elif int(b or a) > len(p.read_text().splitlines()):
            errors.append(f"{f.name}: {path} line {b or a} out of range")
    m = re.search(r"`(test/[\w/\.]+\.t\.sol)` — `(\w+)\(\)`", text)
    if m:
        r = subprocess.run(["forge", "test", "--match-test", m.group(2)],
                           capture_output=True, text=True)
        if r.returncode != 0:
            errors.append(f"{f.name}: PoC {m.group(2)} does not pass")
    elif "No PoC:" not in text:
        errors.append(f"{f.name}: no PoC and no justification")
print("\n".join(errors) or "all findings consistent")
sys.exit(1 if errors else 0)
```

Run it at the final commit. Expected output on a clean report: `all findings consistent`. On an AI-drafted report, expect the first run to flag issues such as out-of-range line references or findings with no PoC.

Note that this checker cannot tell whether line ranges point at the *right* code, only that they exist. A human still reads every location.

## Calibrating severity across a team

Severity disputes are common. Two habits reduce them:

1. **Write the impact and likelihood reasoning in one sentence each** before choosing the label. The label then follows from the matrix.
2. **Keep a team precedent file**: past findings with agreed severities and reasons, grouped by root cause. When a new finding resembles an old one, start from the precedent. An assistant is useful here for retrieval: "find precedents for an oracle staleness issue where the fallback oracle exists".

## Grouping and deduplicating findings

Large reviews produce near-duplicates: the same missing check in four functions, or one rounding error that surfaces in three conversions. Group by **root cause**, not by location. One finding with four locations is easier to fix and to judge than four findings. An assistant can propose groupings ("these three findings all stem from `_convert` rounding down"), but accept a grouping only when the same code change fixes every member. A quick test: apply the proposed fix and rerun every PoC in the group. If one still passes (still demonstrates the bug), it has a different root cause.

## Common pitfalls

- **Inflated severity.** "Loss of funds" without a realistic path is not High. Judges and clients notice.
- **Copying model impact text.** It often overstates or invents preconditions. Rewrite from your PoC.
- **Line references at the wrong commit.** Re-check all references after the client's final push.
- **Bundling root causes.** Two bugs in one finding make fixes and duplicates harder to judge. One root cause per finding.
- **Missing "no PoC" justification.** When a PoC is impractical (for example, governance timing), say why and describe the steps instead.

## Exercises

1. Rewrite a vague past finding using the template.
2. Ask a model to grade the severity of 10 known findings. Compare with the official judgement.
3. Build a script that checks every `Location:` link in a report still points at the right code.

## Further reading

- Code4rena severity categorisation: <https://docs.code4rena.com>
- Sherlock judging guidelines: <https://docs.sherlock.xyz>
- Immunefi vulnerability severity classification: <https://immunefi.com/immunefi-vulnerability-severity-classification-system-v2-3/>
