# 区間の集合演算・関係・数値的属性 詳細設計

## 1. 文書情報

- 対象: Phase 4A
- 前提:
  - `IntervalArithmetic.md`
  - `IntervalArithmetic.Revision3.md`
  - `IntervalNonArithmetic.Roadmap.md`
- 主要参照: `unageek/inari` commit `18b83a571d7681c76067bc38d90a74e8be29f545`
- 作成日: 2026-08-30
- 設計状態: review required

本書では、浮動小数点初等関数backendを必要とせず実装できる集合演算、関係判定、区間の数値的属性、absolute/sign、pointwise min/maxおよび整数値関数を設計する。

## 2. 共通規則

### 2.1 bare interval

全機能はbare `Interval`を対象とする。NaIやdecorationは扱わない。

### 2.2 Empty

Revision 3の規定を使用する。

```text
Interval.Empty.Lower = +Infinity
Interval.Empty.Upper = -Infinity
```

内部ではcanonical NaN 2 laneを維持する。公開端点だけを利用してEmpty判定してはならず、内部の`IsEmpty`判定を先に行う。

### 2.3 infinity

`±Infinity`は実数の区間要素ではなく、下方または上方に非有界であることを表す端点である。

したがって、`Contains(double.PositiveInfinity)`と`Contains(double.NegativeInfinity)`は常に`false`とする。

### 2.4 signed zero

非空結果の下限0は`-0.0`、上限0は`+0.0`へ正規化する。bool関係はzeroの符号に依存しない。

### 2.5 例外

数学的にEmptyとなる演算は例外にせず`Interval.Empty`を返す。

`MidpointRounding`に未定義のenum値を渡す等、API parameter自体が不正な場合だけ`ArgumentOutOfRangeException`等を送出する。

## 3. 公開API候補

```csharp
namespace Devo6.Numerics;

public readonly partial struct Interval
{
    public bool IsBounded { get; }

    public double Width { get; }
    public double Midpoint { get; }
    public double Radius { get; }
    public double Magnitude { get; }
    public double Mignitude { get; }

    public bool Contains(double value);

    public bool IsSubsetOf(Interval other);
    public bool IsInteriorOf(Interval other);
    public bool IsDisjointFrom(Interval other);
    public bool Precedes(Interval other);
    public bool StrictlyPrecedes(Interval other);
    public bool IsWeaklyLessThan(Interval other);
    public bool IsStrictlyLessThan(Interval other);

    public Interval Intersect(Interval other);
    public Interval ConvexHull(Interval other);
    public IntervalOverlap GetOverlap(Interval other);
}

public enum IntervalOverlap
{
    BothEmpty,
    FirstEmpty,
    SecondEmpty,
    Before,
    Meets,
    Overlaps,
    Starts,
    ContainedBy,
    Finishes,
    Equals,
    FinishedBy,
    Contains,
    StartedBy,
    OverlappedBy,
    MetBy,
    After,
}

public static partial class IntervalMath
{
    public static Interval Abs(Interval value);
    public static Interval Sign(Interval value);

    public static Interval PointwiseMin(Interval left, Interval right);
    public static Interval PointwiseMax(Interval left, Interval right);

    public static Interval Floor(Interval value);
    public static Interval Ceiling(Interval value);
    public static Interval Truncate(Interval value);
    public static Interval Round(
        Interval value,
        MidpointRounding mode = MidpointRounding.ToEven);
}
```

`partial`は文書上の機能分割を示す。実装でsource fileをpartial型へ分けるかはコード構成上の判断とする。

## 4. API命名方針

### 4.1 比較演算子を使用しない

区間には次の異なる関係が存在する。

- endpoint-wise less
- set inclusion
- strictly-before
- interior
- overlap relation

そのため、`<`, `<=`, `>`, `>=`をいずれか1つへ割り当てない。すべてnamed methodで提供する。

### 4.2 set min/maxとpointwise min/maxを区別する

- `ConvexHull`: 2区間を包含する最小の区間
- `Intersect`: 集合の共通部分
- `PointwiseMin`: `{ min(x,y) }`の区間像
- `PointwiseMax`: `{ max(x,y) }`の区間像

`Min`や`Max`だけの名称はhullと誤認しやすいため、初版では`Pointwise`を含める。

## 5. 集合演算

`X=[a,b]`、`Y=[c,d]`とする。

### 5.1 `Intersect`

```text
Empty ∩ Y = Empty
X ∩ Empty = Empty

X ∩ Y = [max(a,c), min(b,d)]
```

候補下限が候補上限より大きい場合は`Empty`を返す。

内部表現では、非空operandについてlane-wise minimumとなる。

```text
X.rep = [-a,b]
Y.rep = [-c,d]
min(X.rep,Y.rep)
  = [min(-a,-c), min(b,d)]
  = [-max(a,c), min(b,d)]
```

候補を作成後、外部下限と上限の順序を検証する。

### 5.2 `ConvexHull`

```text
hull(Empty,Y) = Y
hull(X,Empty) = X
hull(X,Y) = [min(a,c), max(b,d)]
```

内部表現ではlane-wise maximumとなる。

```text
max([-a,b],[-c,d])
  = [max(-a,-c), max(b,d)]
  = [-min(a,c), max(b,d)]
```

### 5.3 性質

次をtestで固定する。

```text
X.Intersect(Y) == Y.Intersect(X)
X.ConvexHull(Y) == Y.ConvexHull(X)
X.Intersect(X) == X
X.ConvexHull(X) == X
X.Intersect(Entire) == X
X.ConvexHull(Empty) == X
X.Intersect(Empty) == Empty
```

## 6. 要素包含

### 6.1 `Contains(double)`

```text
Contains(X,x) = !X.IsEmpty
             && IsFinite(x)
             && X.Lower <= x
             && x <= X.Upper
```

結果:

| Interval | value | Result |
|---|---:|---:|
| `Empty` | any | false |
| `Entire` | finite | true |
| `Entire` | `±Infinity` | false |
| any | `NaN` | false |
| `[-0,+0]` | either zero | true |

infinityを非有界端点と実数要素で混同しない。

## 7. 集合関係

### 7.1 補助関係`<′`

無限端点を含むinterior/strict-lessの定義に、次を使用する。

```text
x <′ y  iff
    x < y
    or x == y == -Infinity
    or x == y == +Infinity
```

通常の有限値では`<′`は`<`と同じである。

### 7.2 `IsSubsetOf`

```text
Empty.IsSubsetOf(Y) = true
nonempty.IsSubsetOf(Empty) = false
[a,b].IsSubsetOf([c,d]) = c <= a && b <= d
```

### 7.3 `IsInteriorOf`

IEEE-orientedなinterval interior関係として次を採る。

```text
Empty.IsInteriorOf(Y) = true
nonempty.IsInteriorOf(Empty) = false
[a,b].IsInteriorOf([c,d]) = c <′ a && b <′ d
```

この定義では`Entire.IsInteriorOf(Entire)`は`true`となる。一般集合のtopological interiorと同名だが、interval standard上の関係として使用するため、XML documentationに定義式を明記する。

### 7.4 `IsDisjointFrom`

```text
Empty.IsDisjointFrom(Y) = true
X.IsDisjointFrom(Empty) = true
[a,b].IsDisjointFrom([c,d]) = b < c || d < a
```

端点で接する区間はdisjointではない。

```text
[1,2] and [2,3] -> false
```

### 7.5 `Precedes`

```text
Empty.Precedes(Y) = true
X.Precedes(Empty) = true
[a,b].Precedes([c,d]) = b <= c
```

端点接触を許す。

### 7.6 `StrictlyPrecedes`

```text
Empty.StrictlyPrecedes(Y) = true
X.StrictlyPrecedes(Empty) = true
[a,b].StrictlyPrecedes([c,d]) = b < c
```

### 7.7 `IsWeaklyLessThan`

endpoint-wiseの弱い順序を表す。

```text
Empty vs Empty     -> true
Empty vs nonempty  -> false
nonempty vs Empty  -> false
[a,b] vs [c,d]     -> a <= c && b <= d
```

subsetとは異なる。例えば`[1,2]`は`[2,3]`にsubsetではないが、weakly lessである。

### 7.8 `IsStrictlyLessThan`

```text
Empty vs Empty     -> true
Empty vs nonempty  -> false
nonempty vs Empty  -> false
[a,b] vs [c,d]     -> a <′ c && b <′ d
```

`<′`を使用するため、同じ非有界端点を共有する場合の結果を通常の`<`だけから推測してはならない。

### 7.9 proper subset

`IsProperSubsetOf`は初版public APIへ追加しない。必要な場合は次で表現できる。

```csharp
x.IsSubsetOf(y) && x != y
```

利用頻度が確認された場合にadditive APIとして追加する。

## 8. `IntervalOverlap`

### 8.1 目的

`GetOverlap`はAllen interval relation相当の位置関係に、Empty 3状態を加えた16状態を返す。

`self=[a,b]`、`other=[c,d]`とする。

| State | Condition |
|---|---|
| `BothEmpty` | both Empty |
| `FirstEmpty` | self Empty only |
| `SecondEmpty` | other Empty only |
| `Before` | `b < c` |
| `Meets` | `a < b && b == c && c < d` |
| `Overlaps` | `a < c && c < b && b < d` |
| `Starts` | `a == c && b < d` |
| `ContainedBy` | `c < a && b < d` |
| `Finishes` | `c < a && b == d` |
| `Equals` | `a == c && b == d` |
| `FinishedBy` | `a < c && b == d` |
| `Contains` | `a < c && d < b` |
| `StartedBy` | `a == c && d < b` |
| `OverlappedBy` | `c < a && a < d && d < b` |
| `MetBy` | `c < d && d == a && a < b` |
| `After` | `d < a` |

singletonが関与する場合、`Starts`、`Finishes`等の等端点関係を`Meets`より先に判定する。判定順で結果が変わらないよう、実装は上表の条件を相互排他的なdecision treeへ変換し、全状態fixtureを持つ。

### 8.2 inverse

```text
BothEmpty      <-> BothEmpty
FirstEmpty     <-> SecondEmpty
Before         <-> After
Meets          <-> MetBy
Overlaps       <-> OverlappedBy
Starts         <-> StartedBy
ContainedBy    <-> Contains
Finishes       <-> FinishedBy
Equals         <-> Equals
```

次をproperty testで検証する。

```text
x.GetOverlap(y).Inverse() == y.GetOverlap(x)
```

`Inverse()`をenum extensionとして公開するかはPhase 4AのAPI reviewで決定する。最低限、internal test helperは持つ。

## 9. 区間の数値的属性

### 9.1 `IsBounded`

```text
IsBounded = !IsEmpty
         && IsFinite(Lower)
         && IsFinite(Upper)
```

`Empty`、片側非有界、`Entire`は`false`。

### 9.2 `Width`

```text
Empty -> NaN
[a,b] -> RU(b-a)
unbounded nonempty -> +Infinity
singleton -> +0.0
```

内部表現では次で求められる。

```text
Width = AddUp(_upper, _negatedLower)
```

結果0は`+0.0`へ正規化する。

### 9.3 `Midpoint`

戻り値は区間ではなく、代表点となる1個の`double`である。

```text
Empty                 -> NaN
Entire                -> +0.0
[-Infinity,b]         -> double.MinValue
[a,+Infinity]         -> double.MaxValue
finite [a,b]          -> exact (a+b)/2 に最も近いbinary64
```

有限区間のtie ruleは.NETの通常数値規約に合わせ`ToEven`とする。midpointは包含端点ではないため、上向きまたは下向きへ外向き丸めしない。

有限区間では単純な`(a+b)/2`のoverflowを避ける。

```text
m = (a+b)/2
if a+b overflow:
    m = a/2 + b/2
```

実装は、極端に符号が異なる値、隣接値、subnormalおよびtieを含むfixtureにより、選択した`ToEven`契約を固定する。参考実装との差異がある場合はreference manifestへ記録する。

### 9.4 `Radius`

`m=Midpoint`として、次を満たす最小のbinary64 `r`を返す。

```text
X ⊆ [m-r, m+r]
```

```text
Empty -> NaN
nonempty -> max(SubtractUp(m,Lower), SubtractUp(Upper,m))
```

非有界区間は`+Infinity`となる。結果0は`+0.0`。

### 9.5 `Magnitude`

```text
Empty -> NaN
[a,b] -> max(abs(a),abs(b))
Entire -> +Infinity
```

結果0は`+0.0`。

### 9.6 `Mignitude`

区間内の絶対値の下限を返す。

```text
Empty -> NaN
0 ∈ [a,b] -> +0.0
b < 0 -> abs(b)
0 < a -> abs(a)
```

`Entire`は`+0.0`。

## 10. `Abs`

```text
Abs(Empty) = Empty

0 <= a:
    Abs([a,b]) = [a,b]

b <= 0:
    Abs([a,b]) = [-b,-a]

a < 0 < b:
    Abs([a,b]) = [-0.0, max(-a,b)]
```

外部lowerはcanonical `-0.0`、内部lower laneは`+0.0`。

内部実装:

- nonnegative/Zeroはそのまま返す。
- nonpositiveはlane swapで符号反転像を得る。
- mixedはlane-wise maxで上限を選び、下限を0にする。

## 11. `Sign`

点関数を次とする。

```text
sign(x) = -1 if x < 0
           0 if x = 0
           1 if x > 0
```

signed zeroを`±1`へ写さない。

| Input | Result |
|---|---|
| Empty | Empty |
| `b < 0` | `[-1,-1]` |
| `a < 0 && b == 0` | `[-1,0]` |
| `[0,0]` | `[0,0]` |
| `a == 0 && b > 0` | `[0,1]` |
| `a < 0 < b` | `[-1,1]` |
| `a > 0` | `[1,1]` |
| Entire | `[-1,1]` |

結果端点は整数値のため丸めを必要としない。

## 12. pointwise min/max

`X=[a,b]`、`Y=[c,d]`とする。

### 12.1 `PointwiseMin`

```text
{ min(x,y) | x∈X, y∈Y }
= [min(a,c), min(b,d)]
```

いずれかがEmptyならEmpty。

### 12.2 `PointwiseMax`

```text
{ max(x,y) | x∈X, y∈Y }
= [max(a,c), max(b,d)]
```

いずれかがEmptyならEmpty。

比較とendpoint選択だけであり、算術丸めを行わない。

## 13. 整数値関数

### 13.1 共通

次のpoint functionはすべて単調非減少であるため、非空`[a,b]`に対しendpointへ同じ関数を適用する。

```text
F([a,b]) = [F(a),F(b)]
```

- EmptyはEmpty。
- 非有界端点は同じInfinity端点を維持する。
- finite binary64に対する結果は整数値binary64であり、追加の外向き丸めは不要。
- 結果zeroは区間端点規則へ正規化する。

### 13.2 `Floor`

```text
Floor([a,b]) = [floor(a), floor(b)]
```

### 13.3 `Ceiling`

```text
Ceiling([a,b]) = [ceil(a), ceil(b)]
```

### 13.4 `Truncate`

```text
Truncate([a,b]) = [trunc(a), trunc(b)]
```

### 13.5 `Round`

```csharp
IntervalMath.Round(value, MidpointRounding.ToEven)
IntervalMath.Round(value, MidpointRounding.AwayFromZero)
IntervalMath.Round(value, MidpointRounding.ToZero)
IntervalMath.Round(value, MidpointRounding.ToNegativeInfinity)
IntervalMath.Round(value, MidpointRounding.ToPositiveInfinity)
```

全modeをendpointへ適用する。未知のenum値は`ArgumentOutOfRangeException`。

`ToNegativeInfinity`と`ToPositiveInfinity`はそれぞれ`Floor`と`Ceiling`と同じ結果になるが、.NET利用者が`MidpointRounding`で指定する経路を許可する。

## 14. 内部ファイル構成

```text
src/Devo6.Interval/
  Interval.Relations.cs
  Interval.SetOperations.cs
  Interval.Numeric.cs
  IntervalMath.Basic.cs
  IntervalOverlap.cs
  Internal/
    IntervalRelationKernel.cs
    IntervalSetKernel.cs
    IntervalNumericKernel.cs
```

hot pathへinterface/virtual dispatchを追加しない。

## 15. SIMD方針

Phase 4Aは`[-Lower,Upper]`とlane-wise比較に適する。

候補:

- hull: packed maximum
- intersection: packed minimum + validity mask
- subset: packed less-than-or-equal + all mask
- pointwise min/max: laneごとに異なるmin/maxをshuffleで構成
- Abs: sign class mask + swap/max
- Width:上向きpacked add

scalar referenceとcanonical bitwise equivalentで、benchmark上の改善がある経路だけをproduction dispatchへ含める。

単一bool relationはbranch予測の良いscalarが速い可能性があるため、SIMD化を前提にしない。batch relation APIが必要になった場合はPhase 4A後のadditive API reviewとする。

## 16. TDD順序

1. `Contains`、`IsBounded`
2. `Intersect`、`ConvexHull`
3. subset/disjoint/precedes
4. interior/weak-less/strict-less
5. `IntervalOverlap`全状態
6. Width/Magnitude/Mignitude
7. Midpoint/Radius
8. Abs/Sign
9. PointwiseMin/Max
10. integer functions
11. SIMD differential benchmark

各機能は先に失敗testを追加し、期待するEmpty/Infinity/signed-zeroをbit patternまで確認してから実装する。

## 17. 決定的test matrix

### 17.1 set/relation

- Empty/Empty、Empty/nonempty、nonempty/Empty
- Entire/Entire、bounded/Entire
- disjoint、touching、overlapping、contained、equal
- singleton/singleton
- singletonが他区間のlower/upperと一致
- lower/upper unbounded
- `±0.0`の全入力組合せ

### 17.2 overlap

16 enum状態を最低1caseずつ持つ。

追加で次を固定する。

- 各非対称状態とinverse
- singletonにより`Meets`ではなく`Starts`/`Finishes`等になる境界
- signed zero端点
- Infinity共有端点

### 17.3 numeric

- finite singleton
- 隣接binary64端点
- same-sign巨大値によるmidpoint overflow回避
- opposite-sign cancellation
- minimum subnormal
- one-sided unbounded
- Entire
- Empty
- midpoint tie-to-even
- Radiusが両側距離の大きい方を選ぶcase
- Widthがfinite overflowしてInfinityになるcase

### 17.4 integer mapping

- 正負の非整数
- `±0.5`、`±1.5`の各rounding mode
- `2^52`前後
- signed zero
- Empty/Entire/one-sided unbounded

## 18. Property test

次を検証する。

```text
Intersection commutative/idempotent
Hull commutative/idempotent
Intersection subset of each operand
Each operand subset of Hull
Abs result subset of [0,+Infinity]
Width >= 0 for nonempty
Radius >= 0 for nonempty
Mignitude <= Magnitude for nonempty
GetOverlap inverse consistency
Floor(x) <= Ceiling(x) endpoint-wise
```

bool relationの定義を別関係から循環的に導出せず、直接式と独立matrixの両方で検証する。

## 19. Conformanceとreference

- set operation、boolean relation、numeric function、integer functionの該当ITF1788 caseを固定SHAから抽出する。
- ITF1788にない`IsBounded`やC#固有命名はrepository-defined equivalent matrixを持つ。
- inariとの差分は、semantic差、midpoint tie policy差、C# API adaptationへ分類する。
- midpoint等、標準が複数結果を許す箇所は「同一bit pattern」を無条件の合格条件にせず、採用した本設計契約をprimary oracleとする。

## 20. 完了条件

- 全public候補のXML documentationに数式、Empty、Infinity、signed-zero semanticsが記載されている。
- deterministic/property/conformance testがx64とARM64で成功する。
- canonical result corpusが両architectureで一致する。ただし明示的にplatform-independentではないと承認された項目を作らない。
- scalar referenceとの差分なしに採用済みSIMD backendが動作する。
- API baselineへ追加surfaceが記録される。
- current PR HEADと一致するCI runの証跡がある。
