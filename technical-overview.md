---
title: "ORP (Open Receipt Protocol): Technical Overview"
subtitle: The engineering-depth companion to the ORP Architecture canon
status: DRAFT 2026-08-06
audience: protocol engineers, cryptographers, host and operator implementers, standards reviewers
---

# ORP (Open Receipt Protocol): Technical Overview

> **What this document is.** The engineering-depth companion to the ORP Architecture canon. The Architecture answers what ORP is and why it exists; this document says exactly how each of its concepts works, at the depth needed to build against and to attack. Its scope is the canon's scope: it covers the concepts the Architecture carries and no others. It assumes Architecture-level context and does not re-argue the venture thesis, the operator's legal form, or go-to-market.
>
> **Authority.** The Architecture canon is the sole design authority. Where any statement here conflicts with the canon, the canon wins. Section anchors of the form "Canon: Architecture §n" point to the concept each section deepens. Normative keywords (MUST, SHOULD, MAY) describe what a conforming implementation of the eventual protocol would do — the protocol itself is a proposal, and no implementation program is presumed. Where a statement traces to a ratified decision in the specification repository's ledger it is settled design; wire-level detail beyond that is drafted scaffolding showing the mechanism is buildable, open like everything else to specialist review.
>
> **Flag conventions.** `[PROPOSAL]` marks a best-fitting mechanism offered where the canon is deliberately silent, not a settled fact. `[PLACEHOLDER]` marks an illustrative value or name standing in for something test- or catalog-determined. `[ASSUMPTION]` marks an engineering assumption made to keep the text concrete. Nothing flagged is asserted as canon. All flags are collected in Appendix A. The design bias throughout is to state the shape of a mechanism and leave tuning open, minimizing needless attack surface.
>
> **Naming.** The protocol is referred to as **ORP (Open Receipt Protocol)** throughout. The final name is an open editorial decision and non-blocking; a global rename is trivial.

---

## T1. Claim model & compiler

*Canon: Architecture §5.*

### T1.1 Text is canonical; the embedding is a derived index

A product claim's normative content is its **`canonical_text`**: a single free-text sentence. Everything the matching stack consumes (embeddings, indexes) is a derived representation, reproducible from the canonical text plus a published embedding-model version, and never authoritative. The relationship is source code to compiled binary: when a derived vector and the canonical text disagree, for example after a model upgrade, the text governs.

This carries three properties the rest of the protocol depends on. The text survives model upgrades (the binary recompiles, the source is stable). It is rateable by humans and adjudicable by regulators, which a vector is not. And it keeps a historical match third-party recomputable: anyone holding the canonical text, the demand-claim text, and the pinned model reproduces the distance the operator computed (T4, T5).

### T1.2 Claims are flat and product-bound

A claim is exactly one immutable `canonical_text`, zero or more structured **conditionals** (T3), and a `min_days` follow-up-timer field (T7), belonging to exactly one product. There are no sub-claims and no nesting. **Product claims are the only free text in the protocol**; conditionals and product parameters are structured key/value objects (T3). Abstraction is expressed by registering several independent claims at different granularities, not by hierarchy. A broad claim ("cleans teeth") self-commoditizes because every competitor holds it too, so differentiation is the only path to discoverability.

Claim identity is the canonical text and nothing else. Conditionals never individuate a claim, and outcome timeframes belong in the text: "conversational French in 2 months" and "conversational French in 6 months" are two claims, not one claim with two conditionals.

### T1.3 The claim compiler

Companies author claims with marketing instincts; the compiler removes them before anything is embedded or rated. Submitted text is canonicalized into a boring engineering register (what the product does, for whom, under what conditions), and valuative vocabulary ("best", "great", "powerful", "seamless") is pruned or rejected. The company approves and signs the canonical form.

The compiler's internal procedure is operator-side and may be non-public, but its behavior contract is normative: both the input `source_text` and the output `canonical_text` are retained, and the company signs the output. Canonicalization never puts words in the claimant's mouth without a signature over them. The flow:

1. The product submits `source_text` and proposed structure to a registry.
2. The registry canonicalizes each claim's text and assigns the stable `claim_id` (and, on first registration, `product_id`).
3. The product signs the resulting manifest `body`, canonical text included. A claim is registered only once this signature exists; before it, the artifact is a submission, not a registered manifest.
4. The registry **countersigns (MUST)**. The countersignature is the registration record: it is the only attestation covering the registry-asserted fields (`registered_at`) and it certifies that canonicalization and registration checks passed (T2, T5).

`[ASSUMPTION]` The canonicalizer is itself an LLM stage with a pinned prompt and model version. Its determinism is a quality concern; its injection resistance is a trust dependency, because the canonicalizer's adversary is the submitting company itself, the party that approves and signs the output, so human approval is no defense. The canonicalizer MUST treat `source_text` as untrusted data and never as instruction, and the defense is architectural, never prompt-level: the output is constrained to the register rubric, the registry countersignature certifies that canonicalization checks passed, `source_text` is retained verbatim as audit evidence feeding the fraud layer (T10), and the operator MAY sample-check registered claim texts after the fact. The register rubric it enforces is the authoring contract, a separate normative artifact the canon scopes out of this document (§16).

### T1.4 Immutability and lifecycle

**`canonical_text` is immutable for the life of its `claim_id`.** There is no reword operation; changing what a claim says means registering a new claim with fresh history. Immutability scopes to the text only: a claim's conditionals, `min_days`, and its product's parameters are versionable via a strictly increasing `manifest_version`, each version product-signed and registry-countersigned. Every receipt pins `(claim_id, manifest_version)` (T6), so a historical rating is interpretable against the exact wording and conditional regime the user's recommendation ran against, and a later manifest update never moves the goalposts on a pending follow-up.

Claim lifecycle states and transitions:

| State | Matches? | Public? | Counts toward scoring? | Entered by |
|---|---|---|---|---|
| **submitted** | no | no | no | product submits `source_text` |
| **active** (registered) | yes | yes | yes | product signs + registry countersigns |
| **archived** | no | yes (history + receipts retained) | yes, permanently | product archives ("that wasn't it") |
| **reactivated** → active | yes | yes | yes | product reactivates (only return path) |

The governing rule is **archive, never delete**. A failed claim can be archived, at which point it stops matching and stops counting in scoring, but its canonical text, manifest history, and receipts stay public forever. Archiving is a **priced reset**: it costs a trust penalty on a published schedule, gentle at first, escalating when archival repeats while trust is still regenerating, higher for claims that carried negative ratings (T9.6). Reactivation preserves the `claim_id` and its entire history; there is no reset path in the record itself. **Trust regeneration is the only healer** (T9): a company that failed on a claim recovers by re-earning ratings on new or reactivated claims, or by paying the archive price and waiting out the regeneration.

### T1.5 No redundancy bar, no clustering

Every claim's matching position is its own canonical-text embedding, always (T4). There are no equivalence regions, no equivalence radius, no immutable centers, and no bar on a product registering claims that sit close to its own.

Paraphrase and claim-spray therefore need no dedicated penalty regime; they auto-balance through the scoring layer:

- paraphrased claims split feedback across `claim_id`s, starving each of the rating floor it must clear to publish anything at all (T9);
- claims that never match cost nobody anything but registry space;
- claims that match and get rated are, by definition, serving demand.

Because claims can never be deleted, discoverability is only ever earned by differentiated claims that match real demand and qualify for a rating. A registry MAY impose reasonable submission limits as governance, but the protocol itself does not need them. The residual adversarial surface, probing the registry to find registrable wordings, is a signed submission under the prober's own accountable entity identity, so it self-documents and feeds the fraud layer (T10).

---

## T2. Manifest & product record

*Canon: Architecture §5.*

The claim manifest is the product's public, signed commitment: the object a company puts its name to and the operator countersigns. It is the entire supply-side authoring surface.

### T2.1 Manifest envelope

A manifest is a JSON document with two top-level members, sharing the signing envelope used by receipts (T5):

```json
{ "body": { ... }, "signatures": { "product": {...}, "registry": {...} } }
```

The `body`:

| Field | Req | Description |
|---|---|---|
| `orp_version` | MUST | Document revision the instance was produced under (`"0.1"`) |
| `manifest_version` | MUST | Integer ≥ 1, strictly increasing per product |
| `registered_at` | SHOULD | RFC 3339 timestamp, set by the registry, covered only by the registry countersignature |
| `product` | MUST | Product record (T2.2) |
| `claims` | MUST | Non-empty array of claims (T2.3) |

A registered manifest MUST validate against the published manifest schema and MUST carry both signatures (T2.5).

### T2.2 Product record

The product record holds product-wide identity and facts: everything true of the product rather than of one claim.

| Field | Req | Description |
|---|---|---|
| `product_id` | MUST | `orp:product:{id}`, assigned at first registration, stable across all manifest versions |
| `name` | MUST | Display name. **Presentation only, never an input to matching or scoring** |
| `parameters` | MAY | Array of product-wide parameters (T3), filter-matched, never embedded |
| `jwks_uri` | MUST\* | HTTPS URL of the product's JWK Set (key discovery, T5) |
| `jwks` | MAY\* | Inline JWK Set |
| `homepage` | MAY | HTTPS URL |

\* At least one of `jwks_uri` / `jwks` MUST be present. Signing keys MUST be Ed25519 (`OKP`/`Ed25519`, `alg: EdDSA`). The `orp:` identifier scheme is deliberately unregistered (following the `did:` precedent); a standards-track version of the protocol would seek IANA provisional registration.

### T2.3 The claims array

Each entry is a flat, product-bound claim (T1.2):

| Field | Req | Description |
|---|---|---|
| `claim_id` | MUST | `orp:claim:{id}`, registry-assigned, stable across manifest versions |
| `canonical_text` | MUST | The registered, immutable claim wording (T1) |
| `source_text` | SHOULD | The wording as originally submitted, retained verbatim for audit |
| `conditionals` | MAY | Array of structured conditionals (T3) |
| `min_days` | MUST | Integer 0 to 365: minimum days after the user chooses the product before the host SHOULD elicit (0 = next contact). Drives the follow-up timer (T7) |

There is no sub-claim member, no per-claim stake, and no monetary bond anywhere on the wire: publisher skin-in-the-game is registration governance (legal-entity verification, TOS liability, entity-level bans), not escrowed money. Canonical text SHOULD read as an engineering requirement; registries MUST strip valuative vocabulary at canonicalization (T1.3).

### T2.4 Versioning

`manifest_version` increases by exactly 1 on every registered change to a product's manifest and is append-only and public. It is what every receipt pins so a rating stays interpretable against the exact wording and conditional regime it was produced under (T1.4). Distinct from it, `orp_version` tracks the document revision an instance was produced under: the specification prose, both JSON schemas, and the discovery contract share one version string in lockstep, so a wire artifact always names the exact shape it conforms to. Release mechanics (changelogs, schema `$id`s, breakage policy) are specification-repository housekeeping, not protocol design.

### T2.5 Signatures

The `signatures` member is an object keyed by signing role. On a registered manifest, **both `product` and `registry` are REQUIRED**:

```json
"signatures": {
  "product":  { "protected": "<b64url>", "signature": "<b64url>" },
  "registry": { "protected": "<b64url>", "signature": "<b64url>" }
}
```

The product signature covers the canonical form (T1.3); the registry countersignature is the registration record (T2.1). The registry is an artifact-scoped role: it signs manifests only and never appears among a receipt's signature roles (T5, T6). Construction (JCS-canonicalized body plus detached per-role Ed25519 JWS) is the common envelope defined in T5, so manifests and receipts share one signing profile, one crypto suite, and one key-discovery pattern.

---

## T3. Conditionals & product parameters

*Canon: Architecture §5.*

Conditionals and parameters are the protocol's two **structured** scoping constructs: the only machine-filterable surface in a design whose sole free text is the claim itself. Both draw on one shared key/value catalog and are matched by filtering, never by embedding.

### T3.1 Claim conditionals

A conditional scopes one claim's promise: *"we only promise this claim if or when X."* Its shape:

| Field | Req | Description |
|---|---|---|
| `key` | MUST | `snake_case` identifier from the shared catalog (T3.3) |
| `op` | MUST | One of `eq`, `lt`, `lte`, `gt`, `gte`, `in`, `range` |
| `value` / `values` / `min`,`max` | MUST per `op` | Comparison operand(s) |
| `unit` | MAY | Unit string for numeric values |

The conditionals array is an **implicit AND**. There is deliberately **no OR structure and no logic inside a value**: a disjunction or nested clause would be a predicate language by another name, re-importing the machine-evaluated-eligibility problem the design excludes (T7). Example claim scoping (`[PLACEHOLDER]` keys, illustrative, never reserved):

```json
"conditionals": [
  { "key": "learner_age", "op": "range", "min": 7, "max": 14, "unit": "years" },
  { "key": "practice_sessions_per_week", "op": "gte", "value": 3 },
  { "key": "spaced_repetition_review", "op": "eq", "value": true }
]
```

**A conditional is a pure filter; there is no `required` flag.** It does affect matching, from the demand side: the user or host may filter on the conditional's key (an age or price bound, a platform), and the conditional excludes a claim only against such an explicit demand-side constraint (a query-side `filter`, T4.3). When the query is silent on that key, the conditional does not exclude the claim: the claim still matches, and the host receives the conditional on the matched claim (T4.8) to surface as a disclaimer or to prompt refinement. A product can never compel the query to specify a field, for example the user's age, which would be too obtrusive to demand of every conversation. Honest scoping stays the authoring discipline (narrow enough to hold, wide enough to be matched, T9's GTM channel): over-narrow scope means the host discloses limits that may deter a poorly-fitting user, but it never removes the claim from matching when the demand side is silent.

**Legibility is a registration requirement.** A conditional MUST describe something a human could confirm in ordinary conversation; registries MUST reject conditionals keyed on internal telemetry, event counts, or session identifiers, which a user could not naturally report. A conditional is a public commitment and must be honestly confirmable, which is the point. This bar governs what a conditional may say; it never changes what a conditional does at matching time (T3.4).

**Conditionals do not gate feedback.** A conditional filters matching, never who may rate: letting a company narrow who may rate it points the wrong way, and the eligibility baseline is deliberately low (the user had access and says they used it, T7). Filter-matching conditionals at recommendation time is a different surface entirely, and is machine-evaluated by design (T3.4).

### T3.2 Product parameters

Parameters are product-wide structured facts (not claim scopings), living on the product record (T2.2):

| Field | Req | Description |
|---|---|---|
| `name` | MUST | `snake_case` catalog key |
| `type` | MUST | One of `string`, `number`, `integer`, `boolean`, `enum`, `range` |
| `values` | MUST for `enum` | Allowed values |
| `min`, `max` | MUST for `range` | Numeric bounds (either omittable, not both) |
| `value` | MAY | Fixed value for scalar types |
| `unit` | MAY | Unit string for numeric types |

Parameters participate in matching **as filters only** and MUST NOT be concatenated into any embedded text. Like everything in the manifest body, they are versioned via `manifest_version`. Example (`[PLACEHOLDER]` keys): `price_monthly` (number, EUR), `platforms` (enum: web/ios/android).

### T3.3 One catalog, two scopes

Conditionals and parameters draw on **one shared key/value catalog**. Two consequences the design commits to:

- **The catalog does not exist yet, and every key named in any ORP document is illustrative, never a reservation** (`[PLACEHOLDER]` in every case). Building it, the controlled vocabulary of structured edges (`age`, `location`, `platform`, price, and so on), is the **authoring contract's** job, a separate normative artifact scoped out of this Overview (§16). The catalog covers only the structured edges; claims themselves are never drawn from a catalog, because a controlled vocabulary over claims would collapse the protocol into 1990s key/value string matching and require modeling the entire world of supply and demand upfront.
- **The same key legitimately appears at both scopes.** An age limit can be a product-wide parameter ("this app is 12+") or a conditional scoping one claim ("we only teach reading to ages 7 to 10"). Which scope a fact belongs at is a registration-time question, not a structural one. How registries police misplacement, and cross-claim coherence (a product holding two mutually incompatible claims), are open operator-side governance questions, out of scope here.

With the `required` flag removed (T3.1), conditionals and parameters now have **identical matching behavior**: both are pure filters that bite only against an explicit demand-side constraint on their key, and both are surfaced to the host on the matched candidate. They differ only in scope (one claim versus the whole product), which is exactly what "one catalog, two scopes" means.

### T3.4 Filter placement is an implementation choice

Conditionals and parameters are **hard filters** applied somewhere within the selection pipeline (T4). At which stage (before distance computation, interleaved, or after retrieval) is an implementation-efficiency question, not a protocol rule. This Overview does not fix an ordering.

---

## T4. Demand claims & matching

*Canon: Architecture §6.*

Matching is the discovery half of the protocol. The canon is explicit that this is not where the novelty budget is spent (§14): claim-space matching and coverage aggregation are standard parts, deliberately assembled; the contribution is the accountability loop (T5 to T7), not the matcher. This section specifies the assembly precisely enough to build and attack, and no further.

### T4.1 Three objects: product claim, demand claim, need

| Object | Author | Crosses the wire? |
|---|---|---|
| **Product claim** | product (registered, rateable) | yes, indexed at the operator |
| **Demand claim** | host ("the claim a fitting product would make") | yes, as `demand_claim_text`, embedded verbatim |
| **Need** | the user's natural-language problem | **no, host-side only, ever** |

The division of labor is strict: **the host owns understanding; the protocol owns representation.** The needs-to-demand-claims translation happens entirely host-side; the registry never interprets a need, because interpreting the need would smuggle the solution in as a premise. What crosses the wire is a set of demand claims, plural per query, authored by the host from the conversation.

The host's authoring discipline is **Jobs-to-be-Done** (Christensen): customers are authoritative on problems and unreliable on solutions, so the host expresses granularity by authoring multiple demand claims at different altitudes. Job-altitude is a search dimension, not a registry interpretation.

### T4.2 Verbatim embedding and recomputability

Demand-claim text is embedded **byte-identically** by the operator; there is no operator-side pre-parse of any kind. Product claims and demand claims are embedded by the **same pinned model**, which is precisely what keeps match distances third-party recomputable from the texts alone (T1.1). Register discipline lives at authorship, not in an operator rewrite: the Host SDK requires demand claims to be functional, minimal, single-sentence, and free of valuative vocabulary. Because demand claims are LLM-authored, the register is controllable at the source without a compiler stage.

A consequence the design accepts deliberately: since the raw need never reaches the operator, **the operator cannot audit host translation honesty from its own data.** Hosts are audited by external comparison instead: feeding the same natural-language need to different hosts and comparing the resulting demand claims and recommendations for bias (T12 makes this measurable).

### T4.3 Request

| Field | Req | Description |
|---|---|---|
| `demand_claims` | MUST | Array of host-authored demand-claim strings. Each entry is one demand claim; there is no variant grouping |
| `filters` | MAY | Array of `{ name, op, value }` hard constraints over the parameter/conditional catalog (T3) |
| `locale` | MAY | BCP 47 tag |
| `max_products` | MAY | Host may request fewer than K; the operator caps at K |

Radii, retrieval breadth, and selection weights are **operator-owned and never appear in the request**. There is no per-demand-claim retrieval budget to expose, because retrieval is all-eligible (T4.4).

**Commons contribution requires a verified-human subject.** A request that carries a verified-human subject may contribute its demand claims to the demand commons (T11); a request without one is served and ranked normally but does not feed the published gap/saturation map. Recommendation is ungated on personhood; only the commons aggregates verified-human demand.

### T4.4 Selection pipeline

Given the embedded demand claims, selection proceeds:

1. **Hard filters.** Conditionals and product parameters (T3) constrain the candidate set, each biting only where the demand side (a query `filter`, T4.3) carries an explicit constraint on the same key; a conditional the query is silent on never excludes. Placement within the pipeline is an implementation-efficiency choice.
2. **Per-demand-claim retrieval, all-eligible.** For each demand claim, **every** product claim within the max match radius is eligible. There is **no claim-level cutoff of any kind**: no top-Z, no per-claim budget. Distance is **strictly binary at claim level**: a product claim is either in-radius or not; position within the radius buys nothing.
3. **One product claim per product per demand claim.** A product's several granularity-ladder claims compete against each other only here, collapsing to a single representative per demand claim (T4.5).
4. **Product aggregation, coverage-dominant.** Surviving claims aggregate to products. The selection composite has three inputs: **claim coverage** (how many of the query's demand claims a product matched), the **relevant claim-rating average**, and the **product/company trust multiplier** (T9). Coverage **dominates without being lexicographic**: strong preference for 5-of-5 over 4-of-5, but a 4.8-rated 4-of-5 product beats a 2.2-rated 5-of-5 product. Weights are test-determined and acknowledged as irreducibly preference-laden (T13).
5. **Top-K products returned.** K is small (order 3 to 10, test-determined). The **only cutoff in the entire pipeline is this product-level top-K.**

Oversupply on a generic demand claim ("easy to use", 100,000 matches) is harmless by construction: because the query as a whole describes a product and coverage dominates, candidates matching 1-of-4 demand claims fall away against 4-of-4-plus-filter matches. The composite does the discrimination, never retrieval.

### T4.5 Own-claim tiebreak

When several of one product's claims match a single demand claim in-radius, which one represents the product? The rule is ratified (D26.7):

> **The lowest-rated in-radius claim represents the product.**

The reasoning is adversarial-first. Closest-distance would reward positional wordsmithing; highest-rated would let a product hide a weak claim behind a strong near-synonym. **Lowest-rated makes holding several claims over one region worthless by construction**: nothing is gained by claim-spraying a region, which is what keeps claim-engine-optimization pointless (Architecture §6, §15).

`[PROPOSAL]` **Tiebreak among several claims sharing the low rating:** deterministic and non-gameable, for example the lexicographically smallest `claim_id` (equivalently, oldest registration, since IDs are assigned in order). This detail does not change the design; flagged, not settled.

### T4.6 No distance on the wire

**`match_distance` appears on no wire surface anywhere**: not on the discovery response, not on receipts, not on the authenticated query API. In-radius membership is binary, so a distance would convey no ranking information the composite has not already used, and it needlessly confuses hosts. Distance remains **third-party recomputable** from `demand_claim_text` plus public canonical text under the pinned matcher; that recomputability, not a wire field, is the audit path (T1.1).

### T4.7 No stochastic exposure

There are no probability floors, ceilings, or sampling anywhere in selection. Exposure legitimately reaches **1** (sole claimant of a matched demand) or **0** (an undifferentiated entrant to a saturated region); discovery is earned by differentiation, never granted by randomness. A product claiming exactly what an incumbent claims, contributing no differentiation, deserves no exposure; the newcomer's structural entry path is to serve a demand that is not currently served. Randomization survives in exactly **one** role: a coin-flip tiebreak among candidates genuinely tied at a cut edge.

### T4.8 The discovery response

The operator performs **no LLM step at query time**: marginal cost per query is embedding plus retrieval; no operator-side rerank exists and none may be assumed. The response:

| Field | Req | Description |
|---|---|---|
| `orp_version` | MUST | Document revision |
| `recommendation_ref` | MUST | Opaque operator-scoped id minted at query time, resolvable to the serving instantiation under federation (Architecture §13); the choice-record anchor later receipts reference (T6) |
| `matcher` | MUST | `{ embedding_model, version }`, the pinned model this matching ran under |
| `epoch` | MUST | The rating-publication epoch the response's rating-derived values come from (T10) |
| `supply` | MUST | `{ eligible, full_coverage }`, see T4.9 |
| `products` | MUST | ≤ K product entries |

Per **product entry**: `product_id`, `name` (display only), optional `parameters`, and a `claims` array carrying **matched claims only** (never the full portfolio). Per **claim entry**:

| Field | Req | Description |
|---|---|---|
| `claim_id` | MUST | `orp:claim:{id}` |
| `canonical_text` | MUST | Registered wording. Crosses to the host: the host's final trim requires text and scores in one context |
| `manifest_version` | MUST | Pinned version this recommendation runs against (goalpost stability, T1.4) |
| `demand_claim_index` | MUST | Which demand claim this product claim matched; coverage is host-derived from these, so no coverage field exists to game |
| `rating` | MAY | Per-claim composite, cardinal one-decimal (1.0 to 5.0); absent when below the rating floor (T9). No jitter, no ordinal ranks; published under the band-and-hysteresis discipline (T10.3) |
| `sample_tier` | MUST | Coarse confidence bucket, see T4.10 |
| `conditionals` | MUST when present | Full conditionals array at the pinned version; host context for commitment surfacing, tiebreaks, disclaimers |
| `min_days` | MUST | The claim's follow-up delay (T7) |

**Ordering.** The `products` array is **sorted by the operator's composite**: the host already sees coverage and ratings, so order conveys little it could not derive. Position-bias resistance is a host-conduct property, audited via per-host calibration (T12), and enforced by Host SDK discipline: *hosts MUST NOT overrate the operator's ranking.* Any moderately competitive query converges on a top-K with full coverage and close ratings; 4.5 vs 4.3 is not a deciding factor unless claim texts and conditionals are otherwise equivalent. The host judges on the semantic side and leans on the ranking signal only when the difference is severe: a lone full-coverage candidate, or one product's average strongly dominating the next.

**Never on the response:** the selection composite or any internal score, weight, or multiplier; product- or company-level rating aggregates in any form; demand-side aggregates (query volumes, share-of-chosen) at any granularity; exact rating counts; match distances; any cross-product rater identifier.

### T4.9 Supply metadata and the eligible-count threshold

The response carries two supply-side counts:

- **`eligible`**: total products clearing a minimum claim-match threshold, after filters and before the K-cut.
- **`full_coverage`**: the subset matching every demand claim (secondary information).

Exact values, not tiers; supply-side facts only; returned with a discovery response and available on no standing query surface. They answer the host's two questions: *are these good candidates?* and *would a different query do better?* Bad candidates mean loosen the demand claims or filters, or the market has a gap. Decent candidates but a large `full_coverage` count mean tighten, using the conditionals and parameters visible on the returned candidates as concrete examples of what can be constrained on.

`[PLACEHOLDER]` **The eligible-count threshold is left as an open range:** absolute (for example "at least one matching claim") or proportional (for example "at least half the demand claims"). It is a tuning parameter, not a wire semantic, and does not make or break the protocol. Over-specifying it here would add attack surface for no design gain (T13).

### T4.10 `sample_tier`: the coarse confidence bucket

Every per-claim wire `rating` is accompanied by a **`sample_tier`**: a coarse sample-size and confidence bucket that lets a host narrate honestly ("well established at 4.6" vs "promising but few ratings") **without ever exposing the exact rater count**. The five tiers are frozen labels:

```
unrated | stale | few | moderate | many
```

`unrated` means the claim has never cleared the rating floor (T9); its `rating` field is then absent. `stale` means the claim cleared the floor but has attracted no recent ratings; its last `rating` carries forward under the label (T9.2). `few`, `moderate`, and `many` are the published bands.

`[PLACEHOLDER]` The **numeric bounds** of `few` / `moderate` / `many` are published, versioned, and test-determined, deliberately open here. The exact `n` never crosses any query surface (T10 explains why: it would timestamp each rating's arrival and open a de-anonymization oracle).

### T4.11 Synonym linking `[PROPOSAL]`

The registry maintains a link layer over claim embeddings known to be semantically equivalent: claims in one **synonym family** are treated as co-located for matching, so a product claim matches a demand claim whenever any member of its family is in-radius. Links live registry-side and never change any claim's canonical text or identity (T1.4). The layer's existence is the canon commitment (Architecture §6); everything below is `[PROPOSAL]`-grade mechanism.

- **Admission:** how equivalence is established is open: operator batch analysis over the demand map, product petition, or both.
- **Recomputability:** the link table is published on the public layer (T10.2) and versioned, so a historical match stays third-party recomputable from the pinned model plus the link-table version in force at match time (T4.2).
- **Defect surfacing:** residual encoder defects need no probing to find: for any substantial demand they surface in the demand commons as apparent gaps covered by semantically equivalent claims (T11), and the repairs are a versioned model upgrade or a link added here.
- **Migration:** the procedure for recomputing historical matches across model or link-table versions is an open build item (T13).

### T4.12 Authoring references: Host SDK and Product SDK `[PLACEHOLDER]`

One register, two authoring surfaces. The Product SDK authors claims through the compiler rubric (T1.3); the Host SDK authors demand claims under the same functional discipline (T4.2). The success criterion is fixed by the canon (Architecture §16): independent hosts and products describing the same function land within matching distance without coordination. The normative rubric itself is the authoring contract, a separate artifact this Overview deliberately does not specify (T1.3, T3.3). This section is the reserved anchor for the SDK reference material once that artifact exists; until then, everything SDK-facing in this document is limited to the discipline statements in T1.3 and T4.2.

### T4.13 Synthetic matching evaluation `[PLACEHOLDER]`

A hand-adjudicable, end-to-end test of the authored pipeline before real receipts exist: the author writes and signs off plausible product claims for a test catalog in a familiar consumer domain; an LLM host running the Host SDK prompt turns stated needs into demand claims; the harness measures how well those demand claims find the right products. It exercises SDK prompts, embedder, and radius together, which is what justifies radius, embedder, and authoring-rubric choices before deployment (Architecture §16). It is dev tooling, not protocol: no wire surface, no schema, no operator obligation. This section is the reserved anchor for the harness description once it exists.

---

## T5. Receipt envelope & signing

*Canon: Architecture §7.*

The receipt is the protocol's core novelty: a record about a product that the product cannot write, veto, or shape (Architecture §7, §14). The envelope is what makes that structural. This section specifies the shared signing profile used by both receipts and manifests; T6 specifies the rating-receipt payload on top of it.

### T5.1 A role-addressed multi-signature envelope

A receipt is co-signed by distinct roles: **host and user, both always** (T6). The envelope is a JCS-canonicalized (RFC 8785) `body` plus **detached JWS signatures** (RFC 7515), one per role, keyed by role, **Ed25519 / EdDSA** (RFC 8037). Three properties are load-bearing:

- **Role-addressed co-signatures.** The role-keyed `signatures` map carries the host and user signatures side by side over one body, preserving one-event-one-receipt congruence (T6).
- **Schema-structural role rules.** The role-keyed map lets JSON Schema enforce role requirements structurally (both signatures always present, T6).
- **Readable cleartext body.** No embedded base64 payload, consistent with text canonicality (T1) and regulator adjudicability.

The crypto suite, JWKS key-discovery pattern, and canonicalization family are kept compatible with the ally host ecosystem, so a host already emitting host-witnessed receipts can supply them to ORP as supplementary evidence (T7's deferred contact-event primitive) without ORP taking a normative dependency on any single-vendor spec.

### T5.2 Common envelope structure

Every receipt and every manifest is `{ "body": { ... }, "signatures": { ... } }`. A receipt `body`:

| Field | Req | Description |
|---|---|---|
| `orp_version` | MUST | `"0.1"` |
| `receipt_id` | MUST | `orp:receipt:{id}`, generated by the issuing host; **ULID RECOMMENDED** |
| `receipt_type` | MUST | `rating`, the only receipt type in this version |
| `issued_at` | MUST | RFC 3339 timestamp |
| `product_id` | MUST | `orp:product:{id}` |
| `manifest_version` | MUST | The manifest version the recommendation ran against, so every outcome is interpretable against the exact wording matched (T1.4) |
| `recommendation_ref` | MUST | Opaque operator-scoped id of the served recommendation/choice record this elicitation closes (T4.8) |
| `subject` | MUST | `{ scheme, nullifier, scope }`, the user's product-scoped unique-human pseudonym (T8). `scope` MUST equal `product_id` |
| `host` | MUST | `{ host_id: <https-url>, jwks_uri }` |
| `rating` | MUST | The payload (T6) |

### T5.3 Signature construction

1. Serialize `body` with **JCS** (RFC 8785 JSON Canonicalization Scheme).
2. Each signing party produces a **detached JWS** (RFC 7515) over that serialization: signing input is `BASE64URL(UTF8(protected)) || '.' || BASE64URL(JCS(body))`.
3. The protected header MUST contain `alg: "EdDSA"`, `kid`, and `orp_role`; **`orp_role` MUST equal the signature's key** in the `signatures` object.
4. Each `signatures` entry is `{ "protected": "<b64url>", "signature": "<b64url>" }` under its role key.

```json
"signatures": {
  "host": { "protected": "<b64url>", "signature": "<b64url>" },
  "user": { "protected": "<b64url>", "signature": "<b64url>" }
}
```

Verifiers **MUST reject** a signature whose `orp_role` does not equal its key. Signing roles are **artifact-scoped**: receipts carry `host` / `user`; manifests carry `product` / `registry` (T2.5). **Products sign manifests, never receipts**, which is what makes "a product cannot veto a record about itself" structural rather than a normative promise (T6.2).

### T5.4 Identifiers and key discovery

- **Identifiers.** The `orp:` URI scheme namespaces `product` / `claim` / `receipt` / `template` (T7). Deliberately unregistered (the `did:` precedent); a standards-track version of the protocol would seek IANA provisional registration.
- **Key discovery via explicit `jwks_uri`.** The manifest or receipt carries the full HTTPS URL of the JWK Set; **no well-known path is mandated** (a mandated path adds nothing and breaks publishers without domain-root control). Host keys resolve via the host's JWK Set; **user keys are bound to the subject nullifier by the identity layer** (T8), not fetched by URL.

### T5.5 Key-discovery SSRF considerations

`jwks_uri` values are attacker-suppliable HTTPS URLs that operators and registries fetch: a classic SSRF vector into operator infrastructure. JWKS fetchers:

- SHOULD run **egress-isolated** from internal networks;
- MUST refuse URLs and redirect targets that are not HTTPS;
- MUST refuse resolution to **private, link-local, or loopback** ranges and **cloud-metadata** endpoints, **re-validating after every redirect and DNS resolution** (rebinding defense);
- MUST bound response size and timeout, and SHOULD cache the JWKS.

### T5.6 Test vectors

An implementation program MUST publish envelope **test vectors** (a fixed `body` and key set with the expected JCS byte sequence, signing inputs, and signatures) **before any external implementer builds**. The known interoperability hazard they exist to surface is **JCS number serialization** (RFC 8785's ECMAScript number-to-string rules). The first defense is the schema: a receipt body carries integers (scores 1 to 5), RFC 3339 strings, and opaque ids, and the receipt schema **forbids non-integer JSON numbers outright**, which removes the hazardous serialization cases; RFC 8785 itself recommends representing out-of-range numbers as JSON strings (§3.1). Manifest bodies MAY carry non-integer parameter numbers (T3.2), so the vectors remain the load-bearing check for manifests; implementations SHOULD verify byte-exact agreement against the vectors before producing receipts. `[PLACEHOLDER]` The vector set itself (concrete bodies, keys, expected bytes) is an implementation-program artifact that does not exist yet, and is not reproduced in this Overview.

---

## T6. Rating payload & outcomes

*Canon: Architecture §7.*

The rating receipt records **one elicitation event**: the closing of the loop a recommendation opened. There is exactly one receipt type in this version.

### T6.1 Payload shape

The `rating` payload (T5.2):

| Field | Req | Description |
|---|---|---|
| `matcher` | SHOULD | `{ embedding_model, version }`, the pinned model the recommendation's matching ran under; retained so distances stay third-party recomputable, though no distance appears on the wire (T4.6) |
| `scale` | MUST | `"orp-5"` in this version; applies to every `score` and `proposed_score` |
| `elicitation` | MUST | Elicitation record (T6.3) |
| `outcomes` | MUST | Non-empty array of per-claim scores (T6.4) |

### T6.2 Signatures: both always; scores-only

**Host and user signatures are both always REQUIRED.** A receipt exists only where the user confirmed at least one score. There is no host-only receipt, and scores are the only outcome kind.

Attestation semantics follow the parties, not a per-outcome kind:

- the **user** signature attests exactly the scores as displayed on the confirmation surface under the identified template (T6.3);
- the **host** signature attests the conduct of elicitation: timing, template identity, verbatim display.

**No product signature, structurally.** No receipt's validity references a product signature, so no product can veto, delay, or shape a record about itself; a low score exists regardless of the product's cooperation (T5.3).

**Fabrication bound.** Every receipt MUST reference a `recommendation_ref`, and the operator served every recommendation a receipt can reference (T4.8). A host cannot manufacture follow-ups for recommendations that never happened; per-host receipt counts against served-recommendation counts are an operator-side consistency check (T12). Under the federated end-state the same check runs per member: a member's receipt flow is verifiable by its peers against the recommendations it served (Architecture §13).

### T6.3 Elicitation record

| Field | Req | Description |
|---|---|---|
| `template_id` | MUST | `orp:template:{id}`, protocol-defined confirmation-surface template (T7) |
| `template_version` | MUST | Integer ≥ 1 |
| `locale` | MUST | BCP 47 tag of the rendered surface |
| `presented_at` | MUST | RFC 3339 timestamp of presentation to the user |

The host's signature over the body is its attestation that this template, in this version and locale, was displayed **verbatim**. The surrounding conversation is host discretion; the confirmation surface is not. These four fields are the load-bearing audit anchor: given a template registry, any auditor resolves `(template_id, template_version, locale)` to exact rendered text and checks per-host response-rate and edit-rate statistics (T12).

### T6.4 Outcomes: per-claim, scores-only

Each `outcomes` entry is the attribution record for one *(demand claim, product claim)* match:

| Field | Req | Description |
|---|---|---|
| `claim_id` | MUST | The claim this outcome refers to, the pointer into the registry. Lossless, because `canonical_text` is immutable (T1.4), so the wording can never drift from what was rated |
| `demand_claim_text` | MUST | The host-authored demand claim whose match produced this recommendation (T4.1), byte-identical to the text embedded at query time |
| `score` | MUST | Integer 1 to 5, user-confirmed |
| `proposed_score` | MAY | Integer 1 to 5: the host's pre-confirmation suggestion, if shown. On the wire so per-host anchoring statistics (edit rate, proposal-vs-final skew) are computable and non-repudiable (T7, T12) |

**Attribution is exactly `(claim_id, demand_claim_text)`.** The receipt carries **`claim_id` only as the product-side pointer; it does not also carry `product_claim_text`.** The immutable-text guarantee means `claim_id` losslessly resolves to the exact wording, so a legibility copy on every receipt buys nothing. Match distance is not in the entry either: attribution is exactly the pair above, and distance appears on no wire surface (T4.6).

**Scores are the only outcome kind.** The user either confirms a score or skips; **skips, silence, and interrupted elicitations produce no receipt at all**, because silence cannot be signed. Hosts SHOULD report those as off-receipt operator telemetry (asked-at, no-response) so funnel statistics stay complete (T12).

The array lists **rated claims only**. A claim the host never raised, or the user declined to rate, is omitted rather than marked; the full claim set of the recommendation is recoverable via `recommendation_ref`. There is **no user-supplied whole-product rating**: outcomes are per-claim, and any product-level aggregate is derived operator-side, stays internal, and is never published or queryable (T9). A summary number would be exactly the quotable global object the no-global-scores rule exists to prevent (Architecture §9).

---

## T7. Elicitation timing & conduct

*Canon: Architecture §7.*

Elicitation is how a receipt comes to exist. Its rules are settled; its content (template wording, longitudinal schedules) is deliberately open.

### T7.1 The follow-up loop

1. **Choice starts the clock.** When the user chooses a recommended product, the operator's recommendation record (identified by `recommendation_ref`) anchors the follow-up timer: the matched claim's `min_days` (T2.3).
2. **The host elicits after the delay, conversationally.** The host asks about the recommendation in natural language and MAY propose scores. It reads the claims at the `manifest_version` **pinned in the recommendation record**; a manifest update after the recommendation MUST NOT move the goalposts on a pending follow-up (T1.4).
3. **The confirmation surface closes the loop.** Scores the user confirms are recorded under the identified template; the receipt is issued and submitted to the log (T5, T6, T10). If the user confirms nothing, no receipt is issued and the host reports telemetry instead (T6.4).

Elicitation rides MCP's native elicitation primitive where a host supports it; that primitive is the natural hook for the confirmation surface.

### T7.2 Host-initiated only

Elicitation is host-initiated. A rating receipt **MUST NOT** be issued from a user's spontaneous request to rate a product before the eligibility delay has elapsed; the timer is part of the manipulation cost (T7.6). Spontaneous user commentary before the delay MAY inform the host's later proposal but cannot itself produce a receipt.

### T7.3 Conversational eligibility: the no-machine-gate rule

The eligibility bar is deliberately **low**: the user had access to the product and says they used it. The host judges the rest **conversationally** ("did you get around to trying X or feature B?"), and the human's own answer is the usage evidence. There is **no predicate language, no spec-defined usage-summary, and no machine evaluation of eligibility anywhere**. This is a settled, non-routable rule, and it rests on a structural argument:

> The product architects its own tool and event surface. **Any machine gate over usage events is therefore owned by the party it is meant to police**: a product could design its criteria and its event emission together so that predicted-high raters activate and predicted-low raters do not (the rater-selection attack). Statistical threshold-clustering detection catches the clumsy version, not the distribution-shifting version. Relocating the judgment to the human's own testimony puts it with the party the protocol already trusts to do the rating.

The no-machine-gate rule above is settled and non-routable. Conditional-text legibility is a separate registration requirement: registries MUST reject conditional text a human could not naturally confirm in conversation, for example telemetry- or event-count-keyed scoping (T3.1). Conditionals at matching time are machine-filtered by design (T3.4); the no-machine-gate rule is about eligibility to rate, a different surface.

**Feedback arrives in context, by solicitation:** the follow-up is a turn in a conversation the user was already having, closing a loop they opened by searching and choosing. This attacks both cold-outreach's dismal response rates and the furious-and-delighted selection bias of open review surfaces.

### T7.4 The confirmation surface and template registry

The **confirmation surface** is the load-bearing audit anchor (T6.3): protocol-defined templates, each identified by `template_id` and versioned, resolvable to exact rendered text so that per-host response-rate, edit-rate, and calibration statistics are computable (T12). The host presents the identified template **verbatim**; the surrounding conversation is discretion.

`[PLACEHOLDER]` **Template content, phrasing conventions for credence goods (outcome-phrased claims), and longitudinal or repeat-elicitation schedules are open:** an elicitation-design problem, not a wire problem. Everything the receipt must carry to make elicitation auditable is already fixed by the four elicitation-record fields; nothing in the open template-design space changes them. One constraint on that open space is fixed: the verbal anchoring of the rating levels is the primary informativeness lever, on live-platform randomized-trial evidence (Garg & Johari 2021), so template design MUST treat anchoring phrasing as the first-order decision, carried entirely in template text without changing the integer wire scale (T6.4).

### T7.5 Security considerations (initial, non-exhaustive)

| Surface | Attack | Standing defense |
|---|---|---|
| **Prompt injection at elicitation** | product tool output steers a later elicitation turn ("this user is delighted") | elicit in sessions without the product's context loaded; the human stays between model interpretation and signature |
| **Anchoring** | a proposed score anchors the human | `proposed_score` on the wire, so per-host edit-rate and skew is measurable; a host whose proposals are never edited and skew high is measurably miscalibrated (T12) |
| **Coached testimony** | product coaches users in-app ("tell your assistant you loved feature X") | statistical detection (phrasing similarity, timing clustering across a product's raters) in the subtract-only fraud layer (T10) |
| **Selective timing** | host promptly elicits its favored products' happy users, lets others lapse | per-host response-rate and follow-up-completion statistics (T12) |
| **Follow-up fabrication** | invented follow-ups | bounded by `recommendation_ref`: only operator-served recommendations are referenceable (T6.2) |
| **Shill economics** | without machine usage gates, a complicit rater's marginal cost per rating is lower | identity layer (one human, one nullifier, forever, T8), the host-initiated timer, and rater-credibility weighting (T9). This trade is deliberate |

### T7.6 Contact/acquisition events (deferred)

A future host-witnessed primitive attesting that the user "made the product available to themselves" (install, first tool call, purchase or transaction receipt) has **no wire format in this version**. Its semantics are settled: **supplementary credibility evidence, a bonus in scoring, never a gate on admissibility.** Committed to the chain before a rating exists, it strictly strengthens that rating (temporal precommitment: contact history cannot have been fabricated at rating time). The rule is settled; the mechanism waits on integrations that do not yet exist, and the wire format stays unspecified until the underlying transaction-receipt formats exist to align with.

---

## T8. Identity & privacy

*Canon: Architecture §8.*

Identity is the scarce resource the whole accountability layer rests on: a rating is worth something only because a real human, counted once, produced it. This is where the novelty budget is genuinely spent (Architecture §14), alongside receipts and incentives.

### T8.1 Two-tier scope

Identity is two-tier, and the tiers have opposite visibility:

- **Root, protocol-scoped, never on the wire.** One unique-human identity **per human**, a **zero-knowledge proof-of-personhood nullifier** derived with a protocol-level scope shared by every operator instantiation (Architecture §13, D27), so the same human resolves to the same root at every member: the operator learns only that a unique human exists and which receipts are theirs, nothing else. **No name, contact data, or demographics exist anywhere in the system**; the root is unresolvable to a person by the operator, an auditor, or a subpoena. The root and everything derived from it (the identity graph, per-rater credibility histories) are **operator-only closed data** (T10 layer 4), used solely for fraud prevention and credibility weighting.
- **Wire pseudonym, product-scoped, what receipts carry.** `subject.nullifier` is a **per-product pseudonym derived from the root under operator-held salt**. This projection is **MANDATORY**: cross-product identifiers MUST NOT appear on receipts, so receipts are **unlinkable across products** for products, hosts, auditors outside gated access, and every third party. Linkage exists only at the operator, as closed audited data. **Unlinkability is a wire property, not an operator property.**

`subject.scheme` names the identity scheme and is deliberately **vendor-agnostic** (an open string). Proof-of-personhood via ZK nullifiers is the designed mechanism; a specific provider (for example a WorldID-class implementation) is one candidate, never a dependency.

### T8.2 Why the root is not product-scoped

The root is **held at the operator's closed layer, never product-scoped**. Product-scoping the root (cryptographic unlinkability against the operator itself) would destroy most of the fraud layer: cross-product credibility weighting, ring detection, velocity anomalies, and rater-level ban enforcement, leaving only within-product ballot-stuffing resistance. The operator-scoped split keeps both properties that matter: **no host integration and no future data leak can ever yield per-human rating histories** (the wire is unlinkable), and the operator retains the closed cross-product view the fraud layer needs.

**Privacy as dividend, not compromise:** raters stay fully anonymous and there is no consumer-data product. **Distribution is segmentation**: differentiated claims, and the conditionals surfaced on them, already shape who gets matched and what they are told (T3.1), so companies need no demographics to reach the right population.

### T8.3 Postponable at a cost

Proof-of-personhood is **postponable at a cost, with the minimal substitute named explicitly**: an early deployment MAY substitute minimal LLM-host account verification, **one root per host account instead of one per verified human**. The wire shape is unchanged: the same two-tier `subject` structure, host-vouched instead of ZK-rooted.

- The host submits a stable, host-salted reference for its account holder to the operator (submission metadata, never on receipts).
- The wire `subject` carries a product-scoped derivation of it.
- The user signature is custodial: the host holds the signing key, and the signature attests that the confirmation event occurred, on the host's word.

The cost is stated wherever it applies and never hidden: **such receipts are host-vouched, not human-verified**, and the Sybil floor drops from "one human, one root, forever" to whatever the host's account-fraud defenses are worth. Concretely, the three-point Sybil exposure is:

1. one human can hold multiple accounts on one host (multi-account Sybil, bounded only by the host's account policy);
2. the same human on two hosts is two subjects (cross-host duplication, undetectable until proof-of-personhood nullifiers);
3. a malicious or compromised host can fabricate subjects wholesale (bounded by `recommendation_ref` consistency, per-host statistics, and host standing, T6.2, T7.5, not by identity).

That dropped Sybil floor is exactly the strength by which the open-math anti-Goodhart guarantee for ratings weakens under the substitution (T10.6), which is why the substitution is temporary.

### T8.4 GDPR posture, stated plainly

Operator-side pseudonymous linkage **is personal data**, asserted with no pretense of exemption. Obligations are bounded and architecture-backed:

- **legal basis:** legitimate interest, for fraud prevention;
- **data-subject requests:** authenticated by **proof of nullifier control** (there is no name to match against);
- **erasure:** executed against the **raw store**; the commitment chain holds only digests (T10), so erasure never breaks the chain, and because all recomputation lands at epoch boundaries (T10), erasure timing never leaks through a published value.

Minimization-by-architecture is the backbone of the posture: the linkage never leaves the operator's closed layer, and no BI or query surface ever exposes it. The GDPR-dividend rulings (no demographics, no consumer-data product) are untouched. `[PLACEHOLDER]` The exact binding of user-held signing keys to nullifiers and the detailed DSAR procedure are open items.

---

## T9. Scoring & incentives

*Canon: Architecture §9.*

Scoring is deliberately austere. The design's stance is that most "sophistication" in a reputation score is where the gaming industry grows (the FICO cautionary tale, Architecture §14); ORP keeps the arithmetic minimal and pushes robustness into scarcity (identity, real usage, calendar time, T8, T10) instead of into secret formulas.

### T9.1 Delivery per claim is the whole score

A claim's rating **is what its raters gave it**, subject to only two forces:

- **Recency weighting**: recent ratings outweigh older ones, so standing tracks the most recent deliveries, adjusts at the speed evidence arrives, and there is no decades-long cash-cow milking; and
- **the rating floor**: a claim publishes nothing about itself until it clears a minimum quantity of **distinct raters** (T9.2).

There is **no time-based decay, no ambition term, and no incumbent debuff**; recency weighting, the stale label (T9.2), and the floor already force brand trust to be re-earned. Crucially, **claim scope is the company's discoverability lever, not a scoring input**: a more desirable claim buys more exposure and must then be delivered on, but the claim's semantics never enter the arithmetic, only the ratings do. This is what keeps claim-engine-optimization pointless (T4.5): you cannot phrase your way to a better score, only deliver your way there.

One tuning commitment is elevated above the other knobs: **a claim rated badly enough MUST be worth less to its product than not covering that demand claim at all**, in competitive segments. No dedicated penalty term is required: with unrated claims entering the composite at the default prior (T9.7) and each uncovered demand claim entering as a coverage malus (T4.4), the crossover emerges from the aggregation; a claim rated sufficiently below the prior is net negative against the malus of simply missing the coverage. The prior, the malus, and hence where the crossover lands are open (T9.7, T13.1); the commitment that the crossover exists and bites in competitive segments is not (T13.2). The prior height and the archive-penalty escalation are coupled knobs: a product whose true quality sits below the prior prefers staying unrated, and its reset path, archive-and-reregister back to the prior, is what the escalating schedule prices out (T9.6).

### T9.2 The rating floor (distinct raters, two minimums)

The floor is a **lifetime minimum of distinct raters** a claim must clear before anything about it publishes.

Two properties are load-bearing:

- **The floor counts distinct raters (nullifiers), never rating events** (T8): one rater rating five times unlocks nothing.
- **It is one instrument doing three jobs**: the scoring job (below the floor, no composite), the privacy job (epoch de-anonymization defense, T10), and the spray auto-balance job (paraphrased claims each starve of the floor, T1.5). "Distinct-rater" is a criterion in its name, not three separate mechanisms.

Below the floor, **absence of a rating plus the `sample_tier` = `unrated` signal is the honest output**: the protocol says "not enough data", never a fabricated number.

**Inactivity is labeled rather than erased**: a claim whose ratings stop arriving keeps its last earned value and its `sample_tier` moves to `stale` (T4.10). Staleness is a label transition, not a recomputation; the value carries forward under the epoch discipline (T10.3), moves only when the ratings behind it change, and is shed only by new ratings or by the archive (T9.6). The staleness window scales with a claim's rating activity, so a low-traffic niche claim is not flapped to stale by a window tuned for high-traffic claims (T9.7).

### T9.3 Portfolio aggregation: internal only

Aggregation runs **per-claim → product → company**. The product and company aggregates exist **strictly internally** and serve exactly two purposes: **recommendation-time trust multipliers** (T4.4) and the **claim-spam penalty** ("you can't hide 995 bad claims behind 5 good ones", Architecture §9). Their value is **never stated on any surface, to anyone, at any granularity**. Publishing them would invite witch hunts and brand-versus-brand competition outside the system (marketing through the back door) and would misdescribe products: a 3.5-average product whose five claims a given consumer needs all sit at 4.4+ is excellent for that consumer. Leveraging exactly that differentiation is the point of the protocol; fraud is already handled inside recommendation and needs no audience.

`[PLACEHOLDER]` The aggregation form at each level (how per-claim composites roll up, how spray drags the product aggregate) is an open scoring-semantics question, not fixed here.

### T9.4 No mystery penalties, and the deliberate exception

Everything the **published per-claim ratings** hold against a company is surfaced to that company: the remedy is legible ("remove or change the weak claims"). But the promise is scoped to **scoring outcomes only**. **Conduct penalties sit outside it deliberately**, with one legible exception: the archive penalty follows a published schedule, because archiving is a public act (T9.6). Fraud-layer trust-multiplier movements and fraud-layer actions are not disclosed, because telling a bad actor what tripped a detector is telling it how to pace around one (T10). This is the one place the transparency principle is deliberately narrowed, and it is safe precisely because those actions are subtract-only and logged (T10.5): secrecy can remove fraud, never add score.

### T9.5 Rater-credibility weighting is fraud-only

Rater credibility is keyed **exclusively on behavioral fraud signals** (velocity, coordination, ring structure) and **never on rating valence**. The success criterion is a principle, not a formula:

> **An honest 1-star rater carries exactly the same weight as an honest 5-star rater. Dissent is never expensive.**

This is the boundary between eligibility and admissibility: eligibility qualifies a user to be asked (T7.3); **admissibility** is the protocol layer's own discretion over whether an answer counts, conduct rules (eligibility delay respected, template version current, per-host standing) plus fraud-only credibility weighting. Detector mechanics are fraud-layer internals and out of scope like most fraud measures; the principle is what belongs in the paper.

### T9.6 Clean reputation and experimentation economics

- **No inherited-trust bonus; the trust multiplier is penalty-only.** The product/company trust multiplier is clean reputation, never positive reputation: neutral is 1x, moving downward on conduct penalties and regenerating slowly toward neutral, never above it, so it can demote a bad actor but never lift a product above a clean-record peer. An unproven new claim starts unrated no matter who holds it, and earns exposure only through coverage and the ratings that subsequently arrive. A startup and an incumbent stand equal on a genuinely new claim; an incumbent's advantage is confined to the claims it has already delivered on and been rated for (the good part of brands, trust earned by delivering, never bought), and the newcomer's path is differentiation into unserved demand (T4.7).
- **Experimentation is cheap and self-policing.** Good claims drift to `stale` unless re-confirmed; spray and spam penalties are a secondary, mild backstop, because spray **auto-balances** (T1.5): it splits feedback across `claim_id`s and starves each of the floor. Since claims can't be deleted and archiving is priced (below), discoverability is only ever earned by differentiated claims matching real demand.
- **Archiving is the priced reset.** Archiving stops a claim from matching and from counting in scoring, leaves its history public forever (T1.4), and costs a trust penalty on a **published schedule**: gentle at first, escalating when archival repeats while trust is still regenerating, higher for claims that carried negative ratings. Honest experiments cost almost nothing, buying out a bad record costs real standing, and registration-and-archival churn destroys the trust it feeds on. The schedule is published because archiving is a public act, the one legible conduct penalty in contrast to the undisclosed fraud layer (T9.4).

### T9.7 Open knobs `[PLACEHOLDER]`

Every scoring knob is an **open range**, never fabricated here (consolidated in T13):

> default prior · recency-weight profile · staleness window and its activity scaling · archive-penalty schedule (base, escalation while trust regenerates, trust-regeneration rate) · stale/unrated-claim drag strength · rating-floor level (lifetime).

These are design, measurement, and tuning questions for specialists and operator experimentation; none is a protocol deal-breaker (Architecture §9, §16).

---

## T10. Transparency model

*Canon: Architecture §10, §13, §17.*

The guiding principle: **as transparent as possible without sabotaging the mission.** Trust for adoption comes from structure (the governance lock, audits, inclusion proofs), not from published numbers. "Public" means the structure and verifiability of the three graphs; row-level data follows the residency layers below.

### T10.1 Append-only commitment chain (CT model)

Receipt digests are committed to an **append-only public commitment chain** on the Certificate Transparency model: gated raw data is continuously verifiable against public commitments, and inclusion proofs let anyone check that a given receipt is in the log. The chain holds **digests only**, never receipt contents, which is what lets GDPR erasure run against the raw store without breaking the chain (T8.4). Under the federated end-state every operator instantiation appends to and verifies this one chain, which supplies the split-view monitoring ecosystem the CT model otherwise assumes (Architecture §13); append ordering and conflict handling between members are an open federation item (T13). `[PLACEHOLDER]` **The chain append batch cadence** is an operator parameter: digests attribute nothing, but fine-grained receipt-arrival timing correlated with an epoch jump is correlation material a batch cadence cheaply removes. Archive-never-delete plus permanent receipts on an append-only chain is an unboundedly growing structure; production key-transparency deployments identify log compaction as an operational necessity at scale (Parakeet, NDSS 2023). A compaction or checkpointing story that preserves inclusion-proof verifiability is an open build item (T13).

### T10.2 Four data-residency layers

Open to closed. **Four layers; the operator sells nothing:**

| Layer | Access | Contents |
|---|---|---|
| **1. Public** | no auth | spec, schemas, scoring semantics + reference impl, commitment chain, protocol-health aggregates, **per-host calibration** (T12), the **demand-gap commons** (T11) |
| **2. Query API** | authenticated, uniform free license | per-claim canonical text and **epoch-published per-claim ratings**, never product- or company-level (T9.3). Supply counts are not a standing surface: eligible and full-coverage counts return with a discovery response and nowhere else (T4.9) |
| **3. Own-numbers** | the owning company, free | its own epoch-published numbers, **no fresher feed than anyone else gets** |
| **4. Operator-only** | NDA-auditable | raw receipts, submitted demand-claim texts, identity graph, fraud layer |

There is no company-scoped diagnostics tier: the operator sells nothing to anyone. A company gets its own numbers free at layer 3, and the demand commons (the demand-supply heat map, T11) is addressed to the market, not to any company. Per-host calibration (T12) is the only per-host surface, and it names hosts, not products.

### T10.3 Epoch publication

Every rating-derived value on any query or publication surface updates **only at epoch boundaries**. The rule is normative, not operator policy: even the company's own-numbers tier (layer 3), the sharpest adversary (jump-time correlation against its own user sessions), gets no fresher feed. The boundary is also where the operator's instantiations sync state and recompute over the merged receipt set (Architecture §13), so publication stays joint and no member is a fresher feed than another. Boundary semantics:

- **Conjunctive:** a value recomputes only if **both** a minimum period **P** has elapsed **and** at least **k** new admissible ratings have accumulated since the last update; otherwise it carries forward unchanged. A conjunctive rule is necessary: a pure fixed-period rule leaks at low volume (a period containing one rating is that rating), and a pure threshold rule has no cadence floor.
- **The distinct-rater floor** (T9.2) gates publication: below the floor, no composite at all.
- **Memoryless query surface:** current values only, no history endpoint, no per-epoch deltas, no n-increments. Time-series assembly requires per-epoch polling.
- **All recomputation causes land at boundaries:** new ratings, staleness transitions, quarantine, and erasure all take effect only at recomputation, so a published change is never attributable to a cause class (quarantine and erasure timing never leak).
- **No calibrated noise.** DP-style jitter is formally stronger but is deliberately traded away: **published numbers in a trust protocol must be true.** Epoch plus conjunctive boundaries plus distinct-rater floors plus one-decimal rounding (T4.10) deliver k-anonymity-grade protection: a patient per-epoch observer learns batch-level aggregates but cannot isolate an individual rating below the floor.
- **Banded value with monotone hysteresis:** the published per-claim `rating` is a banded value: it republishes only when the newly recomputed value differs from the published one by more than a stated width, and carries forward otherwise. This coarsens cross-epoch deltas on the same logic as `sample_tier` (T4.10) while keeping every published number true. The selection composite consumes the epoch-published banded values, never the fresher internal composite, so response ordering reveals nothing about a rating that the published surface does not.

`[PLACEHOLDER]` **P, k, the distinct-rater floor level, the rating band width and hysteresis rule, and the chain batch cadence** are test-determined, published, and versioned (T13).

### T10.4 The hard permanent rules and leak-tolerance

- **No published rankings, leaderboards, browse or bulk surface, or quotable benchmark, ever.** This is permanent, not a tuning parameter.
- **Leak-tolerance premise:** whatever crosses the query API is public against a determined adversary. The defense is not technical prevention but **no unauthenticated access + a uniform license + the EU database right + key revocation + staleness** (epoch). The license is an **integrity instrument, not revenue**: it exists to keep the surface honest and attributable, not to sell data.

### T10.5 The one secret layer: subtract-only fraud

The behavioral fraud and quarantine detectors are the **single** component kept secret: heuristics over conduct are the one thing knowledge genuinely defeats (rings pacing under a known velocity threshold). The containment is structural:

> **Detectors are secret; actions never are. Every quarantine is subtract-only and carries a logged justification.** Secrecy can remove fraud from a score, never add score, so a secret layer can never move a score upward, and an auditor can verify that every quarantine carries a recorded justification and sample-test decisions without learning detector internals.

`[PLACEHOLDER]` Under the federated end-state peer instantiations supply the audit by construction, cross-auditing one another's quarantine patterns for target-selection bias (Architecture §13). The external audit instrument for the pre-federation state (how a non-member auditor samples and verifies) is undesigned: a success criterion, not a mechanism (Architecture §16).

### T10.6 Why open math doesn't Goodhart

This is the argument the transparency model rests on (Architecture §10, §17), summarized per surface because it is the sharpest objection the design faces (the FICO precedent, Architecture §14):

| Surface | Why open math is safe |
|---|---|
| **Coverage** (dominant ranking factor) | gaming it means authoring more accurate claims in the functional register; the Goodhart and the desired behavior are the same action. **Self-aligned.** |
| **Ratings** | producing one needs a verified human, a served recommendation, an accepted choice, and a later in-conversation confirmation: scarce real-world inputs (humans, usage, calendar time) that open math does not unlock. Brute force means buying humans: expensive, self-depleting, lands in the fraud layer. **Scarcity-gated.** (This is exactly the guarantee that weakens under host-account substitution, T8.3.) |
| **Trust multipliers & recency weighting** | gamed only by continuously delivering so recent ratings stay good. **Aligned.** |
| **Matching geometry** | the one brute-forceable surface, but there is no payoff: in-radius membership is binary, no cutoff exists, position buys nothing, and demand claims are host-authored independently of user vocabulary (T4). The residual (genuine embedding-model defects) is found by brute force regardless of secrecy, so publishing the model just levels access; the fix is versioned model upgrades, and spray around defects auto-balances (T1.5). |
| **Epoch / floor rules** | knowing them is useless: the memoryless surface and the floor are what make update-timing knowledge worthless for de-anonymization. |
| **Behavioral fraud detectors** | the one place secrecy works, and structurally subtract-only, so it can never corrupt a score upward. |

The pattern: **every open surface is either self-aligned or gated by scarce real-world resources; secrecy is reserved for the single layer where it actually works, and that layer cannot move a score up.**

---

## T11. Demand commons

*Canon: Architecture §11.*

The demand commons is the primary public good the protocol produces: a live, continuously updating map of market demand and supply saturation, addressed to everyone and owned by no one. It is not a per-company service, and under the non-profit mission nothing about it is operator discretion: publishing the gap and saturation map is a mandatory output of running the protocol with a neutral operator.

### T11.1 Search-exhaust, not rating-exhaust

Gap data is derived from the operator's **query logs**, the demand-claim texts hosts submit at matching time (T4), never from receipts. It **names no products**: it is an aggregate over demand, not over supply. Only demand claims carrying a **verified-human subject** contribute (T4.3): the commons aggregates verified-human demand, which makes fabricating phantom gaps a personhood-gated cost rather than a free write, the same Sybil-cost logic that protects ratings. The recommendation service itself stays ungated on personhood; a colluding host with real users can still tilt the map, bounded by per-host demand-claim statistics. Two consequences follow:

- **It is decoupled from the elicitation-rate risk.** The commons exists even if nobody ever rates anything, which makes it the one payoff that survives the design's largest single risk (cold start, T7). A deployment that never achieves rating volume still delivers a live map of unmet demand.
- **It beats stated-preference polling on the two axes that matter.** It is revealed rather than stated, so it does not ask people to predict their own behavior, and it updates continuously rather than per study.

### T11.2 What is published

Unmatched and poorly-matched queries are revealed demand-gap data: over- and under-saturated regions of claim space. The published object is the **demand-supply embedding heat map**, an aggregate over where demand claims land relative to registered supply. It is published at residency layer 1 (T10.2), on the same **epoch-boundary discipline** as every other rating- or query-derived surface (T10.3), so nothing about it becomes a fresher or finer feed than the transparency model allows.

`[PLACEHOLDER]` The concrete aggregation granularity of the heat map (how demand-claim embeddings are bucketed, what saturation metric is published, publication cadence) is a measurement-design question, not a wire question, and is left open. Nothing here changes any wire field.

### T11.3 Intent registration `[PROPOSAL]`

A coordination construct sits on top of the commons as a candidate, not a specification: companies MAY signal internal R&D targeting a published gap ("being built"), with use-it-or-lose-it expiry, and demand pools MAY pre-commit ("notify-when-it-exists"). There is **no monetary stake and no escrow anywhere**. Expiry semantics and pool mechanics are `[PROPOSAL]`-grade: the construct demonstrates the coordination payoff is reachable, and no more.

### T11.4 Conflict-of-interest exclusion

Nobody with ownership of a cataloged product may be part of the non-profit operator. This is a structural-incorruptibility rule that protects the commons at its source: the party publishing the map of the market has no product in the market.

---

## T12. Host calibration

*Canon: Architecture §12.*

Per-host calibration is the protocol's answer to a structural gap: **the operator cannot audit host translation honesty from its own data**, because the user's raw need never reaches it (T4.2). Calibration is the external audit that replaces the one the operator structurally cannot perform, and the single public surface on which **hosts are named** (products never are).

### T12.1 The benchmark measures consistency, not quality

The benchmark measures **consistency, not taste**: similar natural-language needs should yield **similar demand claims and similar recommendations**, both **across hosts** (a host's translations should not diverge systematically from its peers') and **within one host over time** (a host should not drift). It deliberately does not judge whether a host's demand claims are "good": there is no ground-truth translation to grade against, only whether they are stable and peer-consistent.

### T12.2 What it makes visible

Its primary job is **making paid-placement steering visible**. A host quietly steering queries or recommendations toward products that paid it will **diverge from its peers and from its own past**; the divergence is the signal, regardless of how the steering is implemented. Conversely, an adopting host can **point at its calibration number as proof it isn't steering**: the measurable form of "the ad-free pledge gets its proof mechanism" that anchors the adoption thesis. It is the measurable form of "hosts are their own audit surface" (T4.2).

### T12.3 How consistency is measured `[PROPOSAL]`

The canon leaves the how open. A best-fitting, proposal-grade construction:

- **A probe set** of natural-language needs (spanning domains and paraphrase families) is fed to participating hosts, a mystery-shopping harness distinct from real user traffic.
- **Cross-sectional consistency:** for each probe, compare a host's emitted demand claims and its resulting recommendation set against the peer distribution. Divergence metrics are computable in the shared embedding space for demand claims (they are recomputable from text, T4.2) and as set-overlap or rank-agreement over the returned products.
- **Longitudinal consistency:** the same probes re-run over epochs (T10.3) detect within-host drift; a step change concentrated on products with a known commercial relationship is the steering signal.
- **Published as an epoch-boundary, per-host aggregate** at layer 1 (T10.2), hosts named, never per-product, never exposing the probe-level detail that would let a host overfit the harness.

`[PLACEHOLDER]` The concrete divergence metric, probe-set construction and rotation (to resist teaching-to-the-test), significance thresholds, and publication cadence are all open, deliberately, since freezing them early would hand a steering host a fixed target to game. This is a measurement-design problem for specialists, not a wire problem; nothing here changes any wire field.

---

## T13. Parameters as open ranges

*Canon: Architecture §9, §16.*

Every tunable in the protocol is consolidated here as a **flagged open range, never a specification.** This is deliberate and load-bearing: the canon's discipline is that the paper defines success and does not owe every mechanism (§16), and the drafting instruction is to minimize needless attack surface by biasing to proposal-grade placeholders over exhaustive specification. Fixing these numbers here would invent settled facts the design does not have, and hand adversaries a fixed target. Each is a design, measurement, or tuning question for specialists and operator experimentation; **none is a protocol deal-breaker.**

### T13.1 Tunable register

| Parameter | Governs | Open range / constraint | Section |
|---|---|---|---|
| **K** (top-K size) | discovery response breadth | small, order 3 to 10, test-determined | T4.4, T4.8 |
| **max match radius** | claim in-radius membership | test-determined; binary membership, no cutoff | T4.4 |
| **synonym link layer** | equivalence admission and link-table publication | `[PROPOSAL]` open; link table versioned and published for recomputability | T4.11 |
| **eligible-count threshold** | `supply.eligible` | absolute or proportional, **open**; a tuning param, not a wire semantic | T4.9 |
| **`sample_tier` bounds** | few / moderate / many cutoffs | published, versioned, test-determined; exact n never on the wire | T4.10 |
| **own-claim tiebreak** | which own claim represents a product | **lowest-rated** (ratified, D26.7); tie-of-lows broken deterministically (for example oldest `claim_id`) | T4.5 |
| **`min_days`** | earliest follow-up | per-claim 0 to 365; a tuning knob, not load-bearing | T7.1 |
| **epoch period P** | publication cadence floor | conjunctive with k; test-determined | T10.3 |
| **epoch ratings k** | publication volume floor | conjunctive with P; test-determined | T10.3 |
| **distinct-rater floor** (lifetime) | when a claim publishes at all | single lifetime minimum; counts nullifiers, not events | T9.2, T10.3 |
| **rating band width / hysteresis** | published-value coarseness across epochs | test-determined, published, versioned | T10.3 |
| **chain batch cadence** | commitment-chain append timing | operator param; batched to remove arrival-timing correlation | T10.1 |
| **log compaction / checkpointing** | chain growth management | open build item; must preserve inclusion-proof verifiability | T10.1 |
| **selection weights** | coverage vs rating vs trust multiplier | coverage dominates non-lexicographically; weights irreducibly preference-laden | T4.4 |
| **default prior** | unrated-claim starting point | open scoring knob | T9.7 |
| **stale/unrated-claim drag** | spray penalty strength | secondary, mild (spray auto-balances) | T9.6, T9.7 |
| **below-prior crossover** | rating level at which holding a claim is worse than not covering the demand | emergent from default prior + coverage malus + aggregation form, no dedicated term; the commitment that it exists is fixed | T9.1, T4.4 |
| **recency-weight profile** | how strongly recent ratings outweigh old | open | T9.1, T9.7 |
| **staleness window** | when a claim's band moves to stale | scales with rating activity; open | T9.2, T9.7 |
| **archive-penalty schedule** | priced-reset cost and trust regeneration | base, escalation while trust regenerates, regeneration rate; open | T9.6, T9.7 |
| **demand-commons heat-map granularity** | gap/saturation aggregation | open measurement-design question | T11.2 |
| **calibration metric & thresholds** | per-host consistency benchmark | `[PROPOSAL]` divergence in embedding + set space; probe set rotated to resist gaming | T12.3 |
| **federation member count** | operator instantiations at end-state | likely 3 or 5 (DNS-root precedent); launch at 1 (D27) | Architecture §13 |
| **member sync protocol** | state merge, append ordering, conflict handling between members | open federation item, post-paper (with the conformance criteria) | T10.1, T10.3 |

### T13.2 What is not open

For contrast, and so the open ranges above are not mistaken for the design being unsettled, the following are **fixed** by canon and are not tuning questions: text canonicality (T1.1); claim immutability and archive-never-delete (T1.4); flat product-bound claims (T1.2); conditionals as pure filters with no `required` flag, never gating matching on query completeness (T3.1); both signatures always, scores-only receipts (T6.2); the no-machine-gate eligibility rule (T7.3); two-tier identity with mandatory product-scoped wire pseudonyms (T8.1); distance on no wire surface (T4.6); no published product or company aggregates and no published rankings ever (T9.3, T10.4); four residency layers and the operator sells nothing (T10.2); subtract-only fraud (T10.5); no calibrated noise (T10.3); the below-prior commitment, that a badly rated claim is worth less than not covering the demand at all (T9.1). Tuning the numbers never changes these; changing them is a document revision (T2.4).

---

## Appendix A: Decision flags surfaced in this draft

For traceability. Nothing below blocks the white paper; each is either proposal-grade or an illustrative placeholder.

- **Ratified (D26.7): own-claim tiebreak (T4.5).** Lowest-rated in-radius claim represents the product. The tiebreak among several claims sharing the low rating remains proposal-grade: deterministic (oldest `claim_id`), flagged.
- **`[PLACEHOLDER]`: eligible-count threshold (T4.9).** Left as an open range (absolute vs proportional): a tuning detail, not a design fork.
- **Recorded, not open: receipt attribution (T6.4).** `claim_id` only; no `product_claim_text` legibility copy.
- **`[PROPOSAL]`: intent registration (T11.3).** Coordination construct on the commons; expiry and pool semantics proposal-grade, no escrow.
- **`[PROPOSAL]`: synonym linking (T4.11).** The layer's existence is canon; admission, link-table representation, and migration are proposal-grade.
- **`[PLACEHOLDER]`: authoring references and synthetic matching evaluation (T4.12, T4.13).** Reserved anchors for the authoring contract's SDK references and the pre-receipt evaluation harness; both artifacts live outside this Overview (Architecture §16).
- **`[PLACEHOLDER]`: rating band width and hysteresis rule (T10.3).** The banded-publication rule is settled; its width and rule are test-determined.
- **`[PROPOSAL]`: per-host calibration measurement (T12.3)** and **`[PLACEHOLDER]`: all scoring, epoch, and heat-map knobs (T13.1).** Canon-silent tuning surfaces; proposed at shape level, numbers left open.
- **`[ASSUMPTION]`: canonicalizer as a pinned LLM stage (T1.3).** Determinism is a quality concern; injection resistance is a trust dependency, because the canonicalizer's adversary is the submitting company and human approval is no defense (D28). The defense is architectural, never prompt-level, and the operator MAY sample-check registered claim texts.
- **Catalog keys throughout (T3).** Every `key` named is `[PLACEHOLDER]`: illustrative, never a reservation; the catalog is the authoring contract's artifact (Architecture §16).

*End of the ORP (Open Receipt Protocol) Technical Overview, sections T1 to T13. Companion to the Architecture canon; the engineering-depth counterpart to Architecture §§5 to 12.*
