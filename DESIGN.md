# Design

<p align="center">
  <strong>English</strong> &middot;
  <a href="DESIGN.ko.md">한국어</a>
</p>

This document describes how the detector works: its detection model, AMM-replay
enrichment, evidence/confidence taxonomy, and validation methodology. The
[README](README.md) covers install, CLI usage, and the Vigil integration
contract; this is for readers who want to know *why* a given detection fires
(or doesn't) and what guarantees the numbers carry.

## 1. Detection model

A sandwich is three swaps in two participants in two roles. The detector reduces
this to a structural pattern over a stream of `SwapEvent`s, then layers
precision filters and economic enrichment on top.

### 1.1 Same-block detection

```
Block (slot N)
 |
 +-- Parse transactions  -->  Extract swap events per DEX
 |                            (instruction discriminator + token balance deltas)
 |
 +-- Group by pool
 |
 +-- Detect pattern:
       tx[i]  attacker BUYS   -+
       tx[j]  victim   BUYS    +-- same pool, same direction
       tx[k]  attacker SELLS  -+   opposite direction, same signer as tx[i]
```

Implementation: `crates/detector-sameblock/src/detector.rs`. Operates on a
single block's swap events; no cross-slot state.

### 1.2 Cross-slot window detection

```
Slot N:    attacker BUYS in pool P     (frontrun)
Slot N+1:  victim BUYS in pool P       (same direction as frontrun)
Slot N+2:  attacker SELLS in pool P    (backrun -- opposite direction)
```

`FilteredWindowDetector` (`crates/detector-window/src/pipeline.rs`) maintains
a per-pool sliding window of `W` slots. Within-window candidate triplets are
passed through three precision filters before emission:

1. **Bundle provenance** — Jito bundle co-location, ranked
   `AtomicBundle > SpanningBundle > TipRace > Organic`. Same-bundle frontrun
   and backrun mean the attacker paid for atomicity, the strongest provenance
   signal we can read off-chain.
2. **Economic feasibility** — `backrun.amount_out − frontrun.amount_in > tx fees`.
   Loose floor; tightened by AMM-replay enrichment downstream.
3. **Victim plausibility** — size-ratio check against the frontrun (a 1-lamport
   "victim" sandwiched between two large legs is almost certainly a routing
   artifact, not a real sandwich) plus a known-attacker exclusion list for
   wallets verified as MM/arb bots.

Each emitted detection carries a `confidence` ∈ [0, 1] composite of the three
filters' outputs; the gate threshold lives in the pipeline.

### 1.3 Detection rules (both detectors)

| Condition | Why |
|-----------|-----|
| `frontrun.signer == backrun.signer` | Same attacker |
| `frontrun.direction != backrun.direction` | Opposite trades (buy then sell) |
| `victim.direction == frontrun.direction` | Victim pushes price further |
| `victim.signer != attacker` | Different wallet |
| Same pool | Price impact is local to the pool |
| Same slot (same-block) or within W slots (window) | Temporal proximity |

### 1.4 Hybrid parsing

DEX swap events are extracted using a hybrid strategy: instruction
discriminators identify the program and pool address, while pre/post token
balance changes determine swap direction and amounts. The discriminator side
is brittle (Anchor IDL changes), the balance-delta side is brittle (multi-hop
or routed swaps muddle the net delta), so each parser uses both as a
cross-check. See `crates/swap-events/src/dex/`.

## 2. AMM-replay enrichment

Rule-based detection answers *"was this a sandwich?"* but not *"how much did
the victim lose?"* — the naive `backrun.amount_out − frontrun.amount_in`
heuristic ignores price-impact dynamics and breaks for multi-token pairs. The
`pool-state` crate fixes both by replaying each swap through the AMM's own
math:

```
state_0   = pool reserves just before the frontrun  (from tx meta)
state_1   = state_0 after frontrun                  (replay)
state_2   = state_1 after victim                    (replay, "actual")
state_2c  = state_0 after victim, no frontrun       (replay, "counterfactual")
victim_loss  = victim_out(state_2c) − victim_out(state_2)
```

The counterfactual replay (`state_2c`) is what makes the loss number AMM-correct:
it's the output the victim *would have received* in a world where the frontrun
didn't happen, computed from the same pool math the program itself runs.

### 2.1 Per-DEX reserve sources

| AMM family | Reserve source | Replay primitive |
|---|---|---|
| **Constant product** (Raydium V4 / CPMM) | `pre_token_balances` / `post_token_balances` from tx meta | `x * y = k` |
| **Concentrated liquidity** (Whirlpool, Raydium CLMM) | `getAccountInfo` on the pool account for the dynamic snapshot (`sqrt_price`, `liquidity`, current `tick`); tick-array PDAs derived | sqrt-price step + cross-tick walk via the V3 TickArray |
| **Discretised liquidity** (Meteora DLMM) | Pool account + bin-array PDAs for active and adjacent bins | Constant-sum within a bin + cross-bin walk via the bitmap |
| **Bonding curve** (Pump.fun) | `virtual_sol_reserves` / `virtual_token_reserves` recovered from the `TradeEvent` Anchor log Pump.fun emits per swap | Bonding-curve formula |
| **Aggregator** (Jupiter V6) | Resolves the underlying DEX from the route, dispatches to that DEX's replay path | Single-hop only at v1.0; multi-hop deferred |

The constant-product and Pump.fun paths need no extra RPC. V3 / DLMM paths need
`getAccountInfo` on the pool account; the
[`AccountFetcher`](crates/pool-state/src/rpc.rs) trait abstracts the fetch so
archival providers can be plugged in for backfill (no major Solana RPC
currently exposes slot-aware archival account state — `getAccountInfo` always
serves latest-confirmed; `minContextSlot` is a freshness floor, not a slot pin).

### 2.2 Enriched output

When enrichment succeeds, each `SandwichAttack` carries:

- `victim_loss_lamports` — AMM-correct, in the quote token's smallest unit
- `victim_loss_lamports_lower` / `_upper` — confidence interval from per-step
  parser-vs-model residuals
- `attacker_profit` — counterfactual attacker gross profit. Can flip the
  rule-based call: a sandwich that looked profitable by `amount_out − amount_in`
  may show an attacker *loss* once the round-trip is replayed (the rule-based
  engine over-attributes when token decimals differ across legs).
- `price_impact_bps` — frontrun-induced price shift in basis points
- `severity` — bucket from the loss-to-pool-TVL ratio
- One of `amm_replay` (constant product), `clmm_replay` (V3-style: Whirlpool
  and Raydium CLMM share the schema), `dlmm_replay` — the per-DEX raw trace.
  A downstream consumer can recompute the loss step-by-step from this trace
  without re-running the detector.

Attacks on unsupported DEXes (Phoenix today; Jupiter routes through unsupported
underlying pools count too) pass through with the enrichment fields set to
`None`. Multi-hop Jupiter routes surface as `cross_boundary_unsupported` on
the per-DEX heartbeat metric.

## 3. Evidence and confidence

Every detection carries a `DetectionEvidence` block (`crates/swap-events/src/types.rs`)
that records the structured signals the detector considered. This is the
"audit trail" — a downstream consumer can re-judge the call without re-running
the detector.

### 3.1 Signal taxonomy

Signals are grouped into five orthogonal categories, so a detection that fires
across many categories is more credible than one that scores highly in a
single category.

| Category | Example signals |
|---|---|
| **Structural** | `OrderingTight` (front/back tx-index gaps) |
| **Temporal** | `SameBlock`, `CrossSlot { slot_distance }` |
| **Provenance** | `Bundle { provenance, bundle_id }` |
| **Economic** | `NaiveProfit`, `AmmProfit`, `VictimLoss`, `ReplayConfidence`, `InvariantResidual`, `ReservesMatchPostState` |
| **Plausibility** | `VictimSize`, `KnownAttackerVictim`, `AuthorityChain`, `CounterfactualAttackerProfit` |

`ENSEMBLE_CATEGORY_COUNT = 5` is the denominator for `ensemble_agreement`,
the fraction of categories that fired *Pass* on this detection.

### 3.2 Per-signal verdict

Each signal carries a `SignalVerdict`:

- **Pass** — evidence *for* the sandwich call. Ships in
  `--evidence-mode passing` (default).
- **Fail** — signal computed and points *against* the call. Detection still
  emits if the composite confidence passes the gate; ships only in
  `--evidence-mode full` for debugging.
- **Informational** — neutral context (slot distance, AMM intermediates) that
  helps interpretation but doesn't vote.

### 3.3 Tier 3 economic signals

The "Tier 3" signals (`InvariantResidual`, `ReservesMatchPostState`,
`CounterfactualAttackerProfit`) are model-vs-chain consistency checks layered
on top of replay enrichment:

- **`InvariantResidual { residual_bps, step }`** — at each replay step, how far
  the replayed reserves drift from `pre/post_token_balances` for that step.
  A small residual is the strongest single piece of evidence that the replay
  lined up with chain reality.
- **`ReservesMatchPostState { passed, divergence_bps }`** — end-state diff:
  reserves the replay believes the backrun left behind vs the actual
  post-backrun vault balances from the backrun tx's `post_token_balances`.
  Threshold `100 bps` (`crates/pool-state/src/diff_test.rs::PASS_THRESHOLD_BPS`).
- **`CounterfactualAttackerProfit { profit_real }`** — replay-derived attacker
  profit. Used to flag rule-based false positives where the attacker actually
  lost SOL on the round-trip; the most useful single signal for triage.

## 4. Limitations

- **CLOB markets (Phoenix)** — sandwich pattern (limit-order placement) does
  not match the frontrun / victim / backrun model the detector is built
  around. Phoenix retains detection-only coverage; v1 deferred enrichment
  indefinitely. See [CHANGELOG.md](CHANGELOG.md) `[1.0.0]` *Deferred*.
- **Multi-hop aggregator routes** — Jupiter V6 single-hop pivots to the
  underlying DEX's replay; multi-hop routes surface as
  `cross_boundary_unsupported` on the heartbeat metric. The cross-pool
  invariant is non-trivial when intermediate hops touch unsupported pools.
- **Archival pool state** — V3 / DLMM enrichment fetches pool accounts at
  *current* slot. A months-old sandwich corpus replays against the *current*
  pool account, not the slot-of-incident snapshot. End-to-end replay-vs-chain
  validation on historical data needs a Geyser-backed or ledger-replay
  account fetcher; the underlying library functions
  (`pool_state::compare_clmm_replay_to_archival`,
  `diff_attack_against_archival`) are ready, the fetcher implementation is
  not.
- **Token-2022 transfer fees** — the parser doesn't account for transfer-fee
  hooks; for mints that use them, `amount_out` recorded on a `SandwichAttack`
  is the pre-hook value.

## 5. Validation methodology

Three layers, each catches a different bug class.

### 5.1 Golden corpus + adversarial corpus

`crates/eval/tests/{golden,adversarial}` pin manifests of curated mainnet
sandwiches. Golden = canonical positives; adversarial = curated negatives
(hop swaps, MM activity, validator reorders) that should *not* fire. CI runs
both; a regression in either turns a check red.

### 5.2 Property-based tests

`crates/pool-state/tests/k_invariant.rs` and friends use `proptest` to assert
AMM invariants hold across random swap inputs (e.g. `x * y` is monotone after
a fee-adjusted swap; sqrt-price moves in the input direction). Catches
math-side regressions a fixture set can't, especially on the V3 sqrt-price
primitives where edge cases live in the i128 saturation boundaries.

### 5.3 Real-mainnet capture as fixture

When a parser bug surfaces on mainnet, the fix lands with an inline
base64-encoded mainnet fixture pinned as a regression test. v1.0.0 surfaced
two such cases:

- **Pump.fun `TradeEvent` extended payload** (#35) — the on-chain program
  shipped a backwards-compatible upgrade that appended fields, breaking the
  parser's `data.len() == 113` strict-equality check. Lesson: parsers
  decoding external Anchor events must use `data.len() < MIN_PREFIX` plus
  field-shape guards, never strict equality on length. Fixture pinned in
  `parses_real_mainnet_pump_fun_trade_event`.
- **Raydium CLMM pool account index** (#37) — the parser pulled `accounts[1]`
  (the `amm_config` PDA, 117 bytes, shared across pools by fee tier) instead
  of `accounts[2]` (the `pool_state`, 1544 bytes), letting the same-block
  detector group unrelated swaps as cross-mint "sandwiches". Fixed by reading
  the correct index plus four unit tests covering top-level / inner-CPI /
  short-account-list cases.

Both bugs slipped past unit tests because the unit tests' synthetic inputs
matched the parser's assumptions; the mainnet capture broke that. Adding the
real bytes (or, for the CLMM case, the real account ordering) as a fixture is
how we keep them caught next time. New parsers and parser changes must ship
with at least one mainnet-capture test.

### 5.4 Mainnet validation runs

For releases, run the binary over a 1k–10k slot mainnet window and inspect
the per-DEX enrichment matrix. Both v1.0.0 hotfixes were caught this way: the
first by the 1k validation that exposed Pump.fun TradeEvent breakage, the
second by the 10k that exposed CLMM cross-mint false positives at a frequency
the 1k sample missed. Larger sample is the right answer when "the unit tests
pass but production is wrong."

## 6. Where to look

| Question | File |
|---|---|
| How are sandwiches detected? | `crates/detector-sameblock/src/detector.rs`, `crates/detector-window/src/pipeline.rs` |
| How is a swap parsed for DEX X? | `crates/swap-events/src/dex/<x>.rs` |
| What's the AMM-replay math for DEX X? | `crates/pool-state/src/<x>.rs` |
| What signals does evidence carry? | `Signal` enum in `crates/swap-events/src/types.rs` |
| Wire format / Vigil contract? | `crates/swap-events/schema/vigil-v1.json` (canonical), `contrib/vigil-types.ts` (TS mirror) |
| Validation harness? | `crates/eval/tests/`, `crates/pool-state/tests/` |
