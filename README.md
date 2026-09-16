# hypothesis2omics-evidence-rules

Three checks that keep an evidence table honest. One file, no dependencies,
plain dicts in and out.

Part of **Project 10, Hypothesis2Omics** at the NIAID-BRC AI Codeathon 2.0.
A sibling to the other team components, not a replacement for any of them.

```bash
python evidence_rules.py   # four case demo, 30 self-tests
python adapters.py         # worked example against the eligibility engine
```

Both run in about a second and need nothing installed.

---

## Where this sits

The project pipeline runs roughly: hypothesis, then datasets, then eligibility,
then analysis, then a verdict. Different people own different parts.

| Stage | Component | Owner |
|---|---|---|
| Retrieval and orchestration | [yijun-nih/projects/codeathon](https://github.com/yijun-nih/projects/tree/main/codeathon) | Yijun Zhou |
| Dataset eligibility | [akumar901/hypothesis2omics-scientific-validator](https://github.com/akumar901/hypothesis2omics-scientific-validator) | Amar Kumar |
| **Evidence rules** | this repo | Rushikesh Lagad |

Team repo: [NIAID-BRC-Codeathons/hypothesis2omics](https://github.com/NIAID-BRC-Codeathons/hypothesis2omics)

The eligibility engine decides **which datasets we may use**, one dataset at a
time. This repo decides **what the results mean**, across datasets. Those are
different questions and the code does not overlap.

---

## The three checks

**1. Lock the rule before you look.**

`DecisionRule` is frozen at construction. Build it during planning, pass it to
scoring afterwards. Nothing in between can widen alpha, flip the expected
direction, or drop the effect floor once numbers are in.

**2. Count independent groups, not rows.**

Three results from three subseries of one study are one confirmation. The
accessions all look different, which is what makes this easy to get wrong and
expensive to get wrong in public.

**3. Say inconclusive when you mean it.**

A significant effect in the wrong direction is recorded as a refutation rather
than quietly dropped. A required measurement nobody actually made means the
question went unanswered, whatever the rows say.

---

## The gap this fills

An eligibility engine looks at one dataset at a time, so it structurally cannot
ask the set-level question: **of the datasets we cleared, how many separate
studies are they really?**

`adapters.py` answers it. Feed it eligibility verdicts plus whatever dataset
descriptors you hold, and `independence_summary()` tells you, **before any
analysis runs**, whether the eligible set can possibly reach a supporting
verdict. If five eligible datasets collapse to one research programme, no
result from them can clear a two group bar, and the honest move is to widen the
search rather than run the numbers and hope.

Worked output from `python adapters.py`:

```
3. Independence check, before running anything:
   program:HIPC_YF: GSE13485, SDY1264
   bioproject:PRJNA_LAUSANNE: GSE13699
   -> 3 eligible, 2 independent group(s), 2 required

4. Same set, now with results:
   verdict: supports

5. Drop the one dataset from the second group:
   verdict: inconclusive
   because: 2 supporting result(s) but only 1 independent group(s)
```

Same numbers, different verdict, because independence changed.

---

## Using it

```python
from evidence_rules import DecisionRule, integrate
from adapters import groups_from_verdicts, independence_summary

# during planning, before anything has run
rule = DecisionRule(alpha=0.05, direction="up", min_independent_groups=2)

# verdicts from the eligibility engine, records are your dataset descriptors
groups = groups_from_verdicts(verdicts, records)

# sanity check before spending compute
print(independence_summary(verdicts, records, rule)["note"])

# after analysis
result = integrate([rule.score(r) for r in rows], rule, groups)
result["verdict"]    # supports | refutes | inconclusive
result["rationale"]  # one sentence
result["caveats"]    # what the report must disclose
```

`rule.score` copies your row and adds `verdict` and `verdict_rationale`. It
never mutates the original, so it will not disturb anything you already store.

Rows need `effect` and one of `fdr` or `p_value`. Everything else passes
through untouched, so this works with whatever row shape you already use.

### Grouping fields

`independence_group()` reads whichever of these a record has, in order:

```
superseries -> bioproject -> study -> program
then lab plus platform
then the accession itself, meaning assumed independent
```

Pass what you have. Missing fields are skipped, never guessed.

---

## Deliberate limits

**Direction is never inferred from hypothesis text.** "Associated with" does
not state a sign, and guessing one is exactly the assumption this repo exists
to prevent. Set it explicitly or leave it `"either"`.

**Scoring is per row and does no multiple testing across rows.** Testing many
genes? Correct first, pass adjusted values in as `fdr`. `benjamini_hochberg()`
is included if you want it without pulling in scipy.

**Every effect size in the demos is invented.** They exist to show the logic,
not a result. The output says so on the first line.

---

## Notes for the team

Take the parts that help and ignore the rest. Two specific suggestions:

`independence_group()` is worth lifting on its own even if nothing else here is
wanted. It is about forty lines and it has no dependencies.

The eligibility engine's three way split (`eligible` / `uncertain` /
`excluded`, with execution status kept separate from scientific status) is a
better model than a binary pass. If anything merges, it should merge in that
direction.

One open question worth deciding deliberately rather than by accident: a pooled
model with a `trial` covariate and a count of independent replications are two
different ways to handle study effects. They answer different questions. Doing
one and reporting the other is the trap.

MIT licensed. Copy anything.
