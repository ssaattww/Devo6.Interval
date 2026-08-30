# PR #3 追加設計差分レビュー report

## Metadata

- Repository: `ssaattww/Devo6.Interval`
- Pull request: `#3` (`docs: add interval arithmetic detailed design`)
- Review mode: `delta-limited additional design review`
- User-requested boundary: 前回レビュー済みの重複部分は再レビューせず、追加分とその直接影響だけを確認
- Previous reviewed design HEAD: `13cf07cfcdf01205ab4466a99abd380fd1f1d103`
- Previous review-artifact HEAD / delta base: `f60f78e7e1637d587d53c4f926fb05cf4ce0f3b8`
- Current reviewed PR HEAD: `8e6e7499204fccf0643da82f274d1485dc0e3272`
- Reviewed delta: `f60f78e7e1637d587d53c4f926fb05cf4ce0f3b8..8e6e7499204fccf0643da82f274d1485dc0e3272`
- Primary artifact: `doc/Design/detail/IntervalArithmetic.md` design version 4
- Reviewer role: normal reviewer
- Reviewer continuity: 前回のnormal reviewerと同一。今回の追加設計・統合作業は実装していない
- Date: 2026-08-30
- Verdict: **FAIL**
- Active findings: **8**
  - High: 2
  - Medium: 5
  - Low: 1

## Review boundary

今回の主対象は、前回review artifact HEAD `f60f78e7...` 以降に追加された次の範囲である。

- Phase 4A: 集合演算、関係、数値的属性、整数値関数
- Phase 4B: reciprocal、square、sqrt、integer power/root、FMA、区間定数
- Phase 4C: 単調初等関数
- Phase 4D: 周期関数、`Atan2`、general positive-base power
- Phase 4E: `IntervalUnion2`、extended/reverse division、cancellative operations、decorated interval、I/O、splitting
- 複数の詳細設計書を `IntervalArithmetic.md` へ統合し、旧文書を削除した変更
- 追加設計および統合作業のreport / handoff

前回PASS済みの四則演算設計は、次の統合回帰点だけを再確認した。

- Emptyの公開端点 `+Infinity / -Infinity`
- Empty内部canonical NaN表現
- exact-rational finite-overflow branch
- ITF1788 constructor source / `IsSingleton` matrix
- multiplication/divisionのsubnormal thresholdとresidual tie
- ISA/FMA capability分離
- x64/ARM64およびexact-head CI方針

上記は統合版へ保持されており、前回findingを再度起票していない。

## Inspected artifacts

### Current repository artifacts

- `doc/Design/detail/IntervalArithmetic.md`
- `reports/2026-08-30-interval-non-arithmetic-detailed-design.md`
- `reports/2026-08-30-interval-non-arithmetic-detailed-design-handoff.yaml`
- `reports/2026-08-30-interval-detailed-design-consolidation.md`
- `reports/2026-08-30-interval-detailed-design-consolidation-handoff.yaml`
- PR #3 current body and exact-head workflow state

### Pre-consolidation Phase 4 artifacts

統合欠落の確認に限定して、統合前HEAD `01f379255e8677c2cd78a1ea696bf2a925ff7f89` の次を参照した。

- `IntervalNonArithmetic.Roadmap.md`
- `IntervalSetAndNumeric.md`
- `IntervalMathFunctions.md`
- `IntervalAdvancedFeatures.md`

### Pinned references

- `unageek/inari` commit `18b83a571d7681c76067bc38d90a74e8be29f545`
  - `src/basic.rs`
  - `src/elementary.rs`
  - `src/interval.rs`
- accepted basic design and previous review reports

## Positive findings

追加設計には次の妥当な点がある。

1. `Interval`、`IntervalMath`、`IntervalConstants`、`IntervalUnion2`、`DecoratedInterval`、`IntervalContractor`の責務分離は妥当である。
2. bare intervalへNaI、decoration、非連結成分、parser stateを混在させない方針は維持されている。
3. `Contains`、intersection、convex hull、subset、interior、disjoint、precedes、pointwise min/maxの基本式は妥当である。
4. Width、Radius、Magnitude、MignitudeのEmpty / unbounded / signed-zero方針は整合している。
5. reciprocal、square、sqrt、integer power/rootの主要domain分類は妥当である。
6. BCL `Math.*`へ根拠なく固定ULPを加える方式を正式endpoint backendにしない方針は正しい。
7. MPFR directed corpus、pinned reference lock、x64/ARM64 canonical comparison、failure diagnostic artifactの方針は維持されている。
8. parserにexact decimal処理とresource limitを要求し、private memory layoutをwire formatにしない方針は妥当である。
9. 各source実装をTDDで開始し、current HEADと一致するCIだけを完了証跡にする方針は維持されている。
10. 今回のdeltaはdocumentation/reportに限定され、production source、tests、workflowを変更していない。

## Active findings

### F-PR3-010 — High — `IntervalUnion2`のstrict-gap canonicalizationが必要なtwo-component resultを1成分へ潰す

- Origin: `introduced_by_added_design`
- Location:
  - `doc/Design/detail/IntervalArithmetic.md` §53.2 `canonical state`
  - 同 §53.4 `construction`
  - 同 §54 `Extended Division`
  - 同 §55 `Reverse Multiplication`

#### Problem

`Count=2`について次を要求している。

```text
First.Upper < Second.Lower
```

さらに、2成分が接触した場合は1成分へmergeするとしている。

しかし、extended divisionが返す数学的な2成分のclosureは0で接する場合がある。

具体例:

```text
X = [1,2]
Y = Entire = [-Infinity,+Infinity]

{x/y | x∈X, y∈Y, y!=0}
= (-Infinity,0) union (0,+Infinity)
```

現在の§54の式から得る各連結成分のtightなclosed enclosureは次になる。

```text
[-Infinity,-0.0]
[+0.0,+Infinity]
```

binary64比較では`-0.0 == +0.0`であるため、§53のstrict-gap条件を満たさない。現在の`Create2`規則に従うと両成分がmergeされ、`Count=1 / Entire`となる。

同じ問題は次でも発生する。

- `ReciprocalToUnion(Entire)`
- `ReverseMultiply([1,2], Entire)`

pinned `inari`のtwo-output divisionは、各数学的成分のtight enclosureとして`sup(first) <= inf(second)`を許し、`Entire`をfactorとするcaseで`[-Infinity,0]`と`[0,+Infinity]`の2出力を保持している。

#### Impact

- `IntervalUnion2`が導入目的である非連結情報を、代表的なunbounded caseで保持できない。
- `DivideToUnion`と`ReverseMultiply`のcomponent countが参照実装および意図した意味論と異なる。
- 単純に`<=`へ変えるだけでは、2個のclosed `Interval`の集合union自体はzeroを含むため、型が「exact set union」なのか「各成分のclosure enclosure」なのかが曖昧なままになる。

#### Required action

次のいずれかを明示的に選択すること。

1. `IntervalUnion2`を数学的なexact unionとするなら、open endpointを表現できるcomponent型またはboundary metadataを追加する。
2. pinned `inari`同様、各連結成分のtight closed enclosureを保持する型とするなら、その意味論を型・XML documentation・equality・membershipへ明記し、zeroで接する2 enclosureをmergeしないcanonical ruleを定義する。

少なくとも次を固定fixtureへ追加すること。

```text
DivideToUnion([1,2], Entire)
ReciprocalToUnion(Entire)
ReverseMultiply([1,2], Entire)
```

---

### F-PR3-011 — High — `Atan2`のnegative-x branch cut片側接触が定義されず、corner評価では真値を包含しない

- Origin: `introduced_by_added_design`
- Location: `doc/Design/detail/IntervalArithmetic.md` §50 `Atan2`

#### Problem

設計は「negative x branch cutを跨ぐ場合」に`[-π,π]`を返すとしているが、negative x-axisへ下側から接するcaseを定義していない。

具体例:

```text
x = [-2,-1]
y = [-1,0]
```

principal rangeを`(-π,π]`とすると、次が同時に成立する。

- `y=0, x<0`では値は`+π`。
- `y<0`から`0`へ近づくと値は`-π`へ近づく。

したがってbare intervalのtight hullは次である。

```text
[-π,+π]
```

一方、有限corner `atan2(-1,-2)` / `atan2(-1,-1)`とaxis候補`+π`だけでは、lower endpointは`-π`まで到達せず、0へ近いnegative yが生成する角度を包含できない。

pinned `inari`はstrictly-negative xとnonpositive y touching zeroのclassを個別処理し、`[-π,+π]`を返している。

#### Impact

sign-cell implementationが「crossing」を`y.Lower < 0 < y.Upper`だけと解釈すると、上記caseでlower endpointが大き過ぎる非包含resultを返す。区間演算の基本要件である真値包含違反になる。

#### Required action

negative x-axisについて少なくとも次のmatrixを明示すること。

- y strictly negative
- y nonpositive and touching zero
- y exactly Zero
- y nonnegative and touching zero
- y strictly positive
- y crossing zero

次を区別すること。

```text
x<0, y=[negative,0] -> [-π,+π]
x<0, y=Zero         -> Pi
x<0, y=[0,positive] -> second-quadrant range ending at +π
x<0, y crosses zero -> [-π,+π]
```

canonical signed zeroをbranch-cutの上下2点として誤解しない規則と、各caseのfixed fixtureを追加すること。

---

### F-PR3-012 — Medium — general positive-base `Pow`がzero-base境界で禁止した`0^0` / undefined pairの代替を定義していない

- Origin: `introduced_by_added_design`
- Location: `doc/Design/detail/IntervalArithmetic.md` §51 `General Positive-Base Power`

#### Problem

§51は`0^0`をscalar kernelへ直接渡さないとしているが、記載されたendpoint式は`a=0`でその値を要求する。

例1:

```text
X=[0,0.5]
Y=[0,1]
```

正しいhullは`[0,1]`である。`c<=0<d`の式はupper候補として`PowUp(a,c)=PowUp(0,0)`を参照するが、代わりに`1`を追加する分岐がない。

例2:

```text
X=[0,0.5]
Y=[-1,0]
```

正しいhullは`[1,+Infinity]`である。`d<=0`かつ`b<1`の式は`PowUp(a,c)=PowUp(0,-1)`を参照するが、domainではx=0を評価せず、x→0+のlimit `+Infinity`を使用しなければならない。

#### Impact

- 設計どおりでは、禁止したundefined scalar pairを呼ぶか、必要な`1` / `+Infinity`候補を落とすかの二択になる。
- 後者ではunder-enclosureが発生し得る。
- endpoint kernelのpreconditionとinterval extensionの式が矛盾する。

#### Required action

zero boundary用のextended endpoint ruleまたは明示的なinterval分岐を定義すること。

最低限:

```text
x -> 0+, y < 0 : +Infinity limit
x > 0,  y = 0 : 1
x = 0,  y > 0 : 0
```

`PowDown/Up(0,0)`を通常のpoint functionとして定義するのではなく、rectangle closure candidateとして`1`を注入する等、point-domain semanticsとの区別を明記すること。上記2例と`X=[0,2], Y=[0,1]`をfixture化すること。

---

### F-PR3-013 — Medium — 統合時に`DivideToUnion`のone-sided-zero denominator規則が脱落した

- Origin: `introduced_by_consolidation`
- Location: `doc/Design/detail/IntervalArithmetic.md` §54 `Extended Division`

#### Problem

統合版§54は次を定義している。

- Empty / Zero denominator
- zeroを除外するdenominator
- strict zero-crossing denominator `c<0<d`

しかし、次の有効入力classがない。

```text
Y=[0,d], d>0
Y=[c,0], c<0
```

統合前の`IntervalAdvancedFeatures.md`には、このcaseをordinary divisionと同じ1成分として返す明示的な表が存在していた。統合版ではその節が削除されている。

例:

```text
DivideToUnion([1,2],[0,1])
```

はCount=1の`[1,+Infinity]`であるべきだが、現在のdecision treeには該当branchがない。

#### Impact

- `DivideToUnion`の許可入力が全分類されていない。
- ordinary divisionとのhull一致propertyを実装できない。
- 統合report / handoffの「normative content preserved」という主張と一致しない。

#### Required action

統合前のone-sided-zero表をsole normative documentへ復元し、正負・mixed・Zero numeratorを両向きのdenominatorで固定すること。`DivideToUnion(...).ConvexHull == ordinary division`を各caseで検証すること。

---

### F-PR3-014 — Medium — cancellative operationのEmpty totalが`Entire`へ落ち、記載したtightest-result契約を満たさない

- Origin: `introduced_by_added_design`
- Location: `doc/Design/detail/IntervalArithmetic.md` §56 `Cancellative Operations`

#### Problem

現在の規則は「両方nonempty boundedかつwidth条件を満たす場合だけ計算し、それ以外はEntire」としている。

したがって次が`Entire`になる。

```text
CancelSubtract(Empty, [1,2])
CancelSubtract(Empty, Empty)
```

しかし、同節が定義する契約は「`term + z`が`total`を包含するtightest interval z」である。`total=Empty`なら`z=Empty`が条件を満たし、subset orderで最小である。

pinned `inari`も少なくとも`total=Empty`かつ`term`がEmptyまたはbounded common intervalのcaseで`Empty`を返す。

#### Impact

結果は安全側ではあるがtightestではなく、公開methodの定義と不一致になる。Empty入力だけで結果が最大集合へ拡大する。

#### Required action

Empty / common / unboundedの全matrixを明示し、少なくとも次を固定すること。

```text
CancelSubtract(Empty, Empty)   -> Empty
CancelSubtract(Empty, bounded) -> Empty
```

`CancelAdd`にも同じ規則を伝播すること。標準またはinariと異なるcaseを採用する場合は、明示的なsemantic deviationとして記録すること。

---

### F-PR3-015 — Medium — `IntervalUnion2` / `DecoratedInterval`の値等値性を文章で要求する一方、公開APIがそれを提供していない

- Origin: `introduced_by_consolidation_and_added_design`
- Location:
  - `doc/Design/detail/IntervalArithmetic.md` §53.1 `IntervalUnion2 API候補`
  - 同 §57.2、§57.7 `DecoratedInterval`

#### Problem

`IntervalUnion2`は次のように宣言されている。

```csharp
public readonly struct IntervalUnion2 : IEquatable<IntervalUnion2>
```

しかしAPI候補内に次がない。

- `Equals(IntervalUnion2)`
- `Equals(object?)`
- `GetHashCode()`
- `==` / `!=`

このままの宣言を実装すると`IEquatable<IntervalUnion2>`契約を満たせない。統合前の`IntervalAdvancedFeatures.md`にはこれらのmemberが明記されていた。

`DecoratedInterval`についても、§57.7がC# value equality、`NaI == NaI`、collection keyとしての`Equals`/Hashを規定しているが、型宣言に`IEquatable<DecoratedInterval>`、equality member、Hash、operatorが存在しない。

さらに`IntervalUnion2`のindexerについて、`index<0`、`index>=Count`、Count=0/1でのSecond accessの挙動も未定義である。

#### Impact

- 公開API候補が自己矛盾し、そのままbaselineへ落とせない。
- NaIのreflexive C# equalityとIEEE semantic equalityを分離する設計を、利用者が呼び分けられない。
- equalityとHashの整合性をテストできない。

#### Required action

両型のvalue-semantics surfaceを明示すること。

- `IEquatable<T>`
- typed/object `Equals`
- `GetHashCode`
- `==` / `!=`
- canonical stateとsigned zeroを含むHash契約
- indexer / component accessorのinvalid access契約

IEEE semantic equalityは引き続き別named methodとして分離すること。

---

### F-PR3-016 — Medium — decoration propagationにresult interval stateによる上限がなく、invalidな`Com` resultを生成できる

- Origin: `introduced_by_added_design`
- Location: `doc/Design/detail/IntervalArithmetic.md` §57.4、§57.6

#### Problem

設計は基本伝播を次で定義している。

```text
resultDec = min(input decorations, opDec)
```

しかし、result interval自身が許容する最大decorationを考慮していない。

具体例:

```text
X = DecoratedInterval.FromInterval([double.MaxValue,double.MaxValue]) // Com
X + X
```

bare resultはfinite overflowにより概ね次になる。

```text
[double.MaxValue,+Infinity]
```

これはunbounded intervalであり、`Com`ではなく最大でも`Dac`でなければならない。加算の`opDec`を`Com`、input minimumを`Com`とする現在の式だけでは、unbounded resultへ`Com`を付与できてしまう。

同様にEmpty resultは最大`Trv`でなければならない。

pinned `inari`の`set_dec`は、Emptyを`Trv`へ、`Com`を付けようとしたunbounded intervalを`Dac`へcanonicalizeしている。

#### Impact

- IEEE decoration invariantを破る値が作成される。
- equality、serialization、subsequent propagationの基準が不正になる。
- overflowの発生有無により同じoperationのdecoration validityが変わるが、現在の`opDec`だけでは扱えない。

#### Required action

result interval stateによる上限を明示すること。概念上は少なくとも次を必要とする。

```text
maxForResult =
  Trv  if result is Empty
  Dac  if result is unbounded nonempty
  Com  if result is bounded nonempty

resultDec = min(input decorations, opDec, maxForResult)
```

NaI / `Ill`の処理も同じcanonical constructorへ集約し、finite overflowでbounded inputからunbounded resultになるfixtureを追加すること。

---

### F-PR3-017 — Low — sole normative documentでPhase 4A relationのEmpty / inverse境界が一部未定義になった

- Origin: `introduced_by_consolidation`
- Location:
  - `doc/Design/detail/IntervalArithmetic.md` §26.6 `endpoint-wise less`
  - 同 §27 `IntervalOverlap`

#### Problem

統合版はweak endpoint-wise lessについてEmptyの3組合せを定義するが、strict版についてはnonempty式しか記載していない。

統合前文書では次が明示されていた。

```text
Empty vs Empty    -> true
Empty vs nonempty -> false
nonempty vs Empty -> false
```

また、`IntervalOverlap`のinverse listから次が脱落している。

```text
BothEmpty <-> BothEmpty
```

旧詳細ファイルは削除され、現在は統合版がsole normative documentであるため、削除済み文書へ依存できない。

#### Impact

実装者がEmpty strict-lessをweak relationと同じにするか、常にfalseにするかを再判断する必要がある。16-state inverse propertyにも1状態の期待値がない。

#### Required action

削除前のEmpty truth tableと`BothEmpty` inverseを統合版へ復元し、16状態全件のinverse fixtureへ含めること。

## Consolidation assessment

統合版には、以前review済みの四則演算に関する主要な規範は保持されている。一方、追加Phase 4文書から少なくとも次が脱落した。

- extended divisionのone-sided-zero denominator規則
- `IntervalUnion2`のvalue-equality member
- strict endpoint-wise lessのEmpty truth table
- `BothEmpty`のinverse mapping

したがって、consolidation report / handoffの「normative content preserved」「数値意味論を変更していない」という記録は、修正後に更新または訂正記録を追加する必要がある。

## Held / explicitly deferred decisions

次は統合版が明示的に後続subphase reviewへ保留しており、今回のactive findingには数えていない。

- finite Midpointの最終tie policy
- elementary production backendの選択
- Payne-Hanek reducerのtable precisionとproof artifact
- decorated operationの最終public surface
- parser resource-limitの具体値
- binary interchange version 1のreject/canonicalize細則
- batch API overlap / in-place規則

ただし各項目は、対象Phase 4 subphaseの実装開始前に確定しなければならない。

## Required coverage dispositions

| Criterion | Disposition | Findings / evidence |
|---|---|---|
| Added-scope requirement and phase conformance | `checked_finding` | F-PR3-010、013、017 |
| Numerical/set correctness | `checked_finding` | F-PR3-010、011、012、014、016 |
| Public API and value semantics | `checked_finding` | F-PR3-015 |
| Empty / Infinity / signed-zero behavior | `checked_finding` | F-PR3-010、011、014、016、017 |
| Extended/reverse operations | `checked_finding` | F-PR3-010、013、014 |
| Elementary domain / branch-cut handling | `checked_finding` | F-PR3-011、012 |
| Decorated interval semantics | `checked_finding` | F-PR3-015、016 |
| Parsing / interchange / resource safety | `checked_no_finding` | resource-limit requirement、exact decimal、versioned wire方針を確認 |
| TDD / deterministic test strategy | `checked_finding` | 基盤は妥当だがfindingsに対応するfixture追加が必要 |
| Cross-architecture / backend equivalence | `checked_no_finding` | x64/ARM64、MPFR、backend bitwise gateを確認 |
| Previous four-arithmetic regression checkpoints | `checked_no_finding` | Empty endpoint、finite overflow oracle、conformance、rounding thresholds、ISA/FMAを限定確認 |
| Scope discipline | `checked_no_finding` | documentation/reportのみ。source/test/workflow変更なし |
| License / third-party traceability | `checked_no_finding` | pinned SHAとlicense/NOTICE方針あり |
| Security / secrets | `checked_no_finding` | secret追加なし。parser resource abuse対策要求あり |
| Report / handoff accuracy | `checked_finding` | F-PR3-013、015、017によりpreservation claimの訂正が必要 |
| Exact-head CI evidence | `checked_no_finding` | reviewed HEADのmatching pull-request runは0件。CI未実施。他SHA不使用 |

## Validation assessment

### Documentation review

- current unified designをPhase 4Aから4Eまで通読した。
- consolidation前の追加設計は、統合欠落の確認に限定して比較した。
- pinned `inari`のtwo-output division、Atan2、cancellative operation、decoration canonicalizationと照合した。
- 前回review済み四則演算は、統合回帰の高リスク契約だけを確認した。

### Executable validation

未実施。repositoryに実行可能project/test targetがなく、今回の変更もdocumentation-onlyである。

これはbuild/test成功を意味しない。

### Exact-head CI

Reviewed HEAD:

```text
8e6e7499204fccf0643da82f274d1485dc0e3272
```

このSHAを`head_sha`に持つpull_request workflow runは0件である。

```text
CI status: 未実施
other-SHA substitution: false
```

## Verdict

**FAIL**

前回review済みの四則演算coreについて、新しい重複findingは起票していない。FAILの根拠は、追加されたPhase 4設計および統合による欠落だけである。

特に次の2件は、真値包含または追加型の中心目的へ直接影響する。

- `F-PR3-010`: zeroで接するcomponent closureをmergeし、two-component informationを失う。
- `F-PR3-011`: Atan2のnegative-axis片側branch-cutでunder-enclosureになり得る。

PRは`IntervalArithmetic.md`を唯一の規範とし、その統合版review完了をPhase 1開始条件としているため、現状のPR全体はapproveできない。

## Required next action

追加範囲だけを修正する。

1. `IntervalUnion2`のexact-set / component-enclosure semanticsを決定し、zero-touchを処理する。
2. `Atan2`のnegative x-axis matrixを完成する。
3. general Powのzero-boundary candidateを完成する。
4. `DivideToUnion` one-sided-zero tableを復元する。
5. cancellative operationのEmpty matrixを修正する。
6. union/decorated equality surfaceを完成する。
7. decoration result-state capを追加する。
8. relationの統合欠落を復元する。
9. consolidation report / handoffへ訂正記録を追加する。
10. 新しいimmutable HEADに対し、同じreviewerが上記finding identity単位でfix verificationする。

CIは修正後current HEADとrunの`head_sha`が一致するものだけを確認し、存在しなければCI未実施と報告する。

## Merge boundary

mergeは行わない。mergeはrepository ownerが行う。
