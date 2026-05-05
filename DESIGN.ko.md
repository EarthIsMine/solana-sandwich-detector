# Design

<p align="center">
  <a href="DESIGN.md">English</a> &middot;
  <strong>한국어</strong>
</p>

이 문서는 detector의 동작 방식 — 검출 모델, AMM replay enrichment, evidence/confidence 분류, 검증 방법론 — 을 다룹니다. [README](README.ko.md)는 설치 / CLI 사용 / Vigil 통합 계약을 다루고, 이 문서는 *왜* 특정 detection이 발화하는지 (또는 안 하는지) 와 emit되는 수치들이 어떤 보장을 갖는지를 알고 싶은 독자를 위한 자료입니다.

## 1. 검출 모델

샌드위치는 두 참가자가 두 역할로 행하는 세 swap입니다. Detector는 이걸 `SwapEvent` 스트림 위의 구조적 패턴으로 축소한 뒤, 정밀 필터와 경제적 enrichment를 위에 쌓습니다.

### 1.1 Same-block 검출

```
Block (slot N)
 |
 +-- 트랜잭션 파싱 -->  DEX별 swap 이벤트 추출
 |                       (instruction discriminator + 토큰 잔액 변화)
 |
 +-- pool 별 그룹화
 |
 +-- 패턴 검출:
       tx[i]  attacker BUYS   -+
       tx[j]  victim   BUYS    +-- 같은 pool, 같은 방향
       tx[k]  attacker SELLS  -+   반대 방향, tx[i]와 같은 signer
```

구현: `crates/detector-sameblock/src/detector.rs`. 단일 block의 swap 이벤트만 사용; cross-slot 상태 없음.

### 1.2 Cross-slot 윈도우 검출

```
Slot N:    pool P 에서 attacker BUYS    (frontrun)
Slot N+1:  pool P 에서 victim   BUYS    (frontrun과 같은 방향)
Slot N+2:  pool P 에서 attacker SELLS   (backrun — 반대 방향)
```

`FilteredWindowDetector` (`crates/detector-window/src/pipeline.rs`) 는 pool별 `W` 슬롯 슬라이딩 윈도우를 유지하며, 윈도우 내 후보 triplet에 emit 직전 세 가지 정밀 필터를 적용합니다:

1. **번들 출처 (Bundle provenance)** — Jito 번들 동소성. 우선순위 `AtomicBundle > SpanningBundle > TipRace > Organic`. frontrun과 backrun이 같은 번들이라는 건 attacker가 원자성에 비용을 지불했다는 뜻 — 오프체인에서 읽을 수 있는 가장 강한 provenance 신호.
2. **경제적 타당성 (Economic feasibility)** — `backrun.amount_out − frontrun.amount_in > tx 수수료`. 느슨한 floor; AMM-replay enrichment가 다음 단계에서 더 조입니다.
3. **피해자 그럴듯함 (Victim plausibility)** — frontrun 대비 size-ratio 검사 (큰 leg 두 개 사이에 1-lamport "victim"이 끼는 건 거의 routing artifact, 진짜 sandwich 아님) + MM/arb bot으로 확인된 known-attacker 제외 리스트.

각 emit detection은 세 필터 결과의 합성으로 `confidence` ∈ [0, 1] 을 갖습니다. gate threshold는 pipeline 안에 있음.

### 1.3 검출 규칙 (두 detector 공통)

| 조건 | 이유 |
|------|------|
| `frontrun.signer == backrun.signer` | 같은 attacker |
| `frontrun.direction != backrun.direction` | 반대 거래 (사고 → 팔고) |
| `victim.direction == frontrun.direction` | victim이 가격을 더 밀어줌 |
| `victim.signer != attacker` | 다른 지갑 |
| 같은 pool | 가격 영향이 그 pool에 국지적 |
| 같은 slot (same-block) 또는 W 슬롯 내 (window) | 시간적 근접성 |

### 1.4 하이브리드 파싱

DEX swap 이벤트는 하이브리드 전략으로 추출됩니다: instruction discriminator로 프로그램과 pool 주소를 식별하고, pre/post 토큰 잔액 변화로 swap 방향과 금액을 결정합니다. discriminator 쪽은 (Anchor IDL 변경에) 취약하고 잔액 delta 쪽도 (multi-hop / routed swap이 net delta를 흐림) 취약하므로, 각 parser는 둘을 cross-check 로 사용합니다. `crates/swap-events/src/dex/` 참조.

## 2. AMM replay enrichment

규칙 기반 검출은 *"이게 sandwich였나?"* 에는 답하지만 *"피해자는 얼마를 잃었나?"* 에는 답하지 못합니다 — 단순 `backrun.amount_out − frontrun.amount_in` 휴리스틱은 가격 영향 동학을 무시하고 multi-token 페어에서 깨집니다. `pool-state` 크레이트는 각 swap을 AMM 자체 수학으로 replay 해서 둘 다 해결합니다:

```
state_0   = frontrun 직전 pool 잔액           (tx 메타에서)
state_1   = state_0 위에 frontrun 적용         (replay)
state_2   = state_1 위에 victim 적용           (replay, "실제")
state_2c  = state_0 위에 victim 적용, frontrun 없이  (replay, "반사실")
victim_loss  = victim_out(state_2c) − victim_out(state_2)
```

반사실 replay (`state_2c`) 가 손실 수치를 AMM-correct 하게 만드는 핵심 — frontrun 이 없었던 세계에서 victim이 *받았어야 할* 출력을, 프로그램 자체가 돌리는 같은 pool 수학으로 계산합니다.

### 2.1 DEX별 reserves 출처

| AMM 계열 | Reserves 출처 | Replay primitive |
|---|---|---|
| **Constant product** (Raydium V4 / CPMM) | tx 메타의 `pre_token_balances` / `post_token_balances` | `x * y = k` |
| **Concentrated liquidity** (Whirlpool, Raydium CLMM) | pool 계정에 `getAccountInfo` 로 동적 스냅샷 (`sqrt_price`, `liquidity`, 현 `tick`); tick-array PDA 도출 | sqrt-price step + V3 TickArray 통한 cross-tick walk |
| **Discretised liquidity** (Meteora DLMM) | pool 계정 + active/인접 bin 의 bin-array PDA | bin 안에서는 constant-sum + bitmap 통한 cross-bin walk |
| **Bonding curve** (Pump.fun) | Pump.fun 이 swap마다 emit 하는 `TradeEvent` Anchor log 에서 `virtual_sol_reserves` / `virtual_token_reserves` 복원 | 본딩 커브 공식 |
| **Aggregator** (Jupiter V6) | route 에서 underlying DEX 결정 후 그 DEX 의 replay path 로 dispatch | v1 에서 single-hop 만; multi-hop 은 defer |

constant-product 와 Pump.fun 경로는 추가 RPC 불필요. V3 / DLMM 경로는 pool 계정에 `getAccountInfo` 필요; [`AccountFetcher`](crates/pool-state/src/rpc.rs) trait 가 fetch 를 abstraction 해 archival provider 를 끼워 넣을 수 있게 함 (현재 어떤 메이저 Solana RPC 도 슬롯 인지 archival account state 를 노출하지 않음 — `getAccountInfo` 는 항상 latest-confirmed 를 반환하고, `minContextSlot` 은 freshness floor 일 뿐 슬롯 pin 이 아님).

### 2.2 Enriched 출력

Enrichment 성공 시 각 `SandwichAttack` 은 다음을 갖습니다:

- `victim_loss_lamports` — AMM-correct, quote 토큰 최소 단위
- `victim_loss_lamports_lower` / `_upper` — step별 parser-vs-model 잔차에서 도출한 confidence interval
- `attacker_profit` — 반사실 attacker 총 이익. 규칙 기반 판정을 뒤집을 수 있음: `amount_out − amount_in` 으로는 수익으로 보이는 sandwich 가, round-trip 이 올바른 토큰 단위로 replay 되면 attacker *손실* 로 드러나는 케이스 (leg 별 토큰 decimals 가 다를 때 규칙 기반이 과대 귀속).
- `price_impact_bps` — frontrun 이 유발한 가격 변화 (basis points)
- `severity` — loss 대비 pool TVL 비율 버킷
- `amm_replay` (constant product), `clmm_replay` (V3-style: Whirlpool 과 Raydium CLMM 가 schema 공유), `dlmm_replay` 중 하나 — DEX별 raw trace. 다운스트림 소비자가 detector 를 다시 돌리지 않고도 step-by-step 으로 loss 를 재계산 가능.

미지원 DEX (현 Phoenix; Jupiter route 가 미지원 underlying pool 거치는 경우 포함) 의 공격은 enrichment 필드들이 `None` 인 채 통과됩니다. Multi-hop Jupiter route 는 per-DEX heartbeat 메트릭의 `cross_boundary_unsupported` 로 분류됩니다.

## 3. Evidence 와 confidence

모든 detection 은 `DetectionEvidence` 블록 (`crates/swap-events/src/types.rs`) 을 carry — detector 가 고려한 구조화된 신호의 기록. 이게 "감사 추적" 역할: 다운스트림 소비자가 detector 를 다시 돌리지 않고도 판정을 재평가 가능.

### 3.1 신호 분류

신호는 다섯 개의 직교 카테고리로 그룹화됩니다 — 여러 카테고리에 걸쳐 발화한 detection 이 한 카테고리 안에서만 강하게 발화한 것보다 더 신뢰됩니다.

| Category | 예시 신호 |
|---|---|
| **Structural** | `OrderingTight` (front/back tx-index 갭) |
| **Temporal** | `SameBlock`, `CrossSlot { slot_distance }` |
| **Provenance** | `Bundle { provenance, bundle_id }` |
| **Economic** | `NaiveProfit`, `AmmProfit`, `VictimLoss`, `ReplayConfidence`, `InvariantResidual`, `ReservesMatchPostState` |
| **Plausibility** | `VictimSize`, `KnownAttackerVictim`, `AuthorityChain`, `CounterfactualAttackerProfit` |

`ENSEMBLE_CATEGORY_COUNT = 5` 가 `ensemble_agreement` 의 분모 — 이 detection 위에서 Pass 발화한 카테고리의 비율.

### 3.2 신호별 verdict

각 신호는 `SignalVerdict` 를 갖습니다:

- **Pass** — sandwich 판정에 *찬성하는* 증거. `--evidence-mode passing` (기본) 으로 ship.
- **Fail** — 신호가 계산되었지만 판정에 *반대하는* 방향. 합성 confidence 가 gate 를 통과하면 detection 은 emit; debug 용 `--evidence-mode full` 에서만 ship.
- **Informational** — 중립 컨텍스트 (slot 거리, AMM 중간값 등). 해석에 도움되지만 vote 는 안 함.

### 3.3 Tier 3 경제 신호

"Tier 3" 신호 (`InvariantResidual`, `ReservesMatchPostState`, `CounterfactualAttackerProfit`) 는 replay enrichment 위에 layered 된 model-vs-chain 일관성 검사:

- **`InvariantResidual { residual_bps, step }`** — 각 replay step 에서 replay 된 reserves 가 그 step 의 `pre/post_token_balances` 에서 얼마나 drift 했는지. 작은 잔차는 replay 가 chain 현실과 일치한다는 가장 강한 단일 증거.
- **`ReservesMatchPostState { passed, divergence_bps }`** — end-state diff: replay 가 backrun 후라고 믿는 reserves vs backrun tx 의 `post_token_balances` 의 실제 vault 잔액. Threshold `100 bps` (`crates/pool-state/src/diff_test.rs::PASS_THRESHOLD_BPS`).
- **`CounterfactualAttackerProfit { profit_real }`** — replay 도출 attacker 이익. round-trip 에서 attacker 가 실제로 SOL 을 잃은 케이스를 규칙 기반 false positive 로 flag 하는 데 사용 — triage 에 가장 유용한 단일 신호.

## 4. 한계

- **CLOB 마켓 (Phoenix)** — sandwich 패턴 (limit-order placement) 이 detector 가 기반한 frontrun / victim / backrun 모델과 mismatch. Phoenix 는 detection-only 커버리지 유지; v1 enrichment 무한 defer. [CHANGELOG.md](CHANGELOG.md) `[1.0.0]` *Deferred* 참조.
- **Multi-hop aggregator route** — Jupiter V6 single-hop 은 underlying DEX replay 로 pivot; multi-hop 은 heartbeat 메트릭의 `cross_boundary_unsupported` 로 surface. 중간 hop 이 미지원 pool 을 거칠 때 cross-pool 불변량이 비자명.
- **Archival pool state** — V3 / DLMM enrichment 는 pool 계정을 *현재* slot 에서 fetch. 수개월 된 sandwich 코퍼스를 replay 하면 *현재* pool 계정에 대해 돌아감, 사건 발생 슬롯 스냅샷 아님. 과거 데이터 위 end-to-end replay-vs-chain 검증은 Geyser-backed 또는 ledger-replay account fetcher 가 필요; library 함수 (`pool_state::compare_clmm_replay_to_archival`, `diff_attack_against_archival`) 는 준비되어 있고 fetcher 구현이 미구현.
- **Token-2022 transfer fees** — parser 가 transfer-fee hook 을 회계하지 않음; hook 사용하는 mint 의 경우 `SandwichAttack` 에 기록된 `amount_out` 은 hook 적용 *전* 값.

## 5. 검증 방법론

세 layer, 각각 다른 버그 부류를 잡음.

### 5.1 Golden corpus + adversarial corpus

`crates/eval/tests/{golden,adversarial}` 가 큐레이션된 mainnet sandwich 의 manifest 를 박아 둠. Golden = 정통 positive; adversarial = 큐레이션된 negative (hop swap, MM activity, validator reorder) — 발화하면 *안 되는* 케이스. CI 가 둘 다 돌림; 어느 쪽이든 회귀하면 check 가 빨개짐.

### 5.2 Property-based 테스트

`crates/pool-state/tests/k_invariant.rs` 와 친구들이 `proptest` 로 random swap 입력에 대해 AMM 불변량을 assert (예: 수수료 적용 swap 후 `x * y` 단조; sqrt-price 가 입력 방향으로 이동). fixture 셋이 못 잡는 math-side 회귀를 잡음 — 특히 V3 sqrt-price primitive 의 i128 saturation 경계 edge case.

### 5.3 Real-mainnet capture as fixture

mainnet 에서 parser 버그가 surface 되면, fix 가 인라인 base64 인코딩된 mainnet fixture 를 회귀 테스트로 박아서 ship. v1.0.0 이 두 그런 케이스를 surface 함:

- **Pump.fun `TradeEvent` extended payload** (#35) — on-chain 프로그램이 backwards-compatible 업그레이드로 필드를 append, parser 의 `data.len() == 113` strict-equality 검사가 깨짐. 교훈: 외부 Anchor 이벤트를 디코딩하는 parser 는 길이에 strict equality 가 아닌 `data.len() < MIN_PREFIX` + 필드 모양 가드 사용. `parses_real_mainnet_pump_fun_trade_event` 에 fixture pin.
- **Raydium CLMM pool 계정 인덱스** (#37) — parser 가 `accounts[1]` (`amm_config` PDA, 117 byte, fee tier 별로 pool 들이 공유) 을 `accounts[2]` (`pool_state`, 1544 byte) 대신 가져와서, same-block detector 가 무관한 swap 들을 cross-mint "sandwich" 로 묶음. 올바른 인덱스 사용 + top-level / inner-CPI / 짧은 account list edge case 커버하는 unit test 4 개로 fix.

두 버그 모두 unit test 의 합성 입력이 parser 의 가정과 일치해서 unit test 를 통과함; mainnet capture 가 그걸 깨뜨림. 실제 바이트 (또는 CLMM 의 경우 실제 account 순서) 를 fixture 로 추가하는 게 다음에도 잡히게 만드는 방법. 새 parser 와 parser 변경은 최소 한 개의 mainnet-capture test 와 함께 ship 해야 함.

### 5.4 Mainnet 검증 실행

릴리스용으로, 1k–10k slot mainnet 윈도우에 binary 를 돌리고 DEX별 enrichment 매트릭스 검사. v1.0.0 두 hotfix 모두 이 방식으로 잡힘: 첫 번째는 Pump.fun TradeEvent breakage 를 노출한 1k 검증, 두 번째는 1k 표본이 놓친 빈도로 CLMM cross-mint false positive 를 노출한 10k 검증. "unit test 는 통과하는데 production 이 틀린" 상황의 정답은 더 큰 표본.

## 6. 어디를 봐야 하나

| 질문 | 파일 |
|---|---|
| Sandwich 가 어떻게 검출되나? | `crates/detector-sameblock/src/detector.rs`, `crates/detector-window/src/pipeline.rs` |
| DEX X 의 swap 은 어떻게 파싱되나? | `crates/swap-events/src/dex/<x>.rs` |
| DEX X 의 AMM-replay 수학? | `crates/pool-state/src/<x>.rs` |
| Evidence 가 carry 하는 신호는? | `Signal` enum in `crates/swap-events/src/types.rs` |
| Wire format / Vigil 계약? | `crates/swap-events/schema/vigil-v1.json` (canonical), `contrib/vigil-types.ts` (TS mirror) |
| 검증 harness? | `crates/eval/tests/`, `crates/pool-state/tests/` |
