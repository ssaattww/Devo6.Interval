# 区間演算 詳細設計

## 1. 文書情報

- 対象リポジトリ: `ssaattww/Devo6.Interval`
- 対象ライブラリ: `Devo6.Interval`
- 設計対象: IEEE 754 binary64 (`double`) を端点とする区間演算ライブラリ
- 基本設計: `doc/Design/basic/IntervalArithmetic.md`
- 初版: 2026-08-29
- 統合版: 2026-08-30
- 設計版: 4
- 状態: review required

本書を `doc/Design/detail/` 配下の**唯一の詳細設計書**とする。

旧版で分割されていた次の文書の内容は本書へ統合した。

- `IntervalArithmetic.Revision3.md`
- `IntervalNonArithmetic.Roadmap.md`
- `IntervalSetAndNumeric.md`
- `IntervalMathFunctions.md`
- `IntervalAdvancedFeatures.md`

Revision 3で確定した修正事項も本文へ直接反映しており、旧版のprecedence規則は不要とする。

特に次を統合後の唯一の規範とする。

- `Interval.Empty.Lower = +Infinity`
- `Interval.Empty.Upper = -Infinity`
- Empty内部表現はcanonical NaN 2 lane
- exact-rational oracleのfinite overflow処理
- IEEE 1788.1 conformance source mapping
- repository-defined `IsSingleton` matrix
- 四則演算後のPhase 4A～4E設計

既存の四則演算設計に対する過去のreview PASSは、そのreviewed HEADの内容に対する履歴情報であり、統合後の本書へ自動的に引き継がない。本書の実装開始前に、統合後のimmutable HEADを対象として再reviewする。

---

## 2. 設計原則

### 2.1 仕様基準

意味論の基準はIEEE 1788.1-2017を第一候補とし、既存ライブラリとの互換性を重視する。

主要参照実装:

- `unageek/inari`
  - commit: `18b83a571d7681c76067bc38d90a74e8be29f545`
- `mskashi/kv`
  - commit: `c7f8f2324a0e403cca6b39f46088a22843d440db`
- `unageek/ITF1788`
  - commit: `d8c2a64478ebdc9cbde6ccef33eaad3bed60ed81`

優先順位:

1. 数学的なexact oracle
2. 採用したIEEE 1788.1-oriented semantics
3. `inari`の区間意味論・endpoint結果
4. `kv`のcompatibleなdirected-rounding primitive

既存ライブラリ同士で差異がある場合、単純にどちらかへ合わせず、意味論とexact resultを確認して決定する。

### 2.2 真値包含

すべての区間演算は真値を包含する。

有限endpointは可能な限りtightにし、四則演算および正式公開する数学関数では、指定方向へ正しく丸められた最も内側のbinary64を返す。

```text
Lower: round toward -Infinity
Upper: round toward +Infinity
```

### 2.3 公開APIとbackendを分離する

公開APIから次へ依存できないようにする。

- private field layout
- `Vector128<double>`等の物理格納型
- scalar / SIMD backend
- managed / native backend
- CPU feature
- MPFR等のreference implementation

これにより、Phase 2でAPIを固定した後も内部実装を差し替えられる。

### 2.4 暗黙の精度低下を禁止する

同一の公開methodが、実行環境によって次のように意味論を変えない。

```text
環境A: tight endpoint
環境B: 真値は含むが大幅に広いendpoint
```

正式なtight kernelを用意できない関数は、公開を延期する。

### 2.5 bare intervalと拡張情報を分離する

```text
Interval           連結なbare interval
IntervalUnion2     0～2個の連結成分
DecoratedInterval  Interval + Decoration、またはNaI
```

非連結結果、decoration、NaI、parser error、split metadataをbare `Interval`のflagとして混在させない。

---

## 3. 開発フェーズ

### 3.1 全体順序

次の順序で開発する。

1. **Phase 0**: 詳細設計・検証基盤
2. **Phase 1**: SIMDなしmanaged scalar四則演算パイロット
3. **Phase 2**: 基本`Interval` API確定
4. **Phase 3**: 同一意味論のSIMD backend
5. **Phase 4A**: 集合・関係・数値的属性・整数値関数
6. **Phase 4B**: 代数関数・区間定数
7. **Phase 4C**: 単調な初等関数
8. **Phase 4D**: 周期・特異点・多変数関数
9. **Phase 4E**: 非連結結果・decorated interval・I/O・分割

この順序により、API・数値意味論・SIMD最適化・高度関数を分離して検証する。

### 3.2 Phase 0: 設計・検証基盤

完了条件:

- Phase 1が追加の数値アルゴリズム判断なしで開始できる。
- 方向付き四則演算primitiveの全許可入力について返却binary64が一意に決まる。
- conformance、oracle、CI、artifact方針が決まっている。
- 未確定事項がAPI評価または後続backend選択に限定されている。

### 3.3 Phase 1: managed scalarパイロット

対象:

- `Interval`
- constructor / `TryCreate` / `Point`
- `Empty`, `Entire`, `Zero`
- `Lower`, `Upper`
- `IsEmpty`, `IsEntire`, `IsSingleton`
- equality / Hash
- unary `-`
- `+`, `-`, `*`, `/`
- pure-managed directed rounding
- signed zero / Infinity / Emptyの正規化
- exact-rational oracle
- IEEE 1788.1-oriented conformance
- pinned reference corpus
- Linux x64 / ARM64 CI
- failure diagnostic artifact

対象外:

- SIMD
- CPU global rounding-mode変更
- native production dependency
- `sqrt`, `exp`, `log`, `sin`等
- decorated interval / NaI
- exact text parsing
- interval splitting

### 3.4 Phase 2: 基本API確定

確定対象:

- package / assembly / namespace
- constructor/factory
-基本property
- Empty / Entire
-四則演算operator
- equality / Hash
-例外
- signed zero
- diagnostic `ToString`
- scalar conversion/overloadの採否
- generic math interfaceの採否

完了後、基本`Interval` APIへの破壊的変更は原則禁止する。

### 3.5 Phase 3: SIMD backend

実装順:

1. scalar referenceとSIMD differential test基盤
2. SIMD load/store
3. batch add/sub
4. AVX-512 directed mul/div
5. AVX2+FMA、AVX2 without FMA、SSE2、ARM64を個別評価
6. correctnessとbenchmarkの両方を通過した経路のみproduction dispatchへ採用

単一区間operatorを無条件に最大幅vectorへ載せない。

### 3.6 Phase 4A

対象:

- `Contains`
- `Intersect`, `ConvexHull`
- subset/interior/disjoint/precedes等のnamed relation
- `IntervalOverlap`
- `IsBounded`
- `Width`, `Midpoint`, `Radius`, `Magnitude`, `Mignitude`
- `Abs`, `Sign`
- pointwise min/max
- floor/ceiling/truncate/round

### 3.7 Phase 4B

対象:

- reciprocal
- square
- square root
- integer power
- integer root
- fused multiply-add
- tight interval constants

### 3.8 Phase 4C

対象:

- exp/exp2/exp10
- log/log2/log10
- sinh/cosh/tanh
- asinh/acosh/atanh
- asin/acos/atan

### 3.9 Phase 4D

対象:

- sin/cos/tan
- atan2
- positive-base general interval power
- high-precision periodic range reduction

### 3.10 Phase 4E

対象:

- `IntervalUnion2`
- extended division / reciprocal
- reverse multiplication / two-output division
- cancellative addition/subtraction
- `DecoratedInterval` / NaI
- exact/outward parsing
- exact text/binary interchange
- interval splitting

Affine Arithmetic、Taylor Model、root finding、global optimization、constraint solver、automatic differentiationは別の上位package/設計とする。

---

## 4. 対象環境

### 4.1 Target Framework

Phase 1は`net10.0`を対象とする。

古いtarget frameworkはPhase 2以降で利用要件を確認して追加する。

### 4.2 Architecture

Phase 1 correctness target:

- Linux x64
- Linux ARM64

Phase 3候補:

- x64 AVX-512F
- x64 AVX2 + FMA
- x64 AVX2 without FMA
- x64 AVX + FMA without AVX2
- x64 SSE2 without FMA
- ARM64 AdvSimd
- scalar fallback

### 4.3 Runtime dependency

Phase 1 production packageはBCL以外のruntime dependencyを持たない。

`inari`, `kv`, ITF1788, MPFRは、参照・test corpus・将来backend候補であり、Phase 1 production runtimeから呼び出さない。

---

## 5. 基本公開API

### 5.1 namespace

```text
Assembly / package: Devo6.Interval
Namespace:          Devo6.Numerics
Type:               Interval
```

namespaceはPhase 2確定対象。

### 5.2 `Interval`

```csharp
namespace Devo6.Numerics;

public readonly struct Interval : IEquatable<Interval>
{
    public Interval(double lower, double upper);

    public double Lower { get; }
    public double Upper { get; }

    public bool IsEmpty { get; }
    public bool IsEntire { get; }
    public bool IsSingleton { get; }

    public static Interval Empty { get; }
    public static Interval Entire { get; }
    public static Interval Zero { get; }

    public static Interval Point(double value);
    public static bool TryCreate(
        double lower,
        double upper,
        out Interval interval);

    public static Interval operator +(Interval left, Interval right);
    public static Interval operator -(Interval left, Interval right);
    public static Interval operator -(Interval value);
    public static Interval operator *(Interval left, Interval right);
    public static Interval operator /(Interval left, Interval right);

    public bool Equals(Interval other);
    public override bool Equals(object? obj);
    public override int GetHashCode();

    public static bool operator ==(Interval left, Interval right);
    public static bool operator !=(Interval left, Interval right);

    public override string ToString();
}
```

### 5.3 constructor

成功条件:

```text
lower <= upper
lower != +Infinity
upper != -Infinity
lower is not NaN
upper is not NaN
```

不正時:

- constructor: `ArgumentException`
- `TryCreate`: `false`, out=`Interval.Empty`

`lower > upper`を自動Emptyへしない。入力ミスと空集合を区別する。

### 5.4 `Point`

有限`double`のみ受け入れる。

`NaN`および`±Infinity`は実数の点ではないため拒否する。

### 5.5 定数/default

```text
Empty  = empty set
Entire = [-Infinity,+Infinity]
Zero   = [-0.0,+0.0]
```

`default(Interval) == Interval.Zero`を契約とする。

### 5.6 Empty endpoint

内部Emptyはcanonical NaNで識別するが、公開endpointはIEEE-oriented semanticsへ合わせる。

```text
Interval.Empty.Lower -> +Infinity
Interval.Empty.Upper -> -Infinity
```

概念実装:

```csharp
public double Lower
    => IsEmpty ? double.PositiveInfinity : -_negatedLower;

public double Upper
    => IsEmpty ? double.NegativeInfinity : _upper;
```

### 5.7 signed zero

```text
nonempty lower zero -> -0.0
nonempty upper zero -> +0.0
```

### 5.8 四則演算時の例外

数学的な定義域問題を例外で表現しない。

```text
[1,2] / [0,0]  -> Empty
[1,2] / [-1,1] -> Entire
```

`DivideByZeroException`は送出しない。

### 5.9 Phase 1で提供しないもの

```csharp
interval + 1.0
1.0 + interval
(Interval)1.0
```

`INumber<TSelf>`もPhase 1では採用しない。

### 5.10 `ToString`

Phase 1ではdiagnostic用途。

```text
Empty  -> "Empty"
[1,2]  -> "[1, 2]"
Entire -> "[-Infinity, Infinity]"
```

永続化・wire契約にはしない。

---

## 6. 内部表現

### 6.1 negated-lower representation

外部`[lower,upper]`を内部で次として保持する。

```text
[-lower, upper]
```

Phase 1候補:

```csharp
private readonly double _negatedLower;
private readonly double _upper;
```

### 6.2 canonical state

```text
Zero   = [+0.0,+0.0]
Entire = [+Infinity,+Infinity]
Empty  = [canonical-qNaN,canonical-qNaN]
```

EmptyのNaN payloadはpublic contractにしない。

### 6.3 invariant

非空:

```text
!IsNaN(_negatedLower)
!IsNaN(_upper)
Lower <= Upper
Lower != +Infinity
Upper != -Infinity
```

Empty:

```text
IsNaN(_negatedLower)
IsNaN(_upper)
```

片側NaNは禁止。

### 6.4 raw constructor

演算結果用にvalidationを省略するinternal/private constructorを持つ。

呼出側がcanonical stateを保証する。

### 6.5 layout非公開

次をpublic contractにしない。

- size 16 byte
- field順序
- blittable ABI
- public field
- raw byte serializer
- `Vector128<double>` cast

---

## 7. 内部コンポーネント

Phase 1候補:

```text
src/Devo6.Interval/
  Interval.cs
  Interval.Operators.cs
  Interval.Formatting.cs
  Internal/
    DirectedRounding.cs
    ScalarIntervalKernel.cs
    IntervalSignClass.cs
    IntervalCanonicalizer.cs
```

Phase 4追加候補:

```text
  Interval.Relations.cs
  Interval.SetOperations.cs
  Interval.Numeric.cs
  IntervalConstants.cs
  IntervalMath.Basic.cs
  IntervalMath.Algebraic.cs
  IntervalMath.Exponential.cs
  IntervalMath.Hyperbolic.cs
  IntervalMath.Trigonometric.cs
  IntervalMath.Power.cs
  IntervalUnion2.cs
  Decoration.cs
  DecoratedInterval.cs
  DecoratedIntervalMath.cs
  IntervalContractor.cs
  Interval.Parsing.cs
  Interval.Interchange.cs
  Interval.Splitting.cs

  Internal/
    IntervalRelationKernel.cs
    IntervalSetKernel.cs
    IntervalNumericKernel.cs
    DirectedSqrt.cs
    DirectedIntegerPower.cs
    DirectedFma.cs
    DirectedElementary.cs
    PeriodicCriticalPointDetector.cs
    PayneHanekReducer.cs
    AngleArcAccumulator.cs
    ElementaryBackendDispatcher.cs
    IntervalUnion2Canonicalizer.cs
    ExtendedDivisionKernel.cs
    DecorationPolicy.cs
    ExactDecimalParser.cs
    Binary64TextConverter.cs
    IntervalBinaryCodec.cs
```

hot pathにinterface/virtual/delegate dispatchを導入しない。

---

## 8. 方向付き丸め共通契約

### 8.1 primitive

```text
AddUp(x,y)        = min binary64 z such that exact(x+y) <= z
AddDown(x,y)      = max binary64 z such that z <= exact(x+y)
MultiplyUp(x,y)   = min binary64 z such that exact(x*y) <= z
MultiplyDown(x,y) = max binary64 z such that z <= exact(x*y)
DivideUp(x,y)     = min binary64 z such that exact(x/y) <= z
DivideDown(x,y)   = max binary64 z such that z <= exact(x/y)
```

### 8.2 NextUp / NextDown

`Math.BitIncrement` / `Math.BitDecrement`を使用する。

通常演算後に無条件で1 ULP広げない。

### 8.3 symmetry

利用可能な関係:

```text
AddDown(x,y)      = -AddUp(-x,-y)
SubtractDown(x,y) = -SubtractUp(y,x)
MultiplyDown(x,y) = -MultiplyUp(-x,y)
DivideDown(x,y)   = -DivideUp(-x,y)
```

### 8.4 primitiveへ渡さないundefined pair

```text
+Infinity + -Infinity
0 * Infinity
0 / 0
Infinity / Infinity
denominator == 0
NaN operand
```

区間kernelが先に処理する。

### 8.5 finite overflow

exact real resultがfiniteだがbinary64有限範囲外の場合:

```text
positive overflow:
  Up   -> +Infinity
  Down -> +double.MaxValue

negative overflow:
  Up   -> -double.MaxValue
  Down -> -Infinity
```

operand自体がInfinityでexact resultが非有界の場合と区別する。

---

## 9. 加算・減算の方向付き丸め

### 9.1 TwoSum

有限operandかつoverflowなし:

```text
s = roundNearest(x+y)
e = exact(x+y)-s
```

概念コード:

```csharp
static (double Sum, double Error) TwoSum(double x, double y)
{
    double s = x + y;
    double bv = s - x;
    double e = (x - (s - bv)) + (y - bv);
    return (s, e);
}
```

### 9.2 AddUp/Down

```text
AddUp:
  e > 0 -> NextUp(s)
  else  -> s

AddDown:
  e < 0 -> NextDown(s)
  else  -> s
```

### 9.3 subtraction

```text
SubtractUp(x,y)   = AddUp(x,-y)
SubtractDown(x,y) = AddDown(x,-y)
```

---

## 10. 乗算の方向付き丸め

### 10.1 定数

```text
SmallProductThreshold = 2^-969
ProductScale          = 2^537
```

`abs(product) >= 2^-969`は通常残差経路。
`abs(product) < 2^-969`はscaled経路。

### 10.2 FMA residual

```csharp
product = x * y;
error = Math.FusedMultiplyAdd(x, y, -product);
```

通常経路:

```text
Up:
  error > 0 -> NextUp(product)

Down:
  error < 0 -> NextDown(product)
```

### 10.3 scaled path

```text
sx = x * 2^537
sy = y * 2^537
(s,s2) = exact-product-decomposition(sx,sy)
t = (product * 2^537) * 2^537
```

Up補正:

```text
t < s
or t == s && s2 > 0
```

Down補正:

```text
t > s
or t == s && s2 < 0
```

`t == s && s2 == 0`はexact。

### 10.4 overflow

```text
nearest +Infinity:
  Up   -> +Infinity
  Down -> +MaxValue

nearest -Infinity:
  Up   -> -MaxValue
  Down -> -Infinity
```

Infinity operandの場合はexact Infinityとして別分岐。

---

## 11. 除算の方向付き丸め

### 11.1 定数

```text
SmallNumeratorThreshold = 2^-969
LargeDenominatorLimit   = 2^918
DivisionScale           = 2^105
MinimumSubnormal        = 2^-1074
```

### 11.2 denominator正符号化

```text
if y < 0:
    xn = -x
    yn = -y
else:
    xn = x
    yn = y
```

`yn > 0`として比較する。

### 11.3 small numerator

```text
abs(xn) < 2^-969:
    if abs(yn) < 2^918:
        xn *= 2^105
        yn *= 2^105
    else:
        early return
```

境界`abs(xn)==2^-969`は通常経路。
`abs(yn)==2^918`はearly-return側。

### 11.4 large denominator early return

Up:

```text
xn < 0 -> +0.0
xn > 0 -> +2^-1074
```

Down:

```text
xn < 0 -> -2^-1074
xn > 0 -> +0.0
```

### 11.5 normal residual comparison

```text
q  = roundNearest(xn/yn)
r  = roundNearest(q*yn)
r2 = FMA(q,yn,-r)
```

Up補正:

```text
r < xn
or r == xn && r2 < 0
```

Down補正:

```text
r > xn
or r == xn && r2 > 0
```

rounded high partが等しくてもresidualを必ず見る。

---

## 12. 四則演算の区間kernel

全演算でoperandのいずれかがEmptyならEmpty。

### 12.1 add

```text
[a,b] + [c,d]
= [RD(a+c), RU(b+d)]
```

内部:

```text
[-a,b] + [-c,d]
= [-RD(a+c), RU(b+d)]
```

### 12.2 subtract

```text
[a,b] - [c,d]
= [RD(a-d), RU(b-c)]
```

内部では右operandのlane swapを利用できる。

### 12.3 unary minus

```text
-[a,b] = [-b,-a]
[-a,b] -> [b,-a]
```

内部ではlane swap。

### 12.4 sign class

```text
Z: [0,0]
P: 0 <= lower, Zではない
N: upper <= 0, Zではない
M: lower < 0 < upper
```

### 12.5 multiplication table

`A=[a,b]`, `B=[c,d]`:

| A | B | Lower | Upper |
|---|---|---|---|
| Z | * | `0` | `0` |
| * | Z | `0` | `0` |
| P | P | `RD(a*c)` | `RU(b*d)` |
| P | N | `RD(b*c)` | `RU(a*d)` |
| P | M | `RD(b*c)` | `RU(b*d)` |
| N | P | `RD(a*d)` | `RU(b*c)` |
| N | N | `RD(b*d)` | `RU(a*c)` |
| N | M | `RD(a*d)` | `RU(a*c)` |
| M | P | `RD(a*d)` | `RU(b*d)` |
| M | N | `RD(b*c)` | `RU(a*c)` |
| M | M | `min(RD(a*d),RD(b*c))` | `max(RU(a*c),RU(b*d))` |

Zeroを先に処理し`0*Infinity`をprimitiveへ渡さない。

### 12.6 division: denominator excludes zero

B positive `0<c<=d`:

| A | Lower | Upper |
|---|---|---|
| P | `RD(a/d)` | `RU(b/c)` |
| N | `RD(a/c)` | `RU(b/d)` |
| M | `RD(a/c)` | `RU(b/c)` |

B negative `c<=d<0`:

| A | Lower | Upper |
|---|---|---|
| P | `RD(b/d)` | `RU(a/c)` |
| N | `RD(b/c)` | `RU(a/d)` |
| M | `RD(b/d)` | `RU(a/d)` |

reciprocalを一度作って乗算する方式は、二重丸めを避けるため採用しない。

### 12.7 denominator Zero

```text
A/[0,0] -> Empty
```

### 12.8 one-sided zero denominator

`B=[0,d]`, `d>0`:

| A | Result |
|---|---|
| Z | Zero |
| P | `[RD(a/d),+Infinity]` |
| N | `[-Infinity,RU(b/d)]` |
| M | Entire |

`B=[c,0]`, `c<0`:

| A | Result |
|---|---|
| Z | Zero |
| P | `[-Infinity,RU(a/c)]` |
| N | `[RD(b/c),+Infinity]` |
| M | Entire |

### 12.9 denominator crosses zero

```text
c < 0 < d
Zero/B -> Zero
otherwise -> Entire
```

真の像が非連結でも、bare `Interval`はconvex hullを返す。

---

## 13. 等値性・Hash・順序

### 13.1 equality

- Empty同士は等しい。
- 非空はcanonical endpointが等しい場合に等しい。
- `+0.0/-0.0`入力差は影響しない。
- NaIはbare `Interval`に存在しない。

### 13.2 Hash

- Emptyは固定Hash。
- zero endpointはcanonical bit pattern。
- internal NaN payloadを直接Hashしない。

### 13.3 normal comparison operator

`<`, `<=`, `>`, `>=`は基本`Interval`へ定義しない。

区間にはsubset、endpoint-wise less、precedes等の異なる関係があるため、Phase 4Aでnamed APIを提供する。

---

## 14. Exact oracle・reference harness

### 14.1 exact-rational oracle

有限binary64をexactに

```text
significand * 2^exponent
```

へ分解する。

- add/sub/mul: `BigInteger`でexact value
- div: rational numerator/denominator
- production packageへ`BigInteger` oracleを含めない

### 14.2 finite overflow変換

`R`をfinite exact rational、`M`を`double.MaxValue`のexact rationalとする。

BCL nearest resultへ変換する前に必ず比較する。

```text
R > M:
  Up   -> +Infinity
  Down -> +MaxValue

R < -M:
  Up   -> -MaxValue
  Down -> -Infinity
```

`-M <= R <= M`のときのみBCL nearest finite result`n`をexact rational`N`へ変換する。

```text
Up:
  N < R -> BitIncrement(n)
  else  -> n

Down:
  N > R -> BitDecrement(n)
  else  -> n
```

finite overflowとInfinity operandによるexact Infinityを別caseとして記録する。

### 14.3 inari adapter

固定:

```text
repo   = unageek/inari
commit = 18b83a571d7681c76067bc38d90a74e8be29f545
```

用途:

- Empty / Entire
- zero-crossing semantics
- set/arithmetic result
- endpoint differential
- Phase 4関数semantic reference

### 14.4 kv adapter

固定:

```text
repo   = mskashi/kv
commit = c7f8f2324a0e403cca6b39f46088a22843d440db
file   = kv/rdouble-nohwround.hpp
```

用途:

- `add_up/down`
- `mul_up/down`
- `div_up/down`
- `sqrt_up/down`

zero-containing interval divisionはDevo6/inariと意味論が異なるため、kv interval resultをoracleにしない。

### 14.5 reference lock

`tests/ReferenceData/reference-lock.json`に次を記録する。

- inari SHA
- kv SHA
- ITF1788 SHA
- MPFR version
- adapter/generator source hash
- toolchain version
- target triple
- generator command
- corpus SHA-256
- license/NOTICE path

### 14.6 corpus

JSON Lines、数値は16桁hex binary64 bits。

必須metadata:

- schema
- caseId
- operation
- operand bit patterns/state
- expected state/bits
- source
- source revision
- applicability
- expected-difference reason

caseIdでsortし、generator iteration orderへ依存しない。

---

## 15. IEEE 1788.1-oriented conformance

### 15.1 Phase 1 matrix

| concept | API | source | required |
|---|---|---|---|
| empty | `Interval.Empty` | repository matrix | yes |
| entire | `Interval.Entire` | repository matrix | yes |
| numsToInterval | constructor/TryCreate | ITF1788 | yes |
| inf | `Lower` | `libieeep1788_num.itl` | yes |
| sup | `Upper` | `libieeep1788_num.itl` | yes |
| isEmpty | `IsEmpty` | `libieeep1788_bool.itl` | yes |
| isEntire | `IsEntire` | `libieeep1788_bool.itl` | yes |
| isSingleton | `IsSingleton` | repository equivalent matrix | yes |
| equal | `Equals`,`==` | `libieeep1788_bool.itl` | yes |
| neg | unary `-` | `libieeep1788_elem.itl` | yes |
| add/sub/mul/div | operators | `libieeep1788_elem.itl` | yes |

### 15.2 constructor source

固定ITF1788:

```text
commit = d8c2a64478ebdc9cbde6ccef33eaad3bed60ed81
```

使用:

- `itl/libieeep1788_class.itl`
  - bare `b-numsToInterval`
- `itl/ieee1788-constructors.itl`
  - compatible bare numeric supplement

最低限:

```text
(-1,1)                -> [-1,1]
(-Inf,1)              -> [-Inf,1]
(-1,+Inf)             -> [-1,+Inf]
(-Inf,+Inf)           -> Entire
(NaN,NaN)             -> invalid
(1,-1)                -> invalid
(-Inf,-Inf)           -> invalid
(+Inf,+Inf)           -> invalid
```

invalid case:

- constructor -> `ArgumentException`
- TryCreate -> false, out=Empty

### 15.3 IsSingleton repository matrix

固定ITF1788に適切な`isSingleton` corpusが存在しないため、repository-defined matrixを使用する。

| Case | Interval | Expected |
|---|---|---|
| Empty | Empty | false |
| Entire | Entire | false |
| finite singleton | `[1,1]` | true |
| negative singleton | `[-2,-2]` | true |
| Zero | `[-0,+0]` | true |
| signed zero variants | any zero pair | true after normalization |
| bounded non-singleton | `[1,2]` | false |
| lower-unbounded | `[-Inf,2]` | false |
| upper-unbounded | `[1,+Inf]` | false |

契約:

```text
IsSingleton = !IsEmpty && Lower == Upper
```

### 15.4 Empty inf/sup

required:

```text
inf(Empty) = +Infinity
sup(Empty) = -Infinity
```

C# mapping:

```text
inf -> Lower
sup -> Upper
```

### 15.5 manifest/gate

各caseに:

- external/repository-defined source
- path/testcase
- adaptation
- required/deferred/excluded/approved-deviation
- expected

を保存する。

sourceから宣言operationが0件だった場合、黙ってpassせずsource extraction errorとする。

summaryは最低限:

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

---

## 16. 四則演算決定的fixture

### 16.1 threshold

```text
2^-969 previous = 0x035fffffffffffff
2^-969          = 0x0360000000000000
2^-969 next     = 0x0360000000000001

2^918 previous  = 0x794fffffffffffff
2^918           = 0x7950000000000000
2^918 next      = 0x7950000000000001

2^-1074         = 0x0000000000000001
```

### 16.2 large denominator

基準:

```text
x = 0x035fffffffffffff
y = 0x7950000000000000
```

| quotient sign | direction | expected |
|---|---|---|
| positive | Up | `+2^-1074` |
| positive | Down | `+0` |
| negative | Up | `+0` |
| negative | Down | `-2^-1074` |

### 16.3 multiplication scaled witnesses

| Case | xBits | yBits | branch |
|---|---|---|---|
| t-lt-s | `216b5087a9deee3d` | `1e04591a0fee6d8d` | `t<s`, Up correction |
| t-gt-s | `8b8ab461601ec773` | `33c03ee4daaa7148` | `t>s`, Down correction |
| eq-pos | `b8aefe57fced900a` | `88b7778db0690811` | `t==s && s2>0` |
| eq-neg | `b2e664c6cc3b90be` | `8e6b00818bab3ede` | `t==s && s2<0` |
| exact | `3ff0000000000000` | `0000000000000001` | `s2==0` |

### 16.4 division residual tie

| Case | xBits | yBits | qBits | relation |
|---|---|---|---|---|
| positive residual | `35b62b4b61f6a01a` | `6a4b103b1dfd16c0` | `0b5a36846200f80c` | `r==xn && r2>0` |
| negative residual | `0e0db74836096727` | `a9ad3e48c2f627a6` | `a4504233b80eaec4` | `r==xn && r2<0` |

### 16.5 finite overflow

add/sub/mul/divの正負両方向をexact oracleのfinite-overflow pathへ通す。

例:

```text
+MaxValue * 2:
  Up   -> +Infinity
  Down -> +MaxValue

-MaxValue * 2:
  Up   -> -MaxValue
  Down -> -Infinity
```

---

## 17. Property test

Phase 1:

- commutativity of add/mul
- double negation
- Zero identities
- result invariant
- exact sampled point inclusion

分配法則の等号は依存性問題のため要求しない。

Phase 3:

- scalarとSIMDでstate一致
- canonical endpoint bits一致
- scalarより広いだけのSIMD結果は不合格

---

## 18. CI・failure diagnostics

### 18.1 最初の実装PRでworkflowを追加する

現在repositoryに実行可能projectがないため、設計PRではworkflowを追加しない。

Phase 1でproject/testを追加する最初のPRに、診断artifact workflowも同時追加する。

### 18.2 architecture matrix

最低限:

```yaml
strategy:
  matrix:
    include:
      - architecture: x64
        runs-on: ubuntu-24.04
      - architecture: arm64
        runs-on: ubuntu-24.04-arm
```

同一commit、同一test assembly、同一corpusを実行する。

### 18.3 architecture間比較

各jobが`canonical-results.jsonl`をcaseId順で生成。

後続jobでbyte-for-byte比較し、SHA-256と全差分を保存する。

### 18.4 artifact

成功/失敗にかかわらず`if: always()`相当で保存:

- `.trx`等test result
- stdout
- stderr
- diagnostic log
- runtime/OS/architecture/CPU features
- reference-lock
- conformance summary
- canonical result corpus
- mismatch input
- exact result
- Devo6 result
- inari/kv/MPFR resultまたはN/A
- expected difference reason

Phase 4では追加で:

- function/domain/sign/quadrant class
- clipped domain
- endpoint backend
- correction decision
- periodic reduction
- detected critical point/pole/branch cut
- parser exact rational/resource limit
- union/decoration/split state

### 18.5 exact-head CI

CI確認対象は、確認時点のPR current HEAD SHAとrunの`head_sha`が一致するrunだけ。

HEAD更新後は新HEADを再確認する。

matching runがなければ**CI未実施**。

別SHAのrunを代用しない。

---

## 19. API確定ゲート

Phase 2完了条件:

- representative calculationがoperatorで自然に記述できる
- Empty/Entire明示判定
- invalid constructorとEmpty演算結果の違いが明確
- signed zero semantics固定
- §16 fixture成功
- exact oracle差異なし
- Phase 1 conformance required case成功
- inari差異が0または承認済み
- kv primitive差異が0または承認済み
- Linux x64/ARM64 corpus一致
- public API baseline保存
- basic operation allocation 0
- BenchmarkDotNet scalar baseline保存

breaking changeは`doc/Design/BreakingChanges.md`へ記録する。

---

## 20. SIMD設計

### 20.1 capability独立判定

```text
Avx512F.IsSupported
Avx2.IsSupported
Avx.IsSupported
Fma.IsSupported
Sse2.IsSupported
AdvSimd.Arm64.IsSupported
```

FMAをAVX2/SSE2へ暗黙従属させない。

### 20.2 capability matrix

| Environment | Add/Sub | Mul/Div | initial policy |
|---|---|---|---|
| AVX-512F | packed directed | packed directed | 4 interval batch candidate |
| AVX2+FMA | vector TwoSum | vector FMA residual candidate | correctness+benchmark後 |
| AVX2 no FMA | vector TwoSum | scalar fallback | mul/div vector Dekkerは別評価 |
| AVX+FMA | candidate | candidate | benchmark後 |
| SSE2 no FMA | Vector128 TwoSum | scalar fallback | add/sub候補 |
| ARM64 AdvSimd | vector candidate | scalar until exactness proven | differential後 |
| other | scalar | scalar | always |

### 20.3 AVX-512 batch layout

```text
[-L0,U0,-L1,U1,-L2,U2,-L3,U3]
```

上向き丸め付きpacked operationで4区間を処理する。

末尾4未満はscalar。

### 20.4 batch API候補

```csharp
public static class IntervalBatch
{
    public static void Add(
        ReadOnlySpan<Interval> left,
        ReadOnlySpan<Interval> right,
        Span<Interval> destination);

    public static void Subtract(...);
    public static void Multiply(...);
    public static void Divide(...);
}
```

Phase 2基本API freezeとは別のadditive API。

長さ不一致、overlap、in-placeはPhase 3 API reviewで確定する。

### 20.5 production採用条件

- scalarとcanonical bitwise equivalent
- fallback動作
- special/subnormal differential成功
- feature combination test成功
- benchmark上の改善

改善がないkernelはproduction dispatchへ入れない。

---

## 21. Native backend判断

Phase 1、2、Phase 3初期はmanaged-only。

scalar operatorごとのP/Invokeは採用しない。

native再検討gate:

1. Phase 3後のlarge-batch benchmark
2. Phase 4初等関数backend選定

採用条件:

- managedでは利用できないdirected rounding/math function能力
- interop/copy/dispatch込みで実workload改善
-同一set semantics/canonical endpoint
- x64/ARM64/deployment配布可能
- ABI/thread safety/NativeAOT/trimming影響を許容
- license/notice満足

native採用時も公開`Interval`は変更しない。

---

# Part II: Phase 4 詳細設計

## 22. Phase 4共通アーキテクチャ

### 22.1 公開型責務

`Interval`:

- bare interval state
- set relation
- set operation
- numeric property

`IntervalMath`:

- point functionのinterval extension

`IntervalConstants`:

- tight real constants

`IntervalUnion2`:

- 0～2 nonconnected components

`DecoratedInterval`:

- interval + decoration / NaI

`IntervalContractor`:

- reverse/cancellative operations

### 22.2 math function layering

```text
public interval extension
  Empty propagation
  domain clipping
  monotonic/sign/quadrant classification
  extrema/pole/periodic point detection
  connected/union result construction
            ↓
certified scalar endpoint kernel
  FooDown(double)
  FooUp(double)
  constants/range reduction
  overflow/underflow/subnormal
  correct directed binary64 rounding
```

### 22.3 accuracy class

- Exact: relation/set operation等
- Tight directed algebraic: square/sqrt/integer power等
- Tight certified elementary: exp/log/sin等

`Math.Sin`等に証明なしで固定ULPを加える方式を正式backendにしない。

---

## 23. Phase 4A 公開API候補

```csharp
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

名前はPhase 4A API reviewで確定する。

---

## 24. 集合演算

`X=[a,b]`, `Y=[c,d]`。

### 24.1 Intersect

```text
Empty ∩ Y = Empty
X ∩ Empty = Empty
X ∩ Y = [max(a,c),min(b,d)]
```

下限>上限ならEmpty。

内部:

```text
min([-a,b],[-c,d])
= [-max(a,c),min(b,d)]
```

### 24.2 ConvexHull

```text
hull(Empty,Y) = Y
hull(X,Empty) = X
hull(X,Y) = [min(a,c),max(b,d)]
```

内部:

```text
max([-a,b],[-c,d])
= [-min(a,c),max(b,d)]
```

### 24.3 property

- commutative
- idempotent
- intersection subset of operands
- operands subset of hull
- intersection with Entire = self
- hull with Empty = self

---

## 25. Contains

`±Infinity`は非有界端点であって実数要素ではない。

```text
Contains(X,x)
= !X.IsEmpty
  && IsFinite(x)
  && Lower <= x <= Upper
```

| Interval | value | result |
|---|---|---|
| Empty | any | false |
| Entire | finite | true |
| Entire | ±Infinity | false |
| any | NaN | false |
| Zero | ±0 | true |

---

## 26. Relation

### 26.1 extended strict relation `<′`

```text
x <′ y iff
    x < y
 or x == y == -Infinity
 or x == y == +Infinity
```

### 26.2 subset

```text
Empty subset Y = true
nonempty subset Empty = false
[a,b] subset [c,d] iff c<=a && b<=d
```

### 26.3 interior

```text
Empty interior Y = true
nonempty interior Empty = false
[a,b] interior [c,d] iff c <′ a && b <′ d
```

`Entire.IsInteriorOf(Entire)`はtrue。

### 26.4 disjoint

```text
Empty disjoint Y = true
[a,b] disjoint [c,d] iff b<c || d<a
```

端点接触はdisjointではない。

### 26.5 precedes

```text
Empty involved -> true
[a,b] precedes [c,d] iff b<=c
strict precedes iff b<c
```

### 26.6 endpoint-wise less

Weak:

```text
Empty vs Empty = true
Empty vs nonempty = false
nonempty vs Empty = false
[a,b] <=weak [c,d] iff a<=c && b<=d
```

Strict:

```text
[a,b] <strict [c,d] iff a<′c && b<′d
```

subsetとは別概念。

---

## 27. IntervalOverlap

```csharp
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
```

`self=[a,b]`, `other=[c,d]`:

| state | condition |
|---|---|
| BothEmpty | both Empty |
| FirstEmpty | self only Empty |
| SecondEmpty | other only Empty |
| Before | `b<c` |
| Meets | `a<b && b==c && c<d` |
| Overlaps | `a<c && c<b && b<d` |
| Starts | `a==c && b<d` |
| ContainedBy | `c<a && b<d` |
| Finishes | `c<a && b==d` |
| Equals | `a==c && b==d` |
| FinishedBy | `a<c && b==d` |
| Contains | `a<c && d<b` |
| StartedBy | `a==c && d<b` |
| OverlappedBy | `c<a && a<d && d<b` |
| MetBy | `c<d && d==a && a<b` |
| After | `d<a` |

inverse:

```text
FirstEmpty <-> SecondEmpty
Before <-> After
Meets <-> MetBy
Overlaps <-> OverlappedBy
Starts <-> StartedBy
ContainedBy <-> Contains
Finishes <-> FinishedBy
Equals <-> Equals
```

16状態を最低1fixtureずつ持つ。

---

## 28. 数値的属性

### 28.1 IsBounded

```text
!IsEmpty && IsFinite(Lower) && IsFinite(Upper)
```

### 28.2 Width

```text
Empty -> NaN
[a,b] -> RU(b-a)
unbounded -> +Infinity
singleton -> +0
```

内部:

```text
AddUp(_upper,_negatedLower)
```

### 28.3 Midpoint

representative scalarでありenclosure endpointではない。

```text
Empty -> NaN
Entire -> +0.0
[-Infinity,b] -> double.MinValue
[a,+Infinity] -> double.MaxValue
finite -> exact (a+b)/2 の採用丸め規則によるbinary64
```

単純`(a+b)/2`のoverflowを避ける。

```text
if a+b finite:
    (a+b)/2
else:
    a/2+b/2
```

finite midpointのtie policyはPhase 4A API/conformance reviewで最終確認する。実装はcross-architecture deterministic fixtureを持つ。

### 28.4 Radius

`m=Midpoint`として最小binary64 `r`で

```text
X subset [m-r,m+r]
```

を満たす。

```text
Empty -> NaN
r = max(SubtractUp(m,Lower),SubtractUp(Upper,m))
```

### 28.5 Magnitude

```text
Empty -> NaN
max(abs(a),abs(b))
```

### 28.6 Mignitude

```text
Empty -> NaN
0 in X -> +0
b<0 -> abs(b)
a>0 -> abs(a)
```

---

## 29. Abs・Sign・Pointwise Min/Max

### 29.1 Abs

```text
Empty -> Empty
0<=a -> [a,b]
b<=0 -> [-b,-a]
a<0<b -> [-0,max(-a,b)]
```

### 29.2 Sign

```text
sign(x) = -1 x<0
           0 x=0
           1 x>0
```

| input | result |
|---|---|
| Empty | Empty |
| b<0 | `[-1,-1]` |
| a<0,b=0 | `[-1,0]` |
| Zero | `[0,0]` |
| a=0,b>0 | `[0,1]` |
| a<0<b | `[-1,1]` |
| a>0 | `[1,1]` |
| Entire | `[-1,1]` |

### 29.3 pointwise min

```text
[min(a,c),min(b,d)]
```

### 29.4 pointwise max

```text
[max(a,c),max(b,d)]
```

どちらかEmptyならEmpty。

`ConvexHull`と名称・意味を明確に区別する。

---

## 30. 整数値関数

単調非減少なので非空`[a,b]`へendpoint mapping。

```text
Floor([a,b])    = [floor(a),floor(b)]
Ceiling([a,b])  = [ceil(a),ceil(b)]
Truncate([a,b]) = [trunc(a),trunc(b)]
Round([a,b])    = [round(a),round(b)]
```

- Empty -> Empty
- Infinity endpoint維持
- endpoint結果は整数binary64なので追加outward rounding不要
- zeroはcanonical endpointへ正規化
-未知`MidpointRounding` enum -> `ArgumentOutOfRangeException`

---

## 31. Phase 4B 公開API候補

```csharp
public static partial class IntervalMath
{
    public static Interval Reciprocal(Interval value);
    public static Interval Square(Interval value);
    public static Interval Sqrt(Interval value);
    public static Interval Pow(Interval value, int exponent);
    public static Interval Root(Interval value, int degree);
    public static Interval FusedMultiplyAdd(
        Interval left,
        Interval right,
        Interval addend);
}

public static class IntervalConstants
{
    public static Interval Pi { get; }
    public static Interval HalfPi { get; }
    public static Interval TwoPi { get; }
    public static Interval E { get; }
    public static Interval Ln2 { get; }
    public static Interval Ln10 { get; }
    public static Interval Sqrt2 { get; }
}
```

---

## 32. Certified scalar endpoint kernel

Phase 4 math functionsは2層に分離する。

```text
interval extension
  domain / monotonicity / extrema / pole
            ↓
directed scalar function
  FooDown(double)
  FooUp(double)
```

正式backend条件:

1. 全binary64に対する誤差上限と方向補正が証明されたmanaged実装
2. correctly-rounded implementationの検証済み移植
3. MPFR等のdirected native backend

BCL `Math.*`はcandidate seedには使えるが、単体で包含保証の根拠にしない。

---

## 33. IntervalConstants

`π`等をnearest doubleの点区間として偽装しない。

```text
Pi.Lower <= π <= Pi.Upper
```

通常、隣接binary64 2点でtight enclosureとなる。

生成:

- MPFR directed conversion
- endpoint bitsをsource/generated dataへ固定
- build時native/network不要
- integrity jobで再生成一致

三角関数range reduction用にはpublic 2-endpoint constantとは別に高精度`2/π`, `π/2` split tableを使う。

---

## 34. Reciprocal

`X=[a,b]`:

| class | result |
|---|---|
| Empty | Empty |
| Zero | Empty |
| `a<0<b` | Entire |
| `a<0,b=0` | `[-Infinity,RU(1/a)]` |
| `a=0,b>0` | `[RD(1/b),+Infinity]` |
| b<0 | `[RD(1/b),RU(1/a)]` |
| a>0 | `[RD(1/b),RU(1/a)]` |

2成分を保持するversionはPhase 4E。

---

## 35. Square

`X*X`へ委譲しない。依存性問題による拡大を避ける。

```text
Empty -> Empty
0<=a -> [RD(a*a),RU(b*b)]
b<=0 -> [RD(b*b),RU(a*a)]
a<0<b -> [-0,max(RU(a*a),RU(b*b))]
```

---

## 36. Square Root

### 36.1 interval extension

Domain `[0,+Infinity)`。

```text
Empty -> Empty
b<0 -> Empty
a<=0<=b -> [-0,SqrtUp(b)]
0<a -> [SqrtDown(a),SqrtUp(b)]
```

### 36.2 directed algorithm

`kv` no-hardware-roundingを参照。

```text
SmallSqrtInputThreshold = 2^-969
SqrtInputScale          = 2^106
SqrtResultScale         = 2^53
```

```text
r = roundNearest(sqrt(x))

if x < 2^-969:
    xs = x*2^106
    rs = r*2^53
    (p,e)=exactProduct(rs,rs)
    compare p+e with xs
else:
    (p,e)=exactProduct(r,r)
    compare p+e with x
```

Up:

```text
p < target or (p==target && e<0) -> NextUp(r)
```

Down:

```text
p > target or (p==target && e>0) -> NextDown(r)
```

0、min subnormal、threshold previous/equal/next、MaxValue、perfect/non-perfect squareをfixture化する。

---

## 37. Integer Power

API:

```csharp
IntervalMath.Pow(Interval value, int exponent)
```

### 37.1 n=0

```text
Pow(Empty,0)=Empty
Pow(nonempty,0)=[1,1]
```

### 37.2 positive odd

```text
[RD(a^n),RU(b^n)]
```

### 37.3 positive even

`A=Abs(X)`:

```text
[RD(A.Lower^n),RU(A.Upper^n)]
```

### 37.4 negative n

Zeroだけ:

```text
Empty
```

negative even:

```text
A=Abs(X)
[RD(A.Upper^n),RU(A.Lower^n)]
```

zero接触時upperは+Infinity。

negative odd:

```text
strict zero crossing -> Entire
otherwise -> [RD(b^n),RU(a^n)]
```

one-sided zero:

```text
[0,b] -> [RD(b^n),+Infinity]
[a,0] -> [-Infinity,RU(a^n)]
```

`int.MinValue`の絶対値を`int`で取らず、符号+`uint` magnitudeへ分解する。

endpoint primitiveは最終exact `x^n`を正しく方向丸めする。途中の反復丸めだけをtightness根拠にしない。

---

## 38. Integer Root

```csharp
IntervalMath.Root(Interval value, int degree)
```

- degree<=0 -> `ArgumentOutOfRangeException`
- degree=1 -> input
- Empty -> Empty

odd degree:

```text
[RootDown(a,n),RootUp(b,n)]
```

even degree:

```text
b<0 -> Empty
a<=0<=b -> [-0,RootUp(b,n)]
0<a -> [RootDown(a,n),RootUp(b,n)]
```

Newton candidateだけで確定せず、candidate^nと入力のexact relationを検証して隣接補正する。

---

## 39. FusedMultiplyAdd

```text
FMA(X,Y,Z)
= hull({x*y+z})
```

scalar primitive:

```text
FmaDown(x,y,z)=RD(exact(x*y+z))
FmaUp(x,y,z)=RU(exact(x*y+z))
```

`(X*Y)+Z`へ委譲しない。

乗算符号classからlower/upper candidate endpoint pairを選び、Z.Lower/Z.Upperを1回のFMAで加える。

mixed×mixedではlower/upper各2candidateを評価してmin/max。

Zero productはaddend。

Empty operandはEmpty。

propertyとして、同じset semanticsでは原則`FMA(X,Y,Z)`が`(X*Y)+Z`のsubsetまたは同値となることを検証する。

---

## 40. Phase 4C/4D 公開API候補

```csharp
public static partial class IntervalMath
{
    public static Interval Exp(Interval value);
    public static Interval Exp2(Interval value);
    public static Interval Exp10(Interval value);
    public static Interval Log(Interval value);
    public static Interval Log2(Interval value);
    public static Interval Log10(Interval value);

    public static Interval Sinh(Interval value);
    public static Interval Cosh(Interval value);
    public static Interval Tanh(Interval value);
    public static Interval Asinh(Interval value);
    public static Interval Acosh(Interval value);
    public static Interval Atanh(Interval value);

    public static Interval Sin(Interval value);
    public static Interval Cos(Interval value);
    public static Interval Tan(Interval value);
    public static Interval Asin(Interval value);
    public static Interval Acos(Interval value);
    public static Interval Atan(Interval value);
    public static Interval Atan2(Interval y, Interval x);

    public static Interval Pow(
        Interval value,
        Interval exponent);
}
```

---

## 41. 初等関数reference

primary reference corpusは固定MPFR versionで生成する。

binary64をexact入力し、53-bit precisionでRNDD/RNDUを指定する。

pinned `inari`もMPFR RNDD/RNDUを使うためsecondary oracleとする。

lock:

- MPFR version
- wrapper hash
- compiler/target
- input/output corpus hashes
- rounding mode
- inari SHA

---

## 42. 単調関数共通処理

Domainとのintersectionを`[l,u]`とする。

単調増加:

```text
[FDown(l),FUp(u)]
```

単調減少:

```text
[FDown(u),FUp(l)]
```

open boundaryへ接する場合はlimit値を使用する。

例:

```text
Log([0,2]) = [-Infinity,LogUp(2)]
Atanh([-1,0]) = [-Infinity,+0]
```

境界点しか含まずdomain内点がない場合はEmpty。

---

## 43. Exp

`Exp`, `Exp2`, `Exp10`は全実数上単調増加。

```text
[FDown(a),FUp(b)]
```

- Empty -> Empty
- `-Infinity` -> +0 limit
- `+Infinity` -> +Infinity
- finite overflow/underflowをtightに処理
- interval lower zeroはcanonical -0

---

## 44. Log

`Log`, `Log2`, `Log10` domain `(0,+Infinity)`。

```text
b<=0 -> Empty
otherwise:
  lower = -Infinity if a<=0 else LogDown(a)
  upper = +Infinity if b==+Infinity else LogUp(b)
```

`Log([0,0])=Empty`。

---

## 45. Hyperbolic / inverse

### 45.1 monotonic increasing

- Sinh
- Tanh
- Asinh
- Atan

```text
[FDown(a),FUp(b)]
```

limits:

```text
Tanh(-Inf)=-1
Tanh(+Inf)=+1
Atan(-Inf)=-π/2
Atan(+Inf)=+π/2
```

### 45.2 Cosh

```text
b<0 -> [CoshDown(b),CoshUp(a)]
a>0 -> [CoshDown(a),CoshUp(b)]
a<=0<=b -> [1,CoshUp(max(-a,b))]
```

### 45.3 Acosh

Domain `[1,+Infinity)`。

```text
b<1 -> Empty
l=max(a,1)
[AcoshDown(l),AcoshUp(b)]
```

### 45.4 Asin

Domain `[-1,1]`, increasing。

clip後:

```text
[AsinDown(l),AsinUp(u)]
```

### 45.5 Acos

Domain `[-1,1]`, decreasing。

```text
[AcosDown(u),AcosUp(l)]
```

### 45.6 Atanh

Domain `(-1,1)`。

```text
b<=-1 or a>=1 -> Empty
lower = -Infinity if a<=-1 else AtanhDown(a)
upper = +Infinity if b>=1 else AtanhUp(b)
```

---

## 46. Periodic range reduction

Sin/Cos/Tanは全binary64範囲で象限・極を誤らない内部reducerを持つ。

候補: Payne-Hanek型fixed-point reduction。

禁止:

- `Math.PI`との通常除算だけでquadrant決定
- `value % (2*Math.PI)`だけでcritical point決定
- public Pi区間の一端だけで整数kを決定

内部に十分なbit数の`2/π`, `π/2` tableを固定する。

`ContainsPeriodicPoint(X,offset,period)`相当のexact判定を持つ。

---

## 47. Sin

`X=[a,b]`。

lower候補:

- `SinDown(a)`
- `SinDown(b)`
- `-1` if `-π/2+2kπ`を含む

upper候補:

- `SinUp(a)`
- `SinUp(b)`
- `+1` if `+π/2+2kπ`を含む

非有界またはmax/min critical latticeの双方を含む場合`[-1,1]`。

critical点を含む場合、近似endpointではなくexact `±1`を使う。

---

## 48. Cos

lower候補:

- CosDown(a)
- CosDown(b)
- -1 if `π+2kπ`

upper候補:

- CosUp(a)
- CosUp(b)
- +1 if `2kπ`

非有界/十分広い場合`[-1,1]`。

---

## 49. Tan

Domain:

```text
R \ {π/2+kπ}
```

poleなしの1branch内:

```text
[TanDown(a),TanUp(b)]
```

poleへdomain内から近づける場合、bare hullはEntire。

poleしか含まずdomain内点なしならEmpty。

pole detectionは高精度fixed-point reductionを使用する。

---

## 50. Atan2

API orderは.NETに合わせる。

```csharp
IntervalMath.Atan2(y,x)
```

Domain `R²\{(0,0)}`、principal range `(-π,π]`。

rectangle algorithm:

1. Empty operand -> Empty
2. X=Zero,Y=Zero -> Empty
3. rectangleをzero境界でsign cellへ分割
4. originを除外
5. quadrantごとのmonotonic cornerをAtan2Down/Up
6. axis候補として0, ±π/2, πを追加
7. negative x branch cutを跨ぐ場合、bare resultはtight `[-π,π]` enclosure
8. angular rangeを専用accumulatorで統合

sign class matrixを固定fixture化する。

---

## 51. General Positive-Base Power

```csharp
IntervalMath.Pow(Interval value, Interval exponent)
```

integer exponent overloadとは別。

point function:

```text
x^y = 0              if x=0 and y>0
      exp(y*log x)   if x>0
```

Domain:

```text
((0,+Infinity)×R) union ({0}×(0,+Infinity))
```

negative baseは対象外。integer powerを使用する。

baseを`[0,+Infinity]`へclipし、`X=[a,b]`, `Y=[c,d]`。

`d<=0`:

```text
b==0 -> Empty
b<1 -> [PowDown(b,d),PowUp(a,c)]
a>1 -> [PowDown(b,c),PowUp(a,d)]
otherwise -> [PowDown(b,c),PowUp(a,c)]
```

`c>0`:

```text
b<1 -> [PowDown(a,d),PowUp(b,c)]
a>1 -> [PowDown(a,c),PowUp(b,d)]
otherwise -> [PowDown(a,d),PowUp(b,d)]
```

`c<=0<d`:

```text
b==0 -> Zero
lower=min(PowDown(a,d),PowDown(b,c))
upper=max(PowUp(a,c),PowUp(b,d))
```

`0^0`をscalar kernelへ直接渡さない。

---

## 52. Elementary production backend gate

候補:

| candidate | correctness | distribution |
|---|---|---|
| certified managed polynomial/table | proof + hard-case fallback必要 | easiest |
| correctly-rounded managed port | license/port verification | managed |
| native MPFR | direct directed rounding | native binaries |
| CRlibm class | supported functions/platform要確認 | native |

shipping rule:

- core公開functionは全support platformで利用可能
-通常入力で`PlatformNotSupportedException`へしない
-optional nativeしかない場合、core同名APIを未完成公開しない
- backend間canonical endpoint一致

---

## 53. Phase 4E: IntervalUnion2

### 53.1 API候補

```csharp
public readonly struct IntervalUnion2 : IEquatable<IntervalUnion2>
{
    public int Count { get; }
    public bool IsEmpty { get; }

    public Interval First { get; }
    public Interval Second { get; }
    public Interval this[int index] { get; }

    public Interval ConvexHull { get; }
}
```

### 53.2 canonical state

```text
Count=0:
  First=Empty
  Second=Empty

Count=1:
  First=nonempty
  Second=Empty

Count=2:
  First/Second nonempty
  First.Upper < Second.Lower
```

接触/overlapした2成分は1成分へmerge。

### 53.3 default

`default(IntervalUnion2)`はempty union。

Countを明示fieldとして管理し、default `Interval.Zero` fieldを成分と解釈しない。

### 53.4 construction

初版public constructorなし。

internal `Create0/Create1/Create2`でEmpty除去、sort、merge、zero normalization。

---

## 54. Extended Division

```csharp
public static IntervalUnion2 DivideToUnion(
    Interval numerator,
    Interval denominator);
```

semantics:

```text
{x/y | x in X, y in Y, y != 0}
```

共通:

```text
X Empty or Y Empty -> Count0
Y Zero -> Count0
Y excludes zero -> Count1(X/Y)
X Zero and Y has nonzero -> Count1(Zero)
```

zero-crossing denominator `Y=[c,d]`, c<0<d:

strict positive `X=[a,b]`, a>0:

```text
[-Infinity,RU(a/c)] union [RD(a/d),+Infinity]
```

strict negative `X=[a,b]`, b<0:

```text
[-Infinity,RU(b/d)] union [RD(b/c),+Infinity]
```

numerator contains zero:

```text
X==Zero -> Zero
otherwise -> Entire
```

property:

```text
X/Y == DivideToUnion(X,Y).ConvexHull
```

ReciprocalToUnionは`1/X`の専用entryとして共通kernelを使用する。

---

## 55. Reverse Multiplication

```csharp
public static class IntervalContractor
{
    public static IntervalUnion2 ReverseMultiply(
        Interval product,
        Interval factor);
}
```

semantics:

```text
{z | exists y in factor : z*y in product}
```

通常divisionとの違い: factor=0を除外しない。

```text
product Empty or factor Empty -> empty union
0 in product && 0 in factor -> Entire
otherwise -> DivideToUnion(product,factor)
```

例:

```text
ReverseMultiply([1,2],Zero)=empty
ReverseMultiply(Zero,Zero)=Entire
ReverseMultiply([0,2],Zero)=Entire
```

---

## 56. Cancellative Operations

候補:

```csharp
IntervalContractor.CancelSubtract(total,term)
IntervalContractor.CancelAdd(total,term)
```

`total=[a,b]`, `term=[c,d]`。

両方nonempty boundedかつ

```text
exact width(total) >= exact width(term)
```

なら:

```text
[RD(a-c),RU(b-d)]
```

それ以外Entire。

rounded Width propertyだけで比較せず、2Sum expansion等でexact width relationを判定する。

```text
CancelAdd(total,term)=CancelSubtract(total,-term)
```

普通の subtraction/additionと意味が異なることをdocumentationで明記する。

---

## 57. DecoratedInterval

### 57.1 Decoration

```csharp
public enum Decoration : byte
{
    Ill = 0,
    Trv = 4,
    Def = 8,
    Dac = 12,
    Com = 16,
}
```

品質順で、基本propagationはminimum。

### 57.2 API候補

```csharp
public readonly struct DecoratedInterval
{
    public static DecoratedInterval NaI { get; }
    public static DecoratedInterval Empty { get; }
    public static DecoratedInterval Entire { get; }

    public bool IsNaI { get; }
    public bool IsEmpty { get; }
    public Decoration Decoration { get; }

    public static DecoratedInterval FromInterval(Interval interval);
    public bool TryGetInterval(out Interval interval);
    public bool SemanticallyEquals(DecoratedInterval other);
}
```

### 57.3 default

`Ill=0`を利用し、

```text
default(DecoratedInterval).IsNaI == true
```

とする。

### 57.4 FromInterval

```text
bounded nonempty -> Com
unbounded -> Dac
Empty -> Trv
```

### 57.5 NaI

- bare Intervalではない
- TryGetInterval=false, out=Empty
- NaI input operation -> NaI

### 57.6 decoration propagation

operationごとのmaximum possible decoration`opDec`を定義し、

```text
resultDec=min(input decorations,opDec)
```

例:

- add -> input min
- divisor contains zero -> Trv以下
- Sqrt([-1,4]) -> bare [0,2], decorated Trv
- Tan pole crossing -> Trv

`DecorationPolicy`へ集約する。

### 57.7 C# equality vs IEEE semantic equality

C# value equalityはreflexive:

```text
NaI == NaI -> true
```

非NaIはinterval part + decorationを比較。

semantic equality:

```text
NaI.SemanticallyEquals(any) -> false
nonNaI -> interval part equality, decoration ignored
```

collection keyとしての`Equals`とIEEE semantic operationを分離する。

---

## 58. Decorated math API

bare `IntervalMath`と分離する。

```csharp
public static class DecoratedIntervalMath
{
    public static DecoratedInterval Sqrt(DecoratedInterval value);
    public static DecoratedInterval Exp(DecoratedInterval value);
    public static DecoratedInterval Sin(DecoratedInterval value);
}
```

operatorはDecoratedInterval同士に限定する。

---

## 59. Parsing

### 59.1 API候補

```csharp
public static Interval Parse(ReadOnlySpan<char> text);
public static bool TryParse(
    ReadOnlySpan<char> text,
    out Interval interval);

public static Interval ParseExact(ReadOnlySpan<char> text);
public static bool TryParseExact(
    ReadOnlySpan<char> text,
    out Interval interval);
```

`IParsable`/`ISpanParsable`はprovider/syntax review後。

### 59.2 initial syntax

```text
Empty
Entire
[a,b]
[a]
```

endpoint:

- decimal float/integer
- `±Infinity`
- exact hexadecimal binary literal

uncertain form、rational token、decoration suffixは別subphase。

### 59.3 exact decimal parsing

`double.Parse`後に点区間を作らない。

exact decimalを

```text
sign * integerSignificand * 10^decimalExponent
```

として解析する。

`BigInteger`を使用可能。parserはhot arithmetic pathではない。

singleton decimal `x`がbinary64に非exact:

```text
[RoundDown(x),RoundUp(x)]
```

`[a,b]`:

```text
lower=RoundDown(exact a)
upper=RoundUp(exact b)
```

exact real relation `a<=b`を先に確認する。

### 59.4 finite decimal overflow

```text
x > MaxValue -> [MaxValue,+Infinity]
x < -MaxValue -> [-Infinity,-MaxValue]
```

Infinity tokenとfinite decimal overflowを区別する。

### 59.5 invalid

- syntax error
- NaN endpoint
- lower>upper
- lower +Infinity
- upper -Infinity
- ±Infinity singleton

Parse -> `FormatException`候補。
TryParse -> false + Empty。

### 59.6 ParseExact

endpointが要求binary64へexact変換できる場合のみ成功。

exact hexを主要round-trip形式とする。

inexact decimal singletonは失敗。

---

## 60. Formatting

### 60.1 ToString

Phase 1 diagnostic用途のまま。

### 60.2 format候補

```text
G: human-readable valid enclosure
R: exact round-trip interval
X: exact hexadecimal endpoints
```

R/Xはparseでcanonical intervalへ完全復元。

hex例:

```text
Empty
[-0x0p+0,0x0p+0]
[0x1.0000000000000p+0,0x1.0000000000001p+0]
[-Infinity,+Infinity]
```

Gで桁を削る場合も、reparse結果が元区間を包含する。

---

## 61. Binary Interchange

private raw memoryをwire contractにしない。

version 1候補:

```text
byte 0: version=1
byte 1: state=0 Empty / 1 Interval
byte 2..9: external Lower IEEE754 bits little endian
byte 10..17: external Upper bits little endian
```

18 bytes。

- Empty endpoint bytesは0固定
- Entireはexternal ±Infinity
- Zeroはlower -0, upper +0
- invalid state/NaN/orderを拒否/canonicalizeする規則をversion固定

API候補:

```csharp
bool TryWriteLittleEndian(Span<byte> destination);
static bool TryReadLittleEndian(
    ReadOnlySpan<byte> source,
    out Interval interval);
```

platform native endianをwireにしない。

JSON converterは別integrationとして設計する。

---

## 62. Interval Splitting

### 62.1 unionとは別

branch-and-bound splitは:

```text
[a,m]
[m,b]
```

mを共有するため、union型なら1成分へmergeされる。

従ってsplitは別API。

### 62.2 API候補

```csharp
bool TrySplitAt(
    double splitPoint,
    out Interval left,
    out Interval right);

bool TryBisect(
    out Interval left,
    out Interval right);
```

### 62.3 TrySplitAt

成功:

- nonempty
- finite splitPoint
- `Lower < splitPoint < Upper`

結果:

```text
[Lower,splitPoint]
[splitPoint,Upper]
```

failure -> false + both Empty。

unboundedでもfinite strict interior pointなら可。

### 62.4 TryBisect

初版はboundedのみ。

failure:

- Empty
- unbounded
- singleton
- strict interior binary64がない

手順:

1. overflow-safe midpoint candidate
2. candidate<=LowerならBitIncrement(Lower)
3. candidate>=UpperならBitDecrement(Upper)
4. strict interior不可ならfalse
5. TrySplitAt

隣接binary64 endpointの区間は自動二分しない。

unbounded automatic heuristicは初版で採用しない。

---

## 63. Parser resource/security

untrusted textを想定し、実装時に固定する。

- max input length
- max significand digits
- max exponent digits/value
- non-recursive parser
- culture separator ambiguity排除
- exceptionへ入力全文を無制限に含めない
- JSON depth/token limit

binary decoderはlength/stateを先に検証し、片側NaN等のinvalid internal stateを作らない。

---

## 64. Phase 4 TDD順序

### 64.1 Phase 4A

1. Contains / IsBounded
2. Intersect / ConvexHull
3. subset/disjoint/precedes
4. interior/less
5. IntervalOverlap全状態
6. Width/Magnitude/Mignitude
7. Midpoint/Radius
8. Abs/Sign
9. PointwiseMin/Max
10. integer functions
11. SIMD differential/benchmark

### 64.2 Phase 4B-D

1. tight constants
2. Reciprocal
3. Square
4. directed Sqrt + interval Sqrt
5. directed integer Power + interval Power(int)
6. Root
7. directed FMA + interval FMA
8. MPFR adapter/corpus
9. Exp group
10. Log group
11. simple hyperbolic/inverse
12. domain-boundary functions
13. periodic reducer
14. Sin/Cos
15. Tan
16. Atan2
17. general Pow

### 64.3 Phase 4E

1. IntervalUnion2
2. DivideToUnion
3. ReciprocalToUnion
4. ReverseMultiply
5. cancellative operations
6. Decoration/default NaI
7. Decorated construction/equality
8. decorated arithmetic
9. decorated elementary
10. exact decimal parser
11. outward parser
12. exact/human formatter
13. binary interchange
14. TrySplitAt
15. TryBisect

各論理単位で**先に失敗testを追加し、失敗を確認してから実装**する。

---

## 65. Phase 4 deterministic fixtures

### 65.1 set/relation

- Empty/Empty/nonempty組合せ
- Entire
- disjoint/touch/overlap/contain/equal
- singleton
- unbounded
- signed zero全組合せ
- Overlap 16 states + inverse

### 65.2 numeric

- finite singleton
- adjacent binary64
- midpoint overflow avoidance
- cancellation
- subnormal
- one-sided unbounded
- Entire/Empty
- midpoint tie
- radius endpoint asymmetry
- Width overflow

### 65.3 algebraic

- Empty/Entire/Zero
- min subnormal/min normal/MaxValue
- exact/nonexact square
- sqrt `2^-969` prev/equal/next
- odd/even power
- negative exponent
- `int.MinValue`
- FMA cancellation/overflow/subnormal

### 65.4 domain boundary

- Log around 0
- Asin/Acos around ±1
- Acosh around 1
- Atanh around ±1
- even Root negative-only/partial domain

### 65.5 periodic

- every quadrant
- `kπ/2` before/include/after
- 1/2/4 quadrant span
- multiple periods
- huge binary64
- Tan pole before/include/after
- Atan2 sign cells/axis/origin/negative-x branch cut

### 65.6 union/decorated/I/O/split

- union count 0/1/2, merge/sort
- extended division sign classes
- reverse Zero cases
- exact-width cancellation boundary
- default NaI/propagation/decoration degrade
- decimal 0.1
- exact hex
- finite overflow decimal
- malformed/huge parser input
- wire state/version/round-trip
- bounded/subnormal/adjacent/signed-zero split

---

## 66. Phase 4 property tests

- Intersection/Hull commutative/idempotent
- Intersection subset of operands
- Hull contains operands
- Overlap inverse consistency
- Width/Radius nonnegative
- Mignitude <= Magnitude
- Abs result subset `[0,+Infinity]`
- monotonic input subset -> output subset
- Square(X) subset or equal X*X
- FMA subset or equal `(X*Y)+Z`
- Sin/Cos subset `[-1,1]`
- Tanh subset `[-1,1]`
- Exp nonnegative
- Cosh subset `[1,+Infinity]`
- backend canonical bits identical
- DivideToUnion.ConvexHull == ordinary division
- `ParseExact(FormatExact(X)) == X`
- decode(encode(X)) == X
- split children cover original and are subsets

random sample containmentだけをprimary correctness proofにしない。

---

## 67. Phase 4 completion gates

各function/typeをpublic APIへ追加する前に:

- semantics/domain matrix review済み
- endpoint kernel tightness根拠あり
- deterministic/conformance/exact/MPFR corpus success
- Linux x64/ARM64 canonical result一致
- SIMD/native backendがscalarとbitwise equivalent
- failure artifactでbranch/rounding/reductionを追跡可能
- API baseline更新
- sample/documentation更新
- current HEADと一致するCI run成功

Phase 4E追加条件:

- canonical state/default fixed
- parse/format/interchange round-trip success
- parser resource limit固定
- bare/decorated semantic差をdocumentationへ明記

---

## 68. 性能・Thread Safety・AOT

Phase 1から:

- `readonly struct`
- hot arithmetic heap allocationなし
- global rounding mode変更なし
- production arithmeticに`BigInteger`なし
- hot path virtual/interface/delegateなし
-無条件1 ULP拡張なし
-不要なreciprocal経由の二重丸めなし
- raw constructorでvalidation重複回避

`Interval`はimmutable。

Phase 1ではreflection/runtime codegen/dynamic assembly/native resolverを使わず、NativeAOT/trimmingを妨げない。

Phase 4 parserの`BigInteger`利用は許可するが、resource limitを持つ。

---

## 69. 「同等の計算結果」の定義

四則演算およびtight公開関数では:

1. 採用したset-based real semanticsを使用
2. 単一Intervalで非連結を表せない場合はconvex hull
3. finite endpointを指定方向へ正しく丸めた最も内側のbinary64
4. signed zeroをcanonicalize
5. internal Empty NaN payloadは比較しない

reference library差異より、exact oracleと採用意味論を優先する。

`IntervalUnion2`ではconvex hullせず最大2componentを保持する。

---

## 70. License・third-party

`kv`/`inari`コードを翻案・移植する場合:

- source commentへcommit SHA
- MIT copyright/permission notice
- test case移植も出典記録

ITF1788 data/generatorを利用・再配布する場合、license/NOTICEをtest assetへ付随させる。

MPFR/native backend採用時は、そのlicense、binary distribution、NOTICEを別途確認する。

production packageに不要なreference toolを同梱しない。

---

## 71. 未確定事項

Phase 1開始を妨げないもの:

- namespace最終名
- constructor vs factory-only
- scalar overload/conversion
- generic math interface
- `ToString`正式format
- batch API名/overlap/in-place規則
- physical `Vector128<double>` field採否
- AVX2/ARM64 performance採否
- native backend最終判断

Phase 4実装前に各subphaseで確定するもの:

- Phase 4A public relation/numeric naming
- Midpoint tie policyの最終conformance確認
- Elementary production backend
- `IntervalUnion2` public constructionの要否
- decorated operation surface
- parser format詳細
- parser resource-limit具体値
- binary interchange final version-1 schema

方向付き四則演算のbranch/threshold/tie、Empty公開端点、Phase 1 conformance source、x64/ARM64 gateは未確定事項ではない。

---

## 72. 参照

### 72.1 repository

- `doc/Design/basic/IntervalArithmetic.md`

### 72.2 inari

- Repository: `unageek/inari`
- Commit: `18b83a571d7681c76067bc38d90a74e8be29f545`
- License: MIT
- 参照:
  - internal `[-Lower,Upper]`
  - Empty/Entire
  - arithmetic sign classification
  - set/relation/numeric/integer functions
  - elementary interval semantics
  - MPFR RNDD/RNDU
  - two-output division
  - decorated interval

### 72.3 kv

- Repository: `mskashi/kv`
- Commit: `c7f8f2324a0e403cca6b39f46088a22843d440db`
- License: MIT
- 参照:
  - TwoSum/TwoProduct
  - no-hardware-rounding
  - subnormal scaling
  - division residual tie
  - sqrt directed correction

### 72.4 ITF1788

- Repository: `unageek/ITF1788`
- Commit: `d8c2a64478ebdc9cbde6ccef33eaad3bed60ed81`
- 用途: test-only conformance corpus source

### 72.5 .NET

- .NET 10
- `Math.FusedMultiplyAdd`
- `Math.BitIncrement` / `Math.BitDecrement`
- `System.Runtime.Intrinsics.X86.Avx512F`
- `Avx2`, `Avx`, `Fma`, `Sse2`
- `AdvSimd.Arm64`

---

## 73. Review履歴と統合後の扱い

PR #3の過去reviewでは、四則演算設計に対し以下のfindingが解消された履歴がある。

- `F-PR3-001`: directed rounding implementation completeness
- `F-PR3-002`: ISA/FMA capability separation
- `F-PR3-004`: conformance source/matrix
- `F-PR3-005`: oracle responsibilities
- `F-PR3-006`: x64/ARM64 CI matrix
- `F-PR3-007`: deterministic threshold/tie fixtures
- `F-PR3-008`: native backend traceability
- `F-PR3-009`: exact-oracle finite overflow
- `F-PR3-003`: withdrawn review erratum

これらの技術内容は本統合版へ織り込み済みである。

ただし、ファイル統合およびPhase 4設計追加により文書構成とreview対象HEADが変わるため、過去PASSを統合版へ自動継承しない。

統合版の次工程は、新しいPR current HEADを対象とした独立詳細設計reviewである。

---

## 74. 実装開始条件

### Phase 1

本統合版のreview完了後、次の論理単位で開始する。

1. solution/project/x64+ARM64 CI/diagnostic artifact
2. Interval construction/state/normalization
3. exact rational oracle + boundary corpus
4. directed add/sub
5. interval add/sub
6. directed mul
7. interval mul
8. directed div
9. interval div
10. Phase 1 conformance harness
11. inari/kv golden corpus
12. sample/API evaluation report

### Phase 4

Phase 4A以降は次をすべて満たした後に開始する。

- Phase 1～3の必要成果物完了
- Phase 2 basic API freeze完了
- 対象Phase 4 sectionのreview完了
- test/conformance/reference基盤と統合済み
- diagnostic artifact workflowが存在

各source実装はTDDで、失敗testを先にcommit/pushし、失敗を確認してからproduction implementationを追加する。
