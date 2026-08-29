# PR #3 設計レビュー report

## Metadata

- Repository: `ssaattww/Devo6.Interval`
- Pull request: `#3` (`docs: add interval arithmetic detailed design`)
- Review mode: `initial_review`
- Base ref: `main`
- Base SHA: `ad5c058f8a4164b0c7d0763c65246914ea5d1c03`
- Reviewed implementation/design HEAD: `da6e2ae04d35b01acfb307953a093c81c15342b8`
- Commit range: `ad5c058f8a4164b0c7d0763c65246914ea5d1c03..da6e2ae04d35b01acfb307953a093c81c15342b8`
- Reviewer: ChatGPT GPT-5.6 Sol / normal reviewer
- Reviewer continuity: initial review; previous normal-review finding set is not present
- Reviewer independence evidence: this review chat did not implement PR #3 or any review fix
- Date: 2026-08-29
- Verdict: **fail**

技術的な verdict は上記 `reviewed implementation/design HEAD` に対するものである。本 report の repository persistence commit はレビュー結果を保存するための管理変更であり、設計内容への verdict を別 HEAD へ自動的に移すものではない。

## Purpose

PR #3 の詳細設計が、既存の基本設計、利用者が指定した開発フェーズ、参照実装、および後続 Phase 1/Phase 3 の実装開始条件を満たすかを確認した。

特に次を重点確認した。

- scalar pilot -> API freeze -> SIMD -> 四則演算以外、というフェーズ分割
- `[-Lower, Upper]` 内部表現と signed zero / Empty / Entire
- 四則演算の区間意味論と符号分類表
- managed scalar の正しい方向付き丸め
- subnormal / underflow / overflow
- exact-rational oracle と TDD
- AVX-512 / AVX2 / SSE2 / ARM64 の SIMD 境界
- current HEAD に一致する CI evidence
- PR 内 report / handoff の正確性

## Authoritative requirements and design

確認した主な要求・設計:

- SIMD なしの四則演算パイロットを先に作る。
- パイロット後に基本 API を確定する。
- API 確定後に SIMD backend を追加する。
- その後に四則演算以外を追加する。
- 基本設計 `doc/Design/basic/IntervalArithmetic.md` の `double` endpoint、IEEE 1788.1-oriented semantics、外向き丸め、`[-Lower, Upper]`、Empty / Entire を維持する。
- CI を確認するときは PR current HEAD SHA と workflow run の head SHA が一致する run だけを使用し、一致する run がなければ CI 未実施として扱う。

## Changed files inspected

全 changed file を確認した。

1. `doc/Design/detail/IntervalArithmetic.md`
2. `reports/2026-08-29-interval-arithmetic-detailed-design.md`
3. `reports/2026-08-29-interval-arithmetic-detailed-design-handoff.yaml`

直接依存・参照として次も確認した。

- `doc/Design/basic/IntervalArithmetic.md`
- `unageek/inari` pinned commit `18b83a571d7681c76067bc38d90a74e8be29f545`
  - `src/arith.rs`
  - `src/basic.rs`
  - `src/interval.rs`
- `mskashi/kv` pinned commit `c7f8f2324a0e403cca6b39f46088a22843d440db`
  - `kv/rdouble-nohwround.hpp`
- .NET 10 `Math.FusedMultiplyAdd`, `Avx512F`, `FloatRoundingMode`, `Fma`, `Avx2` documentation/runtime surface

## Positive findings

- 四則演算の区間符号分類表は `inari` の pinned implementation と整合している。
- denominator `[0,0]`、0 への片側接触、0 跨ぎを含む division の set-based semantics は `inari` と整合している。
- constructor の有効条件、Empty / Entire、`[-Lower, Upper]`、`default(Interval) == Zero` の方針は基本設計と整合している。
- AVX-512 の packed embedded rounding を `Vector512<double>` の 4 区間 batch path として扱う方針は .NET 10 API surface と整合している。
- exact-rational oracle を production algorithm から独立させる方針、failure artifact 要件、API baseline gate は妥当である。
- current PR の scope は documentation-only に保たれており、`src/**` / `tests/**` への unrelated change はない。

## Findings

### F-PR3-001 — High — subnormal/underflow の方向付き丸め手順が実装可能な粒度まで確定していない

- Origin: `introduced_by_change`
- Location: `doc/Design/detail/IntervalArithmetic.md` §8.6 `乗算`, §8.7 `除算`

#### Description

設計は threshold/scale 定数を示しているが、subnormal 領域で tight な directed result を決める具体的な比較手順が不足している。

乗算では、`abs(product) < 2^-969` の後が「scaled exact relation から判定する」とだけ記載され、scaled product、scaled residual、元の rounded product をどの順序・条件で比較して `NextUp` / `NextDown` を決定するかが一意に決まらない。

除算ではより重大で、pinned `kv` の `div_up` / `div_down` は denominator を正に正規化した後、概ね次の分岐を持つ。

```text
abs(x) < 2^-969:
  abs(y) < 2^918:
    x と y を 2^105 倍して比較処理を継続
  otherwise:
    Up:   x < 0 ? +0 : 2^-1074
    Down: x < 0 ? -2^-1074 : +0
```

この `abs(y) >= 2^918` の early-return は、denominator を `2^105` 倍して overflow させずに tight result を確定するために必要である。現設計は `LargeDenominatorLimit = 2^918` と `MinimumSubnormal = 2^-1074` を列挙するだけで、この分岐と direction/sign ごとの返値を定義していない。

#### Impact

Phase 1 の scalar backend は後続 SIMD backend の reference と定義されているため、ここで 1 ULP または zero/min-subnormal の向きを誤ると、次の基準すべてが誤ったものになる。

- exact directed endpoint
- scalar vs SIMD bitwise equivalence
- reference differential test
- API freeze の correctness gate

また §3.1 の「Phase 1 が追加の基本設計判断なしで開始できる」という Phase 0 完了条件を満たさない。

#### Evidence

- PR design §8.6–8.7 は定数と概念手順までで、direction-specific な scaled comparison / large-denominator branch を定義していない。
- pinned `kv/rdouble-nohwround.hpp` の `mul_up`, `mul_down`, `div_up`, `div_down` にはこれらの具体的な比較・early-return が存在する。

#### Required action

§8.6–8.7 に、少なくとも次を実装可能な pseudocode として追加すること。

1. `MultiplyUp` / `MultiplyDown` の通常領域と scaled 領域の正確な比較条件。
2. `DivideUp` / `DivideDown` の denominator 正符号化。
3. `abs(x) < 2^-969` かつ `abs(y) < 2^918` の scaling 手順。
4. `abs(x) < 2^-969` かつ `abs(y) >= 2^918` の direction/sign 別 `0` / `±2^-1074` early-return。
5. quotient/product comparison に FMA を使う場合の exact relation の判定式。
6. threshold の直前・一致・直後、および正負両方向を固定する test matrix。
7. `kv` の定数を FMA-based algorithm に流用する場合、その安全条件または参照した具体的ロジックを明示すること。

---

### F-PR3-002 — Medium — AVX2/SSE2 と x86 FMA capability が分離されていない

- Origin: `introduced_by_change`
- Location: `doc/Design/detail/IntervalArithmetic.md` §4.2, §15.4–15.5

#### Description

§15.4 は `AVX2 / SSE2 / ARM64` をまとめ、その候補として `vectorized FMA residual` を挙げている。しかし x86 の FMA は AVX2 や SSE2 に内包される capability ではなく、.NET でも `System.Runtime.Intrinsics.X86.Fma` として独立した `IsSupported` 判定を持つ。

したがって、少なくとも次は別 capability として設計する必要がある。

- AVX2 + FMA
- AVX2 without FMA
- SSE2 without FMA
- ARM64 AdvSimd の該当 FMA capability

#### Impact

現記述のまま Phase 3 を実装すると、AVX2/SSE2 backend の multiplication/division が FMA の存在を暗黙前提にする、または非 FMA CPU の fallback 契約が実装時まで未決定になる。

#### Evidence

.NET 10 は `Avx2` と別に `Fma` intrinsic class / `Fma.IsSupported` を提供している。`Fma` は `Avx` を継承する別 instruction-set capability であり、`Avx2.IsSupported` または `Sse2.IsSupported` だけでは FMA availability を保証しない。

#### Required action

Phase 3 の backend capability matrix と dispatch 条件を明記すること。非 FMA x86 については、次のいずれを採るか Phase 3 開始前に明確にすること。

- vectorized TwoProduct/Dekker 等を使用する
- 加減算のみ SIMD 化し、乗除算は scalar fallback とする
- 該当 backend 自体を performance evaluation の結果として採用しない

---

### F-PR3-003 — Low — exact-head CI が存在しない状態を `not_applicable` と記録している

- Origin: `introduced_by_change`
- Location: `reports/2026-08-29-interval-arithmetic-detailed-design-handoff.yaml` `ci` section; PR validation wording

#### Description

reviewed HEAD `da6e2ae04d35b01acfb307953a093c81c15342b8` に一致する workflow run は 0 件である。repository instruction では、この場合は別 SHA の run を代用せず **CI 未実施** と報告する必要がある。

handoff は `ci.required: false` とすること自体は documentation-only scope と両立するが、`ci.conclusion: not_applicable`、`head_sha: unknown` としており、「不要」と「実際に run が存在しない」を分離できていない。

#### Impact

後続 worker が handoff だけを読むと、current-head CI が確認済みで不要だったのか、単に run が存在しなかったのかを区別できない。exact-head evidence の監査性が落ちる。

#### Evidence

GitHub connector で reviewed HEAD に対する workflow runs を確認した結果は 0 件である。別 SHA の run は CI evidence として使用していない。

#### Required action

次回 handoff/report 更新時に、少なくとも次を明示すること。

- CI requirement: documentation-only のため required=false（必要なら維持）
- exact target SHA: `da6e2ae04d35b01acfb307953a093c81c15342b8` または修正後の current HEAD
- matching run: none
- execution status: `CI未実施`
- 他 SHA run を代用していないこと

## Required coverage dispositions

| Criterion | Disposition | Evidence |
|---|---|---|
| Requirement / design conformance | `checked_finding` | Phase order and basic design are consistent; F-PR3-001 blocks Phase 0 completion claim. |
| Correctness / edge cases | `checked_finding` | Sign tables verified against inari; subnormal directed-rounding details incomplete. |
| Scope discipline / unrelated changes | `checked_no_finding` | 3 changed files are design/report/handoff only. |
| Changed files / direct dependencies | `checked_no_finding` | All 3 changed files plus basic design, pinned inari/kv and .NET intrinsic surface inspected. |
| Public API / compatibility | `checked_no_finding` | Pilot API is explicitly provisional until Phase 2; no current contradiction found. |
| SIMD backend / capability design | `checked_finding` | F-PR3-002. |
| Error handling / failure diagnostics | `checked_no_finding` | Empty/Entire semantics and planned failure artifacts are coherent for the stated scope. |
| Security / secrets | `not_applicable` | Documentation-only change; no credentials or security-sensitive execution path added. |
| TDD / test adequacy | `checked_finding` | Test strategy is strong, but F-PR3-001 must be specified so tests and production do not re-derive different underflow rules. |
| Current-HEAD CI evidence | `checked_finding` | Matching workflow run count is 0; CI is unexecuted for the reviewed HEAD. F-PR3-003 records the reporting mismatch. |
| Report / handoff accuracy | `checked_finding` | Main design report is largely faithful; handoff CI state needs correction per F-PR3-003. |
| Regression / maintainability risk | `checked_finding` | Scalar reference ambiguity and SIMD capability ambiguity would propagate into later phases. |

## Validation assessment

### Arithmetic semantics

Result: `supported` except for F-PR3-001.

- multiplication sign-class table checked against the four endpoint candidates and pinned inari implementation
- division positive/negative/zero-contact/cross-zero tables checked against pinned inari implementation
- Empty / Zero special cases checked

### Directed-rounding constants

Result: `partially supported`.

The following constants match the pinned `kv` no-hardware-rounding implementation.

- multiplication threshold `2^-969`
- multiplication scale `2^537`
- division threshold `2^-969`
- large denominator limit `2^918`
- division scale `2^105`
- minimum subnormal `2^-1074`

The constants alone are insufficient because the branch/comparison logic using them is not fully specified.

### .NET intrinsic assumptions

Result: `supported with F-PR3-002`.

- .NET 10 exposes `Avx512F` packed `Vector512<double>` operations with `FloatRoundingMode`, including directed rounding.
- .NET 10 exposes `Math.FusedMultiplyAdd(double, double, double)` as one fused ternary operation.
- x86 FMA availability is exposed independently through `Fma.IsSupported`; it is not implied by AVX2/SSE2.

### Build / tests

Result: `not_applicable` to the documentation content itself.

The repository has no executable project/test target in this PR, so no build/test command was run. This does **not** mean CI was executed successfully.

### Exact-head CI

Result: `unavailable / not executed`.

- Reviewed HEAD: `da6e2ae04d35b01acfb307953a093c81c15342b8`
- Matching workflow runs: `0`
- CI status for this HEAD: **未実施**
- No run from another SHA was substituted.

## Held / deferred items

The following are intentionally deferred by the design and do not independently block this review.

- final namespace and constructor/factory shape: Phase 2
- scalar overload / conversion and generic math: Phase 2
- stable text format: later phase
- batch overlap / in-place rules: Phase 3 detailed design
- AVX2/ARM64 adoption itself: performance evaluation in Phase 3
- decorated interval / extended division: later phase

## Unexplored

None that block the design review. There is no executable code in the reviewed diff, so runtime benchmark/build behavior cannot be exercised at this stage.

## Verdict

**fail**

F-PR3-001 is a high-severity correctness/design-completeness finding and must be fixed before Phase 1 uses this document as an implementation specification. F-PR3-002 and F-PR3-003 should be corrected in the same design revision so the Phase 3 capability contract and exact-head evidence remain unambiguous.

## Next action

1. Return PR #3 to the design implementation chat.
2. Address F-PR3-001, F-PR3-002, F-PR3-003 without broadening scope.
3. Commit/push the fixes in reviewable units.
4. Produce a per-finding closure matrix containing required action, changed design/report path, concrete test/design fixture where applicable, and focused evidence.
5. Reuse this normal review chat for fix verification against the new immutable HEAD.
6. For CI, only use a workflow run whose `head_sha` exactly equals that new PR HEAD; if none exists, report CI未実施.

## Merge boundary

No merge was performed. Merge remains the repository owner's action.