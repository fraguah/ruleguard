# RuleGuard

**Pre-launch ambiguity lint for prediction-market resolution rules, decided by GenLayer validator consensus.**

Most prediction-market fights are not about what happened. They are about what the rule *meant*. RuleGuard moves that argument to before the market opens, when fixing a sentence is free, and then freezes the reviewed text so nobody can "clarify" it after money is in.

---

## Why this exists

Disputes are growing with volume. Reporting in mid-2026 counted more than 1,150 disputed Polymarket markets in the first five months of the year, already above the 2025 total. Two well-known patterns:

| Pattern | What went wrong | RuleGuard category |
|---|---|---|
| *"Did Company X sell bitcoin by May 31?"* | The sale happened in May, but the filing that disclosed it came out on June 1. The rule never said whether the deadline applied to the **event** or to the **disclosure**, and a clarification was added only after the market had moved. | `TIMING_BASIS` |
| *"Will the president wear a suit?"* | Reasonable people disagreed on whether the outfit was a suit, because the rule never defined the word. | `UNDEFINED_TERM` |

Neither needed a smarter judge after the fact. Both needed one more sentence before launch.

## What it does

1. **Submit.** The market author registers `market_id`, a question and the rule text. The contract stores the SHA-256 of the text.
2. **Review.** Anyone calls `review(market_id)`. Each validator's model checks the rule against **six fixed defect categories** and must answer every one.
3. **Verify.** A flag only counts if the model quotes the exact clause it objects to. The contract checks that the quote really is in the rule. Unverifiable flags are dropped deterministically.
4. **Decide.** No flags gives `PASS`, any flag gives `NEEDS_FIX`, with a short suggested fix per flag.
5. **Revise.** The author can revise and get a fresh review. Each version is reviewed **once**, so the author cannot re-roll the model until it says PASS.
6. **Lock.** The author locks a passed rule, which freezes its hash. A flagged rule can only be locked with explicit `accept_warnings=True`, and is then marked `LOCKED_WITH_WARNINGS`.
7. **Launch gate.** A market contract calls `is_launchable(market_id, rule_hash)` before opening. It is true only for a clean lock of exactly that text.

### The six categories

| Category | Flags when… |
|---|---|
| `TIMING_BASIS` | a deadline does not say whether the event or its report/disclosure must happen by then |
| `UNDEFINED_TERM` | the outcome hinges on an undefined, contestable word |
| `SOURCE_GAP` | no source, no priority between conflicting sources, or no fallback |
| `MISSING_TIMEZONE` | a date or time has no timezone |
| `THRESHOLD_EDGE` | a numeric boundary is unclear (>= vs >, rounding, which feed or candle, units) |
| `NON_EXHAUSTIVE` | cancelled, postponed, tied or never-published cases are not covered |

The list is fixed in code on purpose. An open "is this rule ambiguous?" question does not reach stable consensus. Six yes/no questions, each anchored to a quote, can.

## How consensus is used

`review()` uses `gl.vm.run_nondet_unsafe` with a custom validator (the partial-field-matching pattern from the GenLayer docs).

**Leader.** Asks the model, then runs `_extract_flags()`. This is pure, deterministic code that:
- keeps only the six known categories,
- requires all six to be answered (otherwise `[LLM_ERROR]`, forcing leader rotation),
- drops any flag whose quote is not found verbatim in the rule (after whitespace and quote-mark normalisation, minimum 8 characters),
- sorts the result.

**Validator.**
1. Checks the leader's evidence deterministically. Every leader flag must have a known category, a verbatim quote and a bounded fix. This blocks a leader that invents objections or quotes text that does not exist.
2. Re-runs the review with its own model and requires the **same set of flagged categories**. Quotes and fix wording may differ. The decision may not.

A leader that hides real problems (claims PASS) or adds fake ones is rejected. If validators cannot agree, the transaction is undetermined and state does not change.

**Prompt injection.** The rule is fenced as untrusted data and the model is told not to follow instructions inside it. Because the rule text is public and hashed, an injection attempt is itself visible evidence against the market author.

## Interface

```python
submit_rule(market_id: str, question: str, rule_text: str)   # write
revise_rule(market_id: str, rule_text: str)                   # write, author only, before lock
review(market_id: str)                                        # write, anyone, once per version
lock(market_id: str, accept_warnings: bool)                   # write, author only

is_launchable(market_id: str, rule_hash: str) -> bool         # view, for market contracts
get_market(market_id: str) -> dict                            # view: text, hash, version, status, flags
get_status(market_id: str) -> str                             # view
get_version_hash(market_id: str, version: int) -> str         # view, audit trail
hash_text(rule_text: str) -> str                              # view, same hashing as the contract
get_market_count() -> int                                     # view
```

Statuses: `DRAFT → PASS | NEEDS_FIX → LOCKED | LOCKED_WITH_WARNINGS`. `revise_rule` sends a non-locked rule back to `DRAFT`.

### Using it from a market contract

```python
guard = gl.get_contract_at(RULE_GUARD_ADDRESS)
if not guard.view().is_launchable(market_id, rule_hash):
    raise gl.vm.UserError("rule not reviewed, or text differs from the reviewed version")
```

The market stores `rule_hash` at creation. If anyone later edits the displayed rule, the hash no longer matches the locked one.

## Tests

```bash
pip install genlayer-test genvm-linter
genvm-lint check contracts/rule_guard.py     # lint + SDK validation: passes
pytest -q tests                              # 28 direct-mode tests: pass
```

The tests cover:
- the illustrative event-vs-disclosure and undefined-term cases, and a clean rule that passes,
- fabricated and too-short quotes being dropped, and typography-tolerant quote matching,
- incomplete or malformed model output reverting,
- the validator agreeing when only wording differs and disagreeing when the category set differs,
- the validator rejecting a leader that forges quotes, invents categories, hides flags or errors,
- no re-rolls on the same version, author-only revise and lock, explicit acceptance for flagged rules,
- the hash lock refusing an edited text, and the version history,
- the prompt fencing the rule as untrusted data.

**Mutation check.** 14 hand-made mutants of the key guards were run against the suite. 13 are killed. The one survivor removes the validator's unknown-category check, which is redundant: such a leader is already rejected by the re-review comparison.

The model is mocked in direct mode. These tests prove the contract logic, not model quality.

## Live run on GenLayer Studio (real validator models, full consensus)

Contract: `0xD2fe709E2486223A444f123660F9b0aC70deA6C2` (GenLayer Studio, Normal / Full Consensus mode)

| Step | Transaction | Result |
|---|---|---|
| Deploy | `0xdc2656b8bb25a3affdfcf9ff1beffc3743076340521d5e55e8c2624cab682b4a` | FINALIZED |
| `submit_rule("sale-may", …)`, the ambiguous v1 rule | `0x06e3d2af1ea635b42eeebe6f49ebb2dc066c782cbcdf2d528fdf6a1910e77b2b` | FINALIZED |
| `review("sale-may")` on v1 | `0x916e6fbfe675436948bf652df4fb7de3fe56caf123bbc3faf1a0c5507f8c5da9` | FINALIZED, `NEEDS_FIX` |
| `revise_rule("sale-may", …)`, the fixed v2 rule | `0xfa64992909c112d6a2fac72d4bb392397e23a0a133f80f85af0fd292e16d6d25` | FINALIZED |
| `review("sale-may")` on v2 | `0xcc697c5c838f40e906f91bd5e62c6b1de9a70ddacf5dad43e28685ad83583553` | FINALIZED, `PASS` |
| `lock("sale-may", false)` | `0xe350f932e230a3e193b45865eb76f7902588e949d4f872aae269bd03366c33ba` | FINALIZED, `LOCKED` |

**v1 rule (as submitted):**
> This market resolves to Yes if Company X sells any bitcoin by May 31, 2026. Resolution will be based on official company disclosures and on-chain data. Otherwise this market resolves to No.

**What the validators agreed on (stored on-chain, verbatim):**

| Category | Quote found in the rule | Suggested fix (model output) |
|---|---|---|
| `TIMING_BASIS` | "resolves to Yes if Company X sells any bitcoin by May 31, 2026." | Specify if the sale must occur by the deadline or if a disclosure of a prior sale published after the deadline counts. |
| `MISSING_TIMEZONE` | "by May 31, 2026." | Add a specific timezone, such as 11:59 PM UTC, to the deadline. |
| `SOURCE_GAP` | "Resolution will be based on official company disclosures and on-chain data." | Specify which source takes precedence if an official company disclosure contradicts on-chain data analysis. |

This is exactly the event-vs-disclosure gap behind the real dispute, caught before launch.

**v2 rule** fixes all three: UTC deadlines, an explicit "sale date counts, not publication date" clause, SEC EDGAR as primary source with a named fallback, and a "no filing → No" clause. Validators returned no flags, giving `PASS`, version 2.

**Launch gate after lock:**
- `is_launchable("sale-may", "ce05e733…e949")` (the v2 hash) returns `true`
- `is_launchable("sale-may", "814a3038…dafce")` (the v1 hash) returns `false`

**Observed consensus behaviour.** The v1 review needed several leader rotations before a majority agreed on the exact category set. The v2 review needed two. That is the cost of requiring exact set agreement on a judgement task. The contract prefers "no answer yet" to a confident wrong answer, and no state changes until consensus is reached.

## Limits (read before relying on it)

- **PASS is not a guarantee.** It means six specific defect types were not found. A rule can still be disputed for other reasons.
- **Model judgement varies.** Borderline rules may need several leader rotations (seen in the live run) or go undetermined. That is by design: ambiguity about ambiguity should not produce a confident PASS. Only one live run per rule was made, so general accuracy is not measured.
- **Text only.** The contract reviews the wording it is given. It does not fetch the sources the rule names or check that they exist.
- **No legal meaning.** It is a technical pre-check, not legal or compliance advice, and it does not replace the platform's own dispute process.
- **Testnet software.** No audit, no mainnet deployment claimed.

## Files

```
contracts/rule_guard.py    the Intelligent Contract
tests/test_rule_guard.py   direct-mode test suite
```

License: MIT
