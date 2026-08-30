# 区間演算 詳細設計レポート

## Metadata

- Repository: `ssaattww/Devo6.Interval`
- Base branch: `main`
- Working branch: `docs/detailed-interval-arithmetic-design`
- Pull request: `#3`
- Base commit: `ad5c058f8a4164b0c7d0763c65246914ea5d1c03`
- Technical design commit: `aa1a5e5a494ca3fc384250468a4df38014cd2f42`
- Basic design: `doc/Design/basic/IntervalArithmetic.md`
- Detailed design: `doc/Design/detail/IntervalArithmetic.md`
- Date: 2026-08-29

## Purpose

基本設計で決定した IEEE 1788.1-oriented semantics、`[-Lower, Upper]` 内部表現、外向き丸めおよび SIMD 方針を、パイロット実装を開始できる粒度へ詳細化した。

利用者が提示した開発順序を評価し、次のフェーズとして定義した。

1. pure-managed scalar の四則演算パイロット
2. 基本 `Interval` API の確定
3. API を維持した SIMD backend の追加
4. 四則演算以外の機能追加

## Scope

- パイロット版の対象と非対象
- 公開 `Interval` API 候補
- constructor、`Empty`、`Entire`、`Zero`、signed zero の規則
- `[-Lower, Upper]` の物理表現と内部不変条件
- scalar managed の方向付き丸め
- 加算、減算、乗算、除算の endpoint algorithm
- 乗算・除算の符号分類表
- TDD、exact-rational oracle、参照実装比較
- API 確定ゲート
- SIMD backend と batch API の境界
- 四則演算以外の追加順序
- failure diagnostic artifact の要件

## Non-goals

- ライブラリ実装
- project / solution 作成
- CI workflow 作成
- SIMD intrinsics 実装
- native wrapper 実装
- `sqrt`, `exp`, `log`, `sin` 等の実装
- decorated interval / NaI の実装
- parsing / serialization 契約の確定

## Authoritative Requirements

### User instruction

- SIMD なしの四則演算版を最初に作る。
- パイロットの後に API を確定する。
- API 確定後に SIMD 版を作る。
- その後に四則演算以外を追加する。

### Existing basic design

- endpoint type は `double`。
- IEEE 1788.1-2017 を意味論の基準とする。
- `inari` を主要な参照実装とする。
- 内部表現は `[-Lower, Upper]`。
- 外向き丸めを必須とする。
- `Empty` / `Entire` を正式な値とする。
- bare interval と decorated interval を分離する。

## Inspected Evidence

- `doc/Design/basic/IntervalArithmetic.md`
- `.github/workflows/` の現状
- `unageek/inari` の interval representation と MIT License
- `mskashi/kv` の no-hardware-rounding implementation と MIT License
- .NET の support policy、`Math.FusedMultiplyAdd`、AVX-512 intrinsics

Pinned reference commits:

- `inari`: `18b83a571d7681c76067bc38d90a74e8be29f545`
- `kv`: `c7f8f2324a0e403cca6b39f46088a22843d440db`

## Phase Assessment

提示された順序を採用した。

ただし、四則演算パイロットには演算子だけでなく、次を含める必要があると判断した。

- valid / invalid construction
- `Empty`, `Entire`, `Zero`
- endpoint access
- equality / Hash
- signed zero normalization
- unary negation
- diagnostic formatting

これらは四則演算結果の意味と API を評価するために必要であり、後付けすると API freeze 後の breaking change になりやすい。

## Public API Candidate

Pilot candidate:

```text
Assembly / package: Devo6.Interval
Namespace:          Devo6.Numerics
Type:               Interval
```

主な surface:

- `Interval(double lower, double upper)`
- `Point(double)`
- `TryCreate(...)`
- `Lower`, `Upper`
- `Empty`, `Entire`, `Zero`
- `IsEmpty`, `IsEntire`, `IsSingleton`
- binary `+`, `-`, `*`, `/`
- unary `-`
- `IEquatable<Interval>`、`==`, `!=`, `GetHashCode`
- diagnostic `ToString`

次は Phase 1 に含めず、Phase 2 で判断する。

- implicit conversion from `double`
- mixed `double` / `Interval` operators
- `INumber<TSelf>`
- stable parsing / serialization format

## Internal Representation

Phase 1 は private な 2 個の `double` を使用する。

```text
_negatedLower = -Lower
_upper        =  Upper
```

Canonical states:

```text
Zero   = [+0.0, +0.0]
Entire = [+Infinity, +Infinity]
Empty  = [canonical-qNaN, canonical-qNaN]
```

`default(Interval)` は `Zero` と同値になる。

物理 layout、size、SIMD vector cast、blittable ABI は public contract にしない。

## Directed Rounding Design

Phase 1 は CPU の global rounding mode を変更しない。

- addition: TwoSum residual
- multiplication: nearest product + `Math.FusedMultiplyAdd` residual
- division: nearest quotient と exact-product comparison
- correction: residual の符号に応じ、必要な場合だけ `Math.BitIncrement` / `Math.BitDecrement`
- unconditional 1-ULP widening は行わない。
- subnormal 領域は `kv` を参照して scaling する。

設計上の主要定数:

```text
Multiplication threshold = 2^-969
Multiplication scale     = 2^537
Division small threshold = 2^-969
Division large limit     = 2^918
Division scale           = 2^105
Minimum subnormal        = 2^-1074
```

production code では `BigInteger` を使用しない。

## Arithmetic Design

### Addition / subtraction

`[-Lower, Upper]` 表現のまま、両 endpoint を上向き primitive へ変換できる。

### Multiplication

Zero、nonnegative、nonpositive、mixed に分類し、各組合せで必要な endpoint product だけを評価する。

Zero を先に処理し、`0 * Infinity` を raw floating-point primitive へ渡さない。

### Division

- denominator が 0 を含まない場合は endpoint を直接除算する。
- reciprocal interval を作って乗算する二段階方式は、不要な二重丸めを避けるため採用しない。
- denominator `[0,0]` は `Empty`。
- denominator が 0 に片側から接する場合は片側非有界区間または `Entire`。
- denominator が 0 を跨ぎ、numerator が `Zero` でなければ convex hull として `Entire`。

## Equivalent-result Definition

「既存ライブラリと同等」を、単に包含していることとは定義しない。

四則演算では次を要求する。

1. IEEE 1788.1 set-based result
2. 単一区間で表せない場合は convex hull
3. 正しい方向へ丸められた最も内側の binary64 endpoint
4. canonical signed zero
5. Empty の NaN payload は比較対象外

この条件を満たす scalar implementation を、後続 SIMD backend の reference とする。

## Test and TDD Design

Phase 1 は constructor / state から順に失敗テストを追加し、加減算、乗算、除算を実装する。

production algorithm と独立した test oracle を用意する。

- finite binary64 を exact rational として分解する。
- test project だけで `BigInteger` を使用する。
- exact result と BCL nearest result を比較し、期待する directed endpoint を決める。
- `inari` / `kv` は secondary differential oracle とする。

Phase 3 では scalar と SIMD の canonical endpoint bit pattern が一致することを要求する。単に SIMD result が scalar result を包含するだけでは合格としない。

## SIMD Design

AVX-512 の packed embedded rounding は `Vector512<double>` を使用するため、4 区間を一括処理する batch path が主要対象となる。

```text
[-L0,U0,-L1,U1,-L2,U2,-L3,U3]
```

単一 `Interval` operator は scalar より速いことが benchmark で確認された場合だけ SIMD dispatch へ切り替える。

AVX2 / SSE2 / ARM64 は vectorized TwoSum / FMA residual を候補とし、global MXCSR を変更する native shim は初期 SIMD scope に含めない。

## API Freeze Gate

Phase 2 の完了条件:

- representative sample で利用性を確認
- edge-case matrix 成功
- exact-rational oracle 成功
- `inari` / `kv` との差異を説明
- x64 / ARM64 scalar result の canonical equality
- public API baseline を CI で監視
- zero-allocation baseline
- scalar BenchmarkDotNet baseline

性能値自体は API freeze の correctness gate にはしない。

## Failure Diagnostics

Phase 1 で executable project と tests を追加するとき、同じ PR で CI workflow を追加し、少なくとも次を failure 時にも artifact として保存する。

- test result
- standard output
- standard error
- test runner log
- reference mismatch の入力と全 result

本作業は documentation-only であり、現時点の repository には executable target と CI workflow がないため、workflow は追加していない。

## Validation

実行可能な source / project / test / workflow は変更していない。

検証内容:

- 基本設計との整合性を確認した。
- 乗算・除算の符号分類式を全 endpoint 候補と照合した。
- `inari` / `kv` の参照 commit と license を確認した。
- AVX-512 packed rounding と scalar-lane API の違いを設計へ反映した。
- branch は `main` の `ad5c058f8a4164b0c7d0763c65246914ea5d1c03` から作成した。

Build / test execution は対象外である。

## Intentionally Untouched

- `src/**`: 実装作業ではないため未変更
- `tests/**`: 実装作業ではないため未変更
- `.github/workflows/**`: executable/test target がまだないため未変更
- `tasks/**`: user instruction が直接の accepted scope であり、task entry は存在しないため未変更
- basic design: 詳細設計は別ファイルとして追加し、承認済み基本設計は変更していない。

## Remaining Risks and Open Decisions

- namespace の最終確定
- constructor と factory-only API の選択
- scalar conversion / overload
- generic math interface
- stable text format
- batch API と overlap 規則
- AVX2 / ARM64 SIMD の採否
- parsing / serialization
- decorated interval と extended division の導入時期
- reference implementation の挙動差が見つかった場合の個別判定

これらは詳細設計で未認識の事項ではなく、Phase 2 または Phase 3 の明示的な decision gate として残している。

## Next Action

PR #3 の詳細設計を確認し、承認後に Phase 1 の pure-managed scalar パイロット実装を開始する。

## Merge Boundary

本 worker は PR を merge しない。merge は repository owner が行う。
