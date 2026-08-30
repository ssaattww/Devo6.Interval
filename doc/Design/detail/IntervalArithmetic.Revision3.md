# 区間演算 詳細設計 Revision 3

## 1. 文書情報

- 対象リポジトリ: `ssaattww/Devo6.Interval`
- 対象ライブラリ: `Devo6.Interval`
- 基本文書: `doc/Design/detail/IntervalArithmetic.md` 設計版2
- 適用対象review: PR #3 fix verification
- reviewed fix HEAD: `c1826cc8dab3f070bfb5133ece44969a03727e97`
- review report: `reports/2026-08-30-pr-3-design-fix-verification.md`
- 作成日: 2026-08-30
- 設計版: 3

本書は、設計版2に対する規範的な修正である。設計版2と本書が矛盾する場合、次の対象について本書を優先する。

- `Interval.Empty`の`Lower` / `Upper`
- exact-rational oracleのfinite overflow変換
- IEEE 1788.1 Phase 1 conformance corpusのsource mapping
- `IsSingleton`のconformance matrix
- conformance adaptationとacceptance gate
- overflow fixtureのoracle経路
- review finding closure

設計版2のその他の規定は継続して有効である。

本revisionは次のactive findingを閉じるためのものである。

- `F-PR3-004` Medium: conformance corpus/source/adaptationの不整合
- `F-PR3-009` Medium: exact-rational oracleのfinite overflow処理欠落

## 2. Emptyの公開端点

### 2.1 決定

`Interval.Empty`の内部表現は、設計版2どおりcanonical quiet NaNを2 laneに保持する。

```text
internal Empty = [canonical-qNaN, canonical-qNaN]
```

一方、公開端点はIEEE 1788.1-oriented semanticsおよび主要参照実装`inari`に合わせ、次を返す。

```text
Interval.Empty.Lower -> +Infinity
Interval.Empty.Upper -> -Infinity
```

これにより空区間は、公開端点上でも`Lower > Upper`となる。

### 2.2 getter

概念実装を次で固定する。

```csharp
public double Lower
    => IsEmpty ? double.PositiveInfinity : -_negatedLower;

public double Upper
    => IsEmpty ? double.NegativeInfinity : _upper;
```

非空区間については従来どおり、下限0を`-0.0`、上限0を`+0.0`として返す。

### 2.3 NaNとの分離

- `NaN`を通常区間の公開端点として返さない。
- `NaN`は内部のEmpty識別表現および将来のNaI等でのみ使用する。
- Emptyの内部NaN payloadは公開契約およびconformance比較対象にしない。
- `TryCreate`が`false`を返す場合、out値は`Interval.Empty`となり、その公開端点も`+Infinity / -Infinity`となる。

### 2.4 API確定前の変更

本変更はPhase 1 pilot APIに対する修正であり、Phase 2のAPI freeze前に適用する。既存の公開互換性を維持する必要はまだない。

本節は設計版2の§4.5、およびEmptyの公開端点を`NaN / NaN`とするすべての記述を置き換える。内部表現に関するNaN規定は置き換えない。

## 3. Exact-rational oracleのfinite overflow

### 3.1 値分類

oracleは、BCLのnearest resultを調べる前に数学上の結果を次へ分類する。

```text
FiniteRational
PositiveInfinity
NegativeInfinity
Undefined
```

有限operand同士の、実数として定義された加算、減算、乗算、除算は、binary64の有限範囲を超えていても`FiniteRational`として保持する。

operand自体にInfinityを含む演算は、finite overflowとは分離し、区間意味論・特殊値規則に従うsemantic oracleで扱う。`Infinity`をexact rationalへ変換しない。

### 3.2 最大有限値との事前比較

`R`を有限なexact rational result、`M`を`double.MaxValue`のexact rational表現とする。

BCL nearest resultをexact rationalへ変換する前に、必ず次を判定する。

```text
if R > M:
    RoundUp(R)   = +Infinity
    RoundDown(R) = +double.MaxValue

if R < -M:
    RoundUp(R)   = -double.MaxValue
    RoundDown(R) = -Infinity
```

この分岐はproductionのoverflow分岐を流用せず、test-onlyの`BigInteger` rational比較として独立実装する。

### 3.3 有限範囲内の変換

`-M <= R <= M`の場合に限り、BCL nearest result `n`を使用する。この場合`n`は有限値でなければならない。

```text
N = exact rational representation of n

RoundUp:
    if N < R: BitIncrement(n)
    else:     n

RoundDown:
    if N > R: BitDecrement(n)
    else:     n
```

`N == R`では補正しない。

zeroの符号は、oracleがprimitiveのbit patternを検証する場合はprimitive規則に従い、区間端点として比較する場合はDevo6.Intervalのcanonical endpoint規則へ正規化する。

### 3.4 exact Infinityとの区別

次を別caseとして保持する。

```text
finite operands -> finite exact real result outside binary64 range
Infinity operand -> exact resultまたはset semanticsがInfinity
```

両者は最終binary64が同じInfinityになる場合があるが、oracle経路とcase metadataを同一にしない。

### 3.5 再現可能なmetadata

finite overflow caseは、golden corpusへ少なくとも次を保存する。

```text
oraclePath: finite-exact-overflow
exactValueKind: finite-rational-out-of-binary64-range
operandsFinite: true
relationToMaxFinite: greater | less
roundingDirection: up | down
```

必要に応じ、exact numerator / denominatorまたはそのcanonical serialization hashを保存し、production resultだけから期待値を逆算しない。

本節は設計版2の§13.2を置き換える。

## 4. IEEE 1788.1 Phase 1 conformance

### 4.1 Phase 1 operation matrix

| IEEE operation concept | Devo6.Interval API | Source | Phase 1 |
|---|---|---|---|
| `empty` | `Interval.Empty` | repository-defined matrix | required |
| `entire` | `Interval.Entire` | repository-defined matrix | required |
| numeric `numsToInterval` | constructor / `TryCreate` | ITF1788 constructor sources | required |
| `inf` | `Lower` | `libieeep1788_num.itl` | required |
| `sup` | `Upper` | `libieeep1788_num.itl` | required |
| `isEmpty` | `IsEmpty` | `libieeep1788_bool.itl` | required |
| `isEntire` | `IsEntire` | `libieeep1788_bool.itl` | required |
| `isSingleton` | `IsSingleton` | repository-defined equivalent matrix | required |
| `equal` | `Equals`, `==` | `libieeep1788_bool.itl` | required |
| `neg` | unary `-` | `libieeep1788_elem.itl` | required |
| `add` | binary `+` | `libieeep1788_elem.itl` | required |
| `sub` | binary `-` | `libieeep1788_elem.itl` | required |
| `mul` | binary `*` | `libieeep1788_elem.itl` | required |
| `div` | binary `/` | `libieeep1788_elem.itl` | required |

### 4.2 ITF1788 source mapping

固定参照は変更しない。

```text
Repository: unageek/ITF1788
Commit:     d8c2a64478ebdc9cbde6ccef33eaad3bed60ed81
```

Phase 1 generatorは次を使用する。

| Purpose | Pinned source | Selection |
|---|---|---|
| numeric constructor | `itl/libieeep1788_class.itl` | bare `b-numsToInterval` cases from `minimal_nums_to_interval_test` |
| supplemental constructor | `itl/ieee1788-constructors.itl` | Phase 1 numeric bare cases only |
| arithmetic | `itl/libieeep1788_elem.itl` | bare `neg/add/sub/mul/div` |
| boolean | `itl/libieeep1788_bool.itl` | bare `equal/isEmpty/isEntire` |
| endpoint access | `itl/libieeep1788_num.itl` | bare `inf/sup` |

`libieeep1788_bool.itl`から`isSingleton`を抽出しない。固定SHAの同fileおよびrepositoryには、利用可能な`isSingleton` caseが存在しないためである。

### 4.3 numeric constructor case

`libieeep1788_class.itl`のbare numeric constructorについて、少なくとも次をPhase 1 required corpusへ取り込む。

```text
(-1.0, 1.0)                -> [-1.0, 1.0]
(-Infinity, 1.0)           -> [-Infinity, 1.0]
(-1.0, +Infinity)          -> [-1.0, +Infinity]
(-Infinity, +Infinity)     -> Entire
(NaN, NaN)                 -> invalid / UndefinedOperation
(1.0, -1.0)                -> invalid / UndefinedOperation
(-Infinity, -Infinity)     -> invalid / UndefinedOperation
(+Infinity, +Infinity)     -> invalid / UndefinedOperation
```

Devo6.Intervalへの適応は次とする。

- 成功case: constructorおよび`TryCreate=true`を検証する。
- invalid case: constructorが`ArgumentException`、`TryCreate=false`、out値が`Empty`であることを検証する。
- ITF1788の`[empty] signal UndefinedOperation`を、constructorが値としてEmptyを返すこととは解釈しない。
- text constructorおよびdecorated constructorはPhase 4へ延期する。

複数sourceに同一意味のcaseがある場合、source pathとoriginal case identifierを含むcaseIdを使用し、無言で上書きまたは重複除去しない。意図的にdeduplicateする場合はmanifestへ双方のsource locationを残す。

### 4.4 repository-defined `IsSingleton` matrix

外部corpusに存在しないため、次をrepository-defined IEEE 1788.1-equivalent matrixとして固定する。

| Case | Interval | Expected |
|---|---|---|
| empty | `Empty` | `false` |
| entire | `Entire` | `false` |
| finite singleton | `[1.0, 1.0]` | `true` |
| negative singleton | `[-2.0, -2.0]` | `true` |
| zero singleton | `[-0.0, +0.0]` | `true` |
| normalized zero variants | constructor inputs using any `±0.0` pair | `true` after normalization |
| bounded non-singleton | `[1.0, 2.0]` | `false` |
| lower-unbounded | `[-Infinity, 2.0]` | `false` |
| upper-unbounded | `[1.0, +Infinity]` | `false` |

predicate contractは次である。

```text
IsSingleton = !IsEmpty && Lower == Upper
```

有効なbare intervalではInfinity singletonを生成できないため、`[-Infinity,-Infinity]`および`[+Infinity,+Infinity]`をsingleton caseとして扱わず、constructor invalid matrixで検証する。

corpus recordは次を持つ。

```text
source: repository-defined-ieee1788.1-equivalent
sourceRevision: detailed-design-revision-3
operation: isSingleton
```

### 4.5 Emptyの`inf` / `sup`

Phase 1は次をrequired conformance caseとして採用する。

```text
inf(Empty) = +Infinity
sup(Empty) = -Infinity
```

C# API mappingは次である。

```text
inf -> Interval.Lower
sup -> Interval.Upper
```

§2で公開getterをこの意味論へ変更したため、approved deviationは不要である。内部の`[NaN,NaN]`表現はadapterから直接観測せず、conformance比較対象にしない。

### 4.6 adaptation rules

- nonempty lower zeroは`-0.0`、upper zeroは`+0.0`へcanonicalizeする。
- Emptyの公開端点は`+Infinity / -Infinity`として比較する。
- Empty内部NaN payloadは比較しない。
- decorated resultはPhase 1で読み込まない。
- string parsingを必要とするcaseは`deferred-phase-4`とする。
- constructorのUndefinedOperationは、throwing APIとnon-throwing APIをそれぞれ検証する。
- interval resultはstateとcanonical endpoint bit patternで比較する。
- `isSingleton`は外部ITF sourceに由来すると表示せず、repository-defined matrixとして表示する。

### 4.7 manifest

各caseについて次を保存する。

- caseId
- operation
- source type: `itf1788`または`repository-defined`
- source repository / commit / path / testcase
- original expressionまたはrepository-defined case name
- adaptation rule
- applicability: `required`, `deferred-phase-4`, `excluded`, `approved-deviation`
- exclusion / deviation reason
- expected state / value / endpoint bits

存在しないoperationをsource fileから抽出したことにしてはならない。generatorは、宣言されたoperationが0件だった場合に黙って成功せず、manifest generation failureとする。ただしrepository-defined sourceとして明示されたoperationはこの0件checkの対象外とする。

### 4.8 conformance summaryとgate

`conformance-summary.json`は少なくとも次を分離して集計する。

```text
requiredExternal
requiredRepositoryDefined
passed
failed
approvedDeviation
matchedApprovedDeviation
deferred
excluded
sourceExtractionErrors
```

Phase 1 conformance gateは次をすべて要求する。

1. `sourceExtractionErrors == 0`
2. externalおよびrepository-definedのrequired caseがすべて実行された。
3. required caseの`failed == 0`
4. approved deviationが存在する場合、全件がmanifestで事前承認され、actual resultが宣言どおりである。
5. unapproved deviationが0件である。

本revisionではEmpty `inf/sup`を標準・inariへ合わせたため、この項目にapproved deviationは存在しない。

本章は設計版2の§14.2、§14.4、§14.5、§14.6を置き換える。

## 5. Overflow fixtureとoracle経路

### 5.1 finite overflow fixture

加算、減算、乗算、除算の各primitiveで、正負両方向のfinite overflowを固定する。

| Case | Exact expression | Up | Down |
|---|---|---|---|
| add-positive-overflow | `double.MaxValue + double.MaxValue` | `+Infinity` | `+double.MaxValue` |
| add-negative-overflow | `-double.MaxValue + -double.MaxValue` | `-double.MaxValue` | `-Infinity` |
| sub-positive-overflow | `double.MaxValue - (-double.MaxValue)` | `+Infinity` | `+double.MaxValue` |
| sub-negative-overflow | `-double.MaxValue - double.MaxValue` | `-double.MaxValue` | `-Infinity` |
| mul-positive-overflow | `double.MaxValue * 2.0` | `+Infinity` | `+double.MaxValue` |
| mul-negative-overflow | `-double.MaxValue * 2.0` | `-double.MaxValue` | `-Infinity` |
| div-positive-overflow | `double.MaxValue / 2^-1074` | `+Infinity` | `+double.MaxValue` |
| div-negative-overflow | `-double.MaxValue / 2^-1074` | `-double.MaxValue` | `-Infinity` |

各caseは§3の`finite-exact-overflow` oracle pathを通ること自体をassertする。BCL nearest resultがInfinityであることだけをexpected resultの根拠にしてはならない。

### 5.2 special Infinity fixture

次はfinite overflow fixtureと別caseとする。

- `+Infinity + finite`
- `-Infinity + finite`
- nonzero finite `* ±Infinity`
- `±Infinity / finite nonzero`
- finite / `±Infinity`

metadataを次で区別する。

```text
oraclePath: interval-special-value-semantics
exactValueKind: positive-infinity | negative-infinity | finite-zero
operandsFinite: false
```

### 5.3 acceptance

- finite overflow fixtureはexact rationalと`double.MaxValue`の比較によりexpected resultを生成する。
- special Infinity fixtureはexact rational conversionへ入らない。
- 両fixture群が同じexpected bit patternになってもcaseを統合しない。
- x64およびARM64で同じfixture、oracle path、canonical resultを要求する。

本章は設計版2の§15.5を置き換える。

## 6. API確定gateの修正

Phase 2の正確性gateは次を要求する。

- 設計版2 §15および本revision §5の全fixtureが成功する。
- finite-result、finite-underflow、finite-overflowを含むexact rational oracleとの差がない。
- Phase 1 required conformance caseが、external sourceとrepository-defined sourceの双方で全件成功する。
- conformance source extraction errorが0件である。
- unapproved standards deviationが0件である。
- approved deviationが存在する場合はmanifestと実測が一致する。本revision時点でEmpty `inf/sup`のdeviationは存在しない。
- inariとの差異が0件、または差異ごとに意味論と承認記録がある。
- kv primitiveとの互換対象差異が0件、または差異ごとに承認記録がある。
- Linux x64 / Linux ARM64のcanonical result corpusが一致する。

「Phase 1 conformance matrixが全件pass」とは、deferred / excluded caseをpass扱いすることではない。required caseのみを母集団とし、外部source、repository-defined source、approved deviationをsummaryで分離する。

本章は設計版2の§18.2を置き換える。

## 7. TDD実装順への反映

Phase 1では、対象production実装より先に次の失敗testを追加する。

1. `Empty.Lower == +Infinity`および`Empty.Upper == -Infinity`
2. pinned `libieeep1788_class.itl`のbare numeric constructor matrix
3. repository-defined `IsSingleton` matrix
4. ITF1788 `inf/sup` Empty case
5. exact-rational finite overflow conversionのUp/Down matrix
6. add/sub/mul/divのfinite overflow fixtureが`finite-exact-overflow` pathへ入ること
7. special Infinity fixtureがexact rational pathへ入らないこと
8. conformance generatorが存在しないoperation抽出をsource errorとして失敗させること

失敗を確認した後に、getter、oracle、generator、test adapterを実装する。

本PRではdesignのみを変更し、production source、test、workflowは追加しない。

## 8. Finding closure

| Finding | Source severity | Disposition | Evidence |
|---|---:|---|---|
| `F-PR3-004` | Medium | addressed | §2および§4でEmpty endpoint、constructor source、`IsSingleton` matrix、adaptation、gateを一意に定義 |
| `F-PR3-009` | Medium | addressed | §3および§5でfinite exact overflowの独立oracle変換と全四則fixture経路を定義 |
| `F-PR3-001` | High | remains resolved | 設計版2 §9–§10 |
| `F-PR3-002` | Medium | remains resolved | 設計版2 §3.2、§19 |
| `F-PR3-005` | Medium | remains resolved | 設計版2 §13 |
| `F-PR3-006` | Medium | remains resolved | 設計版2 §17 |
| `F-PR3-007` | Medium | remains resolved | 設計版2 §15と本revision §5 |
| `F-PR3-008` | Low | remains resolved | 設計版2 §20 |
| `F-PR3-003` | withdrawn | no action | exhaustive review erratumを維持 |

この表は設計版2の§27を置き換える。

## 9. 参照

- `doc/Design/basic/IntervalArithmetic.md`
- `doc/Design/detail/IntervalArithmetic.md`
- `reports/2026-08-29-pr-3-design-review-exhaustive.md`
- `reports/2026-08-30-pr-3-design-review-follow-up.md`
- `reports/2026-08-30-pr-3-design-fix-verification.md`
- `unageek/inari` commit `18b83a571d7681c76067bc38d90a74e8be29f545`
- `unageek/ITF1788` commit `d8c2a64478ebdc9cbde6ccef33eaad3bed60ed81`
  - `itl/libieeep1788_class.itl`
  - `itl/libieeep1788_elem.itl`
  - `itl/libieeep1788_bool.itl`
  - `itl/libieeep1788_num.itl`
- `mskashi/kv` commit `c7f8f2324a0e403cca6b39f46088a22843d440db`

## 10. 実装開始条件

Phase 1は、本revisionを含む詳細設計へのfix verificationが完了した後に開始する。

実装時は`IntervalArithmetic.md`単独ではなく、本revisionを必ず併読する。将来両文書を統合する場合も、統合PRがreviewされるまでは本revisionの規定を失効させない。
