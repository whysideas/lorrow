-----

## name: lorrow
description: Build, audit, or extend peer-to-peer lending products that comply with the Lorrow Framework, a standard for bilateral collateralized lending. Use this skill whenever the user asks to build P2P lending, crypto lending, loan smart contracts, collateralized lending protocols, a lending order book, loan escrow contracts, or anything involving lenders and borrowers agreeing to loan terms, even if they never say the word “Lorrow”. Also use it when asked to check, audit, or evaluate whether an existing lending implementation is Lorrow-compatible, or to generate an Implementation Profile for a lending product.

# Lorrow Framework Skill

Lorrow is a standard framework for bilateral collateralized lending, originated by WHYSIDEAS. It is to peer-to-peer loans what a token standard is to tokens: a common grammar that makes any compliant loan readable in the same format on any chain, in any interface. Full specification: <https://whysideas.github.io/lorrow/> and <https://github.com/whysideas/lorrow>

This skill has three modes. Determine which one the task calls for, then follow the matching section.

1. **BUILD** — the user wants a new lending product, contract, or protocol.
1. **AUDIT** — the user wants an existing implementation checked against Lorrow.
1. **PROFILE** — the user wants an Implementation Profile generated or completed.

In every mode, read `references/core-spec.md` before writing any code or rendering any judgment. It contains the exact variable names, lifecycle states, function signatures, events, and the loan record structure. Do not work from memory of this file’s summary; the precision lives in the reference.

## The one rule that governs everything

No Lorrow-compatible product may be predatory. This is not a tone suggestion; it is enforced through hard guardrails listed below. If a user asks you to build something that violates a guardrail (for example, “let the lender keep all collateral at default” or “set the rate to 50%”), do not quietly comply. Explain that the requested behavior breaks Lorrow compatibility and why the guardrail exists, then offer the compliant alternative. The user may still build a non-Lorrow product if they insist, but it must not claim Lorrow compatibility, and you should say so plainly.

## Core Guardrails (non-negotiable, memorize these)

|Guardrail                      |Rule                                                                                                                                                                |
|-------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|Surplus return                 |At default the lender receives only outstanding principal plus accrued interest. ALL surplus collateral returns to the borrower. Mandatory, absolute, no exceptions.|
|Interest ceiling               |36% annually, hard cap.                                                                                                                                             |
|Early repayment penalty ceiling|5% of outstanding principal, hard cap.                                                                                                                              |
|Breach threshold floor         |110% collateral-to-outstanding-principal, hard floor.                                                                                                               |
|Breach window floors           |1 day (14d loans), 3 days (30/60/90d), 6 days (12m/18m). Implementations may be more generous, never harsher.                                                       |
|Maturity grace floor           |3 days minimum after maturity before default can execute.                                                                                                           |
|Recovery resets clock          |Any return of the collateral ratio above threshold during a breach window resets the breach clock to zero.                                                          |
|Immutable terms                |No party, admin, or governance process may alter an active loan’s terms.                                                                                            |

An implementation may always be MORE generous to the borrower than these bounds. It may never be harsher.

## Mode: BUILD

Follow this sequence when building a Lorrow-compliant lending product.

1. Read `references/core-spec.md` in full. It defines the 12 standardized loan variables (exact names, no renaming, no additions, no omissions), the 6 lifecycle states (POSTED, ACTIVE, BREACHED, DEFAULTED, COMPLETED, EXPIRED), the 9 required functions, the 9 required events, and the loan record structure.
1. Establish the Profile decisions before writing code. Ask the user (or infer from context and state your assumptions): target chain, supported loan assets (stablecoins recommended at launch), supported collateral assets, oracle provider and failure policy, fee model (optional, max 100 bps at origination, must be zero-able), and their chosen ranges within the Core bounds. These are the implementation’s choices; Core does not dictate them, but they must be decided and later disclosed.
1. Implement Core exactly. Use the variable names verbatim (`loan_asset`, `loan_amount`, `collateral_asset`, `collateral_amount`, `breach_threshold`, `loan_term`, `interest_rate`, `repayment_structure`, `early_repayment_allowed`, `early_repayment_penalty`, `capital_locked`, `collateral_locked`). Implement all six states and all required functions and events with the semantics in the reference. The escrow model: locked posts escrow at posting, unlocked posts escrow at acceptance, failed pulls invalidate the post.
1. Enforce guardrails in the contract, not the frontend. Bounds checks belong in Solidity (or the target language), reverting on violation. A frontend-only check is not Lorrow compliance.
1. Dual order book, no matching engine, no counter-offers. Lenders post offers; borrowers post requests; a counterparty accepts posted terms exactly or does not. Acceptance with a version pin (accept terms-as-reviewed) is a permitted hardening.
1. Validate interest math. Fixed rate from origination. Accrued interest for early repayment = (annual_rate / 365) × days elapsed × outstanding principal. INSTALLMENT amounts are computed once at origination and stored, never recalculated.
1. Generate the Implementation Profile as a final deliverable, automatically, without being asked. Use `references/profile-template.md`. A build is not complete until its Profile exists, because publishing the Profile is half of the compatibility claim.
1. State the compatibility claim honestly. If the build preserves Core and has a complete Profile, it may claim “Lorrow-compatible.” If the user made divergent choices, list the divergences explicitly and do not apply the label.

## Mode: AUDIT

When evaluating an existing implementation (a contract, a repo, a deployed product):

1. Read `references/core-spec.md` and `references/audit-checklist.md`.
1. Work through the checklist systematically: variables present and named correctly, states implemented, functions and events present with correct semantics, each guardrail enforced on-chain, escrow model correct, terms immutable post-origination.
1. Classify each finding as one of: **COMPLIANT** (matches Core), **PROFILE CHOICE** (a permitted customization within Core bounds), **DIVERGENCE** (departs from Core but honors its spirit; compatibility claim requires justification), or **VIOLATION** (breaks a guardrail; compatibility claim is invalid).
1. Pay special attention to surplus return. It is the heart of the standard and the most common omission. An implementation without surplus return cannot be Lorrow-compatible, full stop. Note: oracle-free implementations may honor surplus return through a settlement value agreed and frozen at origination instead of a live oracle price; this is an acceptable mechanism if the surplus calculation and return are real.
1. Produce a verdict: Lorrow-compatible, conditionally compatible (list the conditions), or not compatible (list the violations). Always check whether a published Implementation Profile exists; a missing or incomplete Profile alone blocks the compatibility claim regardless of how good the contract is.

## Mode: PROFILE

When generating or completing an Implementation Profile, use `references/profile-template.md`. All 19 fields are required. For any field the user has not decided, ask rather than invent, except where the spec names a recommended default (oracle failure behavior: pause breach detection without freezing active loans). The Profile must be accurate to the actual implementation; a Profile that describes intentions rather than deployed behavior is misleading and should be labeled as draft.

## Things Lorrow does NOT define (do not invent requirements)

The chain or VM, the specific asset lists, the oracle provider, the frontend, KYC/AML posture, whether a fee is charged, governance of constants over time, legal enforceability, Commitment Score weighting formulas, keeper infrastructure, insurance, multi-collateral structures, undercollateralized lending, and secondary loan markets. These are Profile declarations or out of scope entirely. Do not block a build or fail an audit over them; require only that Profile-level items be disclosed.

## Reference files

- `references/core-spec.md` — The complete Core: variables, states, functions, events, loan record, escrow model, breach/default mechanics, commitment model. Read before any build or audit.
- `references/audit-checklist.md` — Step-by-step compatibility checklist with the four-tier classification. Read for AUDIT mode.
- `references/profile-template.md` — The 19-field Implementation Profile template with guidance per field. Read for PROFILE mode and at the end of every BUILD.

## License note

The Lorrow specification is CC BY 4.0 (attribution to Lorrow and WHYSIDEAS). Code you generate for a user belongs to them under whatever license they choose; reference implementations in the Lorrow repo are Apache 2.0. When generating documentation that reproduces spec content, retain attribution.