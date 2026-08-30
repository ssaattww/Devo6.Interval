# タスク一覧

## 1. 文書情報

- 対象: `ssaattww/Devo6.Interval`
- 基準設計: `doc/Design/detail/IntervalArithmetic.md` 設計版5
- フェーズ定義: `tasks/phases-status.md`
- 状態値: `未着手` / `進行中` / `Blocked` / `完了`

本書は実装・検証をレビュー可能な論理単位へ分解したタスク一覧である。
受け入れ条件は「実装した」ではなく、観測可能なtest、corpus、artifact、API baseline、benchmark等で完了を判定できる形で記載する。

## 2. 共通完了条件

source実装を含むタスクには、個別条件に加えて次を適用する。

- [ ] TDDで失敗testを先に追加し、対象仕様を理由に失敗することを確認してからproduction implementationを追加する。
- [ ] RedとGreenをレビュー可能な小さい論理単位でcommit/pushする。
- [ ] Empty、signed zero、Infinity、subnormal、finite overflow等、対象機能に関係する境界値を決定的fixtureへ含める。
- [ ] exact oracle / MPFR / pinned referenceのうち対象機能に適用可能なものと比較し、未承認差異を残さない。
- [ ] Linux x64 / ARM64でcanonical resultが一致する。複数backendがある場合はbackend間でもcanonical endpoint bitsが一致する。
- [ ] 失敗時にtest result、stdout、stderr、diagnostic log、および原因調査に必要な入力・分岐・reference情報をartifactから取得できる。
- [ ] 対象PRのCIはPR current HEAD SHAとrunの`head_sha`が一致するrunだけを採用する。matching runがなければCI未実施とする。
- [ ] 公開APIを変更した場合はAPI baselineを更新する。破壊的変更の場合は `doc/Design/BreakingChanges.md` に理由と移行方法を記録する。
- [ ] 完了時に対象タスクのstatus、詳細report、PR上の簡易reportを更新する。

infra/documentationのみのタスクにはTDDを要求しないが、内容の整合確認とレビュー可能なcommit単位は維持する。

## 3. タスクサマリー

| ID | Phase | タスク | 状態 | 依存 |
|---|---|---|---|---|
| P0-001 | 0 | 詳細設計版5 fix verification | 進行中 | なし |
| P0-002 | 0 | フェーズ・タスク管理基盤作成 | 進行中 | なし |
| P1-001 | 1 | solution/project/CI/diagnostic artifact基盤 | 未着手 | P0-001, P0-002 |
| P1-002 | 1 | Interval construction/state/normalization | 未着手 | P1-001 |
| P1-003 | 1 | exact-rational oracle・boundary corpus | 未着手 | P1-001 |
| P1-004 | 1 | directed add/sub primitive | 未着手 | P1-003 |
| P1-005 | 1 | interval add/sub/unary minus | 未着手 | P1-002, P1-004 |
| P1-006 | 1 | directed multiply primitive | 未着手 | P1-003 |
| P1-007 | 1 | interval multiplication | 未着手 | P1-002, P1-006 |
| P1-008 | 1 | directed divide primitive | 未着手 | P1-003 |
| P1-009 | 1 | interval division | 未着手 | P1-002, P1-008 |
| P1-010 | 1 | Phase 1 conformance harness | 未着手 | P1-002, P1-005, P1-007, P1-009 |
| P1-011 | 1 | pinned inari/kv corpus・reference lock | 未着手 | P1-010 |
| P1-012 | 1 | sample・API evaluation report | 未着手 | P1-011 |
| P2-001 | 2 | core API review/freeze | 未着手 | P1-012 |
| P2-002 | 2 | conversion/generic math/format判断 | 未着手 | P2-001 |
| P2-003 | 2 | public API baseline・breaking-change運用確定 | 未着手 | P2-002 |
| P3-001 | 3 | scalar/SIMD differential・capability基盤 | 未着手 | P2-003 |
| P3-002 | 3 | SIMD layout/load/store/batch add-sub | 未着手 | P3-001 |
| P3-003 | 3 | AVX-512 directed mul/div candidate | 未着手 | P3-001 |
| P3-004 | 3 | AVX2+FMA mul/div candidate | 未着手 | P3-001 |
| P3-005 | 3 | AVX2 no-FMA/SSE2/ARM64候補評価 | 未着手 | P3-001 |
| P3-006 | 3 | production dispatch・fallback・benchmark gate | 未着手 | P3-002～P3-005 |
| P4A-001 | 4A | Contains / IsBounded | 未着手 | P3-006 |
| P4A-002 | 4A | Intersect / ConvexHull | 未着手 | P4A-001 |
| P4A-003 | 4A | relation named API | 未着手 | P4A-002 |
| P4A-004 | 4A | IntervalOverlap | 未着手 | P4A-003 |
| P4A-005 | 4A | numeric properties | 未着手 | P4A-004 |
| P4A-006 | 4A | Abs / Sign / pointwise min-max | 未着手 | P4A-005 |
| P4A-007 | 4A | Floor/Ceiling/Truncate/Round | 未着手 | P4A-006 |
| P4A-008 | 4A | Phase 4A API/conformance close | 未着手 | P4A-007 |
| P4B-001 | 4B | tight IntervalConstants | 未着手 | P4A-008 |
| P4B-002 | 4B | Reciprocal | 未着手 | P4B-001 |
| P4B-003 | 4B | Square | 未着手 | P4B-002 |
| P4B-004 | 4B | Sqrt | 未着手 | P4B-003 |
| P4B-005 | 4B | integer Pow / Root | 未着手 | P4B-004 |
| P4B-006 | 4B | FusedMultiplyAdd | 未着手 | P4B-005 |
| P4B-007 | 4B | MPFR corpus・elementary endpoint backend qualification | 未着手 | P4B-006 |
| P4C-001 | 4C | Exp / Exp2 / Exp10 | 未着手 | P4B-007 |
| P4C-002 | 4C | Log / Log2 / Log10 | 未着手 | P4C-001 |
| P4C-003 | 4C | Sinh / Cosh / Tanh | 未着手 | P4C-002 |
| P4C-004 | 4C | Asinh/Acosh/Atanh/Asin/Acos/Atan | 未着手 | P4C-003 |
| P4C-005 | 4C | Phase 4C backend/API gate | 未着手 | P4C-004 |
| P4D-001 | 4D | high-precision periodic reducer | 未着手 | P4C-005 |
| P4D-002 | 4D | Sin / Cos | 未着手 | P4D-001 |
| P4D-003 | 4D | Tan | 未着手 | P4D-002 |
| P4D-004 | 4D | Atan2 | 未着手 | P4D-003 |
| P4D-005 | 4D | positive-base general interval Pow | 未着手 | P4D-004 |
| P4D-006 | 4D | Phase 4D backend/API gate | 未着手 | P4D-005 |
| P4E-001 | 4E | IntervalUnion2 | 未着手 | P4D-006 |
| P4E-002 | 4E | DivideToUnion | 未着手 | P4E-001 |
| P4E-003 | 4E | ReciprocalToUnion | 未着手 | P4E-002 |
| P4E-004 | 4E | ReverseMultiply | 未着手 | P4E-003 |
| P4E-005 | 4E | cancellative operations | 未着手 | P4E-004 |
| P4E-006 | 4E | Decoration / default NaI | 未着手 | P4E-005 |
| P4E-007 | 4E | DecoratedInterval equality/canonicalization | 未着手 | P4E-006 |
| P4E-008 | 4E | decorated arithmetic/math | 未着手 | P4E-007 |
| P4E-009 | 4E | exact/outward parser・formatter | 未着手 | P4E-008 |
| P4E-010 | 4E | binary interchange | 未着手 | P4E-009 |
| P4E-011 | 4E | split / bisect | 未着手 | P4E-010 |
| P4E-012 | 4E | Phase 4E security/conformance/API gate | 未着手 | P4E-011 |

---

# Phase 0

## P0-001 詳細設計版5 fix verification

**設計参照:** §1, §51, §52

### 受け入れ条件

- [ ] 設計版5のimmutable HEADを明示してfix verificationを実施する。
- [ ] `F-PR3-010`～`F-PR3-017`をfinding ID単位で再確認し、各findingのrequired actionと本文規範が対応している。
- [ ] 新規blocking/high/medium/low findingがない、または全件修正後に同一reviewerによる必要なclosureを完了する。
- [ ] review verdictとreviewed HEADを詳細reportへ保存する。
- [ ] 詳細設計の状態が実装開始可能であることを確認できるまでPhase 1 source実装を開始しない。

## P0-002 フェーズ・タスク管理基盤作成

**設計参照:** §3, §44, §46, §52

### 受け入れ条件

- [ ] `tasks/phases-status.md` にPhase 0、1、2、3、4A～4Eが順序・依存関係付きで登録されている。
- [ ] 各Phaseに目的、主対象、判定可能な完了条件がある。
- [ ] `tasks/tasks-status.md` にPhase 1実装開始順序とPhase 4 TDD順序を漏れなく反映している。
- [ ] 各実装タスクに、少なくとも機能結果、境界fixture、reference/oracle、architecture/backend整合性のうち適用対象となる具体条件が記載されている。
- [ ] Phase 1最初のタスクにdiagnostic artifact workflow追加が必須として明記されている。
- [ ] タスク一覧と詳細設計版5の間に対象機能の矛盾がない。

---

# Phase 1

## P1-001 solution/project/CI/diagnostic artifact基盤

**設計参照:** §4, §16, §52.1

### 受け入れ条件

- [ ] `net10.0` production projectとtest projectを含むsolutionがbuild可能である。
- [ ] Linux x64 (`ubuntu-24.04`) とLinux ARM64 (`ubuntu-24.04-arm`) のCI jobが同一commit・同一test assembly・同一corpusを実行する。
- [ ] workflowは成功/失敗を問わず `if: always()` 相当でtest result、stdout、stderr、diagnostic logをartifact化する。
- [ ] artifactへruntime、OS、architecture、CPU features、reference-lock、conformance summary、canonical result corpusを含められる構造がある。
- [ ] architecture比較jobがcaseId順の `canonical-results.jsonl` をbyte-for-byte比較し、SHA-256と全差分を保存できる。
- [ ] test failure時に、該当caseIdと入力をartifactから特定できる最小diagnostic形式を定義する。
- [ ] 実行可能projectが追加された最初のPR内でこのworkflowも同時に追加される。

## P1-002 Interval construction/state/normalization

**設計参照:** §5, §6, §12, §14.2～14.4

### 受け入れ条件

- [ ] `public readonly struct Interval : IEquatable<Interval>` が成立する。
- [ ] `default(Interval) == Interval.Zero` がtrueで、Zero公開endpointはlower `-0.0`、upper `+0.0`へcanonicalizeされる。
- [ ] `Interval.Empty.Lower == +Infinity`、`Interval.Empty.Upper == -Infinity`、`Interval.Entire == [-Infinity,+Infinity]`をfixtureで確認する。
- [ ] constructorは`lower > upper`、NaN endpoint、lower `+Infinity`、upper `-Infinity`を`ArgumentException`で拒否する。
- [ ] `TryCreate`は同じinvalid matrixで`false`かつout=`Interval.Empty`を返す。
- [ ] `Point`は有限`double`だけを受け入れ、NaN/±Infinityを拒否する。
- [ ] `IsEmpty`、`IsEntire`、`IsSingleton`がEmpty/Entire/finite singleton/zero variants/unbounded matrixを満たす。
- [ ] Empty同士のequality、signed-zero入力差を無視するnonempty equality、固定Empty Hash、canonical zero Hashを検証する。
- [ ] public APIからprivate layoutやNaN payloadを観測できない。

## P1-003 exact-rational oracle・boundary corpus

**設計参照:** §13, §15

### 受け入れ条件

- [ ] 任意のfinite binary64をexactな `significand * 2^exponent` へ分解できる。
- [ ] add/sub/mulは`BigInteger` exact value、divはexact rational numerator/denominatorとして比較できる。
- [ ] positive finite overflowでUp=`+Infinity`、Down=`+MaxValue`、negative finite overflowでUp=`-MaxValue`、Down=`-Infinity`をoracleが返す。
- [ ] finite overflowとInfinity operand由来のexact Infinityを別caseとして記録する。
- [ ] `-M <= R <= M`ではnearest finite resultをexact rationalへ戻してBitIncrement/BitDecrement要否を判定する。
- [ ] threshold bits `2^-969` previous/exact/next、`2^918` previous/exact/next、minimum subnormalを固定fixtureとして保持する。
- [ ] oracle実装はtest/reference側にのみ存在し、production packageへ`BigInteger`を持ち込まない。

## P1-004 directed add/sub primitive

**設計参照:** §7, §8

### 受け入れ条件

- [ ] `AddUp` / `AddDown`がfinite non-overflow入力でTwoSum residualに基づき必要時だけ隣接doubleへ補正する。
- [ ] exact resultの場合は無条件に1 ULP広げない。
- [ ] `SubtractUp/Down`が設計のsymmetryに従う。
- [ ] finite overflowの4方向結果がexact oracleと一致する。
- [ ] primitiveへ渡してはいけないNaN、`+Infinity + -Infinity`等をcaller contract/testで明示する。
- [ ] randomized finite corpusと決定的tie/overflow fixtureでexact oracleとの差異が0件である。

## P1-005 interval add/sub/unary minus

**設計参照:** §11.1

### 受け入れ条件

- [ ] いずれかoperandがEmptyならadd/sub結果はEmpty、unary minusのEmptyはEmptyである。
- [ ] `[a,b]+[c,d] = [RD(a+c),RU(b+d)]`を満たす。
- [ ] `[a,b]-[c,d] = [RD(a-d),RU(b-c)]`を満たす。
- [ ] `-[a,b]=[-b,-a]`を満たし、double negationで元のcanonical intervalへ戻る。
- [ ] Zero identity、add commutativity、result invariantのproperty testが成功する。
- [ ] unbounded intervalとsigned-zero endpointを含むfixtureがexact oracle/referenceと一致する。

## P1-006 directed multiply primitive

**設計参照:** §9, §15

### 受け入れ条件

- [ ] `abs(product) >= 2^-969`でFMA residual通常経路、`abs(product) < 2^-969`でscaled経路を使用する。
- [ ] `2^-969` previous/exact/nextで経路境界が設計どおり選択される。
- [ ] scaled pathの`t<s`、`t>s`、`t==s && s2>0`、`t==s && s2<0`、exactを固定binary64 witnessで検証する。
- [ ] positive/negative finite overflowのUp/Down結果がexact oracleと一致する。
- [ ] Infinity operandはfinite overflowと別分岐で扱い、`0 * Infinity`をprimitiveへ渡さない。
- [ ] randomized finite corpusでMultiplyUp/Downとexact oracleの差異が0件である。

## P1-007 interval multiplication

**設計参照:** §11.2, §11.3

### 受け入れ条件

- [ ] Z/P/N/M sign classの全組合せが設計表どおりのendpoint候補を選ぶ。
- [ ] Zero classを最初に処理し、`0 * Infinity`をdirected primitiveへ渡さない。
- [ ] M×Mはlowerに`min(RD(a*d),RD(b*c))`、upperに`max(RU(a*c),RU(b*d))`を使用する。
- [ ] mul commutativity、Zero multiplication、result invariantのproperty testが成功する。
- [ ] finite/unbounded/subnormal/overflowを含む決定的matrixがexact oracleおよびcompatibleなreference resultと一致する。

## P1-008 directed divide primitive

**設計参照:** §10, §15

### 受け入れ条件

- [ ] denominatorを正符号化した後にresidual比較を行う。
- [ ] `abs(x) < 2^-969`のsmall numerator経路と`abs(y) >= 2^918`のlarge-denominator early return境界を固定fixtureで検証する。
- [ ] early returnはpositive exact quotientでUp=`+2^-1074`/Down=`+0.0`、negativeでUp=`+0.0`/Down=`-2^-1074`を返す。
- [ ] `r==xn && r2>0`、`r==xn && r2<0`、`r<xn`、`r>xn`、exactの全caseを固定witnessで検証する。
- [ ] denominator zero、0/0、Infinity/Infinity、NaNをprimitiveへ渡さない。
- [ ] positive/negative finite overflowとrandomized finite corpusがexact oracleと一致する。

## P1-009 interval division

**設計参照:** §11.4～11.7

### 受け入れ条件

- [ ] Empty operandを伝播する。
- [ ] denominatorがstrict positive/strict negativeの場合、numerator Z/P/N/Mの各式が設計表と一致する。
- [ ] `A/[0,0] -> Empty`、`Zero/[0,d] -> Zero`、`Zero/[c,0] -> Zero`を満たす。
- [ ] one-sided zero denominator `[0,d]` / `[c,0]`でZ/P/N/Mの結果matrixを全件fixture化する。
- [ ] strict zero-crossing denominatorで`Zero/B -> Zero`、それ以外はbare hullとしてEntireを返す。
- [ ] reciprocalを一度作って乗算する二重丸め実装を採用しない。
- [ ] `[1,2]/[0,0] -> Empty`、`[1,2]/[-1,1] -> Entire`等の代表fixtureが設計どおりである。

## P1-010 Phase 1 conformance harness

**設計参照:** §14

### 受け入れ条件

- [ ] Empty/Entire/numsToInterval/inf/sup/isEmpty/isEntire/isSingleton/equal/neg/add/sub/mul/divをrequired matrixへ登録する。
- [ ] ITF1788の固定commitからconstructor/element/bool等の指定sourceを抽出する。
- [ ] `IsSingleton`はrepository-defined matrixを明示的sourceとして扱う。
- [ ] 各caseにsource、path/testcase、adaptation、required/deferred/excluded/approved-deviation、expectedを保存する。
- [ ] 宣言operationのsource extraction件数が0件の場合、passではなくsource extraction errorで失敗する。
- [ ] required caseが全件成功し、conformance summaryをartifactへ保存する。

## P1-011 pinned inari/kv corpus・reference lock

**設計参照:** §2.1, §13.3, §13.4, §49

### 受け入れ条件

- [ ] `reference-lock.json`にinari SHA、kv SHA、ITF1788 SHA、adapter/generator hash、toolchain/target triple、generator command、corpus SHA-256、license/NOTICE pathを記録する。
- [ ] corpusはJSON Lines、binary64数値は16桁hex bits、caseId sortで生成する。
- [ ] generator iteration orderを変更してもcaseId sort後のcorpus bytesが同じである。
- [ ] inariはinterval semantics/reference、kvはcompatible directed primitive referenceとして役割を分離する。
- [ ] kvのzero-containing interval divisionをoracleとして使用しない。
- [ ] third-party code/testを翻案した箇所はcommit SHAとMIT notice等の出典要件を満たす。

## P1-012 sample・API evaluation report

**設計参照:** §3.3, §5.9, §5.10, §46, §50, §52.1

### 受け入れ条件

- [ ] constructor、Point、Empty/Entire、四則operatorを使うrepresentative sampleを作成する。
- [ ] invalid inputと数学的Empty resultの違いがsampleから明確に理解できる。
- [ ] basic四則演算のheap allocation 0をbenchmarkまたはallocation計測で確認する。
- [ ] namespace、constructor vs factory-only、scalar overload/conversion、generic math、正式formatの未確定事項ごとに採否判断材料をreport化する。
- [ ] Phase 1のcorrectness、conformance、x64/ARM64 corpus一致、known limitationを1つのevaluation reportへまとめる。

---

# Phase 2

## P2-001 core API review/freeze

**設計参照:** §3.3, §5, §46

### 受け入れ条件

- [ ] assembly/package/namespace、constructor/factory、基本property、定数、四則operator、equality/Hash、例外、signed zeroをreviewで確定する。
- [ ] representative sampleが過度なfactory呼出しやbackend依存なしで記述できる。
- [ ] `Interval` public surfaceがinternal negated-lower layoutやSIMD型に依存しない。
- [ ] Phase 1 fixture/conformance結果を変更せずにAPIを確定できる。
- [ ] 基本API freezeのreview記録と確定日/対象HEADを保存する。

## P2-002 conversion/generic math/format判断

**設計参照:** §5.9, §5.10, §50

### 受け入れ条件

- [ ] scalar implicit/explicit conversion、scalar overloadの採否を決定し、採用時は曖昧性とrounding semanticsをtestで固定する。
- [ ] `INumber<TSelf>`等generic math interfaceの採否を、全順序がないことを含む契約適合性で判断する。
- [ ] diagnostic `ToString`と将来のpersistent/wire formatを混同しないsurfaceを確定する。
- [ ] 不採用項目も理由を設計/decision記録へ残し、後続Phaseが再判断を暗黙に行わない。

## P2-003 public API baseline・breaking-change運用確定

**設計参照:** §46

### 受け入れ条件

- [ ] public API baseline fileをrepositoryへ保存する。
- [ ] CIまたは明示検証でbaseline差分を検出できる。
- [ ] Empty/Entire/invalid constructor/signed zero/四則fixture/exact oracle/conformance/x64-ARM64一致がfreeze HEADで成功する。
- [ ] basic operation allocation 0がfreeze HEADで成功する。
- [ ] Phase 2後の破壊的変更は `doc/Design/BreakingChanges.md` 更新なしでは受け入れない運用を明文化する。

---

# Phase 3

## P3-001 scalar/SIMD differential・capability基盤

**設計参照:** §17.1, §17.2

### 受け入れ条件

- [ ] scalar reference backendを常時選択できるtest hookがある。
- [ ] SIMD backendとの同一入力differentialをcaseId単位で比較できる。
- [ ] `Avx512F`、`Avx2`、`Avx`、`Fma`、`Sse2`、`AdvSimd.Arm64`を独立判定する。
- [ ] feature combinationを強制/模擬してfallback選択を検証できるtest構造がある。
- [ ] mismatch artifactにscalar bits、SIMD bits、selected capability pathを保存する。

## P3-002 SIMD layout/load/store/batch add-sub

**設計参照:** §17.3, §17.4

### 受け入れ条件

- [ ] batch layout `[-L0,U0,-L1,U1,...]`をbackend内部で実装し、public layout contractにはしない。
- [ ] load/storeでEmpty/Entire/Zero/normal intervalのcanonical stateが壊れない。
- [ ] batch add/subがscalar resultと全caseでbitwise一致する。
- [ ] AVX-512の4 interval batchで末尾4未満が正しいfallbackへ流れる。
- [ ] basic/special/subnormal corpusでdifferential mismatchが0件である。

## P3-003 AVX-512 directed mul/div candidate

**設計参照:** §17.2～17.4

### 受け入れ条件

- [ ] AVX-512F available環境でpacked directed mul/div candidateを実装する。
- [ ] Empty/zero-containing division/Infinity等の区間level special caseはscalar semanticsと一致する。
- [ ] threshold/subnormal/finite overflow fixtureでscalar canonical bitsと一致する。
- [ ] AVX-512F unavailable時にこのpathへ到達しない。
- [ ] benchmarkはscalar baselineと同一workload/同一corpusで取得する。

## P3-004 AVX2+FMA mul/div candidate

**設計参照:** §17.2

### 受け入れ条件

- [ ] AVX2とFMAを独立featureとして確認し、両方available時だけcandidateを選択する。
- [ ] vector TwoSum add/subとFMA residual mul/div candidateがscalar canonical bitsと一致する。
- [ ] AVX2 available/FMA unavailable環境でFMA pathを誤選択しないfixtureがある。
- [ ] special/subnormal/overflow corpusでdifferential mismatchが0件である。
- [ ] benchmark改善が測定可能な形で記録される。

## P3-005 AVX2 no-FMA/SSE2/ARM64候補評価

**設計参照:** §17.2

### 受け入れ条件

- [ ] AVX2 no-FMAではadd/sub vector pathとmul/div scalar fallbackが正しく動作する。
- [ ] SSE2 no-FMAではVector128 add/sub candidateとmul/div scalar fallbackが正しく動作する。
- [ ] ARM64 AdvSimd candidateはcorrectness proof/differentialが成立したoperationだけをvector化する。
- [ ] AVX+FMA without AVX2を独立feature combinationとして評価する。
- [ ] 各pathでscalar canonical bitsとのdifferential mismatchが0件である。
- [ ] vector化しないoperationも「未実装」ではなく正しいscalar fallbackを使用する。

## P3-006 production dispatch・fallback・benchmark gate

**設計参照:** §17.4, §18

### 受け入れ条件

- [ ] production dispatchはcorrectness gateを通過したkernelだけを候補にする。
- [ ] 同一realistic workloadでscalar比の性能改善を示せないkernelはproduction dispatchへ入れない。
- [ ] capabilityのどの組合せでも正しいbackendかscalar fallbackが選択される。
- [ ] backend選択によってpublic result、exception、signed zero、Empty semanticsが変わらない。
- [ ] scalar/SIMD canonical endpoint bitsが全production corpusで一致する。
- [ ] Phase 3時点ではscalar operatorごとのP/Invokeを導入しない。

---

# Phase 4A

## P4A-001 Contains / IsBounded

**設計参照:** §21.3, §24.1

### 受け入れ条件

- [ ] `Contains`はEmpty/NaN/±Infinityにfalse、finite memberにendpoint-inclusiveでtrueを返す。
- [ ] `Entire.Contains(±Infinity) == false`を固定fixtureで確認する。
- [ ] Zeroは`+0.0`/`-0.0`の両方を含む。
- [ ] `IsBounded`はnonemptyかつ両endpoint finiteの場合のみtrueである。

## P4A-002 Intersect / ConvexHull

**設計参照:** §21.1, §21.2

### 受け入れ条件

- [ ] Empty intersectionはEmpty、disjoint intersectionはEmptyを返す。
- [ ] hull(Empty,Y)=Y、hull(X,Empty)=Xを満たす。
- [ ] intersection/hullのcommutative/idempotent propertyが成功する。
- [ ] `Intersect(X,Y)`はX/Y双方のsubset、X/Yは`ConvexHull(X,Y)`のsubsetである。

## P4A-003 relation named API

**設計参照:** §22

### 受け入れ条件

- [ ] subset/interior/disjoint/precedes/strict precedes/weak less/strict lessをnamed APIとして提供する。
- [ ] `Entire.IsInteriorOf(Entire) == true`をextended strict relation規則で満たす。
- [ ] endpoint接触はdisjointではない。
- [ ] Emptyを含むsubset/interior/precedes/weak/strict matrixを決定的fixtureで全件検証する。
- [ ] `<`, `<=`, `>`, `>=` operatorを基本`Interval`へ追加しない。

## P4A-004 IntervalOverlap

**設計参照:** §23, §44.1 `F-PR3-017`

### 受け入れ条件

- [ ] 16状態すべてに最低1fixtureがある。
- [ ] `BothEmpty`、`FirstEmpty`、`SecondEmpty`を他状態と混同しない。
- [ ] 全状態でinverse mappingが成立する。
- [ ] `BothEmpty` inverseが`BothEmpty`であるreview-regression fixtureを含む。

## P4A-005 numeric properties

**設計参照:** §24.1～24.3

### 受け入れ条件

- [ ] WidthはEmpty=NaN、unbounded=+Infinity、singleton=+0、bounded=`RU(b-a)`である。
- [ ] MidpointはEmpty=NaN、Entire=+0、lower-unbounded=`double.MinValue`、upper-unbounded=`double.MaxValue`を返す。
- [ ] Radiusは採用Midpointに対しXをcoverする最小directed radiusの設計式を満たす。
- [ ] Magnitude/MignitudeはEmpty=NaN、zero crossing時Mignitude=+0を満たす。
- [ ] Width/Radius >= 0、Mignitude <= Magnitudeのproperty testが成功する。

## P4A-006 Abs / Sign / pointwise min-max

**設計参照:** §24.4

### 受け入れ条件

- [ ] Absがpositive/negative/zero-crossing/Empty matrixを満たす。
- [ ] Signはpoint function `-1/0/+1`のinterval extensionであり、signed zeroを±1へ誤変換しない。
- [ ] PointwiseMin/MaxはどちらかEmptyならEmptyを返す。
- [ ] 各結果endpointが設計のpointwise min/max式と一致する。

## P4A-007 Floor/Ceiling/Truncate/Round

**設計参照:** §24.5

### 受け入れ条件

- [ ] Emptyを伝播する。
- [ ] Infinity endpointを維持する。
- [ ] endpointへ同じ単調非減少point functionを適用し、追加outward roundingを行わない。
- [ ] zero endpointをcanonicalizeする。
- [ ] 未知`MidpointRounding`値で`ArgumentOutOfRangeException`を送出する。

## P4A-008 Phase 4A API/conformance close

**設計参照:** §20, §24.2, §46

### 受け入れ条件

- [ ] Phase 4A public namingをreviewで確定する。
- [ ] finite Midpoint tie policyをconformance reviewで確定しfixture化する。
- [ ] Phase 4A deterministic/property/reference testが全件成功する。
- [ ] x64/ARM64、backend間canonical bitsが一致する。
- [ ] public API baselineを更新する。

---

# Phase 4B

## P4B-001 tight IntervalConstants

**設計参照:** §25, §26.1

### 受け入れ条件

- [ ] Pi/HalfPi/TwoPi/E/Ln2/Ln10/Sqrt2をnearest point intervalではなくtight directed endpointsで固定する。
- [ ] MPFR directed conversionで生成したendpoint bitsとrepository固定値が一致する。
- [ ] build時にnetwork/native generatorを要求しない。
- [ ] periodic reduction用の高精度tableとpublic 2-endpoint constantを分離する。

## P4B-002 Reciprocal

**設計参照:** §26.2

### 受け入れ条件

- [ ] Empty->Empty、Zero->Empty、strict zero crossing->Entireを満たす。
- [ ] one-sided zero `[a,0]` / `[0,b]`でInfinity endpointとdirected finite endpointが設計表どおりである。
- [ ] strict positive/negative intervalでdirected reciprocal endpointがtightである。
- [ ] extended 2-component結果をこのbare APIへ混在させない。

## P4B-003 Square

**設計参照:** §26.3

### 受け入れ条件

- [ ] `X*X`へ委譲しない専用kernelを持つ。
- [ ] positive/negative/zero-crossing/Emptyの結果が設計式と一致する。
- [ ] zero-crossing lower endpointはcanonical `-0.0`である。
- [ ] `Square(X)`が同じXに対する`X*X`のsubset-or-equalであるpropertyが成功する。

## P4B-004 Sqrt

**設計参照:** §27.1

### 受け入れ条件

- [ ] negative-onlyはEmpty、zero-crossingはlower=-0、positiveはdirected endpointsを返す。
- [ ] `2^-969`近傍のsmall-input scale経路を固定fixtureで検証する。
- [ ] candidate平方のexact relationから必要時だけNextUp/NextDownする。
- [ ] MPFR directed resultとの差異が0件である。

## P4B-005 integer Pow / Root

**設計参照:** §27.2, §27.3

### 受け入れ条件

- [ ] `Pow(Empty,0)=Empty`、`Pow(nonempty,0)=[1,1]`を満たす。
- [ ] positive odd/even、negative exponent、zero-only/zero-touch/strict zero-crossingを決定的matrixで検証する。
- [ ] `int.MinValue`を安全に符号+`uint` magnitudeへ分解する。
- [ ] Rootで`degree<=0`は`ArgumentOutOfRangeException`、degree=1はinputを返す。
- [ ] even Root negative-onlyはEmpty、odd Rootは全実数上の単調増加結果を返す。
- [ ] candidate^nと入力のexact relationで隣接補正を決定する。

## P4B-006 FusedMultiplyAdd

**設計参照:** §27.4

### 受け入れ条件

- [ ] endpoint primitiveがexact `x*y+z`を1回だけdirected roundingする。
- [ ] `(X*Y)+Z`へ委譲しない。
- [ ] `FMA(X,Y,Z)`が同一set semanticsで`(X*Y)+Z`のsubset-or-equalである。
- [ ] overflow/subnormal/cancellation fixtureがMPFR/exact referenceと一致する。

## P4B-007 MPFR corpus・elementary endpoint backend qualification

**設計参照:** §28, §33

### 受け入れ条件

- [ ] fixed MPFR version、53-bit precision、RNDD/RNDUでbinary64 exact inputからreference corpusを生成する。
- [ ] MPFR version、generator hash、corpus hashをreference lockへ追加する。
- [ ] elementary endpoint backend候補ごとにcorrectness根拠、supported function/platform、distribution/licenseを記録する。
- [ ] BCL `Math.*`単体をcertified backendとして承認しない。
- [ ] Phase 4C/Dのcore公開functionを全support platformで提供できるbackend方針を確定する。

---

# Phase 4C

## P4C-001 Exp / Exp2 / Exp10

**設計参照:** §29.1

### 受け入れ条件

- [ ] Empty伝播、`-Infinity -> +0` limit、`+Infinity -> +Infinity`を満たす。
- [ ] finite overflow/underflow/subnormal endpointをtightに丸める。
- [ ] MPFR RNDD/RNDU corpusとの差異が0件である。

## P4C-002 Log / Log2 / Log10

**設計参照:** §29.2

### 受け入れ条件

- [ ] `b<=0 -> Empty`を満たす。
- [ ] `a<=0<b`ではlower=-Infinity limit、upperをdirected endpointで返す。
- [ ] `b=+Infinity`ではupper=+Infinityを返す。
- [ ] domain boundary近傍とsubnormal positive inputでMPFR corpusとの差異が0件である。

## P4C-003 Sinh / Cosh / Tanh

**設計参照:** §29.3

### 受け入れ条件

- [ ] Sinh/Tanhは単調増加endpoint ruleを満たす。
- [ ] Coshはnegative-only/positive-only/zero-crossingの3classで設計式を満たす。
- [ ] zero crossing Coshのlower endpointはexact 1である。
- [ ] Infinity/subnormal/large finite inputを含むMPFR corpusとの差異が0件である。

## P4C-004 Asinh/Acosh/Atanh/Asin/Acos/Atan

**設計参照:** §29.3

### 受け入れ条件

- [ ] Asinh/Atanは全実数上の単調増加ruleを満たす。
- [ ] Acoshはdomain `[1,+Infinity)`へclipする。
- [ ] Asin/Acosはdomain `[-1,1]`へclipする。
- [ ] Atanhはdomain `(-1,1)`、境界接触時±Infinity limit、`[-1,-1]`/`[1,1]`はEmptyを満たす。
- [ ] Acosの単調減少endpoint順序を誤らない。
- [ ] MPFR corpusとの差異が0件である。

## P4C-005 Phase 4C backend/API gate

**設計参照:** §33, §46

### 受け入れ条件

- [ ] Phase 4C各functionのdomain matrix reviewが完了する。
- [ ] primary MPFR corpusとx64/ARM64 resultsが一致する。
- [ ] 複数backendがある場合canonical endpoint bitsが一致する。
- [ ] core公開functionが全support platformで通常入力に対して利用可能である。
- [ ] failure artifactからfunction/domain/clipped domain/endpoint backend/correction decisionを追跡できる。
- [ ] API baselineを更新する。

---

# Phase 4D

## P4D-001 high-precision periodic reducer

**設計参照:** §30

### 受け入れ条件

- [ ] 全binary64範囲でquadrant/critical point/poleを判定できるfixed high-precision `2/pi`, `pi/2` tableを持つ。
- [ ] `Math.PI`との通常除算や`value % (2*Math.PI)`だけで判定しない。
- [ ] `ContainsPeriodicPoint`相当のexact判定をtest可能な単位へ分離する。
- [ ] huge finite input、subnormal、critical point直前/直後のfixtureでreference reductionと一致する。

## P4D-002 Sin / Cos

**設計参照:** §30.1, §30.2

### 受け入れ条件

- [ ] Sinは`-pi/2+2kpi`を含む場合lower=-1、`+pi/2+2kpi`を含む場合upper=+1をexactに含める。
- [ ] Cosは`pi+2kpi`を含む場合lower=-1、`2kpi`を含む場合upper=+1をexactに含める。
- [ ] 非有界または必要なmax/min lattice双方を含む場合`[-1,1]`を返す。
- [ ] すべての結果が`[-1,1]`のsubsetである。
- [ ] MPFR/reference corpusとの差異が0件である。

## P4D-003 Tan

**設計参照:** §30.3

### 受け入れ条件

- [ ] poleなしの1 branch内で`[TanDown(a),TanUp(b)]`を返す。
- [ ] poleへdomain内から近づける場合bare hullはEntireを返す。
- [ ] poleしか含まずdomain内点がない入力はEmptyを返す。
- [ ] pole直前/直後/接触/crossingのfixtureを固定し、periodic reducer判定をartifactに記録する。

## P4D-004 Atan2

**設計参照:** §31, §44.1 `F-PR3-011`

### 受け入れ条件

- [ ] API orderは`Atan2(y,x)`である。
- [ ] Empty operand -> Empty、X=ZeroかつY=Zero -> Emptyを満たす。
- [ ] 全sign-class直積、axis、originを固定matrixで検証する。
- [ ] strict negative Xに対するYの6class branch-cut matrixを全件fixture化する。
- [ ] `Atan2([-1,0],[-2,-1]) -> [-pi,+pi]`、`Atan2(Zero,[-2,-1]) -> Pi`等のreview-regression fixtureを満たす。
- [ ] signed zeroをbranch cut上下の別点として扱わない。
- [ ] QII/QIII corner evaluationとPi endpointがdirected referenceと一致する。

## P4D-005 positive-base general interval Pow

**設計参照:** §32, §44.1 `F-PR3-012`

### 受け入れ条件

- [ ] domainを`((0,+Infinity)xR) union ({0}x(0,+Infinity))`として扱い、negative baseを対象外にする。
- [ ] scalar `PowDown/Up`へ`0^0`、`0^negative`を渡さないことをinternal hook/testで確認する。
- [ ] strictly positive baseの3 exponent class式を満たす。
- [ ] zero-only baseとzero-touching baseの全subcaseを固定matrixで検証する。
- [ ] `Pow([0,0.5],[0,1]) -> [0,1]`、`Pow([0,0.5],[-1,0]) -> [1,+Infinity]`、`Pow([0,2],[0,1]) -> [0,2]`を満たす。
- [ ] MPFR/referenceとの未承認差異が0件である。

## P4D-006 Phase 4D backend/API gate

**設計参照:** §33, §46

### 受け入れ条件

- [ ] periodic reduction、pole、branch cut、zero-boundaryの分岐がfailure artifactから追跡できる。
- [ ] required deterministic/review-regression fixtureが全件成功する。
- [ ] x64/ARM64と複数backendのcanonical endpoint bitsが一致する。
- [ ] benchmark/correctnessを満たさないoptional backendをcore APIへ混在させない。
- [ ] API baselineを更新する。

---

# Phase 4E

## P4E-001 IntervalUnion2

**設計参照:** §34, §44.1 `F-PR3-010`, `F-PR3-015`

### 受け入れ条件

- [ ] `default(IntervalUnion2)`はCount=0のempty unionである。
- [ ] Count=0/1/2のFirst/Second canonical stateを満たす。
- [ ] Count=2で`First.Upper <= Second.Lower`を許可し、`First.Upper == Second.Lower`だけを理由にmergeしない。
- [ ] strict overlapはinternal validation failureとし、黙ってmergeしない。
- [ ] Count 0/1/2のequality/Hash/operatorがcanonical component列に基づく。
- [ ] invalid indexer accessは`ArgumentOutOfRangeException`である。
- [ ] 初版ではenclosure semanticsを誤解させる`Contains(double)`を公開しない。

## P4E-002 DivideToUnion

**設計参照:** §35, §44.1 `F-PR3-010`, `F-PR3-013`

### 受け入れ条件

- [ ] Empty operandまたはdenominator Zero -> Count0、denominator excludes zero -> Count1 ordinary divisionを満たす。
- [ ] numerator Zeroかつdenominatorにnonzero memberがある場合Count1 Zeroを返す。
- [ ] one-sided denominator `[0,d]` / `[c,0]`のZ/P/N/M matrixを全件固定する。
- [ ] strict zero-crossing denominatorでstrict positive/negative numeratorはCount2、zero-containing numeratorは設計どおりCount1を返す。
- [ ] `DivideToUnion([1,2],Entire)`がzero-touchのCount2 component enclosureを保持し、Entireへmergeしない。
- [ ] 全caseで`X/Y == DivideToUnion(X,Y).ConvexHull`が成立する。

## P4E-003 ReciprocalToUnion

**設計参照:** §35

### 受け入れ条件

- [ ] DivideToUnionの同一kernel semanticsをnumerator=Oneとして再利用する。
- [ ] `ReciprocalToUnion(Entire)`がCount2の`[-Infinity,-0.0]`と`[+0.0,+Infinity]`を返す。
- [ ] Zero -> Count0、strict positive/negative -> Count1、strict zero crossing -> Count2を満たす。
- [ ] `ReciprocalToUnion(value).ConvexHull == Reciprocal(value)`を全domain classのfixtureで確認する。

## P4E-004 ReverseMultiply

**設計参照:** §36, §44.1 `F-PR3-010`

### 受け入れ条件

- [ ] semantics `{z | exists y in factor : z*y in product}`を満たす。
- [ ] product Emptyまたはfactor Empty -> empty unionを返す。
- [ ] `0 in product && 0 in factor -> Count1 Entire`を満たす。
- [ ] それ以外はDivideToUnion相当の結果となる。
- [ ] `ReverseMultiply([1,2],Entire)`がzero-touch Count2 component enclosureを保持する。

## P4E-005 cancellative operations

**設計参照:** §37, §44.1 `F-PR3-014`

### 受け入れ条件

- [ ] Empty/common/unbounded 3x3 matrixを全件固定fixtureで検証する。
- [ ] bounded/common同士はrounded Widthではなくexact width relationを比較する。
- [ ] exact width(total) >= exact width(term)の場合だけ`[RD(a-c),RU(b-d)]`を返し、それ以外はEntireを返す。
- [ ] `CancelSubtract(Empty,Empty) -> Empty`、`CancelSubtract(Empty,bounded) -> Empty`を満たす。
- [ ] `CancelAdd(total,term)=CancelSubtract(total,-term)`で同じEmpty matrixを継承する。

## P4E-006 Decoration / default NaI

**設計参照:** §38.1, §38.3～38.6, §44.1 `F-PR3-016`

### 受け入れ条件

- [ ] `Decoration`の品質順を`Ill < Trv < Def < Dac < Com`として表現できる。
- [ ] `default(DecoratedInterval).IsNaI == true`を満たす。
- [ ] NaI inputはNaIへ伝播し、Illでordinary intervalを生成しない。
- [ ] `FromInterval`はbounded nonempty=Com、unbounded nonempty=Dac、Empty=Trvを返す。
- [ ] result state capによりEmptyは最大Trv、unbounded nonemptyは最大Dac、bounded nonemptyは最大Comとなる。
- [ ] MaxValue singleton同士の加算等でunbounded resultをComとして返さないreview-regression fixtureがある。

## P4E-007 DecoratedInterval equality/canonicalization

**設計参照:** §38.2, §38.7, §44.1 `F-PR3-015`

### 受け入れ条件

- [ ] C# value equalityはreflexiveで`NaI == NaI`がtrueである。
- [ ] NaIは固定Hashを持ち、internal NaN payloadに依存しない。
- [ ] non-NaI equalityはcanonical interval partとDecorationを比較する。
- [ ] `SemanticallyEquals`はNaIにfalse、non-NaIではdecorationを無視したinterval equalityを使用する。
- [ ] `TryGetInterval`はNaIでfalse/out=Emptyを返す。

## P4E-008 decorated arithmetic/math

**設計参照:** §38.5, §38.6, §39

### 受け入れ条件

- [ ] decorated四則演算/mathはbare resultとoperation-specific `opDec`を取得後、共通canonical result capを必ず通す。
- [ ] divisor zero-containing division、domain clipping Sqrt、Tan pole crossing等で`opDec`が設計の上限を超えない。
- [ ] NaI入力を全operationでNaIへ伝播する。
- [ ] bare interval resultとdecorated interval partが同じcanonical endpointsを持つ。

## P4E-009 exact/outward parser・formatter

**設計参照:** §40, §43

### 受け入れ条件

- [ ] `Empty`、`Entire`、`[a,b]`、`[a]`の採用syntaxをparseできる。
- [ ] decimal tokenを`double.Parse`点区間化せず、exact `sign * integerSignificand * 10^exponent`として解析し外向き丸めする。
- [ ] exact lower>upper、NaN endpoint、lower +Infinity、upper -Infinity、Infinity singletonを拒否する。
- [ ] `TryParse`系はinvalid inputでfalse/out=Emptyを返す。
- [ ] exact/round-trip formatはparse後canonical endpoint bitsが元値と一致する。
- [ ] 最大入力長、significand digit数、exponent digit数を固定し、過大入力をbounded resourceで拒否する。
- [ ] exception messageへ入力全文を無制限に含めない。

## P4E-010 binary interchange

**設計参照:** §41

### 受け入れ条件

- [ ] private `[-Lower,Upper]` memory layoutをwire formatへ使用しない。
- [ ] version/state/external Lower bits/external Upper bitsのversioned formatを実装する。
- [ ] Empty/Entire/Zeroをcanonical external bitsでencodeする。
- [ ] invalid version/state/NaN endpoint/reversed endpointをdecoderで拒否する。
- [ ] encode/decode round-tripでcanonical interval bitsが一致する。

## P4E-011 split / bisect

**設計参照:** §42

### 受け入れ条件

- [ ] `TrySplitAt`はnonempty、finite splitPoint、`Lower < splitPoint < Upper`の場合のみ成功する。
- [ ] 成功時は`[Lower,splitPoint]`と`[splitPoint,Upper]`を返しgapを作らない。
- [ ] `TryBisect`初版はbounded non-singletonのみを対象とする。
- [ ] strict interior binary64が存在しない隣接endpoint区間はfalseを返す。
- [ ] unbounded intervalを任意pivotで自動bisectしない。
- [ ] childrenが元区間をcoverし、各childが元区間のsubsetであるproperty testが成功する。

## P4E-012 Phase 4E security/conformance/API gate

**設計参照:** §43～§46

### 受け入れ条件

- [ ] `F-PR3-010`、`F-PR3-013`～`F-PR3-017`のreview-regression fixtureが全件成功する。
- [ ] union/decorated/parser/interchange/splitのdefault、equality、round-trip propertyが成功する。
- [ ] parser/binary decoderのresource/security reviewが完了し、入力上限値が文書化される。
- [ ] x64/ARM64と複数backendのcanonical resultが一致する。
- [ ] failure artifactからunion component、decoration、parser exact rational/resource limit、split stateを追跡できる。
- [ ] public API baselineを更新する。

---

## 4. Phase完了時の更新規則

- Phase内の必須タスクがすべて`完了`になり、`tasks/phases-status.md`の完了条件を全件満たした時だけPhaseを`完了`へ変更する。
- 後続Phase開始後に前Phaseの数値意味論を変更する必要が生じた場合は、通常タスクとして黙って変更せず、設計変更とbreaking-change判定を先に行う。
- API freeze後に基本`Interval` APIを変更する場合は、該当タスクの再open、API baseline差分、`doc/Design/BreakingChanges.md`の要否を必ず確認する。
- CIによる完了判定は必ず対象PR current HEAD SHAと一致するworkflow runを使用する。
