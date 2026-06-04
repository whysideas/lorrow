# Lorrow

A Standard Framework for Bilateral Collateralized Lending.

Read the full specification: <https://whysideas.github.io/lorrow/>

## What Lorrow Is

Lorrow is not a lending product. It is a standard that lending protocols implement around.

Most crypto lending today dissolves the relationship between lender and borrower into an anonymous pool with an algorithmic rate. That is efficient, but it removes the thing that makes credit credit: the agreement. Lorrow gives the agreement back to the two parties while keeping the trustless enforcement that makes on chain lending worth using.

It defines a common grammar for a bilateral loan: the variables, the lifecycle states, the escrow model, and the default rules. Any user can read any Lorrow loan in a common format, on any chain, in any interface. Like a token standard, Lorrow does not dictate what a protocol is for or how it prices risk. It defines what must be common so loans stay legible and borrowers stay protected.

## The Three Layers

**Core** is what must be common. The standardized loan variables, the lifecycle states, the required functions and events, the escrow model, and the guardrails that cannot be removed.

**Profile** is what each implementation customizes and must publish. Chain, assets, oracle policy, risk ranges, fee model, admin controls, and more. The Profile is mandatory. The answers are the implementation’s own, within Core’s bounds.

**Compatibility** is the claim a protocol earns by preserving Core and publishing a complete Profile. To say a protocol is Lorrow compatible means a user can read its terms, lifecycle, escrow model, default process, and reputation signals in the common format, even where its risk rules differ.

## Non-Predatory by Construction

One rule defines the whole standard: no Lorrow compatible protocol should be able to be predatory. This is enforced in Core, not left to good intentions.

The clearest example is surplus return. At default, the lender receives only what they are owed. Any surplus collateral returns to the borrower. Always. Default makes the lender whole. It does not become a windfall.

Around that sit hard ceilings on interest and penalties, a floor on the collateral buffer, breach windows that scale with loan length, and a rule that price recovery resets the default clock so a flash crash cannot destroy an otherwise healthy borrower. A protocol can always be more generous than these bounds. It can never be harsher.

## Status

The framework specification is complete and ready to implement. Reference implementations are not yet published.

This is an invitation. If you run a lending protocol or are building one, you can adopt Core, publish your Profile, and give your users a kind of clarity this space rarely offers. Open an issue to discuss, challenge, or improve the standard. Standards get stronger when they are attacked honestly.

## License

This repository uses two licenses, by design.

The **written specification** (the framework document and its prose, including the hosted spec page) is licensed under Creative Commons Attribution 4.0 International (CC BY 4.0). You may implement, adapt, and redistribute it freely, including commercially, as long as you credit Lorrow and its originator.

Any **code** in this repository (reference implementations, tooling, examples) is licensed under the Apache License 2.0, which includes an explicit patent grant. See the LICENSE file.

Where the two could be read to overlap, the specification text is governed by CC BY 4.0 and the code is governed by Apache 2.0.

Lorrow was originated by WHYSIDEAS.
