# 区間演算 詳細設計

## 1. 文書情報

- 対象リポジトリ: `ssaattww/Devo6.Interval`
- 対象ライブラリ: `Devo6.Interval`
- 設計対象: IEEE 754 binary64 (`double`) を端点とする区間演算ライブラリ
- 基本設計: `doc/Design/basic/IntervalArithmetic.md`
- 初版: 2026-08-29
- 統合版: 2026-08-30
- 設計版: 5
- 状態: fix verification required

本書を `doc/Design/detail/` 配下の唯一の詳細設計書とする。

設計版5では、統合版に対する追加設計レビューの `F-PR3-010` ～ `F-PR3-017` を反映した。過去に分割されていたRevision、Phase 4 roadmap、set/numeric、math functions、advanced featuresの規範は、本書の本文へ統合済みである。

本書の主要な規範は次のとおりである。

- `Interval.Empty.Lower = +Infinity`
- `Interval.Empty.Upper = -Infinity`
- Emptyの内部表現はcanonical quiet NaN 2 lane
- 通常区間の内部表現は`[-Lower, Upper]`
- 有限endpointはtightな外向き丸め
- finite overflowはexact-rational oracleでInfinity operandと分離して判定
- IEEE 1788.1-oriented set semantics
- scalar reference、SIMD、native backend間でcanonical endpointを一致させる
- Phase 4の非連結結果、decoration、NaIをbare `Interval`へ混在させない

過去のreview PASSは、そのimmutable HEADに対する履歴情報である。本書の設計版5は、追加設計reviewのfindingを修正した新しいreview対象であり、fix verification完了前にPASSを継承しない。

---

## 2. 設計原則

### 2.1 仕様基準

意味論の基準はIEEE 1788.1-2017を中心とし、既存ライブラリとの互換性を重視する。

固定参照:

- `unageek/inari`
  - commit: `18b83a571d7681c76067bc38d90a74e8be29f545`
  - license: MIT
- `mskashi/kv`
  - commit: `c7f8f2324a0e403cca6b39f46088a22843d440db`
  - license: MIT
- `unageek/ITF1788`
  - commit: `d8c2a64478ebdc9cbde6ccef33eaad3bed60ed81`

結果判定の優先順位は次とする。

1. 数学的exact oracle
2. 採用したIEEE 1788.1-oriented semantics
3. `inari`の区間意味論・endpoint result
4. `kv`のcompatibleなdirected-rounding primitive

既存ライブラリ同士で差異がある場合は、単純にいずれかへ合わせず、入力domain、集合意味論、exact resultを調査する。

### 2.2 真値包含とtightness

すべての区間演算は真値を包含する。

四則演算および正式公開するtight数学関数では、有限endpointを指定方向へ正しく丸めた最も内側のbinary64とする。

```text
Lower: round toward -Infinity
Upper: round toward +Infinity
```

単に包含しているだけの不要に広い結果を、tight APIの正常結果として許可しない。

### 2.3 公開APIとbackendの分離

公開APIから次へ依存できないようにする。

- private field layout
- `Vector128<double>` / `Vector512<double>`等の物理格納型
- scalar / SIMD backend
- managed / native backend
- CPU feature
- MPFR等のreference implementation

Phase 2で基本APIを固定した後も、同じ意味論を維持して内部backendを差し替えられる構造とする。

### 2.4 暗黙の精度低下を禁止する

同じ公開methodが環境によってtight resultとvalid-but-wide resultを暗黙に切り替えない。

正しく丸められるproduction kernelを用意できない関数は公開を延期する。valid-only APIが必要になった場合は、tight APIとは別の名称・型・accuracy metadataを持つ別設計とする。

### 2.5 bare intervalと拡張情報の分離

```text
Interval           連結なbare interval
IntervalUnion2     最大2個の連結成分に対するtight closed enclosure
DecoratedInterval  Interval + Decoration、またはNaI
```

非連結結果、decoration、NaI、parser error、split metadataをbare `Interval`のflagとして混在させない。

---

## 3. 開発フェーズ

### 3.1 全体順序

1. **Phase 0**: 詳細設計・検証基盤
2. **Phase 1**: SIMDなしmanaged scalar四則演算パイロット
3. **Phase 2**: 基本`Interval` API確定
4. **Phase 3**: 同一意味論のSIMD backend
5. **Phase 4A**: 集合・関係・数値的属性・整数値関数
6. **Phase 4B**: 代数関数・区間定数
7. **Phase 4C**: 単調な初等関数
8. **Phase 4D**: 周期・特異点・多変数関数
9. **Phase 4E**: 非連結結果・decorated interval・I/O・分割

API・数値意味論・最適化・高度関数を分離して検証する。

### 3.2 Phase 1: managed scalarパイロット

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

### 3.3 Phase 2: 基本API確定

確定対象:

- package / assembly / namespace
- constructor/factory
-基本property
- Empty / Entire
- 四則演算operator
- equality / Hash
-例外
- signed zero
- diagnostic `ToString`
- scalar conversion/overloadの採否
- generic math interfaceの採否

完了後、基本`Interval` APIへの破壊的変更は原則禁止する。

### 3.4 Phase 3: SIMD backend

実装順:

1. scalar referenceとSIMD differential test基盤
2. SIMD load/store
3. batch add/sub
4. AVX-512 directed mul/div
5. AVX2+FMA、AVX2 without FMA、SSE2、ARM64を個別評価
6. correctnessとbenchmarkを両方通過した経路のみproduction dispatchへ採用

単一区間operatorを無条件に最大幅vectorへ載せない。

### 3.5 Phase 4A

- `Contains`
- `Intersect`, `ConvexHull`
- subset/interior/disjoint/precedes等のnamed relation
- `IntervalOverlap`
- `IsBounded`
- `Width`, `Midpoint`, `Radius`, `Magnitude`, `Mignitude`
- `Abs`, `Sign`
- pointwise min/max
- floor/ceiling/truncate/round

### 3.6 Phase 4B

- reciprocal
- square
- square root
- integer power
- integer root
- fused multiply-add
- tight interval constants

### 3.7 Phase 4C

- exp/exp2/exp10
- log/log2/log10
- sinh/cosh/tanh
- asinh/acosh/atanh
- asin/acos/atan

### 3.8 Phase 4D

- sin/cos/tan
- atan2
- positive-base general interval power
- high-precision periodic range reduction

### 3.9 Phase 4E

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

Phase 1は`net10.0`を対象とする。古いtarget frameworkはPhase 2以降で利用要件を確認して追加する。

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

`inari`, `kv`, ITF1788, MPFRは参照・test corpus・将来backend候補であり、Phase 1 production runtimeから呼び出さない。

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

有限`double`のみ受け入れる。`NaN`および`±Infinity`は実数の点ではないため拒否する。

### 5.5 定数/default

```text
Empty  = empty set
Entire = [-Infinity,+Infinity]
Zero   = [-0.0,+0.0]
```

`default(Interval) == Interval.Zero`を契約とする。

### 5.6 Empty endpoint

内部Emptyはcanonical NaNで識別し、公開endpointはIEEE-oriented semanticsへ合わせる。

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

scalar overload/conversion、`INumber<TSelf>`はPhase 2で判断する。区間には自然な全順序がないため、通常数値型の契約を無条件に適用しない。

### 5.10 `ToString`

Phase 1ではdiagnostic用途とする。永続化・wire契約にはしない。

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

演算結果用にvalidationを省略するinternal/private constructorを持つ。呼出側がcanonical stateを保証する。

### 6.5 layout非公開

size、field順序、blittable ABI、raw byte representation、`Vector128<double>` castをpublic contractにしない。

---

## 7. 方向付き丸め共通契約

### 7.1 primitive

```text
AddUp(x,y)        = min binary64 z such that exact(x+y) <= z
AddDown(x,y)      = max binary64 z such that z <= exact(x+y)
MultiplyUp(x,y)   = min binary64 z such that exact(x*y) <= z
MultiplyDown(x,y) = max binary64 z such that z <= exact(x*y)
DivideUp(x,y)     = min binary64 z such that exact(x/y) <= z
DivideDown(x,y)   = max binary64 z such that z <= exact(x/y)
```

`NextUp` / `NextDown`には`Math.BitIncrement` / `Math.BitDecrement`を使用し、通常演算後に無条件で1 ULP広げない。

### 7.2 symmetry

```text
AddDown(x,y)      = -AddUp(-x,-y)
SubtractDown(x,y) = -SubtractUp(y,x)
MultiplyDown(x,y) = -MultiplyUp(-x,y)
DivideDown(x,y)   = -DivideUp(-x,y)
```

### 7.3 primitiveへ渡さないundefined pair

```text
+Infinity + -Infinity
0 * Infinity
0 / 0
Infinity / Infinity
denominator == 0
NaN operand
```

区間kernelが先に処理する。

### 7.4 finite overflow

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

## 8. 加算・減算の方向付き丸め

有限operandかつoverflowなしではTwoSumを用いる。

```text
s = roundNearest(x+y)
e = exact(x+y)-s
```

```csharp
static (double Sum, double Error) TwoSum(double x, double y)
{
    double s = x + y;
    double bv = s - x;
    double e = (x - (s - bv)) + (y - bv);
    return (s, e);
}
```

補正:

```text
AddUp:   e > 0 -> NextUp(s),   else s
AddDown: e < 0 -> NextDown(s), else s
```

減算:

```text
SubtractUp(x,y)   = AddUp(x,-y)
SubtractDown(x,y) = AddDown(x,-y)
```

---

## 9. 乗算の方向付き丸め

### 9.1 定数

```text
SmallProductThreshold = 2^-969
ProductScale          = 2^537
```

`abs(product) >= 2^-969`は通常残差経路、`abs(product) < 2^-969`はscaled経路。

### 9.2 FMA residual

```text
product = x*y
error   = FMA(x,y,-product)
```

通常経路:

```text
Up:   error > 0 -> NextUp(product)
Down: error < 0 -> NextDown(product)
```

### 9.3 scaled path

```text
sx = x*2^537
sy = y*2^537
(s,s2) = exact-product-decomposition(sx,sy)
t = (product*2^537)*2^537
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

### 9.4 overflow

```text
nearest +Infinity:
  Up   -> +Infinity
  Down -> +MaxValue

nearest -Infinity:
  Up   -> -MaxValue
  Down -> -Infinity
```

Infinity operandはexact Infinityとして別分岐する。

---

## 10. 除算の方向付き丸め

### 10.1 定数

```text
SmallNumeratorThreshold = 2^-969
LargeDenominatorLimit   = 2^918
DivisionScale           = 2^105
MinimumSubnormal        = 2^-1074
```

### 10.2 denominator正符号化

```text
if y < 0:
    xn = -x
    yn = -y
else:
    xn = x
    yn = y
```

`yn > 0`として比較する。

### 10.3 small numerator

```text
abs(xn) < 2^-969:
    if abs(yn) < 2^918:
        xn *= 2^105
        yn *= 2^105
    else:
        early return
```

`abs(xn)==2^-969`は通常経路、`abs(yn)==2^918`はearly-return側。

large-denominator early return:

| exact quotient sign | Up | Down |
|---|---|---|
| positive | `+2^-1074` | `+0.0` |
| negative | `+0.0` | `-2^-1074` |

### 10.4 normal residual comparison

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

rounded high partが等しくてもresidualを必ず確認する。

---

## 11. 四則演算の区間kernel

全演算でoperandのいずれかがEmptyならEmpty。

### 11.1 add/sub/unary minus

```text
[a,b] + [c,d] = [RD(a+c), RU(b+d)]
[a,b] - [c,d] = [RD(a-d), RU(b-c)]
-[a,b]         = [-b,-a]
```

内部加算:

```text
[-a,b] + [-c,d]
= [-RD(a+c), RU(b+d)]
```

減算は右operandのlane swap、単項マイナスはlane swapを利用できる。

### 11.2 sign class

```text
Z: [0,0]
P: 0 <= lower, Zではない
N: upper <= 0, Zではない
M: lower < 0 < upper
```

### 11.3 multiplication table

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

### 11.4 division: denominator excludes zero

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

reciprocalを一度作って乗算する方式は二重丸めを避けるため採用しない。

### 11.5 denominator Zero

```text
A/[0,0] -> Empty
```

### 11.6 one-sided zero denominator

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

### 11.7 denominator crosses zero

```text
c < 0 < d
Zero/B -> Zero
otherwise -> Entire
```

真の像が非連結でも、bare `Interval`はconvex hullを返す。

---

## 12. 等値性・Hash・順序

### 12.1 equality

- Empty同士は等しい。
- 非空はcanonical endpointが等しい場合に等しい。
- `+0.0/-0.0`入力差は影響しない。
- NaIはbare `Interval`に存在しない。

### 12.2 Hash

- Emptyは固定Hash。
- zero endpointはcanonical bit pattern。
- internal NaN payloadを直接Hashしない。

### 12.3 normal comparison operator

`<`, `<=`, `>`, `>=`は基本`Interval`へ定義しない。subset、endpoint-wise less、precedes等をPhase 4Aのnamed APIとして提供する。

---

## 13. Exact oracle・reference harness

### 13.1 exact-rational oracle

有限binary64をexactに`significand * 2^exponent`へ分解する。

- add/sub/mul: `BigInteger`でexact value
- div: rational numerator/denominator
- production packageへ`BigInteger` oracleを含めない

### 13.2 finite overflow変換

`R`をfinite exact rational、`M`を`double.MaxValue`のexact rationalとする。BCL nearest resultへ変換する前に必ず比較する。

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
Up:   N < R -> BitIncrement(n), else n
Down: N > R -> BitDecrement(n), else n
```

finite overflowとInfinity operandによるexact Infinityを別caseとして記録する。

### 13.3 inari / kv / MPFR

`inari`:

- interval semantics
- Empty/Entire
- zero-crossing
- set/relation/numeric/elementary reference

`kv`:

- `add_up/down`
- `mul_up/down`
- `div_up/down`
- `sqrt_up/down`

kvのzero-containing interval divisionはDevo6/inariと意味論が異なるため、kv interval resultをoracleにしない。

MPFR:

- Phase 4 elementary endpointのprimary directed reference corpus
- fixed version / RNDD / RNDU

### 13.4 reference lock/corpus

`tests/ReferenceData/reference-lock.json`に次を記録する。

- inari SHA
- kv SHA
- ITF1788 SHA
- MPFR version
- adapter/generator hash
- toolchain / target triple
- generator command
- corpus SHA-256
- license/NOTICE path

corpusはJSON Lines、数値は16桁hex binary64 bitsとする。caseIdでsortしgenerator iteration orderへ依存しない。

---

## 14. IEEE 1788.1-oriented conformance

### 14.1 Phase 1 matrix

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

### 14.2 constructor source

固定ITF1788 commit `d8c2a64478ebdc9cbde6ccef33eaad3bed60ed81`から:

- `itl/libieeep1788_class.itl`のbare `b-numsToInterval`
- `itl/ieee1788-constructors.itl`のcompatible bare numeric supplement

を使用する。

最低限:

```text
(-1,1)            -> [-1,1]
(-Inf,1)          -> [-Inf,1]
(-1,+Inf)         -> [-1,+Inf]
(-Inf,+Inf)       -> Entire
(NaN,NaN)         -> invalid
(1,-1)            -> invalid
(-Inf,-Inf)       -> invalid
(+Inf,+Inf)       -> invalid
```

invalid case:

- constructor -> `ArgumentException`
- TryCreate -> false, out=Empty

### 14.3 IsSingleton repository matrix

固定ITF1788に適切な`isSingleton` corpusが存在しないためrepository-defined matrixを使用する。

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

### 14.4 Empty inf/sup

```text
inf(Empty) = +Infinity
sup(Empty) = -Infinity
```

C# mappingは`Lower` / `Upper`。

### 14.5 manifest/gate

caseごとにexternal/repository-defined source、path/testcase、adaptation、required/deferred/excluded/approved-deviation、expectedを保存する。

sourceから宣言operationが0件だった場合、黙ってpassせずsource extraction errorとする。

---

## 15. 四則演算決定的fixture

### 15.1 threshold

```text
2^-969 previous = 0x035fffffffffffff
2^-969          = 0x0360000000000000
2^-969 next     = 0x0360000000000001

2^918 previous  = 0x794fffffffffffff
2^918           = 0x7950000000000000
2^918 next      = 0x7950000000000001

2^-1074         = 0x0000000000000001
```

### 15.2 residual / overflow

multiplication scaled pathで`t<s`, `t>s`, `t==s && s2>0`, `t==s && s2<0`, exactを固定binary64 witnessで試験する。

divisionで`r==xn && r2>0`, `r==xn && r2<0`, `<`, `>`, exactを試験する。

add/sub/mul/divのpositive/negative finite overflowをexact oracleのfinite-overflow pathへ通す。

---

## 16. CI・failure diagnostics

### 16.1 workflow追加時期

現在repositoryに実行可能projectがないため設計PRではworkflowを追加しない。

Phase 1でproject/testを追加する最初のPRに、診断artifact workflowを同時追加する。

### 16.2 architecture matrix

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

### 16.3 architecture間比較

各jobが`canonical-results.jsonl`をcaseId順で生成し、後続jobでbyte-for-byte比較、SHA-256、全差分を保存する。

### 16.4 artifact

成功/失敗にかかわらず保存する。

- test result
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
- expected-difference reason

Phase 4ではさらにfunction/domain/sign/quadrant、clipped domain、endpoint backend、correction decision、periodic reduction、critical point/pole/branch cut、parser exact rational/resource limit、union/decoration/split stateを保存する。

### 16.5 exact-head CI

CI確認対象は確認時点のPR current HEAD SHAとrunの`head_sha`が一致するrunだけとする。

HEAD更新後は新HEADを再確認する。matching runがなければ**CI未実施**。別SHAのrunを代用しない。

---

## 17. SIMD設計

### 17.1 capability独立判定

```text
Avx512F.IsSupported
Avx2.IsSupported
Avx.IsSupported
Fma.IsSupported
Sse2.IsSupported
AdvSimd.Arm64.IsSupported
```

FMAをAVX2/SSE2へ暗黙従属させない。

### 17.2 capability matrix

| Environment | Add/Sub | Mul/Div | initial policy |
|---|---|---|---|
| AVX-512F | packed directed | packed directed | 4 interval batch candidate |
| AVX2+FMA | vector TwoSum | vector FMA residual candidate | correctness+benchmark後 |
| AVX2 no FMA | vector TwoSum | scalar fallback | mul/div vector Dekkerは別評価 |
| AVX+FMA | candidate | candidate | benchmark後 |
| SSE2 no FMA | Vector128 TwoSum | scalar fallback | add/sub候補 |
| ARM64 AdvSimd | vector candidate | scalar until exactness proven | differential後 |
| other | scalar | scalar | always |

### 17.3 AVX-512 batch layout

```text
[-L0,U0,-L1,U1,-L2,U2,-L3,U3]
```

上向き丸め付きpacked operationで4区間を処理する。末尾4未満はscalar。

### 17.4 production採用条件

- scalarとcanonical bitwise equivalent
- fallback動作
- special/subnormal differential成功
- feature combination test成功
- benchmark上の改善

改善がないkernelはproduction dispatchへ入れない。

---

## 18. Native backend判断

Phase 1、2、Phase 3初期はmanaged-only。scalar operatorごとのP/Invokeは採用しない。

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

## 19. Phase 4共通アーキテクチャ

### 19.1 公開型責務

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

- 最大2個の連結成分に対するtight closed enclosure

`DecoratedInterval`:

- interval + decoration / NaI

`IntervalContractor`:

- reverse/cancellative operations

### 19.2 math function layering

```text
public interval extension
  Empty propagation
  domain clipping
  monotonic/sign/quadrant classification
  extrema/pole/periodic point detection
  connected/component-enclosure result construction
            ↓
certified scalar endpoint kernel
  FooDown(double)
  FooUp(double)
  constants/range reduction
  overflow/underflow/subnormal
  correct directed binary64 rounding
```

### 19.3 accuracy class

- Exact: relation/set operation等
- Tight directed algebraic: square/sqrt/integer power等
- Tight certified elementary: exp/log/sin等

`Math.Sin`等に証明なしで固定ULPを加える方式を正式backendにしない。

---

## 20. Phase 4A 公開API候補

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

名称はPhase 4A API reviewで確定する。

---

## 21. 集合演算・Contains

`X=[a,b]`, `Y=[c,d]`。

### 21.1 Intersect

```text
Empty ∩ Y = Empty
X ∩ Empty = Empty
X ∩ Y = [max(a,c),min(b,d)]
```

下限>上限ならEmpty。内部ではlane-wise minimumを利用できる。

### 21.2 ConvexHull

```text
hull(Empty,Y) = Y
hull(X,Empty) = X
hull(X,Y) = [min(a,c),max(b,d)]
```

内部ではlane-wise maximumを利用できる。

### 21.3 Contains

`±Infinity`は非有界端点であって実数要素ではない。

```text
Contains(X,x)
= !X.IsEmpty
  && IsFinite(x)
  && Lower <= x <= Upper
```

`Entire.Contains(±Infinity)`および`Contains(NaN)`はfalse、Zeroは両signed zeroを含む。

---

## 22. Relation

### 22.1 extended strict relation `<′`

```text
x <′ y iff
    x < y
 or x == y == -Infinity
 or x == y == +Infinity
```

### 22.2 subset/interior/disjoint/precedes

```text
Empty subset Y = true
nonempty subset Empty = false
[a,b] subset [c,d] iff c<=a && b<=d
```

```text
Empty interior Y = true
nonempty interior Empty = false
[a,b] interior [c,d] iff c <′ a && b <′ d
```

`Entire.IsInteriorOf(Entire)`はtrue。

```text
Empty disjoint Y = true
[a,b] disjoint [c,d] iff b<c || d<a
```

端点接触はdisjointではない。

```text
Empty involved -> Precedes/StrictlyPrecedes = true
[a,b] precedes [c,d] iff b<=c
strict precedes iff b<c
```

### 22.3 endpoint-wise less

Weak:

```text
Empty vs Empty    -> true
Empty vs nonempty -> false
nonempty vs Empty -> false
[a,b] <=weak [c,d] iff a<=c && b<=d
```

Strict:

```text
Empty vs Empty    -> true
Empty vs nonempty -> false
nonempty vs Empty -> false
[a,b] <strict [c,d] iff a<′c && b<′d
```

subsetとは別概念であり、通常比較演算子へ割り当てない。

---

## 23. IntervalOverlap

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
BothEmpty <-> BothEmpty
FirstEmpty <-> SecondEmpty
Before <-> After
Meets <-> MetBy
Overlaps <-> OverlappedBy
Starts <-> StartedBy
ContainedBy <-> Contains
Finishes <-> FinishedBy
Equals <-> Equals
```

16状態を最低1fixtureずつ持ち、全状態でinverse consistencyを検証する。

---

## 24. 数値的属性・基本関数

### 24.1 IsBounded / Width

```text
IsBounded = !IsEmpty && IsFinite(Lower) && IsFinite(Upper)
```

```text
Width:
  Empty -> NaN
  [a,b] -> RU(b-a)
  unbounded -> +Infinity
  singleton -> +0
```

内部では`AddUp(_upper,_negatedLower)`を利用できる。

### 24.2 Midpoint / Radius

Midpointは代表scalarでありenclosure endpointではない。

```text
Empty -> NaN
Entire -> +0.0
[-Infinity,b] -> double.MinValue
[a,+Infinity] -> double.MaxValue
finite -> exact (a+b)/2 の採用丸め規則によるbinary64
```

finite midpointのtie policyはPhase 4A API/conformance reviewで最終確認する。

Radiusは`m=Midpoint`として最小binary64 `r`で`X subset [m-r,m+r]`を満たす値とし、`max(SubtractUp(m,Lower), SubtractUp(Upper,m))`で求める。

### 24.3 Magnitude / Mignitude

```text
Magnitude:
  Empty -> NaN
  max(abs(a),abs(b))

Mignitude:
  Empty -> NaN
  0 in X -> +0
  b<0 -> abs(b)
  a>0 -> abs(a)
```

### 24.4 Abs / Sign / Pointwise MinMax

```text
Abs:
  Empty -> Empty
  0<=a -> [a,b]
  b<=0 -> [-b,-a]
  a<0<b -> [-0,max(-a,b)]
```

`Sign`のpoint functionは`-1/0/+1`。zeroのsignは0であり、IEEE signed-zeroのbitを`±1`へ変換しない。

```text
PointwiseMin(X,Y)=[min(a,c),min(b,d)]
PointwiseMax(X,Y)=[max(a,c),max(b,d)]
```

どちらかEmptyならEmpty。

### 24.5 integer-valued functions

Floor、Ceiling、Truncate、Roundは単調非減少なのでendpointへ同じpoint functionを適用する。

- Empty -> Empty
- Infinity endpoint維持
- 結果は整数binary64なので追加outward rounding不要
- zeroはcanonical endpointへ正規化
-未知`MidpointRounding` enum -> `ArgumentOutOfRangeException`

---

## 25. Phase 4B API・certified endpoint kernel

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

math functionはinterval extension層とdirected scalar function層を分離する。

正式endpoint backendは、全binary64入力に対する誤差上限・方向補正を証明したmanaged実装、correctly-rounded implementationの検証済み移植、またはMPFR等のdirected backendのみとする。

BCL `Math.*`はcandidate seedには使えるが、単体で包含保証の根拠にしない。

---

## 26. 区間定数・Reciprocal・Square

### 26.1 IntervalConstants

`π`等をnearest doubleの点区間として偽装しない。

```text
Pi.Lower <= π <= Pi.Upper
```

MPFR directed conversionでendpoint bitsを生成してrepositoryへ固定し、build時にnetwork/native generatorを要求しない。

三角関数range reduction用にはpublic 2-endpoint constantとは別に、高精度`2/π`, `π/2` tableを使用する。

### 26.2 Reciprocal

| X class | result |
|---|---|
| Empty | Empty |
| Zero | Empty |
| `a<0<b` | Entire |
| `a<0,b=0` | `[-Infinity,RU(1/a)]` |
| `a=0,b>0` | `[RD(1/b),+Infinity]` |
| b<0 | `[RD(1/b),RU(1/a)]` |
| a>0 | `[RD(1/b),RU(1/a)]` |

2成分を保持するversionはPhase 4E。

### 26.3 Square

`X*X`へ委譲せず依存性問題による拡大を避ける。

```text
Empty -> Empty
0<=a -> [RD(a*a),RU(b*b)]
b<=0 -> [RD(b*b),RU(a*a)]
a<0<b -> [-0,max(RU(a*a),RU(b*b))]
```

---

## 27. Square Root・Integer Power/Root・FMA

### 27.1 Square Root

Domain `[0,+Infinity)`。

```text
Empty -> Empty
b<0 -> Empty
a<=0<=b -> [-0,SqrtUp(b)]
0<a -> [SqrtDown(a),SqrtUp(b)]
```

`kv` no-hardware-roundingを参照し:

```text
SmallSqrtInputThreshold = 2^-969
SqrtInputScale          = 2^106
SqrtResultScale         = 2^53
```

candidate `r=roundNearest(sqrt(x))`について、通常域では`r*r`、small域ではscale後の`(r*2^53)^2`をexact-product decompositionして入力と比較し、必要な場合だけNextUp/NextDownする。

### 27.2 Integer Power

```csharp
IntervalMath.Pow(Interval value, int exponent)
```

```text
Pow(Empty,0)=Empty
Pow(nonempty,0)=[1,1]
```

positive oddは単調増加、positive evenは`Abs(X)`を用いる。

negative exponent:

- Zeroのみ -> Empty
- even -> Absを用い、zero接触時upperは+Infinity
- odd strict zero crossing -> Entire
- odd `[0,b]` -> `[RD(b^n),+Infinity]`
- odd `[a,0]` -> `[-Infinity,RU(a^n)]`

`int.MinValue`の絶対値を`int`で取らず、符号+`uint` magnitudeへ分解する。

### 27.3 Integer Root

```csharp
IntervalMath.Root(Interval value, int degree)
```

- `degree<=0` -> `ArgumentOutOfRangeException`
- `degree=1` -> input
- Empty -> Empty
- odd degree -> 全実数上単調増加
- even degree -> negative-onlyはEmpty、zero crossingではlower=0

Newton candidateだけで確定せず、candidate^nと入力のexact relationを検証して隣接補正する。

### 27.4 FusedMultiplyAdd

```text
FMA(X,Y,Z)=hull({x*y+z})
```

endpoint primitiveは`RD/RU(exact(x*y+z))`を1回だけ丸める。

`(X*Y)+Z`へ委譲しない。積と加算の二重丸めによる不要な拡大を避ける。

---

## 28. Phase 4C/D APIとelementary reference

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

    public static Interval Pow(Interval value, Interval exponent);
}
```

primary reference corpusは固定MPFR versionへbinary64をexact入力し、53-bit precisionでRNDD/RNDUを指定して生成する。pinned `inari`もMPFR RNDD/RNDUを使うためsecondary oracleとする。

---

## 29. 単調関数・domain boundary

Domainとのintersectionを`[l,u]`とする。

単調増加:

```text
[FDown(l),FUp(u)]
```

単調減少:

```text
[FDown(u),FUp(l)]
```

open boundaryへ接する場合はlimit値を使用する。境界点しか含まずdomain内点がない場合はEmpty。

### 29.1 Exp

`Exp`, `Exp2`, `Exp10`は全実数上単調増加。

- Empty -> Empty
- `-Infinity` -> +0 limit
- `+Infinity` -> +Infinity
- finite overflow/underflowをtightに処理

### 29.2 Log

`Log`, `Log2`, `Log10` domain `(0,+Infinity)`。

```text
b<=0 -> Empty
otherwise:
  lower = -Infinity if a<=0 else LogDown(a)
  upper = +Infinity if b==+Infinity else LogUp(b)
```

### 29.3 Hyperbolic / inverse

Sinh、Tanh、Asinh、Atanは単調増加。

Cosh:

```text
b<0 -> [CoshDown(b),CoshUp(a)]
a>0 -> [CoshDown(a),CoshUp(b)]
a<=0<=b -> [1,CoshUp(max(-a,b))]
```

Acosh domain `[1,+Infinity)`、Asin/Acos domain `[-1,1]`、Atanh domain `(-1,1)`をdomain clippingして評価する。

Atanhは境界へ接する場合`±Infinity`のlimitを使用し、`[-1,-1]`と`[1,1]`はEmpty。

---

## 30. Periodic range reduction・Sin/Cos/Tan

Sin/Cos/Tanは全binary64範囲で象限・極を誤らない内部reducerを持つ。候補はPayne-Hanek型fixed-point reduction。

禁止:

- `Math.PI`との通常除算だけでquadrant決定
- `value % (2*Math.PI)`だけでcritical point決定
- public Pi区間の一端だけで整数kを決定

内部に十分なbit数の`2/π`, `π/2` tableを固定し、`ContainsPeriodicPoint(X,offset,period)`相当のexact判定を持つ。

### 30.1 Sin

endpoint候補に加え:

- `-π/2+2kπ`を含む -> exact lower `-1`
- `+π/2+2kπ`を含む -> exact upper `+1`

非有界またはmax/min critical lattice双方を含む場合`[-1,1]`。

### 30.2 Cos

- `π+2kπ`を含む -> lower `-1`
- `2kπ`を含む -> upper `+1`

非有界/十分広い場合`[-1,1]`。

### 30.3 Tan

Domain `R \ {π/2+kπ}`。

poleなしの1branch内では`[TanDown(a),TanUp(b)]`。

poleへdomain内から近づける場合bare hullはEntire。poleしか含まずdomain内点なしならEmpty。

---

## 31. Atan2

API orderは.NETに合わせる。

```csharp
IntervalMath.Atan2(y,x)
```

Domain `R² \ {(0,0)}`、principal range `(-π,π]`。

### 31.1 共通rectangle algorithm

1. いずれかEmpty -> Empty
2. `X=Zero && Y=Zero` -> Empty
3. rectangleをzero境界でsign cellへ分割
4. originをdomainから除外
5. quadrantごとのmonotonic cornerを`Atan2Down/Up`で評価
6. axis候補として`0`, `±π/2`, `+π`を追加
7. negative-x branch cutの接触・crossingは§31.2のmatrixで上書きする
8. angular rangeを`AngleArcAccumulator`で統合する

### 31.2 negative-x branch-cut matrix

`X=[a,b]`がstrictly negative、すなわち`b<0`である場合、`Y=[c,d]`を次の6classへ分類する。

| Y class | condition | bare result rule |
|---|---|---|
| strictly negative | `d<0` | QIII corner enclosure |
| nonpositive touching zero | `c<0 && d==0` | `[-π,+π]` tight closed enclosure |
| Zero | `c==0 && d==0` | `IntervalConstants.Pi` |
| nonnegative touching zero | `c==0 && d>0` | QII lower corner ～ `+π` |
| strictly positive | `c>0` | QII corner enclosure |
| crossing zero | `c<0 && d>0` | `[-π,+π]` tight closed enclosure |

binary64 endpointとして`[-π,+π]`を返す場合は、tight Pi intervalを用いて概念的に次とする。

```text
lower = -IntervalConstants.Pi.Upper
upper = +IntervalConstants.Pi.Upper
```

strictly negative QIII:

```text
lower = Atan2Down(d, a)
upper = Atan2Up(c, b)
```

nonnegative touching zero QII:

```text
lower = Atan2Down(d, b)
upper = IntervalConstants.Pi.Upper
```

strictly positive QII:

```text
lower = Atan2Down(d, b)
upper = Atan2Up(c, a)
```

unbounded endpointでは同じ単調性に基づくlimitをdirected endpoint evaluatorが返す。

### 31.3 signed-zero rule

`+0.0`と`-0.0`は区間集合として同じ実数0である。canonical `Y=Zero`をbranch cutの上下2点として扱わない。

本ライブラリのpoint semanticsでは`x<0, y=0`をprincipal value `+π`として扱う。そのため:

```text
x<0, y=[negative,0] -> [-π,+π]
x<0, y=Zero         -> Pi
x<0, y=[0,positive] -> second-quadrant range ending +π
x<0, y crosses zero -> [-π,+π]
```

を明示的に区別する。

### 31.4 fixture

少なくとも次を固定する。

```text
x=[-2,-1], y=[-1,0] -> [-π,+π]
x=[-2,-1], y=Zero   -> Pi
x=[-2,-1], y=[0,1]  -> QII..π
x=[-2,-1], y=[-1,1] -> [-π,+π]
```

全sign-class直積、axis、origin、negative-x branch cutを固定matrixで検証する。

---

## 32. General Positive-Base Power

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

baseを`[0,+Infinity]`へclipし`X=[a,b]`, `Y=[c,d]`とする。

### 32.1 scalar kernel precondition

`PowDown/Up(x,y)`の通常point kernelは`x>0`だけを受ける。

次を通常point functionとしてkernelへ渡さない。

```text
0^0
0^negative
```

zero-base境界はinterval extension層でclosure candidateとして処理する。

### 32.2 zero-boundary closure candidate

rectangle extremaを求めるとき、次を明示的に候補へ加える。

```text
x -> 0+, y < 0 : +Infinity
x > 0,  y = 0 : 1
x = 0,  y > 0 : 0
```

ここで`1`は`0^0`のpoint valueではなく、`x>0, y=0`に由来するrectangle closure/value candidateである。`+Infinity`も`0^negative`のpoint valueではなく、`x->0+`のlimitである。

### 32.3 strictly positive base `a>0`

この場合は全cornerがpoint-domain内なので従来のmonotonic formulaを使用できる。

`d<=0`:

```text
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
lower=min(PowDown(a,d),PowDown(b,c))
upper=max(PowUp(a,c),PowUp(b,d))
```

### 32.4 zero-only base `b==0`

```text
d > 0  -> Zero
d <= 0 -> Empty
```

exponent intervalにpositive値があれば有効なpairは`0^positive=0`だけである。

### 32.5 zero-touching base `a==0 && b>0`

#### exponent strictly negative: `d<0`

```text
lower = if b<1 then PowDown(b,d)
        else        PowDown(b,c)
upper = +Infinity
```

`b==1`ではlower=1。`b=+Infinity`の場合もdirected powerのlimit規則を用いる。

#### exponent nonpositive and touches zero: `c<0 && d==0`

```text
lower = if b<=1 then 1
        else        PowDown(b,c)
upper = +Infinity
```

#### zero exponent only: `c==0 && d==0`

```text
[1,1]
```

`x=0`はdomain外だが、`x>0`の全点で`x^0=1`である。

#### exponent crosses zero and has negative part: `c<0 && d>0`

```text
[0,+Infinity]
```

positive exponentとbase zeroから0、negative exponentと`x->0+`から+Infinityを得る。

#### nonnegative exponent touching zero: `c==0 && d>0`

```text
lower = 0
upper = if b<=1 then 1
        else        PowUp(b,d)
```

#### strictly positive exponent: `c>0`

```text
lower = 0
upper = if b<1 then PowUp(b,c)
        else if b==1 then 1
        else PowUp(b,d)
```

### 32.6 required fixtures

```text
Pow([0,0.5],[0,1])  -> [0,1]
Pow([0,0.5],[-1,0]) -> [1,+Infinity]
Pow([0,2],[0,1])    -> [0,2]
Pow([0,0],[-1,0])   -> Empty
Pow([0,0],[0,1])    -> Zero
```

fixtureでは`PowDown/Up(0,0)`および`PowDown/Up(0,negative)`が呼ばれていないことをinternal hookで確認する。

---

## 33. Elementary production backend gate

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

## 34. IntervalUnion2

### 34.1 semantic model

`IntervalUnion2`は**数学的なopen/closed endpoint topologyを完全に表すexact set型ではない**。

最大2個の連結成分を持つ数学的結果について、各成分の**tight closed `Interval` enclosure**を保持する型とする。

そのため、exact resultが

```text
(-Infinity,0) union (0,+Infinity)
```

である場合、component enclosureは次となる。

```text
[-Infinity,-0.0]
[+0.0,+Infinity]
```

両closed enclosureは0で接するが、これはexact resultに0が含まれることを意味しない。型は各connected componentのclosure enclosureを保持している。

初版ではこの曖昧性を避けるため`IntervalUnion2.Contains(double)`を公開しない。将来membership APIが必要な場合は`MayContain`等のenclosure semanticsを明示するか、open-boundary metadataを持つ別型を設計する。

### 34.2 API候補

```csharp
public readonly struct IntervalUnion2 : IEquatable<IntervalUnion2>
{
    public int Count { get; }
    public bool IsEmpty { get; }

    public Interval First { get; }
    public Interval Second { get; }
    public Interval this[int index] { get; }

    public Interval ConvexHull { get; }

    public bool Equals(IntervalUnion2 other);
    public override bool Equals(object? obj);
    public override int GetHashCode();

    public static bool operator ==(
        IntervalUnion2 left,
        IntervalUnion2 right);

    public static bool operator !=(
        IntervalUnion2 left,
        IntervalUnion2 right);
}
```

### 34.3 canonical state

```text
Count=0:
  First=Empty
  Second=Empty

Count=1:
  First=nonempty
  Second=Empty

Count=2:
  First/Second nonempty
  First.Lower <= First.Upper
  Second.Lower <= Second.Upper
  First.Upper <= Second.Lower
```

**`First.Upper == Second.Lower`を理由にmergeしてはならない。**

等しい境界は、0を除外したtwo-output divisionのように、distinct connected componentsのclosureが同じ境界へ接するcaseを表し得る。

### 34.4 construction

初版public constructorは提供しない。

internal:

```text
Create0()
Create1(interval)
Create2(first,second, componentsKnownDistinct=true)
```

`Create2`はEmpty除去、順序正規化、signed-zero正規化を行う。

precondition:

```text
First.Upper <= Second.Lower
```

strict overlap `First.Upper > Second.Lower`は、2成分enclosureの生成契約違反である。黙ってmergeせずDebug assertion / internal validation failureとする。数学的結果がconnectedであることをkernelが知っている場合は、callerが最初から`Create1(ConvexHull)`を選択する。

### 34.5 default/accessor

`default(IntervalUnion2)`はCount=0のempty union。

- Count=0: First/SecondはいずれもEmpty
- Count=1: Firstはcomponent、SecondはEmpty
- Count=2:両方component
- indexerは`0 <= index < Count`だけ有効
- invalid indexは`ArgumentOutOfRangeException`

### 34.6 equality/hash

value equalityはCountとcanonical component列を比較する。

- Count=0同士は等しい。
- Count=1はFirstを比較。
- Count=2はFirst/Secondを順序どおり比較。
- `Interval`側のcanonical signed-zero/equalityをそのまま利用する。

HashはCountと有効componentの`Interval.GetHashCode()`を結合し、無効なSecond fieldやinternal NaN payloadへ依存しない。

### 34.7 required zero-touch fixtures

```text
DivideToUnion([1,2], Entire)
  -> Count=2
     [-Infinity,-0.0]
     [+0.0,+Infinity]

ReciprocalToUnion(Entire)
  -> same two component enclosures

ReverseMultiply([1,2], Entire)
  -> same two component enclosures
```

これらをCount=1/Entireへmergeしない。

---

## 35. Extended Division

```csharp
public static IntervalUnion2 DivideToUnion(
    Interval numerator,
    Interval denominator);

public static IntervalUnion2 ReciprocalToUnion(
    Interval value);
```

semanticsの基礎集合:

```text
{x/y | x in X, y in Y, y != 0}
```

戻り値はそのconnected componentごとのtight closed enclosureである。

### 35.1 common

```text
X Empty or Y Empty -> Count0
Y Zero -> Count0
Y excludes zero -> Count1(X/Y)
X Zero and Y has nonzero member -> Count1(Zero)
```

### 35.2 one-sided zero denominator

`Y=[0,d]`, `d>0`:

| X class | Result |
|---|---|
| Z | Count1 Zero |
| P | Count1 `[RD(a/d),+Infinity]` |
| N | Count1 `[-Infinity,RU(b/d)]` |
| M | Count1 Entire |

`Y=[c,0]`, `c<0`:

| X class | Result |
|---|---|
| Z | Count1 Zero |
| P | Count1 `[-Infinity,RU(a/c)]` |
| N | Count1 `[RD(b/c),+Infinity]` |
| M | Count1 Entire |

ここでP/NはZeroを除くnonnegative/nonpositive classであり、`a==0`または`b==0`を許す。

### 35.3 strict zero-crossing denominator

`Y=[c,d]`, `c<0<d`。

strict positive `X=[a,b]`, `a>0`:

```text
First  = [-Infinity,RU(a/c)]
Second = [RD(a/d),+Infinity]
```

strict negative `X=[a,b]`, `b<0`:

```text
First  = [-Infinity,RU(b/d)]
Second = [RD(b/c),+Infinity]
```

numerator contains zero:

```text
X==Zero -> Count1 Zero
otherwise -> Count1 Entire
```

strict positive/negativeかつdenominator=Entireでは、2 component enclosureが0で接することを許す。

### 35.4 ordinary divisionとの関係

すべてのcaseで:

```text
X/Y == DivideToUnion(X,Y).ConvexHull
```

を固定propertyとする。

`ReciprocalToUnion(X)`は同じkernelへnumerator=Oneを渡す専用entryとし、不要なpublic temporaryを作らなくてよい。

---

## 36. Reverse Multiplication

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

通常divisionとの違いはfactor=0を除外しない点。

```text
product Empty or factor Empty -> empty union
0 in product && 0 in factor -> Count1 Entire
otherwise -> DivideToUnion(product,factor)
```

例:

```text
ReverseMultiply([1,2],Zero)=empty
ReverseMultiply(Zero,Zero)=Entire
ReverseMultiply([0,2],Zero)=Entire
ReverseMultiply([1,2],Entire)=two zero-touch component enclosures
```

---

## 37. Cancellative Operations

候補:

```csharp
IntervalContractor.CancelSubtract(total,term)
IntervalContractor.CancelAdd(total,term)
```

通常のsubtraction/additionと異なるstandard-style cancellative operationであり、定義されたprecondition外ではEntireを返す。

`total`のclassをrow、`term`のclassをcolumnとする。

| total \ term | Empty | bounded/common | unbounded |
|---|---|---|---|
| Empty | Empty | Empty | Entire |
| bounded/common | Entire | width条件 | Entire |
| unbounded | Entire | Entire | Entire |

bounded/common同士:

```text
exact width(total) >= exact width(term)
  -> [RD(a-c),RU(b-d)]
otherwise
  -> Entire
```

これにより少なくとも:

```text
CancelSubtract(Empty,Empty)   -> Empty
CancelSubtract(Empty,bounded) -> Empty
```

を固定する。

rounded `Width` propertyだけで比較せず、2Sum expansion等でexact width relationを判定する。

```text
CancelAdd(total,term)=CancelSubtract(total,-term)
```

で同じEmpty matrixを継承する。

---

## 38. DecoratedInterval

### 38.1 Decoration

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

数値順を品質順としてminimumを利用できる。

### 38.2 API候補

```csharp
public readonly struct DecoratedInterval
    : IEquatable<DecoratedInterval>
{
    public static DecoratedInterval NaI { get; }
    public static DecoratedInterval Empty { get; }
    public static DecoratedInterval Entire { get; }

    public bool IsNaI { get; }
    public bool IsEmpty { get; }
    public Decoration Decoration { get; }

    public static DecoratedInterval FromInterval(Interval interval);
    public bool TryGetInterval(out Interval interval);

    public bool Equals(DecoratedInterval other);
    public override bool Equals(object? obj);
    public override int GetHashCode();

    public static bool operator ==(
        DecoratedInterval left,
        DecoratedInterval right);

    public static bool operator !=(
        DecoratedInterval left,
        DecoratedInterval right);

    public bool SemanticallyEquals(DecoratedInterval other);
}
```

### 38.3 default / NaI

`Ill=0`を利用し:

```text
default(DecoratedInterval).IsNaI == true
```

- NaIはbare Intervalではない。
- `TryGetInterval`はfalse、out=Empty。
- NaI input operationはNaI。
- NaIはcanonical single stateとする。

### 38.4 FromInterval

```text
bounded nonempty -> Com
unbounded nonempty -> Dac
Empty -> Trv
```

### 38.5 result-state decoration cap

operationごとのmaximum possible decorationを`opDec`、入力decorationのminimumを`inputDec`とする。

result interval自身が許容する最大decorationを必ず加える。

```text
maxForResult =
  Trv  if result is Empty
  Dac  if result is unbounded nonempty
  Com  if result is bounded nonempty
```

最終結果:

```text
resultDec = min(inputDec, opDec, maxForResult)
```

canonical constructorを1か所に集約する。

```text
CreateCanonical(resultInterval, requestedDecoration):
  requestedDecoration == Ill -> NaI
  result Empty     -> min(requestedDecoration, Trv)
  result unbounded -> min(requestedDecoration, Dac)
  result bounded   -> min(requestedDecoration, Com)
```

NaI入力はこの処理前にNaIへ伝播する。Illでordinary intervalを作らない。

例:

```text
Com [MaxValue,MaxValue] + Com [MaxValue,MaxValue]
  bare result: [MaxValue,+Infinity]
  result cap: Dac
  decorated result must not be Com
```

Empty resultは最大Trv。

### 38.6 operation policy例

- add/sub/mul等の全domain連続operation: `opDec=Com`
- divisorがzeroを含むdivision: `opDec<=Trv`
- `Sqrt([-1,4])`: bare `[0,2]`, `opDec=Trv`
- Tan pole crossing: `opDec=Trv`

operation固有規則は`DecorationPolicy`へ集約する。

### 38.7 C# value equality vs IEEE semantic equality

C# value equalityはreflexiveとする。

```text
NaI == NaI -> true
```

- NaI同士は等しい。
- 非NaIはcanonical interval part + decorationを比較。
- NaIは固定Hash。
- 非NaI Hashは`Interval.GetHashCode()`とDecorationを結合。
- equalityとHashでinternal NaN payloadを観測しない。

`SemanticallyEquals`は別概念とする。

```text
NaI.SemanticallyEquals(any) -> false
nonNaI -> interval part equality, decoration ignored
```

collection keyとしての値等値性とIEEE semantic equalityを分離する。

---

## 39. Decorated math API

bare APIと区別して`DecoratedIntervalMath`を候補とする。

```csharp
public static class DecoratedIntervalMath
{
    public static DecoratedInterval Sqrt(DecoratedInterval value);
    public static DecoratedInterval Exp(DecoratedInterval value);
    public static DecoratedInterval Sin(DecoratedInterval value);
}
```

四則演算operatorは`DecoratedInterval`同士へ追加できるが、すべて§38.5のcanonical result capを通す。

---

## 40. Parsing / Formatting

### 40.1 parsing API候補

```csharp
Interval.Parse(ReadOnlySpan<char> text)
Interval.TryParse(...)
Interval.ParseExact(...)
Interval.TryParseExact(...)
```

初版syntax候補:

```text
Empty
Entire
[a,b]
[a]
```

endpoint token:

- decimal
- integer
- ±Infinity
- exact hexadecimal binary literal

### 40.2 outward decimal parsing

`double.Parse`後に点区間を作らない。

10進tokenをexactに:

```text
sign * integerSignificand * 10^decimalExponent
```

として解析し:

```text
single decimal x -> [RoundDown(x),RoundUp(x)]
[a,b] -> [RoundDown(exact a),RoundUp(exact b)]
```

exact lower<=upperをbinary rounding前に確認する。

finite decimalがbinary64範囲外の場合もfinite overflowとしてtight enclosureを作る。

### 40.3 invalid input

- syntax error
- NaN endpoint
- exact lower > exact upper
- lower +Infinity
- upper -Infinity
- Infinity singleton

`Parse`は最終API reviewで決めた.NET parsing exception、`TryParse`はfalse/out=Empty。

### 40.4 formatting

診断`ToString`と永続化formatを分離する。

候補:

- G: human-readable valid enclosure
- R: exact round-trip
- X: exact hexadecimal endpoint

exact formatのparse round-tripはcanonical intervalへbitwise一致する。

---

## 41. Binary interchange

private `[-Lower,Upper]` memory layoutをwire contractにしない。

version 1候補:

```text
byte 0: version
byte 1: state
byte 2..9: external Lower bits LE
byte 10..17: external Upper bits LE
```

Emptyではendpoint bytesを固定値とし、Entire/Zeroはexternal canonical bitsをencodeする。

invalid version/state/NaN endpoint/reversed endpointをdecoderで拒否する。

---

## 42. Interval splitting

```csharp
bool TrySplitAt(
    double splitPoint,
    out Interval left,
    out Interval right);

bool TryBisect(
    out Interval left,
    out Interval right);
```

`TrySplitAt`成功条件:

```text
nonempty
splitPoint finite
Lower < splitPoint < Upper
```

結果:

```text
[Lower,splitPoint]
[splitPoint,Upper]
```

共有点により実数集合としてgapを作らない。

`TryBisect`初版はbounded non-singletonだけを対象にする。strict interior binary64が存在しない隣接endpoint区間はfalse。unbounded intervalの自動pivotは任意性があるため、利用者が`TrySplitAt`で明示する。

---

## 43. Security/resource considerations

parserはuntrusted inputを想定する。

- 最大入力長
- 最大significand digit数
- 最大exponent digit数
- recursionなし
- culture曖昧性排除
- exceptionへ入力全文を無制限に含めない

具体的limit値はparser実装subphaseのbenchmark/security reviewで固定する。

binary decoderはlength/stateを先に検証し、片側NaN等の内部不変状態を作らない。

---

## 44. TDD・決定的fixture

source実装は各論理単位で先に失敗testを追加し、失敗を確認してからproduction implementationを追加する。

Phase 4A順序:

1. Contains/IsBounded
2. Intersect/Hull
3. relation
4. IntervalOverlap
5. numeric properties
6. Abs/Sign/minmax
7. integer-valued functions

Phase 4B-D順序:

1. constants
2. Reciprocal
3. Square
4. Sqrt
5. integer Pow/Root
6. FMA
7. MPFR reference corpus
8. Exp/Log系
9. hyperbolic/inverse
10. periodic reducer
11. Sin/Cos
12. Tan
13. Atan2
14. general Pow

Phase 4E順序:

1. IntervalUnion2
2. DivideToUnion
3. ReciprocalToUnion
4. ReverseMultiply
5. cancellative operations
6. Decoration/default NaI
7. DecoratedInterval equality/canonicalization
8. decorated arithmetic/math
9. exact parser/formatter
10. interchange
11. split/bisect

### 44.1 review-regression fixtures

`F-PR3-010`:

```text
DivideToUnion([1,2],Entire) -> Count2 zero-touch enclosures
ReciprocalToUnion(Entire) -> Count2 zero-touch enclosures
ReverseMultiply([1,2],Entire) -> Count2 zero-touch enclosures
```

`F-PR3-011`:

negative xとYの6class matrixを全件固定。特に:

```text
Atan2([-1,0],[-2,-1]) -> [-π,+π]
Atan2(Zero,[-2,-1]) -> Pi
Atan2([0,1],[-2,-1]) -> QII..π
Atan2([-1,1],[-2,-1]) -> [-π,+π]
```

`F-PR3-012`:

```text
Pow([0,0.5],[0,1])  -> [0,1]
Pow([0,0.5],[-1,0]) -> [1,+Infinity]
Pow([0,2],[0,1])    -> [0,2]
```

`F-PR3-013`:

one-sided denominator `[0,d]` / `[c,0]`についてZ/P/N/Mを全件固定し、`DivideToUnion(...).ConvexHull == ordinary division`を検証する。

`F-PR3-014`:

cancellative Empty/common/unbounded 3x3 matrixを固定する。

`F-PR3-015`:

- union Count0/1/2 equality/hash/operator
- decorated NaI reflexive equality/hash
- semantic equalityとの相違
- union indexer invalid access

`F-PR3-016`:

```text
Com MaxValue singleton + Com MaxValue singleton
 -> unbounded result
 -> Decoration <= Dac
```

Empty resultがTrvを超えないことも固定する。

`F-PR3-017`:

strict endpoint-wise lessのEmpty 3組合せと`BothEmpty` overlap inverseを固定する。

---

## 45. Property / differential test

Phase 1/3:

- add/mul commutativity
- double negation
- Zero identity
- result invariant
- scalar/SIMD canonical endpoint一致

Phase 4:

- intersection commutative/idempotent
- hull commutative/idempotent
- intersectionは各operandのsubset
-各operandはhullのsubset
- overlap inverse consistency
- Width/Radius >=0
- Mignitude <= Magnitude
- Square(X) subset-or-equal `X*X`
- FMA(X,Y,Z) subset-or-equal `(X*Y)+Z`（同じset semanticsの場合）
- Sin/Cos subset `[-1,1]`
- ordinary division == extended division ConvexHull
- backend間canonical endpoint bits一致
- parse/format and binary interchange round-trip
- split childrenが元区間をcoverし各々subset

sample point inclusionだけをprimary proofにせず、exact/MPFR/conformance corpusを併用する。

---

## 46. API確定・Phase完了ゲート

Phase 2:

- representative calculationがoperatorで自然に記述できる
- Empty/Entire明示判定
- invalid constructorとEmpty演算結果の違いが明確
- signed zero固定
-四則fixture成功
- exact oracle差異なし
- conformance required case成功
- x64/ARM64 corpus一致
- public API baseline保存
- basic operation allocation 0

Phase 4の各function/typeは:

- 意味論/domain matrix review済み
- required deterministic fixture成功
- exact/MPFR/referenceとの差異なしまたは承認済み
- x64/ARM64 canonical一致
- backend間bitwise一致
- failure artifactから分岐を追跡可能
- API baseline更新

を満たすまでpublic化しない。

breaking changeは`doc/Design/BreakingChanges.md`へ理由と移行方法を記録する。

---

## 47. Performance・thread safety・AOT

Phase 1から:

- `readonly struct`
- basic arithmetic heap allocationなし
- global rounding-mode変更なし
- production hot pathに`BigInteger`なし
- hot pathにvirtual/interface/delegate dispatchなし
- unconditional 1 ULP expansionなし
-不要なreciprocal経由二重丸めなし
- raw constructorでvalidation重複回避

`Interval`はimmutable。

Phase 1ではreflection/runtime codegen/dynamic assembly/native resolverを使わずNativeAOT/trimmingを妨げない。

Phase 4 parserの`BigInteger`利用は許可するがresource limitを持つ。

---

## 48. 「同等の計算結果」の定義

四則演算およびtight公開関数では:

1. 採用したset-based real semanticsを使用
2. bare Intervalで非連結を表せない場合はconvex hull
3. finite endpointを指定方向へ正しく丸めた最も内側のbinary64
4. signed zeroをcanonicalize
5. internal Empty NaN payloadは比較しない

reference library差異よりexact oracleと採用意味論を優先する。

`IntervalUnion2`はexact open-set representationではなく、connected componentごとのtight closed enclosureを最大2個保持する。このため、excluded boundaryがcomponent enclosureに含まれる場合があるが、component countと各closure enclosureを保持することを目的とする。

---

## 49. License・third-party

`kv`/`inari`コードを翻案・移植する場合:

- source commentへcommit SHA
- MIT copyright/permission notice
- test case移植も出典記録

ITF1788 data/generatorを利用・再配布する場合license/NOTICEをtest assetへ付随させる。

MPFR/native backend採用時はlicense、binary distribution、NOTICEを別途確認する。

production packageに不要なreference toolを同梱しない。

---

## 50. 未確定事項

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

Phase 4対象subphaseで確定するもの:

- Phase 4A public naming
- Midpoint tie policy最終conformance
- elementary production backend
- decorated operation最終surface
- parser syntax詳細/resource-limit具体値
- binary interchange final version-1 reject/canonicalize細則

`IntervalUnion2`のcomponent-closure-enclosure意味論、Atan2 negative-axis matrix、general Pow zero-boundary規則、one-sided extended division、cancellative Empty matrix、value equality surface、decoration result cap、relation Empty/inverseは設計版5で確定済みであり、未確定事項へ戻さない。

---

## 51. Review finding closure

過去の四則演算finding:

- `F-PR3-001`: resolved history — directed rounding completeness
- `F-PR3-002`: resolved history — ISA/FMA separation
- `F-PR3-004`: resolved history — conformance source/matrix
- `F-PR3-005`: resolved history — oracle responsibilities
- `F-PR3-006`: resolved history — x64/ARM64 CI
- `F-PR3-007`: resolved history — threshold/tie fixtures
- `F-PR3-008`: resolved history — native backend traceability
- `F-PR3-009`: resolved history — finite overflow oracle
- `F-PR3-003`: withdrawn review erratum

追加設計review finding:

| Finding | 設計版5 disposition | Evidence |
|---|---|---|
| `F-PR3-010` High | addressed, verification pending | §34～36でcomponent-closure enclosure、zero-touch Count2、no-touch-mergeを確定 |
| `F-PR3-011` High | addressed, verification pending | §31でnegative-x branch-cut 6class matrixとsigned-zero規則を確定 |
| `F-PR3-012` Medium | addressed, verification pending | §32でzero-boundary closure candidateと`a==0`全分岐を確定 |
| `F-PR3-013` Medium | addressed, verification pending | §35.2でone-sided-zero denominator表を復元 |
| `F-PR3-014` Medium | addressed, verification pending | §37でEmpty/common/unbounded 3x3 matrixを確定 |
| `F-PR3-015` Medium | addressed, verification pending | §34.2/34.5/34.6、§38.2/38.7でvalue semantics/APIを確定 |
| `F-PR3-016` Medium | addressed, verification pending | §38.5でresult-state decoration capを追加 |
| `F-PR3-017` Low | addressed, verification pending | §22.3、§23でEmpty strict-lessとBothEmpty inverseを復元 |

追加設計reviewで指摘された「統合時に全規範を保持した」という過去の記録は正確ではなかった。設計版4ではone-sided extended division等の一部規範が脱落していた。本設計版5および対応レポートを訂正記録とし、過去report自体は履歴として改変しない。

---

## 52. 実装開始条件

### 52.1 Phase 1

本統合設計のreview closure後、次の論理単位で開始する。

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

### 52.2 Phase 4

Phase 4A以降は次をすべて満たした後に開始する。

- Phase 1～3の必要成果物完了
- Phase 2 basic API freeze完了
- 対象Phase 4 sectionのreview完了
- test/conformance/reference基盤と統合済み
- diagnostic artifact workflowが存在

各source実装はTDDで、失敗testを先にcommit/pushし、失敗を確認してからproduction implementationを追加する。

---

## 53. 参照

- `doc/Design/basic/IntervalArithmetic.md`
- `unageek/inari@18b83a571d7681c76067bc38d90a74e8be29f545`
- `mskashi/kv@c7f8f2324a0e403cca6b39f46088a22843d440db`
- `unageek/ITF1788@d8c2a64478ebdc9cbde6ccef33eaad3bed60ed81`
- IEEE 1788.1-2017
- .NET 10 `Math.FusedMultiplyAdd`, `Math.BitIncrement`, `Math.BitDecrement`
- .NET hardware intrinsics `Avx512F`, `Avx2`, `Avx`, `Fma`, `Sse2`, `AdvSimd.Arm64`
