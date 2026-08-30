# 区間代数関数・初等関数 詳細設計

## 1. 文書情報

- 対象: Phase 4B、4C、4D
- 前提:
  - `IntervalArithmetic.md`
  - `IntervalArithmetic.Revision3.md`
  - `IntervalNonArithmetic.Roadmap.md`
  - `IntervalSetAndNumeric.md`
- 主要参照:
  - `unageek/inari` commit `18b83a571d7681c76067bc38d90a74e8be29f545`
  - `mskashi/kv` commit `c7f8f2324a0e403cca6b39f46088a22843d440db`
- 作成日: 2026-08-30
- 設計状態: review required

本書では、四則演算以外の点関数をbare intervalへ拡張する方法と、正しく方向丸めされたscalar endpoint kernelの契約を定義する。

## 2. 公開API候補

```csharp
namespace Devo6.Numerics;

public static partial class IntervalMath
{
    // Algebraic
    public static Interval Reciprocal(Interval value);
    public static Interval Square(Interval value);
    public static Interval Sqrt(Interval value);
    public static Interval Pow(Interval value, int exponent);
    public static Interval Root(Interval value, int degree);
    public static Interval FusedMultiplyAdd(
        Interval left,
        Interval right,
        Interval addend);

    // Exponential / logarithmic
    public static Interval Exp(Interval value);
    public static Interval Exp2(Interval value);
    public static Interval Exp10(Interval value);
    public static Interval Log(Interval value);
    public static Interval Log2(Interval value);
    public static Interval Log10(Interval value);

    // Hyperbolic
    public static Interval Sinh(Interval value);
    public static Interval Cosh(Interval value);
    public static Interval Tanh(Interval value);
    public static Interval Asinh(Interval value);
    public static Interval Acosh(Interval value);
    public static Interval Atanh(Interval value);

    // Trigonometric
    public static Interval Sin(Interval value);
    public static Interval Cos(Interval value);
    public static Interval Tan(Interval value);
    public static Interval Asin(Interval value);
    public static Interval Acos(Interval value);
    public static Interval Atan(Interval value);
    public static Interval Atan2(Interval y, Interval x);

    // Positive-base general power. Phase 4D.
    public static Interval Pow(Interval value, Interval exponent);
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

全signatureは候補であり、各subphaseのAPI review後にbaselineへ追加する。一般`Pow(Interval,Interval)`はinteger powerと同時に公開しない。

## 3. 共通意味論

### 3.1 natural interval extension

点関数`f`の定義域を`D`、入力区間を`X`とする。

bare intervalの結果は原則として次である。

```text
hull({ f(x) | x ∈ X ∩ D })
```

- `X ∩ D`が空なら`Empty`。
- 真の像が非連結でも、戻り値が`Interval`なら最小の連結hullを返す。
- 入力の一部が定義域外だった事実はbare intervalでは保持しない。
- 定義域欠損は将来の`DecoratedInterval`でdecorationへ反映する。

### 3.2 Empty propagation

入力のいずれかがEmptyで、点関数を評価する組が存在しない場合はEmptyを返す。

### 3.3 tight endpoint

有限端点は、真の像を包含する最も内側のbinary64とする。

```text
lower = round toward -Infinity
upper = round toward +Infinity
```

### 3.4 public backend independence

public APIは、managed/native、scalar/SIMD、近似方式、range reduction方式を公開しない。

同一APIを提供するbackendは、canonical endpoint bit patternを一致させる。

## 4. scalar endpoint kernel

### 4.1 構造

```csharp
internal static class DirectedElementary
{
    internal static double SqrtDown(double value);
    internal static double SqrtUp(double value);

    internal static double ExpDown(double value);
    internal static double ExpUp(double value);

    internal static double LogDown(double value);
    internal static double LogUp(double value);

    internal static double SinDown(double value);
    internal static double SinUp(double value);

    // その他も同じ形
}
```

区間拡張層はdomain、単調区間、極値・極を決定し、endpoint kernelは1個のpoint functionを正しい方向へ丸める。

### 4.2 kernel precondition

- NaNを渡さない。
- 定義域外のfinite値を渡さない。
- open boundaryをlimit値として処理する場合は、関数ごとに明示された`±Infinity`等を区間層が直接返す。
- signed zeroに意味がある関数はbit patternを保持する。

### 4.3 正式backendの条件

次のいずれかを満たす実装だけを正式backendとする。

1. 全binary64入力に対する誤差上限と方向補正が証明されているmanaged implementation
2. correctly-rounded implementationの検証済み移植
3. MPFR等、指定方向丸めを提供するnative backend

`Math.Sin`、`Math.Exp`等の戻り値に、証明なしで1 ULPまたは固定ULPを加減する方式は採用しない。

### 4.4 reference oracle

初等関数のprimary reference corpusは、固定versionのMPFRへbinary64入力を正確に渡し、53 bit precisionでRNDD/RNDUを指定して生成する。

`inari`のpinned implementationもMPFRのRNDD/RNDUを使用するため、区間意味論とsecondary differential oracleとして利用する。

reference lockには少なくとも次を保存する。

- MPFR version
- wrapper source hash
- compiler / target
- input corpus hash
- output corpus hash
- rounding mode
- inari commit

## 5. 区間定数

### 5.1 public constant

正確な実数定数を含むtight intervalを返す。

例:

```text
Pi.Lower <= π <= Pi.Upper
Pi.Upper == BitIncrement(Pi.Lower)
```

正確な定数がbinary64で表現可能な場合を除き、点区間にしない。

### 5.2 生成

- MPFRのdirected conversionからendpoint bitsを生成する。
- endpoint bitsをsourceまたはgenerated fileへ固定する。
- build時にネットワークやnative generatorを必要としない。
- generatorと固定値の再生成一致をreference-integrity CIで確認する。

### 5.3 range reduction用定数

三角関数のrange reductionには、publicな2端点定数とは別に、十分なbit数の`2/π`、`π/2` split table等を使用する。

public `Pi`を通常のbinary64除算へ渡すだけで巨大入力の象限を決めない。

## 6. Reciprocal

`X=[a,b]`とする。

| Input class | Result |
|---|---|
| Empty | Empty |
| `[0,0]` | Empty |
| `a < 0 < b` | Entire |
| `a < 0 && b == 0` | `[-Infinity, RU(1/a)]` |
| `a == 0 && b > 0` | `[RD(1/b), +Infinity]` |
| `b < 0` | `[RD(1/b), RU(1/a)]` |
| `a > 0` | `[RD(1/b), RU(1/a)]` |

`1/0`をscalar primitiveへ渡さず、zero contactを区間層で処理する。

通常の`Interval`結果は0を跨ぐ入力を`Entire`へhullする。2成分を保持するreciprocalは`IntervalAdvancedFeatures.md`で設計する。

## 7. Square

```text
Square(Empty) = Empty
```

`X=[a,b]`について:

```text
0 <= a:
    [RD(a*a), RU(b*b)]

b <= 0:
    [RD(b*b), RU(a*a)]

a < 0 < b:
    [-0.0, max(RU(a*a), RU(b*b))]
```

mixed区間の下限0は正確である。乗算operatorで`X*X`と計算しない。同一変数の依存性により不要に広がるためである。

## 8. Square root

### 8.1 区間拡張

定義域は`[0,+Infinity)`。

```text
Sqrt(Empty) = Empty
b < 0       = Empty
a <= 0 <= b = [-0.0, SqrtUp(b)]
0 < a       = [SqrtDown(a), SqrtUp(b)]
```

`Sqrt(+Infinity)=+Infinity`。

### 8.2 directed scalar algorithm

`kv`のno-hardware-rounding implementationを参照し、次を設計仕様とする。

```text
SmallSqrtInputThreshold = 2^-969
SqrtInputScale          = 2^106
SqrtResultScale         = 2^53
```

`x >= 0`について:

```text
r = roundNearest(sqrt(x))

if x < 2^-969:
    xs = x * 2^106
    rs = r * 2^53
    (p,e) = exact-product-decomposition(rs,rs)
    compare p+e with xs
else:
    (p,e) = exact-product-decomposition(r,r)
    compare p+e with x

SqrtUp:
    p < x  or (p == x and e < 0) -> NextUp(r)
    otherwise                     -> r

SqrtDown:
    p > x  or (p == x and e > 0) -> NextDown(r)
    otherwise                     -> r
```

scaled経路では比較対象を`xs`へ読み替える。2の整数乗scaleは対象範囲で正確に行う。

0、最小subnormal、閾値直前/一致/直後、最大有限値、完全平方、非完全平方を固定fixtureにする。

## 9. Integer power

### 9.1 API

```csharp
IntervalMath.Pow(Interval value, int exponent)
```

一般`Pow(Interval,Interval)`とは異なるoverloadである。

### 9.2 scalar primitive

```text
PownDown(double x, int n)
PownUp(double x, int n)
```

は数学的な`x^n`を直接方向丸めする。区間乗算を反復してendpointを作る方式をtight endpointの根拠にしない。

`int.MinValue`の絶対値を`int`で計算しない。内部では符号と`uint` magnitudeへ分解する。

### 9.3 exponent = 0

```text
Pow(Empty,0) = Empty
Pow(nonempty,0) = [1,1]
```

point functionの`x^0=1`を採用し、zeroを含む非空区間も`[1,1]`。

### 9.4 positive odd exponent

単調増加である。

```text
[RD(a^n), RU(b^n)]
```

### 9.5 positive even exponent

`A=Abs(X)`として:

```text
[RD(A.Lower^n), RU(A.Upper^n)]
```

zeroを跨ぐ場合、下限は正確な0になる。

### 9.6 negative exponent

`X=[a,b]`、`n<0`。

```text
X == [0,0] -> Empty
```

nが偶数:

```text
A = Abs(X)
[RD(A.Upper^n), RU(A.Lower^n)]
```

`A.Lower=0`なら上限は`+Infinity`となる。

nが奇数:

```text
a < 0 < b -> Entire
otherwise  -> [RD(b^n), RU(a^n)]
```

片側zero endpointではsigned zeroをscalar kernelへ渡し、次を得る。

```text
[0,b], n<0 odd -> [RD(b^n), +Infinity]
[a,0], n<0 odd -> [-Infinity, RU(a^n)]
```

### 9.7 overflow/underflow

- 結果が有限範囲を超える場合は既存directed overflow規則を使用する。
- 非zeroのexact resultが最小subnormalより小さい場合、方向と符号に応じて0または最小subnormalへ丸める。
- exponentiation-by-squaringを使用する場合も、途中結果の丸めを最終endpointの正しさと混同しない。正しいendpoint kernelとして証明できない構成は採用しない。

## 10. Integer root

### 10.1 API contract

```csharp
IntervalMath.Root(Interval value, int degree)
```

- `degree <= 0`は`ArgumentOutOfRangeException`。
- `degree == 1`は入力を返す。
- EmptyはEmpty。

negative degreeは初版で受け入れない。必要な場合は`Reciprocal(Root(...))`ではなく、tightnessを検討した専用overloadを別途設計する。

### 10.2 odd degree

全実数上で単調増加。

```text
[RootDown(a,n), RootUp(b,n)]
```

### 10.3 even degree

定義域は`[0,+Infinity)`。

```text
b < 0       -> Empty
a <= 0 <= b -> [-0.0, RootUp(b,n)]
0 < a       -> [RootDown(a,n), RootUp(b,n)]
```

### 10.4 endpoint kernel

Newton iterationだけで結果を確定しない。candidateのn乗と入力のexact relationを検証し、必要な隣接binary64補正を行う。overflowを避けるscaled comparison、degreeの偶奇、negative inputのodd rootを固定する。

## 11. Fused multiply-add

### 11.1 意味

```text
FusedMultiplyAdd(X,Y,Z)
= hull({ x*y+z | x∈X, y∈Y, z∈Z })
```

endpoint計算は、数学的な積和を1回だけ丸める`FmaDown` / `FmaUp`を使用する。

```text
FmaDown(x,y,z) = RD(exact(x*y+z))
FmaUp(x,y,z)   = RU(exact(x*y+z))
```

`(X*Y)+Z`を実装として代用しない。積と加算の二重丸めにより広がるためである。

### 11.2 extrema

既存乗算の符号分類表から、積のlower候補endpoint pairとupper候補endpoint pairを得る。

- lower候補には`Z.Lower`を加える。
- upper候補には`Z.Upper`を加える。
- mixed×mixedはlower/upperそれぞれ2候補を評価し、min/maxを取る。
- Zeroとの積はaddendをそのまま返す。
- いずれかがEmptyならEmpty。

### 11.3 acceptance

- scalar FMA endpointはexact rational oracleと一致する。
- subnormal product + large addend、cancellation、overflow、`0*Infinity`の区間分岐を固定する。
- 原則として`FusedMultiplyAdd(X,Y,Z)`は`(X*Y)+Z`のsubsetまたは同値になることをproperty testする。ただし特殊値・Empty semanticsを同じ入力集合として比較する。

## 12. 単調関数の共通実装

### 12.1 単調増加

定義域との共通部分が`[l,u]`となる場合:

```text
[FunctionDown(l), FunctionUp(u)]
```

### 12.2 単調減少

```text
[FunctionDown(u), FunctionUp(l)]
```

### 12.3 open boundary

定義域がopenで、入力が境界へ接する場合はlimit値を使用する。

例:

```text
Log([0,2])   = [-Infinity, LogUp(2)]
Atanh([-1,0]) = [-Infinity, +0]
```

入力が境界点しか含まず、定義域内の点を含まない場合はEmpty。

## 13. Exponential functions

`Exp`、`Exp2`、`Exp10`はいずれも全実数上で単調増加。

```text
Exp([a,b])   = [ExpDown(a), ExpUp(b)]
Exp2([a,b])  = [Exp2Down(a), Exp2Up(b)]
Exp10([a,b]) = [Exp10Down(a), Exp10Up(b)]
```

規則:

- Empty -> Empty
- `-Infinity`側の値は`+0.0`
- `+Infinity`側の値は`+Infinity`
- finite overflow/underflowは方向別にtightに丸める
- 結果lower zeroは区間端点として`-0.0`へ正規化する

## 14. Logarithmic functions

`Log`、`Log2`、`Log10`の定義域は`(0,+Infinity)`で単調増加。

`X=[a,b]`について:

```text
b <= 0 -> Empty

otherwise:
    lower = -Infinity if a <= 0 else LogDown(a)
    upper = +Infinity if b == +Infinity else LogUp(b)
```

base 2、base 10も同じdomain処理を行い、対応するendpoint kernelを使用する。

`Log([0,0])`はEmptyであり、`[-Infinity,-Infinity]`という不正区間を作らない。

## 15. Hyperbolic and inverse functions

### 15.1 全実数上で単調増加

次は共通形を使う。

- `Sinh`
- `Tanh`
- `Asinh`
- `Atan`

```text
F([a,b]) = [FDown(a), FUp(b)]
```

limit:

```text
Tanh(-Infinity) = -1
Tanh(+Infinity) = +1
Atan(-Infinity) = -π/2
Atan(+Infinity) = +π/2
```

`π/2`はtight interval constantの対応端点を使用する。

### 15.2 `Cosh`

偶関数で、負側では減少、正側では増加する。

```text
b < 0:
    [CoshDown(b), CoshUp(a)]

a > 0:
    [CoshDown(a), CoshUp(b)]

a <= 0 <= b:
    [1, CoshUp(max(-a,b))]
```

### 15.3 `Acosh`

定義域は`[1,+Infinity)`、単調増加。

```text
b < 1 -> Empty
l = max(a,1)
[AcoshDown(l), AcoshUp(b)]
```

### 15.4 `Asin`

定義域は`[-1,1]`、単調増加。

```text
Xc = X.Intersect([-1,1])
Xc Empty -> Empty
[AsinDown(Xc.Lower), AsinUp(Xc.Upper)]
```

### 15.5 `Acos`

定義域は`[-1,1]`、単調減少。

```text
Xc = X.Intersect([-1,1])
Xc Empty -> Empty
[AcosDown(Xc.Upper), AcosUp(Xc.Lower)]
```

### 15.6 `Atanh`

定義域は`(-1,1)`、単調増加。

`X=[a,b]`について:

```text
b <= -1 or a >= 1 -> Empty

lower = -Infinity if a <= -1 else AtanhDown(a)
upper = +Infinity if b >= 1 else AtanhUp(b)
```

`[-1,-1]`および`[1,1]`はEmpty。`[-1,0]`および`[0,1]`はそれぞれ片側非有界となる。

## 16. Sine and cosine

### 16.1 range reduction

全binary64範囲で象限を誤らない`PeriodicCriticalPointDetector`を内部実装する。

候補方式はPayne-Hanek型fixed-point reductionとし、`2/π`の十分なbit数のtableを固定する。

禁止する方式:

- `Math.PI`だけで巨大入力を除算してquadrantを決める。
- `value % (2*Math.PI)`だけで臨界点を判定する。
- public `IntervalConstants.Pi`の片側だけを通常除算する。

### 16.2 critical point test

内部APIは、区間が次のlattice pointを含むかを正確に判定する。

```text
ContainsPeriodicPoint(X, offset, period)
```

判定はexact binary64 endpointと高精度constant boundsから、該当する整数`k`の存在を求める。

### 16.3 `Sin`

```text
lower = min(
    SinDown(a),
    SinDown(b),
    -1 if X contains -π/2 + 2kπ)

upper = max(
    SinUp(a),
    SinUp(b),
    +1 if X contains +π/2 + 2kπ)
```

入力が非有界、またはmax/min両critical latticeを含む場合は`[-1,1]`。

### 16.4 `Cos`

```text
lower = min(
    CosDown(a),
    CosDown(b),
    -1 if X contains π + 2kπ)

upper = max(
    CosUp(a),
    CosUp(b),
    +1 if X contains 2kπ)
```

入力が非有界、またはmax/min両critical latticeを含む場合は`[-1,1]`。

### 16.5 exact extrema

critical pointを含む場合、endpoint kernelの近似値ではなく正確な`-1`または`+1`を採用する。

## 17. Tangent

定義域は次である。

```text
R \ { π/2 + kπ | k∈Z }
```

### 17.1 poleなし

入力区間が1つの連続branch内に完全に収まる場合、tanは単調増加である。

```text
[TanDown(a), TanUp(b)]
```

### 17.2 poleを含む

- 入力がpoleへ両側または片側から近づける場合、像の連結hullはEntire。
- 定義域内の点を1つも含まない場合はEmpty。
- binary64点区間は非zeroのπ/2整数倍と数学的に一致しないが、parserや将来の定数区間を考慮し、一般のdomain判定を省略しない。

### 17.3 pole detection

`π/2 + kπ`の存在を、Sin/Cosと同じfixed-point reductionで判定する。近似した`Math.PI`との比較だけでpoleの有無を決めない。

## 18. Atan2

### 18.1 API order

```csharp
IntervalMath.Atan2(y, x)
```

.NETの`Math.Atan2(y,x)`と同じargument orderを使用する。

点関数のdomainは`R² \ {(0,0)}`、principal rangeは`(-π,π]`。

### 18.2 rectangle algorithm

入力rectangle `X×Y`をzero境界で最大4つのsign cellへ分割する。

1. Empty operandならEmpty。
2. `X=Zero`かつ`Y=Zero`ならEmpty。
3. 各sign cellからoriginだけを除外する。
4. quadrantごとにmonotonic cornerを`Atan2Down/Up`で評価する。
5. x/y axisを含む場合は`0`、`±π/2`、`π`のtight constantをcandidateへ加える。
6. negative x-axisの上下を同時に含みbranch cutを跨ぐ場合、bare resultは`[-π,π]`のtight enclosureとする。
7. 各cellのangular rangeを`AngleArcAccumulator`で統合し、通常の数直線min/maxだけで`-π/π`を跨ぐarcを誤処理しない。

### 18.3 sign matrix

実装時に次のx/y classの直積を固定matrixへ展開する。

```text
Empty
Negative
NonPositive touching zero
Zero
NonNegative touching zero
Positive
Mixed
```

matrixはpinned inariの`atan2`分類とITF1788 caseをsecondary referenceとする。全cellを少なくとも1fixtureで通す。

## 19. General positive-base power

### 19.1 point function

一般`Pow(Interval value, Interval exponent)`は、integer exponent overloadと異なり、次のreal functionだけを対象とする。

```text
x^y = 0              if x == 0 and y > 0
      exp(y * log x) if x > 0
```

定義域:

```text
((0,+Infinity) × R) union ({0} × (0,+Infinity))
```

negative baseはこのoverloadのdomain外とする。negative baseの整数乗は`Pow(value,int)`を使用する。

### 19.2 interval algorithm

baseを`[0,+Infinity]`へclipし、`X=[a,b]`、`Y=[c,d]`とする。

Empty operandまたはdomain内の組がない場合はEmpty。

#### `d <= 0`

```text
b == 0 -> Empty
b < 1  -> [PowDown(b,d), PowUp(a,c)]
a > 1  -> [PowDown(b,c), PowUp(a,d)]
otherwise -> [PowDown(b,c), PowUp(a,c)]
```

#### `c > 0`

```text
b < 1  -> [PowDown(a,d), PowUp(b,c)]
a > 1  -> [PowDown(a,c), PowUp(b,d)]
otherwise -> [PowDown(a,d), PowUp(b,d)]
```

#### `c <= 0 < d`

```text
b == 0 -> Zero
lower = min(PowDown(a,d), PowDown(b,c))
upper = max(PowUp(a,c), PowUp(b,d))
```

0 boundaryとnegative exponentではlimitとしてInfinityを扱う。undefinedな`0^0`をscalar kernelへ直接渡さない。

### 19.3 公開順

このAPIは`Exp`、`Log`、directed power kernel、domain/conformance matrixが完成した後にのみ公開する。

## 20. production backend decision

### 20.1 managed core

`Reciprocal`、`Square`、`Sqrt`、integer `Pow`、`FusedMultiplyAdd`は、既存managed directed primitiveを拡張して実装することを第一候補とする。

### 20.2 elementary functions

初回実装前に、関数群単位で次を比較する。

| Candidate | Correctness | Distribution | Performance |
|---|---|---|---|
| certified managed polynomial/table | proofとhard-case fallbackが必要 | 最も容易 | 良好を期待 |
| correctly-rounded code port | license/移植検証が必要 | managed | 実装依存 |
| native MPFR | directed roundingを直接利用 | native binary配布が必要 | call overheadあり |
| native CRlibm等 | 対応関数・platformを確認 | native binary配布が必要 | 実装依存 |

### 20.3 shipping rule

- core packageがfunctionを公開する場合、全support platformで利用可能なbackendを持つ。
- packageがsupportを宣言した環境で、通常入力に`PlatformNotSupportedException`を投げる設計にしない。
- optional native backendしか存在しない場合は、coreの同名APIを未完成状態で公開せず、別package/API境界を設計する。
- backend変更はpublic resultを変更してはならない。

## 21. 内部コンポーネント

```text
src/Devo6.Interval/
  IntervalConstants.cs
  IntervalMath.Algebraic.cs
  IntervalMath.Exponential.cs
  IntervalMath.Hyperbolic.cs
  IntervalMath.Trigonometric.cs
  IntervalMath.Power.cs
  Internal/
    DirectedSqrt.cs
    DirectedIntegerPower.cs
    DirectedFma.cs
    DirectedElementary.cs
    PeriodicCriticalPointDetector.cs
    PayneHanekReducer.cs
    AngleArcAccumulator.cs
    ElementaryBackendDispatcher.cs
```

`ElementaryBackendDispatcher`はpublic optionを持たない。test/benchmarkだけがinternal hookでbackendを固定できる。

## 22. TDD実装順

1. tight constants
2. Reciprocal
3. Square
4. directed Sqrt + interval Sqrt
5. directed integer power + interval Pow(int)
6. integer Root
7. directed FMA + interval FMA
8. MPFR reference adapter/corpus
9. Exp/Exp2/Exp10
10. Log/Log2/Log10
11. Sinh/Tanh/Asinh/Atan
12. Cosh/Acosh/Atanh/Asin/Acos
13. Payne-Hanek reducer
14. Sin/Cos
15. Tan
16. Atan2
17. general positive-base Pow

各項目は、domain/特殊値test、directed endpoint test、interval testの順で失敗を確認してから実装する。

## 23. 決定的fixture

### 23.1 algebraic

- Empty、Entire、Zero
- `±0.0`
- minimum subnormal、minimum normal、maximum finite
- exact square / non-exact square
- square root threshold `2^-969`のprevious/equal/next
- positive/negative power、odd/even、`int.MinValue`
- negative exponentとzero contact/crossing
- FMA cancellation、overflow、subnormal product

### 23.2 domain boundary

- Log: previous/at/next of zero
- Asin/Acos: previous/at/next of`-1`と`+1`
- Acosh: previous/at/next of`1`
- Atanh: previous/at/next of`-1`と`+1`
- Root: even degreeとnegative-only/partial-domain

### 23.3 elementary hard cases

- MPFRでnearest valueの直近に位置するcase
- overflow/underflow threshold
- exact result
- signed-zero result
- endpointが隣接binary64になるcase
- x64/ARM64でBCL seedが異なっても最終endpointが一致するcase

### 23.4 periodic

- 各quadrant
- `kπ/2`の直前/包含/直後
- interval widthが1/2/4 quadrantを跨ぐcase
- multiple periods
-巨大binary64 point/interval
- Tan poleの直前/包含/直後
- Atan2の全sign class、axis、origin、negative-x branch cut

## 24. Property / differential test

- monotone functionは入力subsetに対して出力subsetとなる。
- resultはsampled exact point resultを包含する。
- `Square(x)`は`x*x`のsubsetまたは同値。
- `FusedMultiplyAdd(x,y,z)`は同じset semanticsで`x*y+z`のsubsetまたは同値。
- `Sin`/`Cos`結果は`[-1,1]`のsubset。
- `Tanh`結果は`[-1,1]`のsubset。
- `Exp`結果は非負。
- `Cosh`結果は`[1,+Infinity]`のsubset。
- scalar/SIMD/native backendのcanonical endpoint bitsが一致する。

sample testは正しさのprimary oracleにせず、MPFR directed corpusとdomain matrixを併用する。

## 25. CI artifact

失敗時には既存artifactに加えて次を保存する。

- function / operation
- input endpoint bits
- clipped domain interval
- monotonic/sign/quadrant class
- endpoint backend
- initial approximation bits
- correction decision
- periodic quotient/remainder metadata
- detected critical point / pole / branch cut
- MPFR expected bitsとversion
- actual bits
- constant table revision

## 26. 完了条件

関数群ごとに次を満たすまでpublic APIへ追加しない。

- 意味論とdomain matrixがreview済み。
- endpoint kernelが全許可入力でtightである根拠を持つ。
- deterministic/conformance/MPFR corpusが成功する。
- Linux x64 / ARM64でcanonical resultが一致する。
- 採用するSIMD/native backendがscalar referenceとbitwise equivalent。
- failure artifactから誤ったdomain/reduction/rounding分岐を追跡できる。
- API baselineと利用例が更新される。
- current PR HEADに一致するCI runが成功する。
