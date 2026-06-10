# Lorrow Compatibility Audit Checklist

Use this checklist in AUDIT mode. Work through every item; do not sample. Classify each finding into one of four tiers, then render the verdict.

## Classification tiers

- **COMPLIANT** — matches Core exactly.
- **PROFILE CHOICE** — a customization Core explicitly leaves open (asset lists, oracle choice, ranges within bounds, fee within cap, score weighting, keeper model, UI). No compatibility impact if disclosed in the Profile.
- **DIVERGENCE** — departs from Core’s letter but arguably honors its spirit (e.g., oracle-free surplus return via frozen settlement value; flat fee instead of interest_rate; missing breach machinery in a time-only design). Document it; the compatibility claim requires explicit justification, and the verdict is at most “conditionally compatible” until the divergence is either accepted into the spec or remedied.
- **VIOLATION** — breaks a guardrail or removes a Core requirement (no surplus return; mutable live terms; rate over 36%; penalty over 5%; threshold under 110%; harsher-than-floor windows; no recovery-resets-clock). Compatibility claim is invalid.

## Section A: Loan variables

For each of the 12 variables, check: present, named per spec (or a documented 1:1 mapping), correct type/units (basis points where specified), within Core bounds.

- [ ] loan_asset — distinct from collateral_asset enforced?
- [ ] loan_amount — > 0 enforced?
- [ ] collateral_asset
- [ ] collateral_amount — > 0 enforced?
- [ ] breach_threshold — floor 11000 bps enforced on-chain?
- [ ] loan_term — restricted to the fixed set (or documented subset)? Any invented terms = DIVERGENCE.
- [ ] interest_rate — ceiling 3600 bps enforced on-chain? (A flat-fee model with no rate = DIVERGENCE; evaluate whether the effective cost can exceed a 36% annualized equivalent — if it can without bound, escalate to VIOLATION.)
- [ ] repayment_structure — three options present?
- [ ] early_repayment_allowed — false actually reverts early payoff?
- [ ] early_repayment_penalty — ceiling 500 bps enforced?
- [ ] capital_locked / collateral_locked — both posting modes supported and recorded?

## Section B: Lifecycle

- [ ] All six states reachable and correctly gated (POSTED, ACTIVE, BREACHED, DEFAULTED, COMPLETED, EXPIRED).
- [ ] BREACHED → ACTIVE transition exists for both cure payment and price recovery.
- [ ] Cure payments never charged the early penalty.
- [ ] closed_early recorded on pre-maturity completion.
- [ ] Terminal states truly terminal.
- [ ] No state in which terms can be modified.

A design with no BREACHED state at all (pure time-based settlement, no oracle) is a structural DIVERGENCE, not automatically a violation — but check Section D’s surplus item with extra care, because such designs are where surplus return most often goes missing.

## Section C: Functions and events

- [ ] All nine functions present with spec semantics (names may map; semantics may not).
- [ ] All nine events emitted at the correct moments.
- [ ] LoanDefaulted carries both debtToLender and surplusToBorrower (this is how surplus return is audited on-chain; its absence is a red flag).
- [ ] executeDefault callable only when the window/grace conditions genuinely hold.
- [ ] Unlocked-post acceptance failure invalidates the post.

## Section D: Guardrails (the verdict mostly lives here)

- [ ] **Surplus return.** Trace the default path in code. Does surplus actually move to the borrower? Via live oracle valuation, or via an origination-frozen settlement value (acceptable)? If the lender can ever keep collateral worth more than the debt — including via a “pure pledge” mode being the default or the only mode — that is a VIOLATION. If a forfeit mode exists but is opt-in alongside a real surplus-return mode, classify the forfeit option itself as a VIOLATION risk and flag that loans using it are not Lorrow loans; the product can only claim compatibility for the surplus-return path, and the Profile must say so.
- [ ] Interest ceiling enforced on-chain.
- [ ] Penalty ceiling enforced on-chain.
- [ ] Threshold floor enforced on-chain.
- [ ] Breach windows at or above per-term floors; recovery resets the clock.
- [ ] Maturity grace at or above 3 days.
- [ ] Immutability: no admin function, upgrade path, or governance action can alter an active loan’s terms. Upgradeable contracts are not disqualifying per se, but the upgrade mechanism must be unable to touch live loan records — verify, do not assume.

## Section E: Escrow

- [ ] Locked posts escrow at posting; cancel returns funds.
- [ ] Unlocked posts pull at acceptance; failed pull = EXPIRED.
- [ ] Neutral escrow: neither party nor admin can extract escrowed funds outside the defined flows.

## Section F: Profile

- [ ] A published Implementation Profile exists (public, findable from the product’s interface).
- [ ] All 19 fields present and accurate to deployed behavior.
- A missing or materially incomplete Profile blocks the compatibility claim on its own, regardless of contract quality. The claim is Core + Profile, both required.

## Verdict template

```
# Lorrow Compatibility Audit: <implementation name>
Audited: <contract addresses / repo / commit>
Date: <date>

## Verdict: [COMPATIBLE | CONDITIONALLY COMPATIBLE | NOT COMPATIBLE]

## Summary
<2-4 sentences>

## Findings
| Item | Tier | Notes |
|---|---|---|
...

## Conditions / remediations (if conditional)
...

## Divergences requiring spec-author decision (if any)
...
```

When a DIVERGENCE genuinely honors the spirit through a different mechanism, say so generously and precisely — the standard wants adoption, not gatekeeping — but never launder a VIOLATION into a divergence. Surplus return in particular admits alternative mechanisms but not alternative outcomes.