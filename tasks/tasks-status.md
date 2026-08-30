# タスク一覧

## 1. 文書情報

- 対象: `ssaattww/Devo6.Interval`
- 基準設計: `doc/Design/detail/IntervalArithmetic.md` 設計版5
- フェーズ定義: `tasks/phases-status.md`
- 状態値: `未着手` / `進行中` / `Blocked` / `完了`

本書は実装・検証をレビュー可能な論理単位へ分解したタスク一覧である。
各受け入れ条件は、実装者が詳細設計本文を再解釈しなくても、入力、期待結果、例外、bit一致、artifact、review evidenceのいずれかで合否を判定できる形で記載する。

## 2. 共通判定規則

source実装を含むタスクには個別条件に加えて次を適用する。

- [ ] Red commitでは、追加したtest名、実行command、終了コード、失敗messageをreportへ記録し、対象仕様が未実装であることを理由に最低1件失敗する。
- [ ] Green commitはRed commitより後の別commitとし、同じfocused test commandが終了コード0になる。
- [ ] Red/Greenを含む各commitは1つの論理単位に限定し、無関係なcleanupを同じcommitへ含めない。
- [ ] binary64 endpointの期待値は、NaN payloadをpublic contractにしない箇所を除き、`BitConverter.DoubleToInt64Bits`相当の64bit値でfixtureへ固定する。
- [ ] exact oracle / MPFR / pinned referenceのうち対象taskで指定されたreferenceとの未承認差異は0件とする。
- [ ] x64/ARM64比較対象taskでは、caseId順のcanonical result fileのSHA-256が一致する。
- [ ] 複数backend比較対象taskでは、同一caseIdのcanonical endpoint bitsがbackend間で一致する。
- [ ] failure artifact対象taskでは、失敗caseの`caseId`からinput、selected branch、expected、actual、referenceを1つのartifact set内で追跡できる。
- [ ] public API変更taskではPublic API baseline差分を保存する。未承認破壊的変更がある場合はtaskを完了にしない。
- [ ] CI証拠は確認時点のPR current HEAD SHAとworkflow runの`head_sha`が一致するrunだけを採用する。0件なら `CI未実施` とreportへ記録する。
- [ ] task完了時はstatus、詳細report、PR簡易reportを更新する。

infra/documentation/review-gateのみのtaskはRed/Greenを要求しない。それ以外の条件は適用する。

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
| P1-013 | 1 | hot-path・NativeAOT・trimming final gate | 未着手 | P1-012 |
| P2-001 | 2 | core API review/freeze | 未着手 | P1-013 |
| P2-002 | 2 | conversion/generic math/format判断 | 未着手 | P2-001 |
| P2-003 | 2 | public API baseline・breaking-change運用確定 | 未着手 | P2-002 |
| P3-001 | 3 | scalar/SIMD differential・capability基盤 | 未着手 | P2-003 |
| P3-002 | 3 | SIMD layout/load/store/batch add-sub | 未着手 | P3-001 |
| P3-003 | 3 | AVX-512 directed mul/div candidate | 未着手 | P3-001 |
| P3-004 | 3 | AVX2+FMA mul/div candidate | 未着手 | P3-001 |
| P3-005 | 3 | AVX2 no-FMA/SSE2/ARM64候補評価 | 未着手 | P3-001 |
| P3-006 | 3 | production dispatch・fallback・benchmark gate | 未着手 | P3-002～P3-005 |
| P4A-000 | 4A | Phase 4A implementation preflight | 未着手 | P3-006 |
| P4A-001 | 4A | Contains / IsBounded | 未着手 | P4A-000 |
| P4A-002 | 4A | Intersect / ConvexHull | 未着手 | P4A-001 |
| P4A-003 | 4A | relation named API | 未着手 | P4A-002 |
| P4A-004 | 4A | IntervalOverlap | 未着手 | P4A-003 |
| P4A-005 | 4A | numeric properties | 未着手 | P4A-004 |
| P4A-006 | 4A | Abs / Sign / pointwise min-max | 未着手 | P4A-005 |
| P4A-007 | 4A | Floor/Ceiling/Truncate/Round | 未着手 | P4A-006 |
| P4A-008 | 4A | Phase 4A API/conformance close | 未着手 | P4A-007 |
| P4B-000 | 4B | Phase 4B implementation preflight | 未着手 | P4A-008 |
| P4B-001 | 4B | tight IntervalConstants | 未着手 | P4B-000 |
| P4B-002 | 4B | Reciprocal | 未着手 | P4B-001 |
| P4B-003 | 4B | Square | 未着手 | P4B-002 |
| P4B-004 | 4B | Sqrt | 未着手 | P4B-003 |
| P4B-005 | 4B | integer Pow / Root | 未着手 | P4B-004 |
| P4B-006 | 4B | FusedMultiplyAdd | 未着手 | P4B-005 |
| P4B-007 | 4B | MPFR corpus・elementary endpoint backend qualification | 未着手 | P4B-006 |
| P4C-000 | 4C | Phase 4C implementation preflight | 未着手 | P4B-007 |
| P4C-001 | 4C | Exp / Exp2 / Exp10 | 未着手 | P4C-000 |
| P4C-002 | 4C | Log / Log2 / Log10 | 未着手 | P4C-001 |
| P4C-003 | 4C | Sinh / Cosh / Tanh | 未着手 | P4C-002 |
| P4C-004 | 4C | Asinh/Acosh/Atanh/Asin/Acos/Atan | 未着手 | P4C-003 |
| P4C-005 | 4C | Phase 4C backend/API gate | 未着手 | P4C-004 |
| P4D-000 | 4D | Phase 4D implementation preflight | 未着手 | P4C-005 |
| P4D-001 | 4D | high-precision periodic reducer | 未着手 | P4D-000 |
| P4D-002 | 4D | Sin / Cos | 未着手 | P4D-001 |
| P4D-003 | 4D | Tan | 未着手 | P4D-002 |
| P4D-004 | 4D | Atan2 | 未着手 | P4D-003 |
| P4D-005 | 4D | positive-base general interval Pow | 未着手 | P4D-004 |
| P4D-006 | 4D | Phase 4D backend/API gate | 未着手 | P4D-005 |
| P4E-000 | 4E | Phase 4E implementation preflight | 未着手 | P4D-006 |
| P4E-001 | 4E | IntervalUnion2 | 未着手 | P4E-000 |
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

- [ ] review reportに `reviewed HEAD=<40桁SHA>` と対象file `doc/Design/detail/IntervalArithmetic.md` を記録する。
- [ ] `F-PR3-010`～`F-PR3-017`を各finding IDごとに `required action / section / evidence / disposition` の4列で記録する。
- [ ] verdictが `pass` で、`unresolved blocking/high/medium/low findings=0` と記録される。
- [ ] review対象HEADとreport記載HEADが一致しない場合はpass扱いにしない。
- [ ] matching current-HEAD workflow runが0件なら `CI未実施` と記録し、別SHA runを引用しない。

## P0-002 フェーズ・タスク管理基盤作成

**設計参照:** §3, §44, §46, §52

### 受け入れ条件

- [ ] `tasks/phases-status.md` にPhase 0/1/2/3/4A/4B/4C/4D/4Eの9行が存在する。
- [ ] `tasks/tasks-status.md` にPhase 1 §52.1の12実装項目、Phase 4 §44のTDD順序、Phase 4A～4Eのpreflight gateが存在する。
- [ ] source taskの受け入れ条件に最低1つ、具体的な入力と期待結果、または実行commandとexit/output条件が記載される。
- [ ] `P4A-001`, `P4B-001`, `P4C-001`, `P4D-001`, `P4E-001`の直接依存先が各`P4?-000`である。
- [ ] `P1-001`に§16.4のfailure diagnostic fieldがすべて列挙される。
- [ ] `P1-013`に§47のhot-path/NativeAOT/trimming条件が列挙される。
- [ ] PR review finding `F-PR5-001`～`F-PR5-010`が本書末尾の対応表から各修正taskへ追跡できる。

---

# Phase 1

## P1-001 solution/project/CI/diagnostic artifact基盤

**設計参照:** §4, §16, §52.1

### 受け入れ条件

- [ ] solutionに`net10.0` production projectとtest projectが含まれ、`dotnet build`と`dotnet test`が終了コード0となる。
- [ ] workflow matrixに `runs-on: ubuntu-24.04` x64 と `runs-on: ubuntu-24.04-arm` ARM64の2 jobがあり、同一commit・同一test assembly・同一reference corpusを実行する。
- [ ] test commandのstdoutとstderrを別fileへ保存し、test result fileも保存する。
- [ ] artifact upload stepはsuccess/failure双方で実行される `if: always()` 相当の条件を持つ。
- [ ] artifactには `test result / stdout / stderr / diagnostic log / runtime / OS / architecture / CPU features / reference-lock / conformance summary / canonical-results.jsonl` が含まれる。
- [ ] 数値case diagnostic entryは最低限 `caseId`, `inputBits`, `selectedBranch`, `exactResult`, `devo6ResultBits`, `inariResult`, `kvResult`, `mpfrResult`, `expectedDifferenceReason` のfieldを持つ。
- [ ] reference未導入時は空欄にせず文字列 `N/A` を保存する。
- [ ] `expectedDifferenceReason`は通常一致caseで`null`、承認差異caseだけ非nullとする。
- [ ] x64/ARM64各jobがcaseId順`canonical-results.jsonl`を生成し、比較jobがbyte-for-byte diffとSHA-256をartifactへ保存する。
- [ ] failure sample entryとして `caseId=diagnostic-probe-001` 相当のfixtureを用意し、artifactからinput、branch、expected、actualを一意に特定できることを確認する。

## P1-002 Interval construction/state/normalization

**設計参照:** §5, §6, §12

### 受け入れ条件

- [ ] `[1,2]`の内部論理stateが`[-1,2]`、`[-2,-1]`が`[+2,-1]`であることをinternal test/assertionで確認する。
- [ ] `Interval.Empty`は内部2 laneの両方が同一repository-defined `CanonicalNaNBits`を持つqNaNで、片側だけNaNのraw stateはinternal validationで失敗する。
- [ ] `default(Interval) == Interval.Zero` がtrueで、Zeroのpublic Lower bits=`0x8000000000000000`、Upper bits=`0x0000000000000000`となる。
- [ ] `Interval.Empty.Lower`は`+Infinity`、`Upper`は`-Infinity`を返す。
- [ ] `Interval.Entire`は`[-Infinity,+Infinity]`、`IsEntire=true`となる。
- [ ] constructor成功fixture `(-1,1)`, `(-Inf,1)`, `(-1,+Inf)`, `(-Inf,+Inf)` が期待区間を返す。
- [ ] constructor失敗fixture `(NaN,1)`, `(0,NaN)`, `(1,-1)`, `(-Inf,-Inf)`, `(+Inf,+Inf)` は`ArgumentException`となる。
- [ ] 同じ失敗fixtureの`TryCreate`は`false`かつ`out=Interval.Empty`となる。
- [ ] `Point(1)`は`[1,1]`、`Point(NaN)`, `Point(+Inf)`, `Point(-Inf)`は拒否される。
- [ ] Empty同士はequal、zero signed-bit入力差はcanonical equality/hashへ影響せず、equal値は同じhashを返す。
- [ ] raw result constructor経由でZero/Entire/Empty/normal区間を生成した後もpublic invariantが成立し、operator実装からpublic validating constructorを呼ばない。
- [ ] `ToString()`はEmpty、Entire、`[1,2]`で例外を送出せず非空文字列を返し、round-trip/永続化保証はtest名・API docで要求しない。
- [ ] public APIからprivate field順序、physical size、qNaN payload、SIMD型を取得する契約を追加しない。

## P1-003 exact-rational oracle・boundary corpus

**設計参照:** §13, §15

### 受け入れ条件

- [ ] finite binary64を`significand * 2^exponent`へexact分解し、`1.0`, `-0.0`, min subnormal, MaxValueのround-trip bitsが一致する。
- [ ] add/sub/mul oracleは`BigInteger` exact value、div oracleはexact rational numerator/denominatorを使用する。
- [ ] positive finite overflow `R > MaxValue`でUp=`+Infinity`, Down=`+MaxValue`、negative finite overflow `R < -MaxValue`でUp=`-MaxValue`, Down=`-Infinity`となる。
- [ ] operand自体がInfinityのcaseとfinite exact result overflowを別caseId/branchとして記録する。
- [ ] `2^-969` previous/exact/next bits=`0x035fffffffffffff/0x0360000000000000/0x0360000000000001`をfixtureへ固定する。
- [ ] `2^918` previous/exact/next bits=`0x794fffffffffffff/0x7950000000000000/0x7950000000000001`をfixtureへ固定する。
- [ ] min subnormal bits=`0x0000000000000001`をfixtureへ固定する。
- [ ] production projectからoracle implementation/`BigInteger` test helperへのruntime referenceが0件である。

## P1-004 directed add/sub primitive

**設計参照:** §7, §8

### 受け入れ条件

- [ ] finite non-overflow `TwoSum(x,y)`が `s=RN(x+y)` と `e=exact(x+y)-s`を返すfixtureをexact oracleと比較する。
- [ ] `AddUp`: `e>0 -> BitIncrement(s)`、`e<=0 -> s` を固定witnessで確認する。
- [ ] `AddDown`: `e<0 -> BitDecrement(s)`、`e>=0 -> s` を固定witnessで確認する。
- [ ] exact addition witnessではUp/Downとも`BitIncrement/BitDecrement`なしで`s`を返す。
- [ ] `SubtractUp(x,y)=AddUp(x,-y)`、`SubtractDown(x,y)=AddDown(x,-y)`のcanonical bitsがexact oracleと一致する。
- [ ] positive/negative finite overflowのUp/Down結果がP1-003のoverflow ruleと一致する。
- [ ] undefined pair `+Inf + -Inf`をprimitiveへ渡すpublic pathが0件である。

## P1-005 interval add/sub/unary minus

**設計参照:** §11.1

### 受け入れ条件

- [ ] `[1,2]+[3,4]`が`[RD(4),RU(6)]=[4,6]`となる。
- [ ] `[1,2]-[3,4]`が`[RD(-3),RU(-1)]=[-3,-1]`となる。
- [ ] `-[1,2]=[-2,-1]`、`-Empty=Empty`、`-Entire=Entire`となる。
- [ ] operandのいずれかがEmptyならadd/sub結果はEmptyとなる。
- [ ] zero endpointを含む結果でlower zero bits=`-0.0`、upper zero bits=`+0.0`へcanonicalizeされる。
- [ ] internal add結果はlowerを外部表現へ戻して再格納せず、negated-lower stateのまま生成されることをinternal testで確認する。
- [ ] exact oracle corpus全caseでendpoint bitsが一致する。

## P1-006 directed multiply primitive

**設計参照:** §7, §9, §15

### 受け入れ条件

- [ ] `abs(product)>=2^-969`の通常経路で`error=FMA(x,y,-product)`を使用するfixtureを持つ。
- [ ] 通常経路Upは`error>0 -> BitIncrement(product)`、それ以外`product`、Downは`error<0 -> BitDecrement(product)`、それ以外`product`となる。
- [ ] `abs(product)<2^-969`のscaled経路は`ProductScale=2^537`を使用する。
- [ ] scaled Upは `t<s` または `t==s && s2>0` のwitnessで`BitIncrement(product)`、それ以外`product`となる。
- [ ] scaled Downは `t>s` または `t==s && s2<0` のwitnessで`BitDecrement(product)`、それ以外`product`となる。
- [ ] `t==s && s2==0` witnessはUp/Downとも無補正で`product`を返す。
- [ ] fixed witnessを `t<s`, `t>s`, `t==s&&s2>0`, `t==s&&s2<0`, exact の5 caseIdで保存し、input bitsとexpected output bitsをfixtureへ固定する。
- [ ] positive overflowはUp=`+Inf`, Down=`+MaxValue`、negative overflowはUp=`-MaxValue`, Down=`-Inf`となる。
- [ ] `0*Infinity`, NaN operandをdirected primitiveへ渡すpublic pathが0件である。

## P1-007 interval multiplication

**設計参照:** §11.2, §11.3

### 受け入れ条件

- [ ] sign classを `Z=[0,0]`, `P=0<=lowerかつ非Z`, `N=upper<=0かつ非Z`, `M=lower<0<upper` の4classへ分類する。
- [ ] Z×任意、任意×ZはZeroを返し、`0*Infinity`をprimitiveへ渡さない。
- [ ] P×P: lower=`RD(a*c)`, upper=`RU(b*d)`。
- [ ] P×N: lower=`RD(b*c)`, upper=`RU(a*d)`。
- [ ] P×M: lower=`RD(b*c)`, upper=`RU(b*d)`。
- [ ] N×P: lower=`RD(a*d)`, upper=`RU(b*c)`。
- [ ] N×N: lower=`RD(b*d)`, upper=`RU(a*c)`。
- [ ] N×M: lower=`RD(a*d)`, upper=`RU(a*c)`。
- [ ] M×P: lower=`RD(a*d)`, upper=`RU(b*d)`。
- [ ] M×N: lower=`RD(b*c)`, upper=`RU(a*c)`。
- [ ] M×M: lower=`min(RD(a*d),RD(b*c))`, upper=`max(RU(a*c),RU(b*d))`。
- [ ] 4×4 class matrixの各row/columnに最低1 fixtureを持ち、exact oracle endpoint bitsと一致する。
- [ ] Empty operandはEmptyを返す。

## P1-008 directed divide primitive

**設計参照:** §7, §10, §15

### 受け入れ条件

- [ ] denominatorを正符号化し、内部比較時は`yn>0`を満たす。
- [ ] `abs(xn)<2^-969 && abs(yn)<2^918`は`DivisionScale=2^105`でxn/yn双方をscaleする。
- [ ] `abs(xn)==2^-969`はnormal path、`abs(yn)==2^918`はlarge-denominator early-return側へ入る固定fixtureを持つ。
- [ ] positive exact quotientのlarge-denominator early returnはUp=`+2^-1074`, Down=`+0.0`となる。
- [ ] negative exact quotientのlarge-denominator early returnはUp=`+0.0`, Down=`-2^-1074`となる。
- [ ] normal pathで`q=RN(xn/yn)`, `r=RN(q*yn)`, `r2=FMA(q,yn,-r)`を記録するfixtureを持つ。
- [ ] DivideUpは `r<xn` または `r==xn && r2<0` なら`BitIncrement(q)`、それ以外`q`となる。
- [ ] DivideDownは `r>xn` または `r==xn && r2>0` なら`BitDecrement(q)`、それ以外`q`となる。
- [ ] `r==xn && r2==0` exact witnessはUp/Downとも無補正で`q`を返す。
- [ ] `r<xn`, `r>xn`, `r==xn&&r2<0`, `r==xn&&r2>0`, exact の5 caseIdでinput/expected bitsを固定する。
- [ ] positive/negative finite overflowはP1-003 ruleと一致する。
- [ ] denominator zero、`0/0`, `Inf/Inf`, NaN operandをprimitiveへ渡すpublic pathが0件である。

## P1-009 interval division

**設計参照:** §11.4～§11.7

### 受け入れ条件

- [ ] denominator positive `0<c<=d`かつnumerator Pではlower=`RD(a/d)`, upper=`RU(b/c)`となる。
- [ ] denominator positive `0<c<=d`かつnumerator Nではlower=`RD(a/c)`, upper=`RU(b/d)`となる。
- [ ] denominator positive `0<c<=d`かつnumerator Mではlower=`RD(a/c)`, upper=`RU(b/c)`となる。
- [ ] denominator negative `c<=d<0`かつnumerator Pではlower=`RD(b/d)`, upper=`RU(a/c)`となる。
- [ ] denominator negative `c<=d<0`かつnumerator Nではlower=`RD(b/c)`, upper=`RU(a/d)`となる。
- [ ] denominator negative `c<=d<0`かつnumerator Mではlower=`RD(b/d)`, upper=`RU(a/d)`となる。
- [ ] `[1,2]/[0,0]=Empty`, `[0,0]/[0,0]=Empty`となり`DivideByZeroException`を送出しない。
- [ ] `Zero/[0,d]=Zero`, `Zero/[c,0]=Zero`, `Zero/[c,d]` with `c<0<d` = Zeroとなる。
- [ ] `[1,2]/[0,2]=[RD(1/2),+Infinity]`となる。
- [ ] `[-2,-1]/[0,2]=[-Infinity,RU(-1/2)]`となる。
- [ ] `[1,2]/[-2,0]=[-Infinity,RU(1/-2)]`となる。
- [ ] `[-2,-1]/[-2,0]=[RD(-1/-2),+Infinity]`となる。
- [ ] numerator Mかつone-sided zero denominatorはEntireとなる。
- [ ] strict zero-crossing denominator `c<0<d`はnumerator ZeroならZero、それ以外Entireとなる。
- [ ] reciprocalを一度構築してmultiplyする実装経路をbasic divisionから使用しない。
- [ ] Empty operandはEmptyとなる。

## P1-010 Phase 1 conformance harness

**設計参照:** §14

### 受け入れ条件

- [ ] manifestに `empty, entire, numsToInterval, inf, sup, isEmpty, isEntire, isSingleton, equal, neg, add, sub, mul, div` の14 conceptがrequiredとして存在する。
- [ ] ITF1788 source extractionで宣言operationが0件ならtest successではなく`source extraction error`で終了コード非0となる。
- [ ] constructor fixture `(-1,1)`, `(-Inf,1)`, `(-1,+Inf)`, `(-Inf,+Inf)`が成功する。
- [ ] constructor fixture `(NaN,NaN)`, `(1,-1)`, `(-Inf,-Inf)`, `(+Inf,+Inf)`がinvalidとしてconstructor=`ArgumentException`, TryCreate=`false/out=Empty`となる。
- [ ] IsSingleton matrixでEmpty=false, Entire=false, `[1,1]`=true, `[-2,-2]`=true, Zero=true, `[1,2]`=false, `[-Inf,2]`=false, `[1,+Inf]`=falseとなる。
- [ ] `inf(Empty)=+Infinity`, `sup(Empty)=-Infinity`を確認する。
- [ ] manifest各caseに `source/path/testcase/adaptation/status/expected` が存在し、required/deferred/excluded/approved-deviationを区別する。
- [ ] required case failure数が0となる。

## P1-011 pinned inari/kv corpus・reference lock

**設計参照:** §13.3, §13.4

### 受け入れ条件

- [ ] `tests/ReferenceData/reference-lock.json`にinari SHA=`18b83a571d7681c76067bc38d90a74e8be29f545`, kv SHA=`c7f8f2324a0e403cca6b39f46088a22843d440db`, ITF1788 SHA=`d8c2a64478ebdc9cbde6ccef33eaad3bed60ed81`, MPFR versionまたは`N/A`を保存する。
- [ ] lockにadapter/generator hash、toolchain/target triple、generator command、corpus SHA-256、license/NOTICE pathを保存する。
- [ ] corpusはJSON Lines、caseId昇順、binary64値は16桁hex bitsで保存する。
- [ ] generatorを同一lock条件で2回実行し、corpus SHA-256が一致する。
- [ ] kvのzero-containing interval division resultをoracleとして使用するcaseが0件である。
- [ ] exact oracleとadopted semanticsがreference libraryと異なるcaseは`expectedDifferenceReason`を必須とする。

## P1-012 sample・API evaluation report

**設計参照:** §5, §46, §52.1

### 受け入れ条件

- [ ] public APIだけで `new Interval(1,2)+Interval.Point(3)` を実行し結果 `[4,5]`をassertするsampleがある。
- [ ] public APIだけで `new Interval(1,2)/new Interval(-1,1)` を実行し`IsEntire=true`、例外なしをassertするsampleがある。
- [ ] `new Interval(2,1)` が`ArgumentException`となり、`Interval.Empty`生成経路と区別されるsample/testがある。
- [ ] sample codeからinternal namespace、raw constructor、SIMD/backend型、reference adapterへの参照が0件である。
- [ ] reportに上記3scenarioのsource path、command、actual result、pass/failを記録する。
- [ ] reportにPhase 2で決定が必要な `namespace / constructor-vs-factory / scalar conversion / generic math / formal format` の5項目を列挙する。

## P1-013 hot-path・NativeAOT・trimming final gate

**設計参照:** §47

### 受け入れ条件

- [ ] BenchmarkDotNetでbasic `+`, `-`, `*`, `/` のAllocated columnが全て`0 B`となる。
- [ ] JIT disassemblyまたはDisassemblyDiagnoserでbasic operator pathにinterface dispatch、delegate invoke、virtual indirect dispatchが存在しないことをevidence fileへ保存する。
- [ ] production source/ILにreflection、`System.Reflection.Emit`、dynamic assembly generation、runtime code generation、native resolver登録を使用するpathが0件である。
- [ ] Linux x64で `dotnet publish -c Release -r linux-x64 -p:PublishAot=true` が成功し、publish binaryのbasic 3scenario smokeが終了コード0となる。
- [ ] Linux ARM64でもNativeAOT publish/smokeが終了コード0となる。
- [ ] `PublishTrimmed=true` publishでbasic 3scenario smokeが終了コード0となる。
- [ ] operator disassembly/call graphでpublic validating constructor呼出しがなく、internal raw result constructionを使用することを確認する。
- [ ] production hot pathに`BigInteger`呼出しが0件、global floating-point rounding mode変更callが0件である。

---

# Phase 2

## P2-001 core API review/freeze

**設計参照:** §5, §46

### 受け入れ条件

- [ ] API decision recordにassembly/package、namespace、`Interval` type、constructor/factory、Lower/Upper、state properties/constants、operators、equality/hash、exception、signed-zero、ToStringの各項目を単一決定として記録し`TBD`を0件にする。
- [ ] compile fixtureでP1-012の3scenarioが変更なしでpassする。
- [ ] `Interval.Empty.Lower=+Inf`, `Upper=-Inf`, `default(Interval)==Zero`がAPI freeze fixtureでpassする。
- [ ] normal comparison operator `< <= > >=` がpublic baselineに存在しない。
- [ ] review reportにpublic API diffを添付し、unresolved API finding=0かつverdict=passとなる。

## P2-002 conversion/generic math/format判断

**設計参照:** §5.9, §5.10, §50

### 受け入れ条件

- [ ] scalar implicit/explicit conversionを`Adopt`または`Reject`の1値で記録する。
- [ ] Adoptならexact C# signatureと成功/失敗fixtureを記載し、Rejectならbaselineにconversion operatorが0件であることをtestする。
- [ ] generic math候補interfaceが`<`, `<=`, `>`, `>=`またはtotal-order比較契約を必須とする場合、そのinterfaceは`Reject`としpublic baselineに当該interface実装と4比較operatorを追加しない。
- [ ] total-order比較を要求しないgeneric math interfaceをAdoptする場合、adoptするinterface名と全required member signatureをdecision recordへ列挙し、compile fixtureで全memberを解決できることを確認する。
- [ ] diagnostic `ToString`と永続化formatを同一契約にするか分離するかを1値で記録し、formal formatを採用する場合はround-trip fixtureを追加する。
- [ ] scalar conversion/generic math/formatの3項目すべてにdecision owner/date/rationale/test pathがあり`TBD`が0件となる。

## P2-003 public API baseline・breaking-change運用確定

**設計参照:** §46

### 受け入れ条件

- [ ] public API baseline fileをrepositoryに保存し、CI/local gateでbaseline差分0を確認するcommandを記録する。
- [ ] public APIを1 signature変更したnegative fixture/validationでbaseline gateが終了コード非0になることを確認する。
- [ ] `doc/Design/BreakingChanges.md` entry templateに `old signature / new signature / reason / migration example / version` の5 fieldを定義する。
- [ ] unapproved baseline差分がある状態ではPhase 3 taskを開始不可とする依存関係を文書化する。
- [ ] final API baseline commit SHAをreportへ記録する。

---

# Phase 3

## P3-001 scalar/SIMD differential・capability基盤

**設計参照:** §17.1, §17.2

### 受け入れ条件

- [ ] capability modelに`Avx512F`, `Avx2`, `Avx`, `Fma`, `Sse2`, `AdvSimd.Arm64`の6独立flagが存在する。
- [ ] test seamで`Avx2=true/Fma=false`と`Avx2=false/Fma=true`を別々に表現できる。
- [ ] scalar referenceとcandidate backendへ同一input corpusを渡し、caseIdごとにendpoint bit diffを出力できる。
- [ ] diff 1件を注入したnegative fixtureでgateが終了コード非0となり、caseId/expected/actual/backendをartifactへ出す。
- [ ] fallback backend名をdiagnostic logへ記録できる。

## P3-002 SIMD layout/load/store/batch add-sub

**設計参照:** §17.3

### 受け入れ条件

- [ ] 4 interval AVX-512 layoutを`[-L0,U0,-L1,U1,-L2,U2,-L3,U3]`の8 lane順で固定する。
- [ ] `[1,2],[-3,4],[0,0],Entire`をload/storeしexternal endpoint bitsが元入力と一致する。
- [ ] batch Add/SubをN=`1,2,3,4,5,7,8,32`でscalar referenceと比較し全endpoint bits一致する。
- [ ] N%4=1/2/3のtailをscalarで処理し、out-of-range read/writeがないことをboundary testで確認する。
- [ ] Emptyを含むbatchで対応elementだけEmptyが伝播し、隣接elementを汚染しない。

## P3-003 AVX-512 directed mul/div candidate

**設計参照:** §17.2～§17.4

### 受け入れ条件

- [ ] AVX-512F supported時だけcandidateを選択し、unsupported時はcandidate entryを実行しない。
- [ ] packed directed roundingでmul/divのlower/upperを計算し、scalar referenceとnormal/subnormal/overflow/Infinity/zero-containing interval corpusでbitwise一致する。
- [ ] 4 interval batchとtail N=1～3の両方でbitwise一致する。
- [ ] correctness mismatchが1件でもあればcandidate status=`rejected_correctness`としproduction dispatchへ登録しない。
- [ ] candidate result reportにCPU feature、commit SHA、corpus SHA-256、mismatch countを記録する。

## P3-004 AVX2+FMA mul/div candidate

**設計参照:** §17.2, §17.4

### 受け入れ条件

- [ ] candidate選択条件が`Avx2.IsSupported && Fma.IsSupported`である。
- [ ] vector FMA residual pathのmul/divがscalar directed primitiveとnormal/subnormal/overflow/threshold corpusでbitwise一致する。
- [ ] FMA=true/AVX2=false test seamではこのcandidateを選択しない。
- [ ] mismatch 0件の場合のみstatus=`qualified_correctness`、1件以上なら`rejected_correctness`とする。
- [ ] performance採否は本taskで決めずP3-006の事前固定benchmark gateだけで決める。

## P3-005 AVX2 no-FMA/SSE2/ARM64候補評価

**設計参照:** §17.2

### 受け入れ条件

- [ ] `AVX2=true,FMA=false`ではadd/sub vector候補を評価し、mul/div candidateがP3-001と同一required differential corpusでmismatch=0にならない限りmul/divはscalar fallbackとする。
- [ ] `SSE2=true,FMA=false`ではVector128 add/sub候補を評価し、mul/div candidateが同じrequired corpusでmismatch=0にならない限りmul/divはscalar fallbackとする。
- [ ] ARM64 AdvSimd candidateはscalar differential全required corpusでmismatch=0となるまでproduction candidate statusにしない。
- [ ] 各feature combinationに `selected backend / add-sub status / mul-div status / mismatch count / fallback` をmatrixで保存する。
- [ ] unsupported combinationでbasic public APIが`PlatformNotSupportedException`を送出しない。

## P3-006 production dispatch・fallback・benchmark gate

**設計参照:** §17.4, §47

### 受け入れ条件

- [ ] benchmark policy commitをcandidate result測定commitより前に作成し、履歴で順序を確認できる。
- [ ] policyはBenchmarkDotNet/.NET 10、scalar baseline、同一input corpus、batch N=`4,32,256,4096`、operation=`Add,Sub,Mul,Div`、metric=`median ns/interval`、allocationを固定する。
- [ ] corpus seedと各input bits fileのSHA-256をpolicy/reportへ記録し、scalar/candidateで同じfileを使用する。
- [ ] production採用条件をN>=256全workloadのscalar比median幾何平均`<=0.95`、各workload`<=1.02`、allocation増加`0 B`に固定する。
- [ ] correctness mismatch>0のcandidateは性能結果にかかわらず不採用となる。
- [ ] 上記performance条件を1つでも満たさないcandidateはproduction dispatch tableへ登録しない。
- [ ] dispatch tableの各entryにrequired ISA/FMA、fallback backend、correctness report path、benchmark report pathを記録する。
- [ ] force-disable全SIMD testでscalar fallbackが選択され、canonical corpus SHA-256がscalar baselineと一致する。
- [ ] public API baseline diffが0件である。

---

# Phase 4A

## P4A-000 Phase 4A implementation preflight

**設計参照:** §19～§24, §46, §52.2

### 受け入れ条件

- [ ] Phase 1～3の必須taskが全件`完了`、P2 public API baseline gateがpassである。
- [ ] review reportにPhase 4A対象§19～§24、`reviewed HEAD=<40桁SHA>`, `verdict=pass`, `unresolved findings=0`を記録する。
- [ ] review report pathとreviewed HEADをP4A-000 reportへ記録する。
- [ ] diagnostic artifact workflow fileが`.github/workflows/`配下に存在する。
- [ ] smoke fixture `Entire.Contains(+Infinity)==false` と `Entire.Contains(0.0)==true` が既存harnessでpassする。
- [ ] Midpoint tie policyとPhase 4A public namingを単一値でdecision recordへ固定し`TBD`を0件にする。
- [ ] P4A-001の最初のRed commitのparent時点でP4A-000 status=`完了`である。

## P4A-001 Contains / IsBounded

**設計参照:** §21.3, §24.1

### 受け入れ条件

- [ ] `Empty.Contains(0)==false`, `Entire.Contains(0)==true`, `Entire.Contains(+Inf)==false`, `Entire.Contains(-Inf)==false`, `Entire.Contains(NaN)==false`となる。
- [ ] `Zero.Contains(+0.0)==true`かつ`Zero.Contains(-0.0)==true`となる。
- [ ] `[1,2].Contains(1)==true`, `.Contains(2)==true`, `.Contains(0)==false`, `.Contains(3)==false`となる。
- [ ] `IsBounded`はEmpty=false, Entire=false, `[1,2]`=true, `[-Inf,2]`=false, `[1,+Inf]`=falseとなる。

## P4A-002 Intersect / ConvexHull

**設計参照:** §21.1, §21.2

### 受け入れ条件

- [ ] `Intersect(Empty,Y)=Empty`と`Intersect(X,Empty)=Empty`となる。
- [ ] `Intersect([1,3],[2,4])=[2,3]`、`Intersect([1,2],[3,4])=Empty`となる。
- [ ] `ConvexHull(Empty,[1,2])=[1,2]`、`ConvexHull([1,2],Empty)=[1,2]`となる。
- [ ] `ConvexHull([1,3],[2,4])=[1,4]`となる。
- [ ] fixed-seed propertyでintersection/hullのcommutativeとidempotentがpassする。

## P4A-003 relation named API

**設計参照:** §22

### 受け入れ条件

- [ ] `Empty.IsSubsetOf(Y)==true`、nonempty `.IsSubsetOf(Empty)==false`となる。
- [ ] `[2,3].IsSubsetOf([1,4])==true`, `[0,3].IsSubsetOf([1,4])==false`となる。
- [ ] `Empty.IsInteriorOf(Y)==true`、`Entire.IsInteriorOf(Entire)==true`となる。
- [ ] `Empty.IsDisjointFrom(Y)==true`; `[1,2]`と`[2,3]`はdisjoint=false、`[1,2]`と`[3,4]`はtrueとなる。
- [ ] Empty involved `Precedes`/`StrictlyPrecedes`はtrueとなる。
- [ ] weak/strict endpoint-wise lessのEmpty matrixはEmpty/Empty=true、Empty/nonempty=false、nonempty/Empty=falseとなる。
- [ ] public API baselineに`operator <, <=, >, >=`が存在しない。

## P4A-004 IntervalOverlap

**設計参照:** §23

### 受け入れ条件

- [ ] enumに`BothEmpty, FirstEmpty, SecondEmpty, Before, Meets, Overlaps, Starts, ContainedBy, Finishes, Equals, FinishedBy, Contains, StartedBy, OverlappedBy, MetBy, After`の16値が存在する。
- [ ] 16 stateそれぞれを発生させる最低1 fixtureをcaseId付きで固定する。
- [ ] `BothEmpty`のinverseが`BothEmpty`、`FirstEmpty<->SecondEmpty`, `Before<->After`, `Meets<->MetBy`, `Overlaps<->OverlappedBy`, `Starts<->StartedBy`, `ContainedBy<->Contains`, `Finishes<->FinishedBy`, `Equals<->Equals`となる。
- [ ] 全fixtureで`GetOverlap(X,Y)`とinverse(`GetOverlap(Y,X)`)が一致する。

## P4A-005 numeric properties

**設計参照:** §24.1～§24.3

### 受け入れ条件

- [ ] `Width(Empty)=NaN`, `Width([1,1])=+0.0`, `Width([1,2])=RU(1)`, `Width(Entire)=+Inf`となる。
- [ ] `Midpoint(Empty)=NaN`, `Midpoint(Entire)=+0.0`, `Midpoint([-Inf,2])=double.MinValue`, `Midpoint([1,+Inf])=double.MaxValue`となる。
- [ ] finite midpointのtie fixtureはP4A-000で固定したtie policyのexpected bitと一致する。
- [ ] Radiusは`max(SubtractUp(mid,Lower),SubtractUp(Upper,mid))`と同じbitを返しnegative値にならない。
- [ ] `Magnitude(Empty)=NaN`, `Magnitude([-2,1])=2`, `Mignitude([-2,1])=+0.0`, `Mignitude([2,3])=2`となる。

## P4A-006 Abs / Sign / pointwise min-max

**設計参照:** §24.4

### 受け入れ条件

- [ ] `Abs(Empty)=Empty`, `Abs([1,2])=[1,2]`, `Abs([-2,-1])=[1,2]`, `Abs([-2,1])=[-0.0,2]`となる。
- [ ] `Sign(Empty)=Empty`, `Sign([-2,-1])=[-1,-1]`, `Sign(Zero)=Zero`, `Sign([1,2])=[1,1]`, `Sign([-2,3])=[-1,1]`となる。
- [ ] `PointwiseMin([1,3],[2,4])=[1,3]`, `PointwiseMax([1,3],[2,4])=[2,4]`となる。
- [ ] min/maxはいずれかEmptyならEmptyとなる。
- [ ] zeroを含む結果のlower/upper signed-zero bitsがcanonical ruleと一致する。

## P4A-007 Floor/Ceiling/Truncate/Round

**設計参照:** §24.5

### 受け入れ条件

- [ ] Empty inputは4関数すべてEmptyを返す。
- [ ] Infinity endpointを持つ区間でInfinity endpointを保持する。
- [ ] `Floor([1.2,2.8])=[1,2]`, `Ceiling([1.2,2.8])=[2,3]`, `Truncate([-1.8,2.8])=[-1,2]`となる。
- [ ] P4A-000 decision recordにサポートする全`MidpointRounding` enum値と各1 tie input/expected integerを表で固定し、Round testがその全rowをpassする。
- [ ] 未知`MidpointRounding` enum値をcast入力した場合`ArgumentOutOfRangeException`となる。
- [ ] integer result endpointに追加ULP拡張を行わない。

## P4A-008 Phase 4A API/conformance close

**設計参照:** §44～§46

### 受け入れ条件

- [ ] P4A-001～007のfocused testsが全件passする。
- [ ] property tests `Intersect commutative/idempotent`, `Hull commutative/idempotent`, `Intersect subset operands`, `operands subset Hull`, `Width/Radius>=0`, `Mignitude<=Magnitude`が固定seedでpassする。
- [ ] x64/ARM64 canonical result SHA-256が一致する。
- [ ] qualified backend間endpoint bits mismatch=0となる。
- [ ] public API baselineを更新しunapproved diff=0となる。
- [ ] failure artifactでfunction名、input bits、branch、expected/actual/referenceを追跡できる。

---

# Phase 4B

## P4B-000 Phase 4B implementation preflight

**設計参照:** §25～§27, §46, §52.2

### 受け入れ条件

- [ ] P4A-008が`完了`である。
- [ ] review reportに§25～§27、reviewed HEAD、verdict=pass、unresolved findings=0を記録する。
- [ ] diagnostic workflowが存在し、P4A final run artifactを1件取得できる。
- [ ] input=`[-2,1]`, expected=`[-0.0,4]`のSquare smoke Red testをharnessへ登録し実行commandをreportへ記録する。
- [ ] P4B-001 Red commit前にP4B-000 status=`完了`とする。

## P4B-001 tight IntervalConstants

**設計参照:** §26.1

### 受け入れ条件

- [ ] `Pi, HalfPi, TwoPi, E, Ln2, Ln10, Sqrt2`の7 constantについてMPFR RNDD/RNDU生成lower/upper bitsをfixtureへ固定する。
- [ ] generatorは各constantで`lower <= exact value <= upper`を確認する。
- [ ] exact valueがbinary64で表現不能なconstantをnearest doubleのsingletonとして保存しない。
- [ ] generator version、MPFR version、command、output SHA-256をreference-lockへ記録する。
- [ ] production buildはMPFR/network/native generatorを必要とせず固定bitsだけでconstantを構築する。

## P4B-002 Reciprocal

**設計参照:** §26.2

### 受け入れ条件

- [ ] `Reciprocal(Empty)=Empty`, `Reciprocal(Zero)=Empty`, `Reciprocal([-1,1])=Entire`となる。
- [ ] `Reciprocal([0,2])=[RD(1/2),+Inf]`となる。
- [ ] `Reciprocal([-2,0])=[-Inf,RU(1/-2)]`となる。
- [ ] strict positive `[1,2] -> [RD(1/2),RU(1)]`、strict negative `[-2,-1] -> [RD(-1),RU(-1/2)]`となる。
- [ ] endpoint bitsがdirected divide primitiveと一致する。

## P4B-003 Square

**設計参照:** §26.3

### 受け入れ条件

- [ ] `Square(Empty)=Empty`, `Square([1,2])=[RD(1),RU(4)]`, `Square([-2,-1])=[RD(1),RU(4)]`となる。
- [ ] `Square([-2,1])=[-0.0,4]`となる。
- [ ] implementationがpublic `X*X`へ単純委譲しないことをcall graph/source testで確認する。
- [ ] fixed-seed propertyで`Square(X).IsSubsetOf(X*X)==true`となる。

## P4B-004 Sqrt

**設計参照:** §27.1

### 受け入れ条件

- [ ] `Sqrt(Empty)=Empty`, `Sqrt([-4,-1])=Empty`, `Sqrt([-1,4])=[-0.0,2]`, `Sqrt([1,4])=[1,2]`となる。
- [ ] `SmallSqrtInputThreshold=2^-969`, `SqrtInputScale=2^106`, `SqrtResultScale=2^53`をfixtureで確認する。
- [ ] `2^-969` previous/exact/nextの3caseでselected branchとMPFR directed output bitsを固定する。
- [ ] candidate補正前後でexact-product comparisonを行い、MPFR resultがcandidateより上側ならUpを`BitIncrement`、下側ならDownを`BitDecrement`し、exact一致時は無補正となるfixed witnessを持つ。

## P4B-005 integer Pow / Root

**設計参照:** §27.2, §27.3

### 受け入れ条件

- [ ] `Pow(Empty,0)=Empty`, `Pow([2,3],0)=[1,1]`となる。
- [ ] positive odd/even exponentをそれぞれnegative/positive/zero-crossing intervalで固定fixture化する。
- [ ] negative exponentでZero-only=Empty、odd strict zero crossing=Entire、one-sided zero touchは設計endpoint+Infinity ruleとなる。
- [ ] `int.MinValue` exponentをoverflowせず処理するfixtureがpassする。
- [ ] `Root(value,degree<=0)`は`ArgumentOutOfRangeException`、degree=1はinput identityとなる。
- [ ] even Rootのnegative-only input=Empty、zero crossing lower=-0.0、odd Rootはnegative domainを含めmonotonic resultとなる。
- [ ] Root candidate^degreeとinput exact relationによる隣接補正がMPFR/reference bitsと一致する。

## P4B-006 FusedMultiplyAdd

**設計参照:** §27.4

### 受け入れ条件

- [ ] endpoint primitiveはexact `x*y+z`を1回だけdirected roundingする。
- [ ] implementationがpublic `(X*Y)+Z`へ単純委譲しないことをcall graph/source testで確認する。
- [ ] 二重丸めで差が出る固定witnessを持ち、FMA resultがexact oracle endpoint bitsと一致する。
- [ ] fixed-seed propertyで`FMA(X,Y,Z).IsSubsetOf((X*Y)+Z)==true`となる。
- [ ] Empty operand伝播とInfinity/zero undefined pair handlingを固定fixtureでpassする。

## P4B-007 MPFR corpus・elementary endpoint backend qualification

**設計参照:** §18, §25, §28, §33

### 受け入れ条件

- [ ] MPFR version、precision=53、RNDD/RNDU、generator hash、corpus SHA-256をreference-lockへ保存する。
- [ ] managed backendを正式採用する場合、全required endpoint corpusでMPFR bits mismatch=0とする。
- [ ] native backendを使用しない場合はreportへ`native_backend_gate=N/A`と具体理由を記録する。
- [ ] native backendを採用する場合、interop/copy/dispatch込みbenchmarkをP3-006と同じpolicyで測定し採用閾値をpassする。
- [ ] native採用時、Linux x64/ARM64配布assetがCIから利用でき、missing binaryによる通常API failure=0となる。
- [ ] native採用時、ABI smoke、32 parallel concurrent-call stress、NativeAOT publish、trimming publishが全て終了コード0となる。
- [ ] native採用時、license/NOTICE/binary redistribution条件をrepository fileへ保存する。
- [ ] managed/nativeでpublic `Interval` API baseline diff=0、同一corpus endpoint bits mismatch=0となる。

---

# Phase 4C

## P4C-000 Phase 4C implementation preflight

**設計参照:** §28～§29, §33, §46, §52.2

### 受け入れ条件

- [ ] P4B-007が`完了`である。
- [ ] §28～§29と§33のreview reportがreviewed HEAD、verdict=pass、unresolved findings=0を持つ。
- [ ] endpoint backendがP4B-007でqualified済みである。
- [ ] smoke fixture `Log([-1,1])=[-Inf,+0.0]`をMPFR harnessで比較可能である。
- [ ] P4C-001 Red commit前にP4C-000 status=`完了`とする。

## P4C-001 Exp / Exp2 / Exp10

**設計参照:** §29.1

### 受け入れ条件

- [ ] Empty inputはEmptyを返す。
- [ ] lower=-Infinity endpointはlower result=+0.0、upper=+Infinity endpointはupper result=+Infinityとなる。
- [ ] `Exp([0,0])=[1,1]`, `Exp2([0,0])=[1,1]`, `Exp10([0,0])=[1,1]`となる。
- [ ] 各関数でMPFR corpusからunderflow境界直前/直後1caseずつ、overflow境界直前/直後1caseずつをfixtureへ固定しRNDD/RNDU bitsと一致する。
- [ ] finite normal/subnormal corpusでMPFR RNDD/RNDU bits mismatch=0となる。

## P4C-002 Log / Log2 / Log10

**設計参照:** §29.2

### 受け入れ条件

- [ ] `b<=0`のinputはEmptyとなる。
- [ ] `a<=0<b`ではlower=-Infinityとなる。
- [ ] `b=+Infinity`ではupper=+Infinityとなる。
- [ ] `[1,1]`はLog系でexact zero endpointを返す。
- [ ] positive finite endpoint corpusでMPFR RNDD/RNDU bitsと一致する。

## P4C-003 Sinh / Cosh / Tanh

**設計参照:** §29.3

### 受け入れ条件

- [ ] Sinh/Tanhはmonotonic increasing endpoint ruleでMPFR bitsと一致する。
- [ ] `Cosh([-3,-2])=[CoshDown(-2),CoshUp(-3)]`、`Cosh([2,3])=[CoshDown(2),CoshUp(3)]`となる。
- [ ] `Cosh([-2,3])=[1,CoshUp(3)]`となる。
- [ ] Empty/±Infinity endpoint fixtureを各関数でpassする。

## P4C-004 Asinh/Acosh/Atanh/Asin/Acos/Atan

**設計参照:** §29.3

### 受け入れ条件

- [ ] Asinh/Atanは全実数monotonic increasingとしてMPFR bitsと一致する。
- [ ] Acoshは`b<1 -> Empty`, `a<1<=b`ではlower=0 limit, positive valid rangeはdirected endpointsとなる。
- [ ] Asin/Acosは`[-1,1]`へclipし、intersectionがEmptyならEmptyとなる。
- [ ] Atanhは`(-1,1)`へclipし、`[-1,-1]`と`[1,1]`はEmpty、boundary接触では±Infinity limitを返す。
- [ ] domain clipping前後のinput/branchをdiagnostic artifactへ記録する。

## P4C-005 Phase 4C backend/API gate

**設計参照:** §33, §45, §46

### 受け入れ条件

- [ ] P4C-001～004 required MPFR corpus mismatch=0となる。
- [ ] approved deviationがある場合はcaseId/expected/actual/reason/approverがmanifestに全件存在する。
- [ ] x64/ARM64 canonical result SHA-256が一致する。
- [ ] qualified backend間bits mismatch=0となる。
- [ ] normal inputで`PlatformNotSupportedException`件数=0となる。
- [ ] public API baselineのunapproved diff=0となる。

---

# Phase 4D

## P4D-000 Phase 4D implementation preflight

**設計参照:** §28, §30～§33, §46, §52.2

### 受け入れ条件

- [ ] P4C-005が`完了`である。
- [ ] §30～§33 review reportにreviewed HEAD、verdict=pass、unresolved findings=0を記録する。
- [ ] high-precision reducer table format/bit length/generator hashをreviewで固定し`TBD`を0件にする。
- [ ] smoke fixture `Atan2(Zero,[-2,-1])=Pi` と `Sin([0,HalfPi])=[0,1]` をreference harnessへ登録する。
- [ ] P4D-001 Red commit前にP4D-000 status=`完了`とする。

## P4D-001 high-precision periodic reducer

**設計参照:** §30

### 受け入れ条件

- [ ] reducerが固定high-precision `2/pi` と `pi/2` tableを使用し、`Math.PI`通常除算だけでquadrantを決定するpathが0件である。
- [ ] `% (2*Math.PI)`だけでperiodic critical point/poleを決定するpathが0件である。
- [ ] exact critical lattice直前/一致/直後の固定binary64 witnessでquadrant/pole判定がreferenceと一致する。
- [ ] MaxValue近傍を含むlarge-magnitude corpusでMPFR/reference quadrant判定 mismatch=0となる。
- [ ] reducer diagnosticにinput bits、reduced quadrant、integer k、critical/pole decisionを保存する。

## P4D-002 Sin / Cos

**設計参照:** §30.1, §30.2

### 受け入れ条件

- [ ] Sin intervalが`-pi/2+2kpi`を含むfixtureでlower=-1、`+pi/2+2kpi`を含むfixtureでupper=+1となる。
- [ ] Cos intervalが`pi+2kpi`を含むfixtureでlower=-1、`2kpi`を含むfixtureでupper=+1となる。
- [ ] `Sin(Entire)=[-1,1]`, `Cos(Entire)=[-1,1]`となる。
- [ ] input=`[-IntervalConstants.Pi.Upper,+IntervalConstants.Pi.Upper]`でSin/Cosとも`[-1,1]`となる。
- [ ] branch内endpoint-only fixtureでMPFR directed endpoint bitsと一致する。
- [ ] fixed-seed propertyでresult subset `[-1,1]`が全件trueとなる。

## P4D-003 Tan

**設計参照:** §30.3

### 受け入れ条件

- [ ] poleを含まない1 branch inputは`[TanDown(a),TanUp(b)]`となる。
- [ ] poleへdomain内から両側/片側で接近可能なinputはbare result=Entireとなる。
- [ ] input intersectionがpole pointだけでdomain内点0件ならEmptyとなる。
- [ ] pole直前/直後/跨ぎfixtureでreducer pole decisionがreferenceと一致する。
- [ ] diagnostic artifactにpole index kとbranch decisionを保存する。

## P4D-004 Atan2

**設計参照:** §31, §44.1 F-PR3-011

### 受け入れ条件

- [ ] operand EmptyならEmpty、`X=Zero && Y=Zero`ならEmptyとなる。
- [ ] strictly negative XでY strictly negative/nonpositive-touch-zero/Zero/nonnegative-touch-zero/strictly-positive/crossing-zeroの6classを各1fixture以上持つ。
- [ ] `Atan2([-1,0],[-2,-1])`はlower=`-IntervalConstants.Pi.Upper`, upper=`+IntervalConstants.Pi.Upper`となる。
- [ ] `Atan2(Zero,[-2,-1])=IntervalConstants.Pi`となる。
- [ ] `Atan2([0,1],[-2,-1])`はlower=`Atan2Down(1,-1)`, upper=`IntervalConstants.Pi.Upper`となる。
- [ ] `Atan2([-1,1],[-2,-1])`は`[-IntervalConstants.Pi.Upper,+IntervalConstants.Pi.Upper]`となる。
- [ ] signed zeroをbranch cut上下の別点として扱わずY=Zeroをprincipal value +piとする。
- [ ] sign-class直積、axis、origin、negative-x branch cut corpusでMPFR/reference mismatch=0となる。

## P4D-005 positive-base general interval Pow

**設計参照:** §32, §44.1 F-PR3-012

### 受け入れ条件

- [ ] baseを`[0,+Infinity]`へclipし、clip後EmptyならEmptyを返す。negative-only baseはEmpty、negative-to-positive crossing baseは非negative部分だけを評価する。
- [ ] `Pow([0,0.5],[0,1])=[0,1]`となる。
- [ ] `Pow([0,0.5],[-1,0])=[1,+Infinity]`となる。
- [ ] `Pow([0,2],[0,1])=[0,2]`となる。
- [ ] `Pow([0,0],[-1,0])=Empty`、`Pow([0,0],[0,1])=Zero`となる。
- [ ] zero-touch base`[0,b]`, `b>0`かつ`d<0`ではlower=`b<1 ? PowDown(b,d) : PowDown(b,c)`, upper=`+Infinity`となる。
- [ ] zero-touch baseかつ`c<0 && d==0`ではlower=`b<=1 ? 1 : PowDown(b,c)`, upper=`+Infinity`となる。
- [ ] zero-touch baseかつ`c==0 && d==0`では`[1,1]`となる。
- [ ] zero-touch baseかつ`c<0<d`では`[0,+Infinity]`となる。
- [ ] zero-touch baseかつ`c==0<d`ではlower=0, upper=`b<=1 ? 1 : PowUp(b,d)`となる。
- [ ] zero-touch baseかつ`c>0`ではlower=0, upper=`b<1 ? PowUp(b,c) : (b==1 ? 1 : PowUp(b,d))`となる。
- [ ] internal hookで`PowDown/Up(0,0)`および`PowDown/Up(0,negative)`call count=0を確認する。
- [ ] strictly-positive-base corner formula corpusでMPFR mismatch=0となる。

## P4D-006 Phase 4D backend/API gate

**設計参照:** §33, §45, §46

### 受け入れ条件

- [ ] P4D-001～005 required fixtureが全件passする。
- [ ] Sin/Cos/Tan/Atan2/PowのMPFR/reference required case mismatch=0となる。
- [ ] x64/ARM64 canonical result SHA-256が一致する。
- [ ] qualified backend間bits mismatch=0となる。
- [ ] public API baselineのunapproved diff=0となる。

---

# Phase 4E

## P4E-000 Phase 4E implementation preflight

**設計参照:** §34～§43, §46, §52.2

### 受け入れ条件

- [ ] P4D-006が`完了`である。
- [ ] §34～§43 review reportにreviewed HEAD、verdict=pass、unresolved findings=0を記録する。
- [ ] parser syntaxをexact grammarとしてdecision recordへ固定し、hex literalはC99-style `0x<hex>[.<hex>]p[+-]<decimal exponent>`をaccepted syntaxに含める。
- [ ] parser resource limit `max input length / max significand digits / max exponent digits / max exception excerpt length`を正整数のnumeric literalとしてpreflight recordへ固定し`TBD`を0件にする。
- [ ] binary interchange v1を採用する場合`length=18`, byte0=version, byte1=state, byte2..9=Lower LE, byte10..17=Upper LEをpreflight recordへ固定する。
- [ ] Parse rejected inputで送出するexception typeまたはerror contractを1つに固定し、TryParseはfalse/out=Emptyとするdecisionを記録する。
- [ ] smoke fixture `DivideToUnion([1,2],Entire).Count==2`をexisting harnessへ登録する。
- [ ] P4E-001 Red commit前にP4E-000 status=`完了`とする。

## P4E-001 IntervalUnion2

**設計参照:** §34, §44.1 F-PR3-010/F-PR3-015

### 受け入れ条件

- [ ] `default(IntervalUnion2).Count==0`, `IsEmpty==true`, `First==Empty`, `Second==Empty`となる。
- [ ] Count=1はFirst nonempty/Second Empty、Count=2はFirst/Second nonemptyかつ`First.Upper<=Second.Lower`となる。
- [ ] `First.Upper==Second.Lower`のCount=2をmergeしない固定fixtureを持つ。
- [ ] `First.Upper>Second.Lower`のinternal Create2はdebug assertionまたはrepository-defined internal validation exceptionのいずれか1つへ固定し、そのnegative fixtureがpassする。
- [ ] Count0/1/2 equality/hash/operatorはcanonical component列だけに依存し、unused field/NaN payloadへ依存しない。
- [ ] indexerは`0<=index<Count`以外で`ArgumentOutOfRangeException`となる。
- [ ] public `Contains(double)`を初版baselineへ追加しない。

## P4E-002 DivideToUnion

**設計参照:** §35, §44.1 F-PR3-010/F-PR3-013

### 受け入れ条件

- [ ] numerator/denominator Emptyまたはdenominator ZeroはCount0となる。
- [ ] denominator excludes zeroはCount1で`First == ordinary division`となる。
- [ ] numerator Zeroかつdenominatorにnonzero memberありはCount1 Zeroとなる。
- [ ] `Y=[0,d], d>0`: X=Z -> Count1 Zero、X=P -> Count1 `[RD(a/d),+Infinity]`、X=N -> Count1 `[-Infinity,RU(b/d)]`、X=M -> Count1 Entireとなる。
- [ ] `Y=[c,0], c<0`: X=Z -> Count1 Zero、X=P -> Count1 `[-Infinity,RU(a/c)]`、X=N -> Count1 `[RD(b/c),+Infinity]`、X=M -> Count1 Entireとなる。
- [ ] strict zero-crossing denominatorでstrict positive XはFirst=`[-Infinity,RU(a/c)]`, Second=`[RD(a/d),+Infinity]`となる。
- [ ] strict zero-crossing denominatorでstrict negative XはFirst=`[-Infinity,RU(b/d)]`, Second=`[RD(b/c),+Infinity]`となる。
- [ ] `DivideToUnion([1,2],Entire)`はCount2でFirst=`[-Inf,-0.0]`, Second=`[+0.0,+Inf]`となる。
- [ ] numerator contains zeroかつnonzero intervalではstrict crossing denominator result=Count1 Entireとなる。
- [ ] 全fixtureで`ordinary division == DivideToUnion(...).ConvexHull`となる。

## P4E-003 ReciprocalToUnion

**設計参照:** §35

### 受け入れ条件

- [ ] Zero -> Count0、strict positive/negative -> Count1、strict zero crossing -> Count2となる。
- [ ] `ReciprocalToUnion(Entire)`はCount2 `[-Inf,-0.0]` / `[+0.0,+Inf]`となる。
- [ ] implementationはDivideToUnionの同一kernel semanticsをnumerator=Oneとして使用する。
- [ ] `Reciprocal(value)==ReciprocalToUnion(value).ConvexHull`がbare semantics適用caseで成立する。

## P4E-004 ReverseMultiply

**設計参照:** §36

### 受け入れ条件

- [ ] product/factor EmptyはCount0となる。
- [ ] `0 in product && 0 in factor`ならCount1 Entireとなる。
- [ ] `ReverseMultiply([1,2],Zero)=Count0`, `ReverseMultiply(Zero,Zero)=Count1 Entire`, `ReverseMultiply([0,2],Zero)=Count1 Entire`となる。
- [ ] `ReverseMultiply([1,2],Entire)`はCount2 `[-Inf,-0.0]` / `[+0.0,+Inf]`となる。
- [ ] 上記特殊case以外はDivideToUnion相当resultとなる。

## P4E-005 cancellative operations

**設計参照:** §37, §44.1 F-PR3-014

### 受け入れ条件

- [ ] `CancelSubtract(Empty,Empty)=Empty`となる。
- [ ] `CancelSubtract(Empty,bounded)=Empty`、`CancelSubtract(Empty,unbounded)=Entire`となる。
- [ ] `CancelSubtract(bounded,Empty)=Entire`、`CancelSubtract(bounded,unbounded)=Entire`となる。
- [ ] `CancelSubtract(unbounded,Empty)=Entire`、`CancelSubtract(unbounded,bounded)=Entire`、`CancelSubtract(unbounded,unbounded)=Entire`となる。
- [ ] bounded/common同士でexact width(total)>=exact width(term)なら`[RD(a-c),RU(b-d)]`となる。
- [ ] bounded/common同士でexact width(total)<exact width(term)ならEntireとなる。
- [ ] width比較にrounded public `Width`だけを使用せずexact expansion/rational relationで判定する。
- [ ] `CancelAdd(total,term)==CancelSubtract(total,-term)`が上記9class fixtureで成立する。

## P4E-006 Decoration / default NaI

**設計参照:** §38.1, §38.3～§38.6, §44.1 F-PR3-016

### 受け入れ条件

- [ ] `Decoration` underlying typeが`byte`である。
- [ ] `(byte)Ill=0`, `(byte)Trv=4`, `(byte)Def=8`, `(byte)Dac=12`, `(byte)Com=16`となる。
- [ ] `default(DecoratedInterval).IsNaI==true`となる。
- [ ] `FromInterval([1,2]).Decoration=Com`, `FromInterval(Entire).Decoration=Dac`, `FromInterval(Empty).Decoration=Trv`となる。
- [ ] NaI input operationはNaIを返し、Ill decorationでordinary interval stateを生成しない。
- [ ] result capはEmpty<=Trv、unbounded nonempty<=Dac、bounded nonempty<=Comとなる。
- [ ] `Com [MaxValue,MaxValue] + Com [MaxValue,MaxValue]`のunbounded result decorationがComにならない。

## P4E-007 DecoratedInterval equality/canonicalization

**設計参照:** §38.2, §38.7, §44.1 F-PR3-015

### 受け入れ条件

- [ ] `NaI == NaI`、`NaI.Equals(NaI)`がtrueとなる。
- [ ] NaI hashが固定でinternal NaN payload差に依存しない。
- [ ] non-NaI equalityはcanonical interval partとDecorationの両方一致時だけtrueとなる。
- [ ] `NaI.SemanticallyEquals(any)==false`となる。
- [ ] non-NaI `SemanticallyEquals`はdecoration差を無視しinterval part equalityだけで判定する。
- [ ] `NaI.TryGetInterval(out x)`はfalseかつx=Emptyとなる。

## P4E-008 decorated arithmetic/math

**設計参照:** §38.5, §38.6, §39

### 受け入れ条件

- [ ] decorated add/sub/mul/div/mathの全entryが共通`CreateCanonical(resultInterval, requestedDecoration)`相当を通る。
- [ ] NaI operandを含む全operationはNaIとなる。
- [ ] divisor zero-containing divisionのopDec<=Trv、`Sqrt([-1,4])`のopDec=Trv、Tan pole crossingのopDec=Trvとなる。
- [ ] bare result interval partとdecorated result interval partのcanonical endpoint bitsが一致する。
- [ ] Empty result decorationがTrvを超えず、unbounded resultがComにならない。

## P4E-009 exact/outward parser・formatter

**設計参照:** §40, §43, F-PR5-006

### 受け入れ条件

- [ ] accepted fixture `Empty`, `Entire`, `[1,2]`, `[1]`, `[-Infinity,1]`, `[1,+Infinity]`をparseできる。
- [ ] decimal `[0.1]`はnearest singletonではなくexact decimal 0.1の`[RoundDown,RoundUp]`となる。
- [ ] C99-style hex `[0x1p+0]`はexact `[1,1]`となる。
- [ ] `[0x0.0000000000001p-1022]`はmin positive subnormal exact singletonとなる。
- [ ] `[-0x0p+0]`はzero singletonとしてcanonical lower=-0.0/upper=+0.0となる。
- [ ] rejected fixture `[+Infinity,1]`, `[1,-Infinity]`, `[Infinity]`, `[NaN,1]`, exact lower>upperはP4E-000で固定したParse error contractになる。
- [ ] `TryParse`系は全rejected fixtureでfalse/out=Emptyとなる。
- [ ] parserはInvariantCulture固定とし、`[1,5]`をlocale decimal 1.5として解釈せずlower=1/upper=5として扱う。
- [ ] parser implementationにrecursive descent/self-recursive callを使用せず、nesting syntaxを受け入れない。
- [ ] P4E-000で固定したmax input length/significand digits/exponent digitsについて各limit値の入力はaccepted、limit+1はP4E-000で固定したerror contractとなる。
- [ ] exception messageに入力全文を無制限に含めず、excerpt lengthはP4E-000で固定したmax exception excerpt length以下となる。
- [ ] exact round-trip formatを採用した場合、format->parse後のcanonical endpoint bitsが元値と一致する。

## P4E-010 binary interchange

**設計参照:** §41, §43, F-PR5-007

### 受け入れ条件

- [ ] version 1 encoded lengthは常に18 byteである。
- [ ] byte0=version、byte1=state、byte2..9=external Lower bits little-endian、byte10..17=external Upper bits little-endianとなる。
- [ ] encoded dataにprivate `[-Lower,Upper]` raw memory layoutを直接copyしない。
- [ ] Zero round-tripはLower external bits=`0x8000000000000000`、Upper bits=`0x0000000000000000`となる。
- [ ] Entire round-tripはLower=-Infinity, Upper=+Infinityのexternal bitsとなる。
- [ ] Emptyはversion/stateでcanonical emptyを表し、internal NaN payloadをwireへ露出しない。
- [ ] 17 byteと19 byte inputはversion/state/endpoint decodeより前のlength checkでrejectする。
- [ ] unknown version、unknown state、NaN endpoint、lower>upper、lower=+Inf、upper=-Infをrejectする。
- [ ] reject pathで片側NaN等のinvalid internal Interval stateを生成しない。
- [ ] normal/Zero/Entire/Emptyのencode->decode round-tripでcanonical public stateが一致する。

## P4E-011 split / bisect

**設計参照:** §42

### 受け入れ条件

- [ ] `TrySplitAt`はEmpty、NaN/Infinity splitPoint、`splitPoint<=Lower`, `splitPoint>=Upper`でfalseとなる。
- [ ] `[0,2]`を1でsplitするとleft=`[0,1]`, right=`[1,2]`となる。
- [ ] split childrenは共有split pointを持ちgapを作らない。
- [ ] `TryBisect`はbounded non-singletonのみ成功対象とする。
- [ ] adjacent binary64 endpointsでstrict interior valueがないintervalはTryBisect=falseとなる。
- [ ] unbounded intervalのTryBisectはfalseとなり任意pivotを自動選択しない。
- [ ] fixed-seed propertyで`left/right subset original`かつ`ConvexHull(left,right)==original`となる。

## P4E-012 Phase 4E security/conformance/API gate

**設計参照:** §43～§46

### 受け入れ条件

- [ ] `F-PR3-010`, `F-PR3-013`～`F-PR3-017`のreview-regression fixtureが全件passする。
- [ ] union/decorated equality/default/round-trip、parser exact/outward、binary 18-byte layout、split cover propertyが全件passする。
- [ ] parser resource limitのlimit/limit+1 testとbinary 17/19 byte reject testがpassする。
- [ ] x64/ARM64 canonical result SHA-256が一致する。
- [ ] qualified backend間canonical bits mismatch=0となる。
- [ ] failure artifactからunion component、decoration、parser exact rational/resource-limit decision、binary reject reason、split stateをcaseId単位で追跡できる。
- [ ] public API baselineのunapproved diff=0となる。

---

## 4. PR #5 review finding対応表

| Finding | 修正先 | 具体化した証拠 |
|---|---|---|
| F-PR5-001 High | P4A-000/P4B-000/P4C-000/P4D-000/P4E-000 | reviewed HEAD/verdict/report path/diagnostic workflow/smoke fixture/Red開始禁止 |
| F-PR5-002 Medium | P1-001 | caseId/inputBits/branch/exact/Devo6/inari/kv/MPFR/expected-difference fields |
| F-PR5-003 Medium | P1-002 | `[-Lower,Upper]`, canonical qNaN 2 lane,片側NaN禁止, raw constructor, ToString |
| F-PR5-004 Medium | P1-006/P1-008 | `NextUp/NextDown/無補正`条件と5 branch witnessを明記 |
| F-PR5-005 Medium | P1-012/P2-001/P3-006 | API 3scenario、benchmark workload/metric/閾値を固定 |
| F-PR5-006 Medium | P4E-000/P4E-009 | accepted/rejected parser fixture、hex syntax、InvariantCulture、recursion/resource limit |
| F-PR5-007 Medium | P4E-010 | 18 byte、byte offset、little-endian、length-first reject |
| F-PR5-008 Medium | P1-013 | allocation/disassembly/NativeAOT/trimming/raw-constructor gate |
| F-PR5-009 Low | P4E-006 | `Decoration : byte`, 0/4/8/12/16を固定 |
| F-PR5-010 Medium | P4B-007 | managed N/Aまたはnative interop性能/ABI/thread/AOT/trimming/distribution/license gate |

## 5. Phase完了時の更新規則

- Phase内の必須taskが全て`完了`になり、`tasks/phases-status.md`の完了条件を全件満たした時だけPhaseを`完了`へ変更する。
- Phase 4のpreflight reviewed design HEADが更新された場合、対応`P4?-000`を再openし、新HEADのreview passまでsource実装を停止する。
- API freeze後にbasic `Interval` APIを変更する場合、P2-003 gateをfailさせ、baseline更新とbreaking-change判定を先に行う。
- CIによる完了判定は必ず対象PR current HEAD SHAと一致するworkflow runのみ使用する。
- workerはmergeしない。
