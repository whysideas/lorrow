# Lorrow Implementation Profile Template

Every Lorrow-compatible implementation must publish this Profile: public, complete, accurate to deployed behavior, and linked from the product’s interface. Generate it at the end of every BUILD automatically. All 19 fields are required. If a field is undecided, ask the user; do not invent. If the Profile describes intended rather than deployed behavior, label it DRAFT prominently.

```markdown
# Lorrow Implementation Profile

**Standard:** Lorrow Framework Specification v1.0 (https://whysideas.github.io/lorrow/)
**Status:** [DEPLOYED | DRAFT]

| # | Field | Declaration |
|---|---|---|
| 1 | Protocol name | <name and version> |
| 2 | Chain / VM | <chain(s), contract addresses, verification links> |
| 3 | Supported loan assets | <exact list> |
| 4 | Supported collateral assets | <exact list> |
| 5 | Collateral ratio range | <implementation's breach_threshold range; must be ≥ 110%> |
| 6 | Interest rate range | <implementation's band; must be ≤ 36% annually> |
| 7 | Oracle provider | <provider used for valuation, breach, and LTV — same source for all three. "None" permitted only with an oracle-free surplus mechanism declared in field 10> |
| 8 | Oracle fallback policy | <secondary source or procedure> |
| 9 | Oracle failure behavior | <recommended default: pause breach detection without freezing active loans> |
| 10 | Default settlement method | <how collateral is valued at default and how surplus is returned to the borrower — be specific; this field carries the standard's core promise> |
| 11 | Liquidation method, if any | <any mechanism beyond the standard breach-and-default flow, or "None"> |
| 12 | Maturity grace period | <must be ≥ 3 days> |
| 13 | Breach window policy | <windows per term; must be ≥ 1d/3d/6d floors> |
| 14 | Commitment Score formula | <how the five standardized inputs are weighted, or "Not yet implemented"> |
| 15 | Fee model | <every fee: kind, size in bps, recipient, when charged. Origination protocol fee max 100 bps. "None" if zero> |
| 16 | Upgradeability / admin | <admin controls and upgrade mechanisms and their limits; state explicitly that active loans are untouchable> |
| 17 | Compliance / KYC posture | <any identity requirements, or "None"> |
| 18 | Keeper model | <who runs reportBreach/checkRecovery/executeDefault; open or permissioned> |
| 19 | Frontend / operator | <who operates the interface; operator disclosures> |
```

## Field guidance

- **Field 5 / 6 / 12 / 13:** these declare the implementation’s chosen position within Core bounds. If a declared value sits outside a Core bound, the Profile is self-disqualifying — flag it immediately rather than publishing it.
- **Field 10** is the most important field in the document. A vague answer here (“standard default handling”) is unacceptable. Name the valuation source (oracle at default time, or origination-frozen settlement value), show the formula (debt = outstanding principal + accrued interest; surplus = collateral value − debt; surplus → borrower), and state what happens when collateral is worth less than the debt (lender absorbs the shortfall; borrower owes nothing further).
- **Field 15:** include every fee a user can encounter, including optional interface fees, placement/boost fees, and third-party service fees, even if defaulted to zero. Undisclosed fees discovered later void the Profile’s accuracy.
- **Field 16:** if the contract is upgradeable, describe exactly why upgrades cannot alter active loan records. If they can, the implementation violates the immutability guardrail and should not publish a compatibility claim.

## Machine-readable companion (recommended, optional)

Alongside the markdown, emit `lorrow-profile.json` with the same 19 fields keyed `protocol_name`, `chain_vm`, `loan_assets`, `collateral_assets`, `collateral_ratio_range`, `interest_rate_range`, `oracle_provider`, `oracle_fallback`, `oracle_failure_behavior`, `default_settlement`, `liquidation_method`, `maturity_grace`, `breach_windows`, `commitment_score_formula`, `fee_model`, `upgradeability_admin`, `compliance_kyc`, `keeper_model`, `frontend_operator`, plus `lorrow_version: "1.0"` and `status`. This enables automated Profile discovery and cross-implementation comparison, which is the standard’s long-term goal.