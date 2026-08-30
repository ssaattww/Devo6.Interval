# フェーズ一覧

## 1. 文書情報

- 対象: `ssaattww/Devo6.Interval`
- 基準設計: `doc/Design/detail/IntervalArithmetic.md` 設計版5
- 詳細タスク: `tasks/tasks-status.md`
- 状態値: `未着手` / `進行中` / `Blocked` / `完了`

本書は開発フェーズ単位の目的、依存関係、成果物、完了条件を管理する。
個別の実装単位と受け入れ条件は `tasks/tasks-status.md` を正とする。

## 2. フェーズサマリー

| Phase | 名称 | 状態 | 依存 |
|---|---|---|---|
| 0 | 詳細設計・検証方針確定 | 進行中 | なし |
| 1 | managed scalar 四則演算パイロット | 未着手 | Phase 0 |
| 2 | 基本 `Interval` API 確定 | 未着手 | Phase 1 |
| 3 | SIMD backend | 未着手 | Phase 2 |
| 4A | 集合・関係・数値的属性・整数値関数 | 未着手 | Phase 3 |
| 4B | 代数関数・区間定数 | 未着手 | Phase 4A |
| 4C | 単調な初等関数 | 未着手 | Phase 4B |
| 4D | 周期・特異点・多変数関数 | 未着手 | Phase 4C |
| 4E | 非連結結果・decorated interval・I/O・分割 | 未着手 | Phase 4D |

---

## 3. Phase 0: 詳細設計・検証方針確定

### 目的

Phase 1の実装を開始しても仕様判断が揺れない状態にし、数値検証、参照実装、CI診断の方針を確定する。

### 主成果物

- `doc/Design/basic/IntervalArithmetic.md`
- `doc/Design/detail/IntervalArithmetic.md`
- `tasks/phases-status.md`
- `tasks/tasks-status.md`
- 設計レビュー結果および必要な修正記録

### 完了条件

- [ ] 詳細設計版5に対するfix verificationが完了し、`F-PR3-010`～`F-PR3-017`を含む未解決findingが残っていない。
- [ ] `Interval`、`IntervalUnion2`、`DecoratedInterval`の責務境界とPhase 1～4Eの対象範囲が設計書上で矛盾していない。
- [ ] `inari`、`kv`、ITF1788の参照commitと結果判定優先順位が固定されている。
- [ ] exact-rational oracle、reference corpus、conformance matrix、x64/ARM64比較の検証方針が設計書に定義されている。
- [ ] Phase 1最初のPRで、test result、stdout、stderr、diagnostic log等を保存するdiagnostic artifact workflowを追加することが必須タスクとして登録されている。
- [ ] Phase 1～4Eのタスク、依存関係、受け入れ条件が `tasks/tasks-status.md` に登録されている。
- [ ] Phase 0では実行可能projectが存在しないため、CI workflowを先行追加しない方針が維持されている。

---

## 4. Phase 1: managed scalar 四則演算パイロット

### 目的

SIMDを使用せず、pure-managed実装だけで区間の状態、外向き丸め、四則演算、検証基盤を成立させる。
このフェーズで数値意味論と基本操作の実装可能性を検証する。

### 対象

- `net10.0`
- `Interval`
- constructor / `TryCreate` / `Point`
- `Empty` / `Entire` / `Zero`
- `Lower` / `Upper`
- `IsEmpty` / `IsEntire` / `IsSingleton`
- equality / Hash / diagnostic `ToString`
- unary `-`, `+`, `-`, `*`, `/`
- pure-managed directed rounding
- exact-rational oracle
- IEEE 1788.1-oriented conformance
- pinned reference corpus
- Linux x64 / ARM64 CI
- diagnostic artifacts

### 完了条件

- [ ] solution、production project、test projectが `net10.0` でbuild/test可能である。
- [ ] Linux x64とLinux ARM64で同一commit、同一test assembly、同一corpusを実行するCIが存在する。
- [ ] CIは成功/失敗にかかわらず、少なくともtest result、stdout、stderr、diagnostic log、runtime/OS/architecture/CPU feature、reference-lock、conformance summary、canonical result corpusをartifactとして保存する。
- [ ] `Interval`のcanonical stateが設計どおりで、`default(Interval) == Interval.Zero`、Empty endpoint、signed zero、不正constructor入力が決定的fixtureで検証される。
- [ ] add/sub/mul/divのdirected rounding primitiveがexact oracleに対してtightであり、通常入力に対する無条件1 ULP拡張を行わない。
- [ ] subnormal、threshold前後、residual tie、positive/negative finite overflow、Infinity operandを含む決定的fixtureが成功する。
- [ ] 四則演算kernelがEmpty伝播、zero classification、zero-containing denominator等の設計matrixを満たす。
- [ ] ITF1788/repository-defined conformanceのPhase 1 required caseが全件成功する。
- [ ] `tests/ReferenceData/reference-lock.json` とcanonical corpusが生成・固定され、参照SHA、generator hash、corpus SHA-256を追跡できる。
- [ ] x64/ARM64で生成したcanonical resultsがcaseId順でbyte-for-byte一致する。
- [ ] production packageはBCL以外のruntime dependency、CPU global rounding-mode変更、production hot pathの`BigInteger`を持たない。
- [ ] basic四則演算でheap allocationが0であることを計測で確認する。
- [ ] sample/API evaluation reportが作成され、Phase 2のAPI判断材料が揃っている。

---

## 5. Phase 2: 基本 `Interval` API 確定

### 目的

Phase 1の実使用結果を基に、基本 `Interval` APIをfreezeし、Phase 3以降でbackendを変更しても公開契約が変わらない状態にする。

### 確定対象

- package / assembly / namespace
- constructor / factory
- property
- Empty / Entire / Zero
- 四則演算operator
- equality / Hash
- 例外
- signed zero
- `ToString`
- scalar conversion / overload
- generic math interface

### 完了条件

- [ ] namespace、constructor/factory方針、scalar overload/conversion、generic math採否、diagnostic/正式formatの境界が明文化されている。
- [ ] representative calculationがoperator中心で自然に記述できることをsampleで確認する。
- [ ] invalid constructor入力と、数学的演算結果としてのEmptyがAPI上明確に区別される。
- [ ] Empty/Entire/signed zero/equality/Hash/四則演算の契約がPhase 1 fixtureと一致する。
- [ ] exact oracle、conformance required case、x64/ARM64 corpusで未解決差異がない。
- [ ] public API baselineがrepositoryへ保存され、自動または明示的な差分確認ができる。
- [ ] basic operationのheap allocation 0が維持される。
- [ ] Phase 2完了後の基本 `Interval` API破壊的変更は原則禁止とし、必要時は `doc/Design/BreakingChanges.md` に理由と移行方法を記録する運用が確定している。

---

## 6. Phase 3: SIMD backend

### 目的

Phase 2でfreezeした公開APIとscalar意味論を変更せず、SIMD経路を追加して実workloadを高速化する。

### 対象候補

- AVX-512F
- AVX2 + FMA
- AVX2 without FMA
- AVX + FMA
- SSE2
- ARM64 AdvSimd
- scalar fallback

### 完了条件

- [ ] scalar referenceとSIMD backendのdifferential test基盤が存在する。
- [ ] capabilityは`Avx512F`、`Avx2`、`Avx`、`Fma`、`Sse2`、`AdvSimd.Arm64`を独立判定し、FMAを他ISAへ暗黙従属させない。
- [ ] SIMD load/storeとbatch add/subが実装され、scalar結果とcanonical endpoint bitsが一致する。
- [ ] mul/divの各候補backendはspecial value、subnormal、overflow、threshold fixtureを含むdifferential testに成功する。
- [ ] unsupported feature combinationでは必ず正しいfallbackへ遷移し、公開APIの結果が変わらない。
- [ ] x64/ARM64および利用可能な各backend間でcanonical endpointがbitwise一致する。
- [ ] production dispatchへ入るkernelは、correctness gateと実workload benchmarkの両方を通過している。
- [ ] benchmark上改善が確認できないkernelはproduction dispatchへ採用しない。
- [ ] SIMD導入による基本 `Interval` 公開APIの変更がない。

---

## 7. Phase 4A: 集合・関係・数値的属性・整数値関数

### 目的

bare `Interval`の集合操作、関係、数値的属性、整数値関数を追加する。

### 完了条件

- [ ] `Contains`、`IsBounded`、`Intersect`、`ConvexHull`が設計されたEmpty/Infinity semanticsを満たす。
- [ ] subset/interior/disjoint/precedes/endpoint-wise less等はnamed APIとして実装され、通常比較operatorを導入しない。
- [ ] `IntervalOverlap` 16状態すべてに最低1件の決定的fixtureがあり、全状態でinverse consistencyが成立する。
- [ ] `Width`、`Midpoint`、`Radius`、`Magnitude`、`Mignitude`がEmpty/Entire/unbounded/singletonを含むmatrixを満たす。
- [ ] `Abs`、`Sign`、pointwise min/max、Floor/Ceiling/Truncate/Roundがcanonical zeroとEmpty伝播を満たす。
- [ ] intersection/hullのcommutative/idempotent、subset関係等のproperty testが成功する。
- [ ] Midpoint tie policyとPhase 4A public namingがreviewで確定している。
- [ ] exact/referenceとの差異なし、x64/ARM64一致、backend間bitwise一致、API baseline更新を満たす。

---

## 8. Phase 4B: 代数関数・区間定数

### 目的

correctly directedな代数関数とtight interval constantsを追加する。

### 完了条件

- [ ] `IntervalConstants`のPi/HalfPi/TwoPi/E/Ln2/Ln10/Sqrt2がMPFR directed conversion由来の固定endpointで真値をtightに包含する。
- [ ] `Reciprocal`がzero-only、zero-touch、strict zero-crossing、positive/negative domain matrixを満たす。
- [ ] `Square`が`X*X`への単純委譲ではなく依存性拡大を避け、`Square(X)`が`X*X`のsubset-or-equalである。
- [ ] `Sqrt`がdomain clipping、subnormal、小入力scale、directed correctionを含む決定的fixtureを満たす。
- [ ] integer `Pow`/`Root`が偶奇、negative exponent、zero crossing、`int.MinValue`、invalid degreeを含むmatrixを満たす。
- [ ] `FusedMultiplyAdd`が`(X*Y)+Z`への単純委譲ではなく1回丸めのendpoint primitiveを使用し、同一semanticsでsubset-or-equalを満たす。
- [ ] MPFR reference corpus生成・固定手順が整備され、Phase 4C/Dで利用可能である。
- [ ] exact/MPFR/referenceとの差異なし、x64/ARM64一致、backend間bitwise一致、API baseline更新を満たす。

---

## 9. Phase 4C: 単調な初等関数

### 目的

domain clippingとcertified directed endpoint kernelで評価可能な単調初等関数を追加する。

### 対象

- Exp / Exp2 / Exp10
- Log / Log2 / Log10
- Sinh / Cosh / Tanh
- Asinh / Acosh / Atanh
- Asin / Acos / Atan

### 完了条件

- [ ] 各functionのdomain、open boundary、Infinity limit、overflow/underflow/subnormal matrixが設計どおりである。
- [ ] endpoint backendは誤差上限・方向補正を証明したmanaged実装、検証済みcorrectly-rounded port、またはdirected native backendのいずれかであり、`Math.*`単体を包含保証の根拠にしない。
- [ ] MPFR RNDD/RNDU primary corpusとの差異がない、または承認済み差異としてcase単位で記録されている。
- [ ] clippingによりdomain内点が存在しない場合はEmpty、open boundaryへ接する場合は正しいlimitを返す。
- [ ] 全support platformでcore公開functionが利用でき、通常入力を`PlatformNotSupportedException`へ逃がさない。
- [ ] x64/ARM64一致、backend間bitwise一致、failure artifact追跡、API baseline更新を満たす。

---

## 10. Phase 4D: 周期・特異点・多変数関数

### 目的

高精度range reduction、pole/branch cut、複数変数domainを必要とする関数を追加する。

### 対象

- periodic reducer
- Sin / Cos / Tan
- Atan2
- positive-base general interval Pow

### 完了条件

- [ ] 全binary64範囲で象限・critical point・pole判定を誤らないhigh-precision periodic reducerが存在し、`Math.PI`や`% (2*Math.PI)`だけに依存しない。
- [ ] Sin/Cosはcritical latticeを検出し、必要時にexact `-1` / `+1`を含むtight enclosureを返す。
- [ ] Tanはpoleなしのbranch、pole crossing、pole-only inputを区別し、Entire/Emptyを設計どおり返す。
- [ ] Atan2は全sign-class直積、axis、origin、negative-x branch cut 6class matrixを満たし、signed-zeroを集合として正規化する。
- [ ] general positive-base Powはzero-base boundaryをinterval extension層で処理し、`0^0`/`0^negative`をscalar point kernelへ渡さない。
- [ ] review-regression fixture `F-PR3-011`、`F-PR3-012`を含む決定的fixtureが成功する。
- [ ] MPFR/referenceとの差異なし、x64/ARM64一致、backend間bitwise一致、failure artifact追跡、API baseline更新を満たす。

---

## 11. Phase 4E: 非連結結果・decorated interval・I/O・分割

### 目的

bare `Interval`では表現できない2成分結果、decoration/NaI、exact I/O、分割機能を別型・別APIとして追加する。

### 対象

- `IntervalUnion2`
- extended division / reciprocal
- reverse multiplication
- cancellative operations
- `DecoratedInterval` / NaI
- decorated arithmetic/math
- exact/outward parsing / formatting
- binary interchange
- split / bisect

### 完了条件

- [ ] `IntervalUnion2`はCount 0/1/2のcanonical state、value equality/Hash/indexerを持ち、closureが同一点で接する2 componentを勝手にmergeしない。
- [ ] `DivideToUnion` / `ReciprocalToUnion`がzero-only、one-sided zero、strict zero-crossingを含むmatrixを満たし、`ordinary division == DivideToUnion(...).ConvexHull`が成立する。
- [ ] `ReverseMultiply`がfactor=0を除外しないcontractor semanticsを満たす。
- [ ] cancellative operationがEmpty/common/unbounded 3x3 matrixとexact width relationを満たす。
- [ ] `default(DecoratedInterval).IsNaI == true`、NaI/value equality/semantic equalityが設計どおり分離される。
- [ ] decoration result capがbounded/unbounded/Emptyのresult stateを反映し、unbounded resultを`Com`、Empty resultを`Dac/Com`として返さない。
- [ ] parserはdecimalをexact rationalとして外向き丸めし、NaN/reversed/invalid Infinity等を拒否する。
- [ ] parserは入力長、significand桁数、exponent桁数等のresource limitを持ち、untrusted inputに対するsecurity reviewを通過する。
- [ ] exact formatおよびbinary interchangeのround-tripがcanonical intervalへbitwise一致する。
- [ ] `TrySplitAt` / `TryBisect`はchildrenが元区間をcoverし、それぞれが元区間のsubsetである。
- [ ] `F-PR3-010`、`F-PR3-013`～`F-PR3-017`のreview-regression fixtureがすべて成功する。
- [ ] x64/ARM64一致、backend間bitwise一致、failure artifact追跡、API baseline更新を満たす。

---

## 12. 全Phase共通ゲート

Phase 1以降のsource実装はTDDを基本とし、論理単位ごとに失敗testを先に追加して失敗を確認してからproduction implementationを追加する。

各公開function/typeは、次をすべて満たすまで完了扱いにしない。

- [ ] 設計上の意味論/domain matrixがreview済みである。
- [ ] required deterministic fixtureが成功する。
- [ ] exact oracle / MPFR / referenceとの差異がない、または承認済み差異として追跡可能である。
- [ ] Linux x64 / ARM64のcanonical endpoint/result corpusが一致する。
- [ ] 複数backendがある場合はcanonical endpoint bitsが一致する。
- [ ] failure artifactから入力、分岐、exact result、実装結果、reference result、差異理由を追跡できる。
- [ ] public API変更がある場合はAPI baselineが更新されている。
- [ ] 破壊的変更が必要な場合は `doc/Design/BreakingChanges.md` に理由と移行方法が記録されている。
- [ ] 対象PRのCI確認ではPR current HEAD SHAとworkflow runの`head_sha`が一致している。matching runがなければCI未実施として扱う。
