# Lorrow Core Specification Reference

This file is the operational distillation of the Lorrow Framework Specification v1.0 for implementation and audit work. Canonical source: <https://whysideas.github.io/lorrow/>

## Contents

1. Design principles
1. The 12 loan variables
1. The 6 lifecycle states
1. Required functions (9)
1. Required events (9)
1. Loan record structure
1. Escrow and commitment model
1. Breach and default mechanics
1. Commitment Score inputs
1. Order book rules
1. Interest calculation
1. Optional fee hook

-----

## 1. Design principles (preserve all five)

1. **A standard, not a product.** Lorrow defines common nouns, verbs, states, and disclosures. Implementations customize within bounds and declare their choices.
1. **Bilateral agreement.** Every loan is between a specific lender and a specific borrower. No pools, no protocol-set rates, no anonymous counterparties.
1. **Immutable terms.** Once a loan is created on-chain, no variable changes. No repricing, no governance interference with live loans, no admin keys over active agreements.
1. **Standardized variables.** Every loan uses exactly the variable set below. No additions, removals, or renames. A loan object missing any field is not a valid Lorrow loan.
1. **Transparent commitment.** Locked versus unlocked posting is visible and feeds an on-chain reputation signal.

## 2. The 12 loan variables

All set by the poster (the party creating the offer or request). Where a Core bound exists, the implementation chooses its own narrower range inside it and publishes that range in its Profile.

|Variable                 |Type                         |Core bound                             |Semantics                                                                                                                              |
|-------------------------|-----------------------------|---------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
|`loan_asset`             |address                      |Profile-defined list                   |Asset the borrower receives. Must differ from collateral_asset. Stablecoins recommended at launch.                                     |
|`loan_amount`            |uint256                      |> 0                                    |Principal, denominated in loan_asset.                                                                                                  |
|`collateral_asset`       |address                      |Profile-defined list                   |Asset pledged by the borrower.                                                                                                         |
|`collateral_amount`      |uint256                      |> 0                                    |Amount escrowed. LTV is derived for display (loan_amount / current collateral value) but never stored as a variable.                   |
|`breach_threshold`       |uint16 (basis points)        |>= 11000 (110%) hard floor             |Collateral-to-outstanding-principal ratio below which a breach begins. No Core ceiling.                                                |
|`loan_term`              |uint32 (seconds)             |Fixed set: 14d, 30d, 60d, 90d, 12m, 18m|Implementations may support a subset; may not invent new terms.                                                                        |
|`interest_rate`          |uint16 (basis points, annual)|0 to 3600 (0% to 36%) hard ceiling     |Fixed for the life of the loan.                                                                                                        |
|`repayment_structure`    |uint8 enum                   |0=LUMP, 1=INSTALLMENT, 2=BALLOON       |LUMP: principal+interest at maturity. INSTALLMENT: equal payments over term. BALLOON: interest-only during term, principal at maturity.|
|`early_repayment_allowed`|bool                         |true/false                             |If false, contract reverts early payoff attempts.                                                                                      |
|`early_repayment_penalty`|uint16 (basis points)        |0 to 500 (0% to 5%) hard ceiling       |Only meaningful if early_repayment_allowed. Charged on outstanding principal at early payoff.                                          |
|`capital_locked`         |bool                         |true/false                             |Lender escrowed loan_asset at posting. Feeds Commitment Score.                                                                         |
|`collateral_locked`      |bool                         |true/false                             |Borrower escrowed collateral at posting. Feeds Commitment Score.                                                                       |

## 3. The 6 lifecycle states

|State        |Entry trigger                                                                             |What is true in this state                                                                                                                        |
|-------------|------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------|
|POSTED (0)   |postOffer or postRequest                                                                  |Visible on order book. Escrowed if locked post.                                                                                                   |
|ACTIVE (1)   |acceptOffer or acceptRequest                                                              |Loan created, capital disbursed to borrower, collateral in escrow, clocks running. Also re-entered from BREACHED on cure/recovery.                |
|BREACHED (2) |Collateral ratio < breach_threshold                                                       |Breach clock running (window sized by term). Cure or price recovery returns to ACTIVE and resets clock to zero.                                   |
|DEFAULTED (3)|Breach clock ran full window continuously below threshold, OR unpaid past maturity + grace|Collateral valued; debt portion to lender; surplus to borrower; loan closed; recorded in both wallet histories. Terminal.                         |
|COMPLETED (4)|Full repayment at or before maturity                                                      |Principal + interest (+ early penalty if applicable) to lender; collateral back to borrower; `closed_early` flag set if before maturity. Terminal.|
|EXPIRED (5)  |cancelPost, or unlocked-post acceptance fails on pull                                     |Escrow returned, post removed. Terminal.                                                                                                          |

CURED is not a state; it is the BREACHED→ACTIVE transition. EARLY_REPAID is not a state; it is COMPLETED with `closed_early = true`.

## 4. Required functions

```
postOffer(params)        — lender posts. If capital_locked, pulls loan_amount into escrow. Emits OfferPosted.
postRequest(params)      — borrower posts. If collateral_locked, pulls collateral into escrow. Emits RequestPosted.
acceptOffer(offerId)     — borrower accepts a lender offer. Pulls any unescrowed side; creates loan; disburses
                           loan_amount to borrower; starts clocks. Emits LoanCreated.
acceptRequest(requestId) — lender accepts a borrower request. Same mechanics as acceptOffer. Emits LoanCreated.
repay(loanId, amount)    — borrower pays. Partial payments reduce outstanding principal (and can cure a breach).
                           INSTALLMENT payments validate against the stored schedule. Early full payoff reverts if
                           early_repayment_allowed is false; otherwise charges accrued interest + penalty. On full
                           repayment: release collateral, set COMPLETED. Emits RepaymentMade / LoanCompleted.
reportBreach(loanId)     — anyone/keeper. If ratio < threshold and not already BREACHED: set BREACHED, start
                           breach clock sized by term. Emits BreachTriggered.
checkRecovery(loanId)    — anyone/keeper. If BREACHED and ratio >= threshold: reset clock, set ACTIVE.
                           Emits BreachCleared.
executeDefault(loanId)   — anyone/keeper. Valid only if breach window fully elapsed continuously below threshold,
                           or maturity + grace elapsed unpaid. Values collateral; transfers debt portion
                           (outstanding principal + accrued interest) to lender; RETURNS SURPLUS TO BORROWER;
                           sets DEFAULTED. Emits LoanDefaulted.
cancelPost(postId)       — poster only, pre-acceptance. Returns escrow, removes post. Emits PostCancelled.
```

Cure payments during BREACHED are principal reductions, never early repayment; they never trigger the early penalty.

## 5. Required events

```
OfferPosted(offerId, lender, params, locked)
RequestPosted(requestId, borrower, params, locked)
LoanCreated(loanId, lender, borrower, params, timestamp)
RepaymentMade(loanId, amount, remainingPrincipal, timestamp)
BreachTriggered(loanId, collateralValue, threshold, windowSeconds, timestamp)
BreachCleared(loanId, collateralValue, timestamp)
LoanDefaulted(loanId, debtToLender, surplusToBorrower, timestamp)
LoanCompleted(loanId, totalInterestPaid, earlyClosure, timestamp)
PostCancelled(postId, fundsReturned, timestamp)
```

LoanDefaulted MUST carry both debtToLender and surplusToBorrower so surplus return is auditable on-chain.

## 6. Loan record structure

```
loanId:                     bytes32
lender:                     address
borrower:                   address
loan_asset:                 address
loan_amount:                uint256
collateral_asset:           address
collateral_amount:          uint256
breach_threshold:           uint16     // basis points, e.g. 12000 = 120%
loan_term:                  uint32     // seconds
interest_rate:              uint16     // basis points annually
repayment_structure:        uint8      // 0=LUMP 1=INSTALLMENT 2=BALLOON
early_repayment_allowed:    bool
early_repayment_penalty:    uint16     // basis points
capital_locked_at_post:     bool
collateral_locked_at_post:  bool
origination_timestamp:      uint256
maturity_timestamp:         uint256
outstanding_principal:      uint256
state:                      uint8      // 0..5 per lifecycle
breach_clock_start:         uint256    // 0 if not breached
breach_window:              uint32     // seconds, derived from loan_term
closed_early:               bool
```

## 7. Escrow and commitment model

- **Locked post:** funds (lender capital or borrower collateral) escrowed at posting. Cancellation returns them immediately.
- **Unlocked post:** statement of intent; contract pulls at acceptance. If the pull fails (insufficient balance/approval), acceptance fails and the post is invalidated (EXPIRED).
- Locked posts raise Commitment Score; failed unlocked acceptances lower it. Both are valid post types; the protocol surfaces the difference rather than forbidding either.

## 8. Breach and default mechanics

**Breach windows (Core floors, by term):**

|Loan term        |Minimum breach window|
|-----------------|---------------------|
|14 days          |1 day                |
|30 / 60 / 90 days|3 days               |
|12 / 18 months   |6 days               |

- The window measures **continuous** time below threshold, not cumulative. Any recovery above threshold (by cure payment OR price movement) resets the clock to zero. This is mandatory; it protects borrowers from wicks and flash crashes.
- **Maturity default:** unpaid at maturity starts a grace period (Core floor: 3 days). After it, executeDefault is valid.
- **Surplus return (the heart of the standard):** at default, the lender receives only outstanding principal + accrued interest, valued against the collateral. Everything beyond that returns to the borrower. If collateral is worth less than the debt, the lender absorbs the shortfall and the borrower owes nothing further. The same oracle source must be used for breach detection, recovery detection, and LTV display.
- **Oracle-free designs:** an implementation with no oracle may honor surplus return via a settlement value agreed by both parties and frozen at origination (the collateral’s deemed worth in principal terms). On default, surplus is computed against that frozen value and returned. This is an acceptable Profile-level mechanism if surplus genuinely flows back to the borrower. Pure pledge/forfeit (lender keeps everything) violates the guardrail and voids compatibility.
- Recommended oracle-failure default: pause breach detection; do not freeze active loans.

## 9. Commitment Score inputs (standardized; weighting is Profile-defined)

1. Locked post ratio — share of a wallet’s posts that were locked.
1. Completion rate — share of accepted loans reaching COMPLETED (computed separately for lender and borrower roles).
1. Breach recovery rate — of breaches entered, share returned to ACTIVE rather than DEFAULTED (borrower history).
1. Failed acceptance rate — share of unlocked posts that failed at the pull (strongly negative).
1. Loan volume — lifetime USD volume as a confidence weight.

Score is a signal, never a gate. No minimum score to post or accept.

## 10. Order book rules

- Dual book: lender offers and borrower requests, separate.
- No algorithmic matching. No counter-offers, no negotiation. Accept posted terms exactly or do not. (Version-pinned acceptance — taking terms exactly as reviewed — is a permitted hardening.)
- Each listing shows all variables, the poster’s Commitment Score, locked status, and live-derived LTV.
- Filtering is UI-level and unconstrained.

## 11. Interest calculation

- Accrues on outstanding principal from origination_timestamp.
- INSTALLMENT amounts computed once at origination from rate and term, stored, never recalculated.
- Early payoff accrued interest = (annual_rate / 365) × days elapsed × outstanding principal at payoff.

## 12. Optional fee hook

Implementations may charge one protocol fee at origination: a basis-point cut of loan_amount, recipient set at deployment, visible to both parties pre-acceptance, hard-capped at 100 bps, zero allowed. Any other fee structures (interface fees, placement boosts, third-party service fees) are Profile-level inventions: permitted if fully disclosed in the Profile, but they are not part of Core.