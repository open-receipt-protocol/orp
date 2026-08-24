# Open Receipt Protocol (ORP)

**Competing on Quality, Not Attention**: accountable product discovery through AI hosts.

ORP (Open Receipt Protocol) is a reciprocal public-good protocol for accountable discovery. Companies register claims about their products. LLM Hosts facilitate recommendations and feedback. Verified humans contribute the claim-specific judgments. Those contributions build graphs of supply, demand and accountability. The structure, commitments, and aggregate signals of all three are public; individual conversations and identities stay private throughout. A neutral non-profit operator or a federation of such maintain the graphs but may sell no influence over them.

The contribution is the accountability loop: signed rating receipts that reference the exact claims a recommendation was made on, produced by verified humans, tied to a recommendation the operator itself served, and held by an operator with nothing to sell. A host that runs its own version of this cannot write these receipts, because everyone can see whose thumb is on the scale. This is what makes the quality signal hard to fake and hard to buy retroactively.

![The discovery-then-accountability loop](figures/orp-discovery-accountability-loop.svg)

## Read

- **[One-page abstract](ABSTRACT.md)** ([PDF](abstract.pdf)): start here.
- **[White paper](whitepaper.md)** ([PDF](whitepaper.pdf)): the full thesis. The centralization problem, the parties and their incentives, the core mechanism, identity and privacy, scoring, the transparency model, operator neutrality, and a per-surface attack map.
- **[Technical Overview](technical-overview.md)**: the engineering-depth companion. How each concept works, at the depth needed to build against and to attack.

## Status

Draft v1.0 thesis (2026-08-07). In the paper's own words: it is "a draft contribution written in a finished register, not a finished architecture: it argues a thesis and shows one way to build it, and it does not pretend to have found every flaw in its own design or an optimal solution. It invites the scrutiny a claim-accountability protocol should hold itself to."

Adversarial review is welcome: [open an issue](https://github.com/open-receipt-protocol/orp/issues).

## Author & license

Niklas Henkel. Contact via [issues](https://github.com/open-receipt-protocol/orp/issues).

Text and figures are licensed under [CC BY 4.0](LICENSE).
