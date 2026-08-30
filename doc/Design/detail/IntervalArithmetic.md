# 区間演算 詳細設計

## 1. 文書情報

- 対象リポジトリ: `ssaattww/Devo6.Interval`
- 対象ライブラリ: `Devo6.Interval`
- 設計対象: IEEE 754 binary64 (`double`) を端点とする bare interval
- 基本設計: `doc/Design/basic/IntervalArithmetic.md`
- 初版: 2026-08-29
- 最終更新: 2026-08-30
- 設計版: 2

本書は、基本設計で決定した区間意味論、`[-Lower, Upper]` 内部表現、外向き丸めおよび SIMD 方針を、実装可能な粒度まで具体化する。

設計版2では、PR #3 の exhaustive design review で示された次の有効指摘を反映した。

- `F-PR3-001`: 乗除算の方向付き丸めを、subnormal、overflow、残差tieを含む疑似コードへ確定
- `F-PR3-002`: AVX2/SSE2 と x86 FMA capability を分離
- `F-PR3-004`: IEEE 1788.1 conformance matrix と固定 corpus を定義
- `F-PR3-005`: exact rational、inari、kv の責務と再現可能な参照harnessを定義
- `F-PR3-006`: Linux x64 / Linux ARM64 のCI matrixを定義
- `F-PR3-007`: 閾値、scaled比較、残差tie、overflowの決定的fixtureを定義
- `F-PR3-008`: native backendを後続benchmark gateへ明示的に延期

`F-PR3-003` は exhaustive review で `withdrawn_erratum` とされたため、有効な実装指摘としては扱わない。

## 2. 開発フェーズ

次の順序で開発する。

1. SIMDを使用しないmanaged scalar四則演算パイロットを作成する。
2. パイロットの利用結果を基に、基本`Interval` APIを確定する。
3. 確定したAPIを変更せず、SIMD backendを追加する。
4. 四則演算以外の区間操作および初等関数を段階的に追加する。

この順序により、区間意味論と公開APIの問題を、CPU命令およびSIMD最適化の問題から分離する。

### 2.1 Phase 0: 設計・検証基盤

本書の確定をPhase 0とする。

成果物:

- 公開API候補
- 内部不変条件
- 四則演算アルゴリズム
- 方向付き丸めアルゴリズム
- standards conformance matrix
- TDDの実装順序と決定的fixture
- exact rational oracle
- pinned reference harness
- x64 / ARM64 CI matrix
- API確定条件
- SIMD導入条件

完了条件:

- Phase 1が追加の数値アルゴリズム判断なしで開始できる。
- すべての許可された方向付き丸めprimitive入力について、返すbinary64値が一意に定まる。
- Phase 1に残す未確定事項が公開APIの評価項目に限定されている。

### 2.2 Phase 1: managed scalarパイロット

目的:

- SIMDやnative libraryに依存せず、区間意味論とAPIを検証する。
- 四則演算についてtightな外向き丸め結果を返す。
- 後続SIMD実装の基準となるscalar reference backendを作る。

対象:

- `Interval`値型
- 区間生成と入力検証
- `Empty`、`Entire`、`Zero`
- `Lower`、`Upper`
- `IsEmpty`、`IsEntire`、`IsSingleton`
- 等値比較とHash
- 加算、減算、単項マイナス、乗算、除算
- managed方向付き丸め
- signed zeroと特殊値の正規化
- 診断用文字列表現
- exact rational oracle
- standards conformance test
- pinned reference corpus comparison
- Linux x64 / Linux ARM64 CI
- 失敗診断artifact
- 最小サンプル

対象外:

- SIMD intrinsics
- processまたはthreadの浮動小数点丸めモード変更
- native library呼び出し
- `sqrt`、`pow`、`exp`、`log`、`sin`、`cos`等
- decorated interval / NaI
- 厳密な文字列からの区間変換
- 区間分割、Affine Arithmetic、Taylor Model

### 2.3 Phase 2: 基本API確定

Phase 1の利用性、特殊値、例外、命名および性能特性を確認し、基本`Interval` APIを固定する。

確定対象:

- package / assembly / namespace
- `Interval`の生成方法
- 基本プロパティ
- `Empty` / `Entire`の扱い
- 演算子
- 等値性とHash
- 例外種別
- signed zeroの公開規則
- `ToString`の契約範囲
- scalar値との変換・演算
- generic math interface

Phase 2完了後は、基本`Interval` APIに対する破壊的変更を原則として行わない。後続のSIMDや初等関数は、内部差し替えまたは追加APIとする。

### 2.4 Phase 3: SIMD backend

目的:

- Phase 1のscalar referenceと同じ意味論およびcanonical endpoint bit patternを維持したまま高速化する。

実装順:

1. scalar referenceとSIMD backendの差分試験基盤を追加する。
2. SIMD向けロード・ストアを追加する。
3. 加算・減算のbatch kernelを追加する。
4. AVX-512の方向付き丸め付き乗算・除算batch kernelを追加する。
5. AVX2 + FMA、AVX2 without FMA、SSE2、ARM64を個別に評価する。
6. 正確性と性能の両ゲートを通過した経路だけをproduction dispatchへ含める。

単一区間演算を無条件に最大幅vectorへ載せない。単一区間operatorは、benchmarkでscalarより有利と確認できた場合に限り内部backendを切り替える。

### 2.5 Phase 4: 四則演算以外

次の順序で小さな単位に分ける。

1. 集合・判定操作
   - `Contains`
   - subset判定
   - overlap判定
   - intersection
   - convex hull
2. 基本代数関数
   - reciprocal
   - absolute value
   - square
   - square root
   - integer power
   - fused multiply-add
3. 単調関数
   - exponential
   - logarithm
   - hyperbolic functions
4. 周期・特異点を持つ関数
   - sine
   - cosine
   - tangent
   - inverse trigonometric functions
5. 拡張機能
   - extended division
   - decorated interval / NaI
   - 厳密な文字列変換
   - interval splitting

周期関数、厳密な文字列変換、decorated intervalはAPIと正確性の負荷が大きいため、四則演算直後の同一作業単位には含めない。

## 3. 対象環境

### 3.1 Target Framework

Phase 1は`net10.0`を対象とする。

- パイロット段階では複数target frameworkの検証行列を増やさない。
- 後続のhardware intrinsicsも同じtarget framework上で評価する。
- 古いtarget frameworkの追加はPhase 2で利用者要件として判断する。

### 3.2 対象アーキテクチャ

Phase 1:

- Linux x64
- Linux ARM64

両アーキテクチャで同じpure-managed scalar algorithmを実行する。

Phase 3:

- x64 AVX-512F
- x64 AVX2 + FMA
- x64 AVX2 without FMA
- x64 AVX + FMA without AVX2
- x64 SSE2 without FMA
- ARM64 AdvSimd
- 未対応環境のscalar fallback

### 3.3 Runtime dependency

Phase 1のproduction packageはBCL以外のruntime dependencyを持たない。

`inari`、`kv`、ITF1788およびそれらのtoolchainは、test corpus生成・差分検証にのみ使用し、production packageから呼び出さない。

## 4. 公開API候補

Phase 1では以下をpilot APIとし、Phase 2完了時に固定する。

### 4.1 Assemblyとnamespace

```text
Assembly / package: Devo6.Interval
Namespace:          Devo6.Numerics
Type:               Interval
```

### 4.2 型定義

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

### 4.3 生成

`new Interval(lower, upper)`は次を満たす場合に成功する。

```text
lower <= upper
lower != +Infinity
upper != -Infinity
lower is not NaN
upper is not NaN
```

不正な場合は`ArgumentException`を送出する。`lower > upper`を自動的に`Empty`へ変換しない。

`TryCreate`は不正入力で`false`を返し、out値へ`Interval.Empty`を設定する。

`Point(value)`は有限の`double`のみを受け入れる。`NaN`および`±Infinity`は実数の点ではないため拒否する。

### 4.4 定数とdefault

```text
Empty  = 空集合
Entire = [-Infinity, +Infinity]
Zero   = [-0.0, +0.0]
```

`default(Interval)`は`Interval.Zero`と同値にする。

### 4.5 Lower / Upper

非空区間では正規化済みの端点を返す。空区間では両方とも`double.NaN`を返す。

```text
Interval.Empty.Lower -> NaN
Interval.Empty.Upper -> NaN
```

下限が0の場合は`-0.0`、上限が0の場合は`+0.0`を返す。

### 4.6 演算時の例外

公開APIから生成された`Interval`は常に有効である。数学的な定義域問題は区間値で表現し、四則演算時に`DivideByZeroException`等を送出しない。

```text
[1, 2] / [0, 0]  -> Empty
[1, 2] / [-1, 1] -> Entire
```

### 4.7 Phase 1で提供しないAPI

次はPhase 2で判断する。

```csharp
interval + 1.0
1.0 + interval
(Interval)1.0
```

`INumber<TSelf>`もPhase 1では実装しない。区間には自然な全順序がなく、通常数値型の契約をそのまま満たさないためである。

### 4.8 文字列表現

Phase 1の`ToString()`は診断用途とする。

```text
Empty  -> "Empty"
[1, 2] -> "[1, 2]"
Entire -> "[-Infinity, Infinity]"
```

数値はinvariant cultureとround-trip formatを使用する。永続化、通信、round-trip parsingの契約とはしない。

## 5. 内部表現

### 5.1 Phase 1の物理表現

```csharp
public readonly struct Interval
{
    private readonly double _negatedLower;
    private readonly double _upper;
}
```

外部区間`[lower, upper]`を次のように保持する。

```text
_negatedLower = -lower
_upper        =  upper
```

### 5.2 canonical state

```text
Zero   = [+0.0, +0.0]
Entire = [+Infinity, +Infinity]
Empty  = [canonical-qNaN, canonical-qNaN]
```

EmptyのNaN payloadは公開契約にしないが、production実装では固定bit patternを使用する。

### 5.3 内部不変条件

非空区間:

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

片側だけNaNの状態を作成してはならない。

### 5.4 zero正規化

constructor入力と演算結果の両方で次へ正規化する。

```text
external lower zero -> -0.0
external upper zero -> +0.0
internal Zero       -> [+0.0, +0.0]
```

### 5.5 raw constructor

演算結果用にvalidationを省略するprivate/internal constructorを設ける。

```csharp
private Interval(
    double negatedLower,
    double upper,
    RawIntervalTag _);
```

呼び出し側は、Emptyまたは有効なcanonical internal representationであることを保証する。

### 5.6 レイアウト非公開

内部が2個の`double`であること、size、field order、blittable ABIはpublic contractにしない。public field、raw bytes、`Vector128<double>`へのpublic castも提供しない。

## 6. 内部コンポーネント

### 6.1 予定構成

```text
src/Devo6.Interval/
  Devo6.Interval.csproj
  Interval.cs
  Interval.Operators.cs
  Interval.Formatting.cs
  Internal/
    DirectedRounding.cs
    ScalarIntervalKernel.cs
    IntervalSignClass.cs
    IntervalCanonicalizer.cs

tests/Devo6.Interval.Unit.Tests/
  ConstructionTests.cs
  EqualityTests.cs
  AdditionTests.cs
  SubtractionTests.cs
  MultiplicationTests.cs
  DivisionTests.cs
  DirectedRoundingTests.cs
  SpecialValueTests.cs
  Conformance/
    Ieee1788Phase1Tests.cs
    ConformanceCaseReader.cs
  ReferenceOracle/
    Binary64Rational.cs
    DirectedRoundingOracle.cs
    GoldenCaseReader.cs

tests/ReferenceData/
  ieee1788-phase1.jsonl
  directed-rounding-boundaries.jsonl
  reference-differential.jsonl
  reference-lock.json

tools/reference-oracles/
  inari-adapter/
  kv-adapter/
  generate-reference-data.*
```

### 6.2 責務

`Interval`:

- public value semantics
- validation
- properties
- operators entry point
- equality / Hash

`ScalarIntervalKernel`:

- Empty propagation
- 符号分類
- 四則演算の区間アルゴリズム
- canonical raw interval生成

`DirectedRounding`:

- `AddUp` / `AddDown`
- `MultiplyUp` / `MultiplyDown`
- `DivideUp` / `DivideDown`
- overflow / underflow / subnormal処理
- `NextUp` / `NextDown`

`IntervalCanonicalizer`:

- endpoint zero正規化
- Empty canonicalization

`IntervalSignClass`:

```text
Zero        [0, 0]
NonNegative 0 <= Lower かつZeroではない
NonPositive Upper <= 0 かつZeroではない
Mixed       Lower < 0 < Upper
```

Phase 1のhot pathにはinterface、virtual call、delegate tableを導入しない。

## 7. 方向付き丸めの共通契約

### 7.1 定義

許可されたbinary64 operandに対して次を返す。

```text
AddUp(x, y)        = 最小のdouble zで、実数としてのx+y <= z
AddDown(x, y)      = 最大のdouble zで、z <= 実数としてのx+y
MultiplyUp(x, y)   = 最小のdouble zで、実数としてのx*y <= z
MultiplyDown(x, y) = 最大のdouble zで、z <= 実数としてのx*y
DivideUp(x, y)     = 最小のdouble zで、実数としてのx/y <= z
DivideDown(x, y)   = 最大のdouble zで、z <= 実数としてのx/y
```

`NextUp`は`Math.BitIncrement`、`NextDown`は`Math.BitDecrement`を使用する。通常演算後に無条件で1 ULP広げない。

### 7.2 下向き演算の関係

実装では次の恒等関係を利用できる。

```text
AddDown(x, y)      = -AddUp(-x, -y)
SubtractDown(x, y) = -SubtractUp(y, x)
MultiplyDown(x, y) = -MultiplyUp(-x, y)
DivideDown(x, y)   = -DivideUp(-x, y)
```

本書では分岐の完全性を明確にするため、乗除算のUp/Down条件をそれぞれ記載する。

### 7.3 primitiveへ渡さない組合せ

区間kernelは次を事前分岐し、方向付き丸めprimitiveへ渡さない。

```text
+Infinity + -Infinity
0 * Infinity
0 / 0
Infinity / Infinity
denominator == 0
NaN operand
```

Debug buildでは前提条件をassertする。

### 7.4 overflowの方向別結果

有限な正確値がbinary64の有限範囲を超える場合:

```text
positive overflow:
  Up   -> +Infinity
  Down -> +double.MaxValue

negative overflow:
  Up   -> -double.MaxValue
  Down -> -Infinity
```

operandそのものがInfinityで正確値も非有界の場合は、該当するsigned Infinityを返す。

## 8. 加算・減算の方向付き丸め

### 8.1 TwoSum

有限operandかつoverflowなしの場合、TwoSumで次を得る。

```text
s = roundNearest(x + y)
e = exact(x + y) - s
```

概念コード:

```csharp
static (double Sum, double Error) TwoSum(double x, double y)
{
    double s = x + y;
    double bVirtual = s - x;
    double e = (x - (s - bVirtual)) + (y - bVirtual);
    return (s, e);
}
```

### 8.2 AddUp / AddDown

```text
AddUp:
  error > 0 -> NextUp(sum)
  error <= 0 -> sum

AddDown:
  error < 0 -> NextDown(sum)
  error >= 0 -> sum
```

Infinityとfinite overflowは§7.4に従って先に処理する。

### 8.3 減算

```text
SubtractUp(x, y)   = AddUp(x, -y)
SubtractDown(x, y) = AddDown(x, -y)
```

## 9. 乗算の方向付き丸め

### 9.1 定数

```csharp
SmallProductThreshold = 2^-969;
ProductScale          = 2^537;
```

`abs(product) >= 2^-969`は通常残差経路、`abs(product) < 2^-969`はscaled経路とする。閾値一致は通常経路である。

### 9.2 TwoProductFma

```csharp
static (double Product, double Error) TwoProductFma(double x, double y)
{
    double product = x * y;
    double error = Math.FusedMultiplyAdd(x, y, -product);
    return (product, error);
}
```

通常経路では、選択した閾値によりnonzero residualがbinary64で表現可能な範囲にあることを前提とする。この前提はexact rational oracleの境界試験で固定する。

### 9.3 MultiplyUp

以下を実装仕様とする。

```text
MultiplyUp(x, y):
  require x,y are not NaN
  require not (x == 0 and y is Infinity)
  require not (y == 0 and x is Infinity)

  if x == 0 or y == 0:
      return x * y

  if x or y is Infinity:
      return signed Infinity of x*y

  r = roundNearest(x * y)

  if r == +Infinity:
      return +Infinity                 // finite positive overflow

  if r == -Infinity:
      return -double.MaxValue          // finite negative overflow

  if abs(r) >= 2^-969:
      r2 = FMA(x, y, -r)
      if r2 > 0:
          return NextUp(r)
      return r

  sx = x * 2^537
  sy = y * 2^537
  (s, s2) = TwoProductFma(sx, sy)
  t = (r * 2^537) * 2^537

  if t < s or (t == s and s2 > 0):
      return NextUp(r)
  return r
```

scaled経路では`2^537`による各operandのscaleと`r`の2段scaleがoverflowしないことを、分岐条件とnonzero binary64 operandの指数範囲から保証する。2の整数乗によるscaleは、その範囲内では正確である。

### 9.4 MultiplyDown

```text
MultiplyDown(x, y):
  同じpreconditionとzero / Infinity分岐を行う

  r = roundNearest(x * y)

  if r == +Infinity:
      return +double.MaxValue          // finite positive overflow

  if r == -Infinity:
      return -Infinity                 // finite negative overflow

  if abs(r) >= 2^-969:
      r2 = FMA(x, y, -r)
      if r2 < 0:
          return NextDown(r)
      return r

  sx = x * 2^537
  sy = y * 2^537
  (s, s2) = TwoProductFma(sx, sy)
  t = (r * 2^537) * 2^537

  if t > s or (t == s and s2 < 0):
      return NextDown(r)
  return r
```

### 9.5 scaled比較の意味

```text
t        = roundNearest(x*y)を2^1074倍した値
s + s2   = x*yを2^1074倍したexact product decomposition
```

したがって:

- `t < s`、または`t == s && s2 > 0`: `r`は真値より小さい。
- `t > s`、または`t == s && s2 < 0`: `r`は真値より大きい。
- `t == s && s2 == 0`: `r`は正確である。

このtie条件を省略してはならない。

## 10. 除算の方向付き丸め

### 10.1 定数と厳密な境界

```csharp
SmallNumeratorThreshold = 2^-969;
LargeDenominatorLimit   = 2^918;
DivisionScale           = 2^105;
MinimumSubnormal        = 2^-1074;
```

境界判定は次で固定する。

```text
abs(xn) < 2^-969  -> small numerator分岐
abs(xn) == 2^-969 -> 通常経路

small numeratorかつabs(yn) < 2^918  -> 両operandを2^105倍
small numeratorかつabs(yn) >= 2^918 -> zero/min-subnormal early return
```

`abs(yn) == 2^918`はearly-return側である。これにより`yn * 2^105`のoverflowを避ける。

### 10.2 denominator正符号化

finite nonzero denominatorについて、比較前に必ず`yn > 0`へ正規化する。

```text
if y < 0:
    xn = -x
    yn = -y
else:
    xn = x
    yn = y
```

`xn/yn`は元の`x/y`と等しい。

### 10.3 special value

区間kernelでundefined pairを除外した後、primitiveは次を扱う。

```text
x == 0, finite nonzero y -> signed zero
finite nonzero x, y is Infinity -> signed zero
x is Infinity, finite nonzero y -> signed Infinity
```

finite値同士のoverflowは§7.4に従う。

### 10.4 DivideUp

```text
DivideUp(x, y):
  require x,y are not NaN
  require y != 0
  require not (x is Infinity and y is Infinity)

  handle zero and Infinity cases

  normalize to xn / yn with yn > 0

  if abs(xn) < 2^-969:
      if abs(yn) < 2^918:
          xn = xn * 2^105
          yn = yn * 2^105
      else:
          if xn < 0:
              return +0.0
          else:
              return +2^-1074

  q = roundNearest(xn / yn)

  if q == +Infinity:
      return +Infinity

  if q == -Infinity:
      return -double.MaxValue

  r  = roundNearest(q * yn)
  r2 = FMA(q, yn, -r)

  if r < xn or (r == xn and r2 < 0):
      return NextUp(q)
  return q
```

`r == xn`だけではexact relationを判定できない。`r2 < 0`ならexactな`q*yn`は`xn`より小さく、`q`は真の商より小さいため上向き補正が必要である。

### 10.5 DivideDown

```text
DivideDown(x, y):
  require x,y are not NaN
  require y != 0
  require not (x is Infinity and y is Infinity)

  handle zero and Infinity cases

  normalize to xn / yn with yn > 0

  if abs(xn) < 2^-969:
      if abs(yn) < 2^918:
          xn = xn * 2^105
          yn = yn * 2^105
      else:
          if xn < 0:
              return -2^-1074
          else:
              return +0.0

  q = roundNearest(xn / yn)

  if q == +Infinity:
      return +double.MaxValue

  if q == -Infinity:
      return -Infinity

  r  = roundNearest(q * yn)
  r2 = FMA(q, yn, -r)

  if r > xn or (r == xn and r2 > 0):
      return NextDown(q)
  return q
```

`r2 > 0`ならexactな`q*yn`は`xn`より大きく、`q`は真の商より大きいため下向き補正が必要である。

### 10.6 scaling後の残差表現可能性

small numeratorをscaleする場合、最小nonzero binary64の`2^-1074`は`2^-969`へ移動する。これにより`q*yn`のproduct decompositionで必要な最下位bitがbinary64のsubnormal範囲に残る。

`yn < 2^918`のstrict比較により、`yn * 2^105`はfinite範囲に収まる。large denominatorはscaleせず、符号と方向からtightなzero/min-subnormalを直接返す。

## 11. 区間四則演算

以下では`RD`を下向き丸め、`RU`を上向き丸めとする。operandのどちらかがEmptyならEmptyを返す。

### 11.1 加算

```text
[a, b] + [c, d]
= [RD(a+c), RU(b+d)]
```

内部表現:

```text
[-a, b] + [-c, d]
= [RU(-a-c), RU(b+d)]
= [-RD(a+c), RU(b+d)]
```

### 11.2 減算

```text
[a, b] - [c, d]
= [RD(a-d), RU(b-c)]
```

内部表現では右operandのlane交換に相当する。

```text
result.negatedLower = RU((-a) + d)
result.upper        = RU(b + (-c))
```

### 11.3 単項マイナス

```text
-[a, b] = [-b, -a]
[-a, b] -> [b, -a]
```

内部表現ではlane交換で得る。

### 11.4 乗算の符号分類

```text
Z: [0,0]
P: 0 <= lower かつZではない
N: upper <= 0 かつZではない
M: lower < 0 < upper
```

`A=[a,b]`、`B=[c,d]`とする。

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
| M | M | `min(RD(a*d), RD(b*c))` | `max(RU(a*c), RU(b*d))` |

Zeroを先に処理し、`0 * Infinity`をendpoint primitiveへ渡さない。

### 11.5 除算: denominatorが0を含まない場合

`A=[a,b]`、`B=[c,d]`とする。

Bが正、`0 < c <= d`:

| A | Lower | Upper |
|---|---|---|
| P | `RD(a/d)` | `RU(b/c)` |
| N | `RD(a/c)` | `RU(b/d)` |
| M | `RD(a/c)` | `RU(b/c)` |

Bが負、`c <= d < 0`:

| A | Lower | Upper |
|---|---|---|
| P | `RD(b/d)` | `RU(a/c)` |
| N | `RD(b/c)` | `RU(a/d)` |
| M | `RD(b/d)` | `RU(a/d)` |

reciprocal intervalを一度作って乗算せず、端点を直接除算する。

### 11.6 denominatorが`[0,0]`

```text
A / [0,0] -> Empty
```

### 11.7 denominatorが0に片側から接する場合

`B=[0,d]`, `d>0`:

| A | Result |
|---|---|
| Z | `Zero` |
| P | `[RD(a/d), +Infinity]` |
| N | `[-Infinity, RU(b/d)]` |
| M | `Entire` |

`B=[c,0]`, `c<0`:

| A | Result |
|---|---|
| Z | `Zero` |
| P | `[-Infinity, RU(a/c)]` |
| N | `[RD(b/c), +Infinity]` |
| M | `Entire` |

### 11.8 denominatorが0を跨ぐ場合

```text
c < 0 < d
Zero / B -> Zero
その他    -> Entire
```

nonzero numeratorの結果集合は非連続になり得るため、bare `Interval`ではconvex hullとしてEntireを返す。2区間を返すextended divisionはPhase 4の別APIとする。

## 12. 等値性、Hash、正規化

### 12.1 等値性

- Empty同士は等しい。
- 非空区間はcanonical Lower / Upperが等しい場合に等しい。
- 入力時の`+0.0`と`-0.0`の差は等値性へ影響しない。
- NaIはbare `Interval`に存在しない。

### 12.2 Hash

- Emptyは固定Hashとする。
- zero endpointはcanonical bit patternでHashする。
- NaN payloadをHashへ直接使用しない。

### 12.3 順序比較

`<`, `<=`, `>`, `>=`はPhase 1で定義しない。subset、precedes、element-wise orderは別概念である。

## 13. Oracleと参照harness

### 13.1 責務の優先順位

結果判定の優先順位を次で固定する。

1. **Exact rational oracle**: 数学的なprimary oracle
2. **IEEE 1788.1 Phase 1 conformance matrix**: 区間演算の要求意味論
3. **inari adapter**: set-based interval semanticsとendpoint differential oracle
4. **kv adapter**: 互換な有限endpointに対するdirected-rounding primitive oracle

既存libraryの結果がexact rational oracleと矛盾する場合、既存libraryへ無条件に合わせない。入力意味論、標準適合範囲、adapter、既存libraryの実装を調査する。

### 13.2 Exact rational oracle

テスト専用に有限binary64をexact rationalへ分解する。

```text
significand * 2^exponent
```

- 加算、減算、乗算は`BigInteger`でexact valueを生成する。
- 除算は分子・分母のrationalとして保持する。
- BCL nearest resultをexact rationalへ変換し、真値との大小でUp/Down補正を決める。
- production projectへ`BigInteger` oracleを含めない。

### 13.3 inari adapter

固定参照:

```text
Repository: unageek/inari
Commit:     18b83a571d7681c76067bc38d90a74e8be29f545
```

Rustのtest-only CLI adapterを`tools/reference-oracles/inari-adapter/`へ置く。

- stdin: JSON Lines
- stdout: JSON Lines
- 入力: operationとoperand endpoint bit pattern
- 出力: state、lower bits、upper bits、error
- EmptyのNaN payloadは比較しない。
- signed zeroはDevo6.Intervalのcanonical ruleへ変換して比較する。

inariは、Empty / Entire、zero-crossing division、set-based arithmetic、tight endpointのsecondary oracleとする。

### 13.4 kv adapter

固定参照:

```text
Repository: mskashi/kv
Commit:     c7f8f2324a0e403cca6b39f46088a22843d440db
File:       kv/rdouble-nohwround.hpp
```

C++のtest-only CLI adapterを`tools/reference-oracles/kv-adapter/`へ置く。

比較対象:

- `add_up` / `add_down`
- `mul_up` / `mul_down`
- `div_up` / `div_down`
- primitive前提条件を満たす有限または明示的に互換な特殊operand

kvの通常`interval`除算はdenominatorが0を含む場合にDevo6.Interval / inariのset-based semanticsと互換でないため、その区間演算結果をoracleにしない。該当caseは`expected-difference: kv-zero-containing-denominator`として記録し、kv adapterを実行しない。

### 13.5 reference lock

`tests/ReferenceData/reference-lock.json`に次を保存する。

- inari commit SHA
- kv commit SHA
- ITF1788 commit SHA
- adapter source hash
- generator source hash
- Rust toolchain version
- C++ compilerおよびversion
- target triple
- generator command
- output corpus SHA-256
- source license / notice path

reference更新は専用PRで行い、lockとgenerated corpusの差分を同時にreviewする。

### 13.6 corpus形式

JSON Linesを使用し、数値はdecimal文字列ではなく16桁hexのbinary64 bit patternで保持する。

```json
{"schema":1,"caseId":"mul-scaled-t-lt-s","kind":"rounding","operation":"mulUp","xBits":"216b5087a9deee3d","yBits":"1e04591a0fee6d8d","expectedBits":"...","source":"exact-rational"}
{"schema":1,"caseId":"div-cross-zero","kind":"interval","operation":"div","left":{"lowerBits":"3ff0000000000000","upperBits":"4000000000000000"},"right":{"lowerBits":"bff0000000000000","upperBits":"3ff0000000000000"},"expected":{"state":"entire"},"source":"ieee1788-matrix"}
```

実際のcorpusでは次を必須fieldとする。

- schema
- caseId
- operation
- operand bit patternsまたはstate
- expected state / endpoint bits
- source
- source revision
- applicability
- expected-difference reason

caseは`caseId`でsortし、generatorの実行順やdictionary順に依存させない。

### 13.7 実行方式

通常CIはrepositoryへcommitされたgolden corpusをC# testから読み込む。毎回Rust/C++ toolchainをproduction test jobへ導入しない。

reference-integrity jobは次を行う。

1. `reference-lock.json`の固定SHAをcheckoutする。
2. adapterを固定toolchainでbuildする。
3. generator inputを各adapterへ渡す。
4. JSONLをcanonicalizeして再生成する。
5. commit済みcorpusとbyte-for-byte比較する。
6. 差分、adapter stdout/stderr、toolchain情報をartifactへ保存する。

## 14. IEEE 1788.1 conformance strategy

### 14.1 基準

意味論の基準は基本設計どおりIEEE 1788.1-2017とする。外部ITF1788 corpusは補助的な入力源であり、標準そのものの代替ではない。

ITF1788固定参照:

```text
Repository: unageek/ITF1788
Commit:     d8c2a64478ebdc9cbde6ccef33eaad3bed60ed81
```

inariがsubmoduleとして固定する同じSHAを使用する。

### 14.2 Phase 1適用matrix

| IEEE operation concept | Devo6.Interval API | Phase 1 |
|---|---|---|
| `empty` | `Interval.Empty` | required |
| `entire` | `Interval.Entire` | required |
| numeric `numsToInterval` | constructor / `TryCreate` | required |
| `inf` | `Lower` | required |
| `sup` | `Upper` | required |
| `isEmpty` | `IsEmpty` | required |
| `isEntire` | `IsEntire` | required |
| `isSingleton` | `IsSingleton` | required |
| `equal` | `Equals`, `==` | required |
| `neg` | unary `-` | required |
| `add` | binary `+` | required |
| `sub` | binary `-` | required |
| `mul` | binary `*` | required |
| `div` | binary `/` | required |

### 14.3 Phase 4へ延期する項目

- text constructor / exact text I/O
- reciprocal、square、sqrt、fma、power
- transcendental functions
- set operationsと各種relation
- reduction
- extended / two-output division
- decorated interval / NaI / decoration propagation

延期項目をPhase 1 conformance failureとして数えない。matrix上で`deferred-phase-4`と明示する。

### 14.4 ITF1788入力

Phase 1 generatorは固定SHAから次の互換caseを抽出・適応する。

- `itl/ieee1788-constructors.itl`
- `itl/libieeep1788_elem.itl`の`neg/add/sub/mul/div`
- `itl/libieeep1788_bool.itl`の`equal/isEmpty/isEntire/isSingleton`
- `itl/libieeep1788_num.itl`の`inf/sup`

ITF1788がIEEE 1788-2015向けである点を考慮し、IEEE 1788.1と本基本設計に一致するcaseだけをPhase 1 required corpusへ採用する。除外caseはoperation、理由、source locationをmanifestへ残す。

### 14.5 adaptation rules

- lower zeroは`-0.0`、upper zeroは`+0.0`へcanonicalizeする。
- EmptyのNaN payloadは比較しない。
- decorated resultはPhase 1で読み込まない。
- string parsingを必要とするcaseはPhase 4へ延期する。
- constructor errorはC#の`ArgumentException`または`TryCreate=false`へ対応付ける。
- interval resultはstateとcanonical endpoint bit patternで比較する。

### 14.6 conformance evidence

各architecture jobは次を生成する。

- `conformance-summary.json`
- passed / failed / deferred / excluded件数
- failed caseのsource revisionとcaseId
- actual / expected endpoint bits
- adaptation rule

これらを`if: always()`でartifactへ保存する。

## 15. 決定的テストfixture

random/property testとは別に、以下を固定caseとして実装する。

### 15.1 閾値の直前・一致・直後

```text
2^-969 previous = bits 0x035fffffffffffff
2^-969          = bits 0x0360000000000000
2^-969 next     = bits 0x0360000000000001

2^918 previous  = bits 0x794fffffffffffff
2^918           = bits 0x7950000000000000
2^918 next      = bits 0x7950000000000001

2^-1074         = bits 0x0000000000000001
```

`x=1.0`との乗算、および正負符号を反転したcaseで、`2^-969`のbelow/equal/aboveがscaled/normalの期待経路へ入ることを確認する。

除算では`abs(xn)`のbelow/equal/aboveと、`abs(yn)`のbelow/equal/aboveを直積で組み合わせる。`2^918`の一致はlarge-denominator early-return側であることを固定する。

### 15.2 large-denominator early return

`xBits=0x035fffffffffffff`、`yBits=0x7950000000000000`を基準に正負を作る。

| Exact quotient sign | Direction | Expected primitive result |
|---|---|---|
| positive | Up | `+2^-1074` |
| positive | Down | `+0.0` |
| negative | Up | `+0.0` |
| negative | Down | `-2^-1074` |

`yBits=0x794fffffffffffff`ではscale継続側へ入り、`yBits=0x7950000000000001`ではearly-return側へ入ることを確認する。

### 15.3 乗算scaled比較

次の固定binary64 operandをcorpusへ保存する。

| Case | xBits | yBits | Required branch |
|---|---|---|---|
| `mul-scaled-t-lt-s` | `216b5087a9deee3d` | `1e04591a0fee6d8d` | `t < s`; UpのみNextUp |
| `mul-scaled-t-gt-s` | `8b8ab461601ec773` | `33c03ee4daaa7148` | `t > s`; DownのみNextDown |
| `mul-scaled-eq-pos-residual` | `b8aefe57fced900a` | `88b7778db0690811` | `t == s && s2 > 0`; UpのみNextUp |
| `mul-scaled-eq-neg-residual` | `b2e664c6cc3b90be` | `8e6b00818bab3ede` | `t == s && s2 < 0`; DownのみNextDown |
| `mul-scaled-exact` | `3ff0000000000000` | `0000000000000001` | `t == s && s2 == 0`; 補正なし |

expected endpoint bitsはexact rational oracleで生成し、表のbranch assertionと結果assertionを両方行う。production methodをpublicにしないため、branch coverageはinternal test hookまたはcoverage instrumentationで確認する。

### 15.4 除算rounded-product tie

次の固定caseで、rounded high productがnumeratorと等しくてもresidualで補正が変わることを確認する。

| Case | xBits | yBits | qBits | Relation after denominator normalization | Expected correction |
|---|---|---|---|---|---|
| `div-tie-positive-residual` | `35b62b4b61f6a01a` | `6a4b103b1dfd16c0` | `0b5a36846200f80c` | `r == xn && r2 > 0` | DownのみNextDown |
| `div-tie-negative-residual` | `0e0db74836096727` | `a9ad3e48c2f627a6` | `a4504233b80eaec4` | `r == xn && r2 < 0` | UpのみNextUp |

さらに`r < xn`、`r > xn`、`r == xn && r2 == 0`を各1case固定する。

### 15.5 overflow

最低限次を方向別に固定する。

```text
+double.MaxValue * 2:
  Up   -> +Infinity
  Down -> +double.MaxValue

-double.MaxValue * 2:
  Up   -> -double.MaxValue
  Down -> -Infinity

+double.MaxValue / 2^-1074:
  Up   -> +Infinity
  Down -> +double.MaxValue

-double.MaxValue / 2^-1074:
  Up   -> -double.MaxValue
  Down -> -Infinity
```

Infinity operandによるexact Infinityと、finite operandによるoverflowを別caseで確認する。

### 15.6 exact result

2の整数乗同士、`1*2^-1074`、`1/2`等について、residualが0で不要な`NextUp` / `NextDown`を行わないことを固定する。

## 16. Property testとcross-backend test

### 16.1 property test

ランダムな有効区間に対して次を検証する。

- resultがexact sampled valueを包含する。
- `x + y == y + x`
- `x * y == y * x`
- `-(-x) == x`
- `x + Zero == x`
- nonempty `x`について`x * Zero == Zero`
- resultの内部不変条件が保たれる。

分配法則の等号は依存性問題により一般には要求しない。

### 16.2 Phase 3 differential test

同じ入力をscalarと各SIMD backendへ渡し、次を要求する。

- Empty / Entire stateが一致する。
- nonempty canonical endpoint bit patternが一致する。
- CPU非対応時はscalar fallbackと一致する。
- scalarより広いが包含しているだけのSIMD結果は不合格とする。

## 17. CIとfailure diagnostics

Phase 1で最初のexecutable projectとtest projectを追加するPRに、同時にCI workflowを追加する。

### 17.1 architecture matrix

同じcommit、同じtest assembly、同じreference corpusを次で実行する。

```yaml
strategy:
  matrix:
    include:
      - architecture: x64
        runs-on: ubuntu-24.04
      - architecture: arm64
        runs-on: ubuntu-24.04-arm
```

runner labelが将来変更された場合は同等のLinux x64 / Linux ARM64 runnerへ置換できるが、両architectureで同一suiteを実行するgate自体は省略しない。

### 17.2 各architectureの実行内容

- deterministic unit tests
- directed-rounding boundary fixtures
- exact rational oracle tests
- IEEE 1788.1 Phase 1 conformance tests
- committed golden corpus comparison
- property tests with fixed seed
- canonical result corpus生成

### 17.3 architecture間比較

各jobはcaseId順にsortした`canonical-results.jsonl`を生成する。このfileには結果bit patternだけを含め、runtime metadataは別fileにする。

後続jobはx64 / ARM64の`canonical-results.jsonl`をbyte-for-byte比較し、SHA-256も記録する。不一致時は最初の差分だけで打ち切らず、全caseId差分をartifactへ保存する。

### 17.4 failure artifact

各architecture jobは成功・失敗に関係なく`if: always()`で少なくとも次を保存する。

- `.trx`等のtest result
- test runner標準出力
- test runner標準エラー
- diagnostic verbosity log
- runtime / OS / architecture / CPU feature情報
- reference-lock snapshot
- conformance summary
- canonical result corpus
- mismatch case input
- exact rational result
- Devo6.Interval result
- inari resultまたは`not-applicable`
- kv primitive resultまたは`not-applicable`
- expected-difference reason

### 17.5 exact-head CI

PRのCI確認は、確認時点のPR current HEAD SHAとworkflow runの`head_sha`が一致するrunだけを対象とする。HEAD更新後は新HEADのrunを確認する。一致するrunがない場合はCI未実施とし、別SHAのrunを代用しない。

本PRはdocumentation-onlyであり、現時点のrepositoryにはexecutable targetとworkflowがないため、本設計変更ではworkflowを追加しない。

## 18. API確定ゲート

### 18.1 利用性

- 代表的な計算例をoperatorで記述できる。
- Empty / Entireを明示的に判定できる。
- 不正constructorと数学的にEmptyになる演算を区別できる。
- signed zeroが予期しない分岐を生じさせない。

### 18.2 正確性

- §15の全fixtureが成功する。
- exact rational oracleとの差がない。
- Phase 1 conformance matrixが全件passする。
- inariとの差異が0件、または各差異が承認済みである。
- kv primitiveとの互換対象差異が0件、または各差異が承認済みである。
- Linux x64 / Linux ARM64のcanonical result corpusが一致する。

### 18.3 API baseline

- public API一覧をbaseline fileとして保存する。
- 以後のPRでpublic API差分をCI検出する。
- breaking changeは`doc/Design/BreakingChanges.md`へ理由と移行方法を記録する。

### 18.4 性能baseline

- 基本演算がheap allocationを発生させない。
- scalar operationのBenchmarkDotNet baselineを保存する。
- Debug assertionがRelease hot pathに残らない。

性能値自体はAPI確定のcorrectness gateにはしない。

## 19. SIMD capabilityとdispatch

### 19.1 capabilityは独立判定する

x86 FMAをAVX2またはSSE2の付随機能として扱わない。少なくとも次を個別に判定する。

```text
Avx512F.IsSupported
Avx2.IsSupported
Avx.IsSupported
Fma.IsSupported
Sse2.IsSupported
AdvSimd.Arm64.IsSupported
```

backendは型名ではなく、operationごとの利用可能kernelとして選ぶ。

### 19.2 capability matrix

| Environment | Add/Sub | Mul/Div | Initial production policy |
|---|---|---|---|
| x64 AVX-512F | packed embedded rounding | packed embedded rounding | 4区間batch候補 |
| x64 AVX2 + FMA | vector TwoSum候補 | vector FMA residual候補 | correctness/benchmark通過後に採用 |
| x64 AVX2 without FMA | vector TwoSum候補 | scalar fallback | mul/divのvector Dekkerは別評価 |
| x64 AVX + FMA without AVX2 | vector TwoSum候補 | vector FMA residual候補、bit correctionは制約あり | benchmark後に採否決定 |
| x64 SSE2 without FMA | Vector128 TwoSum候補 | scalar fallback | add/subのみ候補 |
| ARM64 AdvSimd | vector TwoSum候補 | fused residualのexactness確認まではscalar fallback | 個別差分試験後に採用 |
| その他 | scalar | scalar | 常時利用可能 |

AVX2 without FMAおよびSSE2では、Phase 3初期実装のmul/divをscalar fallbackとする。vectorized TwoProduct/Dekkerは、正確性証明とbenchmarkで有効性が確認できた後の追加候補とする。

ARM64のfused multiply-add/subtract intrinsicはx86の`Fma.IsSupported`とは別経路である。`AdvSimd.Arm64.IsSupported`と実際に使用するintrinsicを個別にgateし、scalar referenceとのbitwise differential testを通過するまでmul/div production dispatchへ入れない。

### 19.3 AVX-512 batch layout

4区間を1個の`Vector512<double>`へ配置する。

```text
[-L0,U0,-L1,U1,-L2,U2,-L3,U3]
```

加算:

```text
result = Add(left, right, ToPositiveInfinity)
```

結果はそのまま4個の内部区間表現になる。末尾が4区間未満の場合はscalarで処理する。

### 19.4 batch API候補

```csharp
public static class IntervalBatch
{
    public static void Add(
        ReadOnlySpan<Interval> left,
        ReadOnlySpan<Interval> right,
        Span<Interval> destination);

    public static void Subtract(
        ReadOnlySpan<Interval> left,
        ReadOnlySpan<Interval> right,
        Span<Interval> destination);

    public static void Multiply(
        ReadOnlySpan<Interval> left,
        ReadOnlySpan<Interval> right,
        Span<Interval> destination);

    public static void Divide(
        ReadOnlySpan<Interval> left,
        ReadOnlySpan<Interval> right,
        Span<Interval> destination);
}
```

これはPhase 2の基本`Interval` API freezeとは別のadditive APIである。長さ不一致、overlap、in-place可否はPhase 3開始時のAPI reviewで確定する。

### 19.5 dispatch順序

概念上のx64 dispatch:

```text
AVX-512F exact packed kernel
  -> AVX2 + FMA exact kernel
  -> AVX2 add/sub-only kernel + scalar mul/div
  -> AVX + FMA exact candidate
  -> SSE2 add/sub-only kernel + scalar mul/div
  -> scalar
```

ARM64はAdvSimd kernelまたはscalarを選ぶ。利用者へbackend選択optionを最初から公開しない。test / benchmarkだけがinternal hookでbackendを固定できるようにする。

### 19.6 SIMD完了条件

- scalar backendとcanonical bitwise equivalent
- 非対応CPUでscalar fallback
- Empty、zero、Infinity、subnormalを含む差分試験成功
- 各capability組合せを強制したtest成功
- 単一区間またはbatchで測定可能な性能改善
- 性能改善のない経路をproduction dispatchへ含めない

## 20. Native backendの判断

Phase 1、Phase 2およびPhase 3の初期production implementationはmanaged-onlyとする。scalar operatorごとのP/Invokeは採用しない。

native backendの選択肢自体は破棄せず、次のdecision gateへ延期する。

1. Phase 3完了後のlarge-batch benchmark
2. Phase 4で超越関数の正しいmanaged実装方式を決める時点

採用条件:

- managed実装では利用できない方向付き丸めまたは数学関数能力が必要
- interop、copy、dispatchを含めても実利用workloadで有意な改善がある
- scalar referenceと同じset semanticsとcanonical endpointを返す
- x64 / ARM64 / deployment targetの配布方法を維持できる
- ABI、例外、thread safety、NativeAOT、trimmingへの影響を受容できる
- license / third-party noticeを満たす

nativeを採用する場合も公開`Interval`型は変更せず、large batchまたは重い初等関数の内部backendに限定する。判定結果はdesign updateとbenchmark reportに残す。

## 21. 四則演算以外の設計原則

四則演算以外は`IntervalMath`を第一候補とする。

```csharp
public static class IntervalMath
{
    public static Interval Sqrt(Interval value);
    public static Interval Exp(Interval value);
    public static Interval Log(Interval value);
    public static Interval Sin(Interval value);
}
```

bare intervalは関数定義域との共通部分を評価する。

```text
Sqrt([-1,4])  -> [0,2]
Sqrt([-4,-1]) -> Empty
```

定義域の一部が欠けた事実は、将来decorated intervalで表現する。

## 22. 性能、thread safety、AOT

Phase 1から次を守る。

- `readonly struct`
- heap allocationなし
- global rounding mode変更なし
- productionの`BigInteger`利用なし
- hot pathのvirtual/interface/delegate dispatchなし
- 無条件1 ULP拡張なし
- reciprocalを経由した二重丸めなし
- internal raw constructorによるvalidation重複回避

`Interval`はimmutable value typeとする。reflection、runtime code generation、dynamic assembly、native resolverをPhase 1で使用しない。NativeAOTとtrimmingを妨げない構造とする。

## 23. 「同等の計算結果」の定義

四則演算では次を要求する。

1. IEEE 1788.1-oriented set-based resultを使用する。
2. 単一区間で表せない場合はconvex hullを返す。
3. 各有限端点を指定方向へ正しく丸めた最も内側のbinary64とする。
4. signed zeroをDevo6.Intervalのcanonical ruleへ正規化する。
5. EmptyのNaN payloadを比較対象外とする。

この条件を満たす場合、nonempty endpointは原則として一意である。inari / kvとの差異よりexact rational oracleと採用した区間意味論を優先する。

## 24. 参照実装移植とlicense

`kv`または`inari`のコードを翻案・移植する場合:

- 参照元commit SHAをsource commentへ記録する。
- MIT Licenseのcopyrightとpermission noticeをthird-party noticeへ含める。
- 参照元test caseを移植した場合も出典を記録する。

ITF1788のtest dataやgeneratorを利用・再配布する場合は、そのlicenseとNOTICEをtest assetに付随させる。production packageに不要なexternal test toolを同梱しない。

## 25. 未確定事項

Phase 1開始を妨げない項目だけを残す。

- namespaceを`Devo6.Numerics`で確定するか
- constructorとfactory-only APIのどちらを正式採用するか
- scalar overload / conversion
- generic math operator interface
- `ToString`の正式format
- public batch APIの名称とoverlap規則
- `Vector128<double>`を物理fieldとして保持するか
- AVX2 / ARM64 kernelのperformance採否
- parsing / serialization format
- decorated intervalとextended divisionの公開時期
- native backend decision gateの最終結果

方向付き丸めの分岐、閾値、tie条件、Phase 1 conformance範囲、x64 / ARM64 gateは未確定事項に含めない。

## 26. 参照

### 26.1 基本設計

- `doc/Design/basic/IntervalArithmetic.md`

### 26.2 inari

- Repository: `unageek/inari`
- Pinned commit: `18b83a571d7681c76067bc38d90a74e8be29f545`
- License: MIT

### 26.3 kv

- Repository: `mskashi/kv`
- Pinned commit: `c7f8f2324a0e403cca6b39f46088a22843d440db`
- Reference file: `kv/rdouble-nohwround.hpp`
- License: MIT

### 26.4 ITF1788

- Repository: `unageek/ITF1788`
- Pinned commit: `d8c2a64478ebdc9cbde6ccef33eaad3bed60ed81`
- Usage: test-only conformance corpus source / generator

### 26.5 .NET

- .NET releases and support
- `Math.FusedMultiplyAdd`
- `Math.BitIncrement` / `Math.BitDecrement`
- `Avx512F`
- `Avx2`
- `Avx`
- `Fma`
- `Sse2`
- `AdvSimd.Arm64`

## 27. Review finding closure

| Finding | Disposition | Design evidence |
|---|---|---|
| F-PR3-001 High | addressed | §9、§10でscaled比較、tie、large-denominator、overflow、閾値一致を確定 |
| F-PR3-002 Medium | addressed | §3.2、§19でISA/FMAを独立判定しfallback matrixを定義 |
| F-PR3-004 Medium | addressed | §14でpinned ITF1788、Phase 1 matrix、adaptation、artifactを定義 |
| F-PR3-005 Medium | addressed | §13でoracle責務、adapter、lock、JSONL、kv非互換除外を定義 |
| F-PR3-006 Medium | addressed | §17でLinux x64 / ARM64 matrixとarchitecture間corpus比較を定義 |
| F-PR3-007 Medium | addressed | §15で閾値bit pattern、scaled4分岐、division tie、overflow fixtureを定義 |
| F-PR3-008 Low | addressed | §20でmanaged-only範囲とnative再検討gateを定義 |
| F-PR3-003 withdrawn | no implementation action | exhaustive reviewのerratumに従いactive findingから除外 |

## 28. Phase 1実装開始条件

本書のfix verification完了後、次の単位でPhase 1を開始する。

1. solution / project / x64・ARM64 CI・diagnostic artifact基盤
2. `Interval`生成・状態・正規化
3. exact rational oracleと固定boundary corpus
4. managed directed roundingの加減算
5. 区間加算・減算
6. managed directed roundingの乗算
7. 区間乗算
8. managed directed roundingの除算
9. 区間除算
10. IEEE 1788.1 Phase 1 conformance harness
11. inari / kv reference adapterとgolden corpus
12. sample / API評価report

各論理単位はTDDで、先に失敗testを追加し、失敗を確認してから実装する。変更はreview可能な小さなcommitとしてpushする。