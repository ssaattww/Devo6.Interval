# 区間演算 詳細設計

## 1. 文書情報

- 対象リポジトリ: `ssaattww/Devo6.Interval`
- 対象ライブラリ: `Devo6.Interval`
- 設計対象: IEEE 754 binary64 (`double`) を端点とする bare interval
- 基本設計: `doc/Design/basic/IntervalArithmetic.md`
- 作成日: 2026-08-29

本書は、基本設計で決定した区間意味論、`[-Lower, Upper]` 内部表現、外向き丸めおよび SIMD 方針を、実装可能な粒度まで具体化する。

## 2. 開発フェーズの評価と結論

次の順序で開発する。

1. SIMD を使用しない四則演算対応のパイロット版を作成する。
2. パイロット版の利用結果を基に、基本 `Interval` API を確定する。
3. 確定した API を変更せず、SIMD バックエンドを追加する。
4. 四則演算以外の区間操作および初等関数を段階的に追加する。

この順序は妥当である。区間意味論と公開 API の問題を、SIMD 最適化の問題から分離できるためである。

ただし、パイロット版の対象は演算子 `+`, `-`, `*`, `/` だけには限定しない。API を評価するため、次の最小限の値型契約も同じ段階で実装する。

- 区間生成と入力検証
- `Empty`, `Entire`, `Zero`
- `Lower`, `Upper`
- `IsEmpty`, `IsEntire`, `IsSingleton`
- 等値比較と Hash
- 単項マイナス
- 特殊値および signed zero の正規化
- 診断用文字列表現

これらがない状態では、演算結果、例外、特殊値の扱いを利用側から評価できず、API 確定後に破壊的変更が発生しやすい。

## 3. フェーズ定義

### 3.1 Phase 0: 設計・検証基盤

本書の作成を Phase 0 とする。

成果物:

- 公開 API 候補
- 内部不変条件
- 四則演算アルゴリズム
- 方向付き丸めアルゴリズム
- TDD のテスト分類
- 参照実装との比較方法
- API 確定条件
- SIMD 導入条件

完了条件:

- Phase 1 が追加の基本設計判断なしで開始できる。
- 未確定事項が Phase 1 の実装を妨げない範囲に限定されている。

### 3.2 Phase 1: managed scalar パイロット版

目的:

- SIMD や native ライブラリに依存せず、区間意味論と API を検証する。
- 四則演算について、真値を包含するだけでなく、正しく方向丸めされた binary64 端点を返す。
- 後続 SIMD 実装の基準となる scalar reference backend を作る。

対象:

- `Interval` 値型
- 区間生成と基本状態
- 加算、減算、単項マイナス、乗算、除算
- managed の方向付き丸め
- 単体テスト、参照値テスト、ランダム試験
- 最小サンプル
- CI と失敗診断 artifact

対象外:

- SIMD intrinsics
- CPU の丸めモード変更
- native ライブラリ呼び出し
- `sqrt`, `pow`, `exp`, `log`, `sin`, `cos` 等
- decorated interval / NaI
- 厳密な文字列からの区間変換
- 区間分割、Affine Arithmetic、Taylor Model

### 3.3 Phase 2: 基本 API 確定

目的:

- Phase 1 の利用性、特殊値、例外、命名および性能特性を確認し、基本 API を固定する。

確定対象:

- package / assembly / namespace
- `Interval` の生成方法
- 基本プロパティ
- `Empty` / `Entire` の扱い
- 演算子
- 等値性と Hash
- 例外種別
- signed zero の公開規則
- `ToString` の契約範囲
- scalar 値との変換・演算を提供するか
- generic math interface を提供するか

Phase 2 完了後は、基本 `Interval` API に対する破壊的変更を原則として行わない。後続の SIMD や初等関数は、既存 API の内部差し替えまたは追加 API とする。

### 3.4 Phase 3: SIMD バックエンド

目的:

- Phase 1 の scalar reference と同じ意味論および端点を維持したまま高速化する。

実装順:

1. scalar reference と SIMD backend の差分試験基盤を追加する。
2. SIMD に適した内部ロード・ストアを追加する。
3. 加算・減算から SIMD 化する。
4. 乗算・除算の符号分類を SIMD 化する。
5. AVX-512 の方向付き丸めを使用する batch kernel を追加する。
6. AVX2 / SSE2 / ARM64 について、誤差補償型 SIMD の効果を計測して採否を決定する。

AVX-512 の packed embedded rounding は `Vector512<double>` に対して利用できる。一方、`Vector128<double>` の embedded-rounding API は scalar lane 用であり、2 lane を同時に方向丸めする packed API ではない。

したがって、AVX-512 の利点を直接得やすいのは次の形式である。

```text
[-L0, U0, -L1, U1, -L2, U2, -L3, U3]
```

これは 4 区間を 1 個の `Vector512<double>` に配置する batch 処理である。単一の `Interval` 演算を無条件に `Vector512` へ載せる設計にはしない。単一演算は benchmark で scalar より有利と確認できた場合に限り SIMD backend へ切り替える。

### 3.5 Phase 4: 四則演算以外

次の順序で小さな単位に分ける。

1. 集合・判定操作
   - `Contains`
   - subset 判定
   - overlap 判定
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

周期関数と decorated interval は API と正確性の負荷が大きいため、四則演算直後の同一作業単位には含めない。

## 4. 対象環境

### 4.1 Target Framework

Phase 1 の対象を `net10.0` とする。

理由:

- 2026-08-29 時点で .NET 10 は LTS である。
- 後続の hardware intrinsics を同じ target framework 上で利用できる。
- 複数 target framework の同時対応による検証行列を、パイロット段階では増やさない。

古い target framework の追加は Phase 2 で利用者要件として判断する。

### 4.2 対象アーキテクチャ

Phase 1:

- x64
- ARM64

Phase 1 は pure managed とし、同じ scalar algorithm を使用する。

Phase 3:

- x64 AVX-512: packed embedded-rounding backend
- x64 AVX2 / SSE2: 誤差補償型 SIMD の検証対象
- ARM64 AdvSimd: 誤差補償型 SIMD の検証対象
- 未対応 CPU: Phase 1 scalar backend

### 4.3 Runtime dependency

Phase 1 の production package は BCL 以外の runtime dependency を持たない。

`kv` と `inari` は参照実装およびテスト比較対象とし、production runtime からは呼び出さない。

## 5. 公開 API 候補

Phase 1 では以下の API を実装候補とする。Phase 2 までは pilot API とし、Phase 2 完了時に固定する。

### 5.1 Assembly と namespace

```text
Assembly / package: Devo6.Interval
Namespace:          Devo6.Numerics
Type:               Interval
```

`Devo6.Interval.Interval` という重複した完全修飾名を避けるため、pilot namespace は `Devo6.Numerics` とする。namespace は Phase 2 の確定対象である。

### 5.2 型定義

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

### 5.3 生成

#### 通常区間

```csharp
var x = new Interval(lower, upper);
```

次の条件を満たす場合に生成できる。

```text
lower <= upper
lower != +Infinity
upper != -Infinity
lower is not NaN
upper is not NaN
```

不正な場合は `ArgumentException` を送出する。

`lower > upper` は自動的に `Empty` へ変換しない。入力ミスと数学上の空区間生成を区別するためである。空区間は `Interval.Empty` を使用する。

#### TryCreate

```csharp
if (Interval.TryCreate(lower, upper, out var interval))
{
    // valid interval
}
```

不正な場合は `false` を返し、`interval` には `Interval.Empty` を設定する。

#### 点区間

```csharp
var x = Interval.Point(value);
```

有限の `double` のみを受け入れ、`[value, value]` を生成する。

`NaN`, `+Infinity`, `-Infinity` は実数の点ではないため拒否する。

### 5.4 定数

#### Empty

```csharp
Interval.Empty
```

空集合を表す。

#### Entire

```csharp
Interval.Entire
```

```text
[-Infinity, +Infinity]
```

を表す。

#### Zero

```csharp
Interval.Zero
```

```text
[-0.0, +0.0]
```

を正規形とする。

`default(Interval)` は内部フィールドがともに `+0.0` となるため、`Interval.Zero` と同値にする。この性質をテストで固定する。

### 5.5 Lower / Upper

非空区間では正規化済みの端点を返す。

空区間では両方とも `double.NaN` を返す。

```text
Interval.Empty.Lower -> NaN
Interval.Empty.Upper -> NaN
```

下限が 0 の場合は `-0.0`、上限が 0 の場合は `+0.0` を返す。

### 5.6 状態プロパティ

```text
IsEmpty     Empty の場合に true
IsEntire    [-Infinity, +Infinity] の場合に true
IsSingleton 非空かつ Lower == Upper の場合に true
```

`Empty.IsSingleton` は `false` とする。

### 5.7 演算子と例外

公開 API で生成された `Interval` は常に有効であるため、四則演算は数学的な定義域問題を例外で表現しない。

例:

```text
[1, 2] / [0, 0]  -> Empty
[1, 2] / [-1, 1] -> Entire
```

演算時に `DivideByZeroException` は送出しない。

### 5.8 scalar double との演算

次は Phase 1 では提供しない。

```csharp
interval + 1.0
1.0 + interval
(Interval)1.0
```

理由:

- implicit conversion が `NaN` や infinity で例外を生じる API になる。
- overload 数が増え、API 評価範囲が広がる。
- `Interval.Point(value)` で明示的に代替できる。

Phase 2 で利用性を確認して追加可否を決定する。

### 5.9 generic math

`INumber<TSelf>` は区間に自然な全順序や通常の数値変換を要求するため、Phase 1 では実装しない。

個別の operator interface についても Phase 2 で判断する。

### 5.10 文字列表現

Phase 1 の `ToString()` は診断用途とする。

候補:

```text
Empty       -> "Empty"
[1, 2]      -> "[1, 2]"
Entire      -> "[-Infinity, Infinity]"
```

数値は invariant culture と round-trip format を使用する。

文字列形式を永続化・通信・round-trip parsing の契約としては扱わない。正式な format / parse 契約は Phase 4 で設計する。

## 6. 内部表現

### 6.1 Phase 1 の物理表現

```csharp
public readonly struct Interval
{
    private readonly double _negatedLower;
    private readonly double _upper;
}
```

外部区間 `[lower, upper]` を次のように保持する。

```text
_negatedLower = -lower
_upper        =  upper
```

### 6.2 正規表現

#### 非空区間

```text
[-Lower, Upper]
```

#### Zero

```text
[+0.0, +0.0]
```

外部取得時は次になる。

```text
Lower = -(+0.0) = -0.0
Upper = +0.0
```

#### Entire

```text
[+Infinity, +Infinity]
```

#### Empty

固定した quiet NaN bit pattern を 2 lane に格納する。

```text
[canonical-qNaN, canonical-qNaN]
```

NaN payload は公開 API の契約にはしないが、同一 runtime 内での決定性を得るため production 実装では固定する。

### 6.3 内部不変条件

非空区間は次を満たす。

```text
!IsNaN(_negatedLower)
!IsNaN(_upper)
Lower <= Upper
Lower != +Infinity
Upper != -Infinity
```

Empty は次を満たす。

```text
IsNaN(_negatedLower)
IsNaN(_upper)
```

内部処理で片方だけ NaN の状態を作成してはならない。

### 6.4 raw constructor

演算結果用に validation を省略する private/internal constructor を設ける。

```csharp
private Interval(
    double negatedLower,
    double upper,
    RawIntervalTag _);
```

raw constructor を呼ぶ側は次を保証する。

- Empty または有効な正規化済み内部表現である。
- zero の符号が正規化済みである。
- 片側 NaN を含まない。

### 6.5 レイアウト非公開

内部が 2 個の `double` であること、16 byte であること、フィールド順序は public API 契約にしない。

次を公開しない。

- public field
- `ref` での内部表現取得
- `Vector128<double>` への public cast
- raw bytes を前提とする serializer
- blittable ABI の保証

これにより Phase 3 で格納型を変更できる。

## 7. 内部コンポーネント

### 7.1 予定ファイル構成

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
  ReferenceOracle/
    Binary64Rational.cs
    DirectedRoundingOracle.cs
```

### 7.2 責務

#### Interval

- public value semantics
- validation
- properties
- operators の entry point
- equality / Hash

#### ScalarIntervalKernel

- Empty propagation
- 符号分類
- 四則演算の区間アルゴリズム
- 正規化済み raw interval の生成

#### DirectedRounding

- scalar binary64 の `AddUp`, `MultiplyUp`, `DivideUp`
- 必要な down-direction helper
- overflow / underflow / subnormal 処理
- `NextUp`, `NextDown`

#### IntervalCanonicalizer

- lower zero を `-0.0` へ正規化
- upper zero を `+0.0` へ正規化
- Empty の canonical NaN 化

#### IntervalSignClass

区間を次に分類する。

```text
Zero        [0, 0]
NonNegative 0 <= Lower
NonPositive Upper <= 0
Mixed       Lower < 0 < Upper
```

Zero を先に判定し、`NonNegative` と `NonPositive` の重複を避ける。

### 7.3 hot path の dispatch

Phase 1 では operator から `ScalarIntervalKernel` を直接呼ぶ。

interface、virtual method、delegate table は hot path に導入しない。

Phase 3 では `Avx512F.IsSupported` 等の intrinsic support check を static method 内で使用し、JIT が非対応分岐を除去できる構造を優先する。

## 8. 方向付き丸め

### 8.1 契約

`DirectedRounding` は、許可された binary64 operand に対して次を返す。

```text
AddUp(x, y)      = 最小の double z で x + y <= z
MultiplyUp(x, y) = 最小の double z で x * y <= z
DivideUp(x, y)   = 最小の double z で x / y <= z
```

下向き丸めは符号反転で導出する。

```text
AddDown(x, y)      = -AddUp(-x, -y)
SubtractDown(x, y) = -SubtractUp(y, x)
MultiplyDown(x, y) = -MultiplyUp(-x, y)
DivideDown(x, y)   = -DivideUp(-x, y)
```

zero sign は返却後に区間端点として正規化する。

### 8.2 特殊 operand と禁止 operand

方向付き丸め primitive は、通常計算より前に infinity と zero を分岐する。

- 加算で片方が infinity、もう片方が有限値または同符号 infinity の場合は、その infinity を返す。
- 乗算で片方が infinity、もう片方が nonzero の場合は符号付き infinity を返す。
- 除算で numerator が infinity、denominator が有限 nonzero の場合は符号付き infinity を返す。
- 除算で numerator が有限、denominator が infinity の場合は符号付き zero を返し、区間側で canonical zero に正規化する。
- 除算で numerator が zero、denominator が有限 nonzero の場合は符号付き zero を返す。

次の IEEE 754 上 undefined な組み合わせは、区間アルゴリズム側で分岐し、方向付き丸め primitive へ渡さない。

```text
+Infinity + -Infinity
0 * Infinity
0 / 0
Infinity / Infinity
```

`DirectedRounding` の Debug build では前提条件を assert する。

### 8.3 NextUp / NextDown

隣接 binary64 の取得には `Math.BitIncrement` と `Math.BitDecrement` を使用する。

通常演算の結果を無条件で 1 ULP 外側へ広げる方式は採用しない。

理由:

- 演算結果が正確だった場合にも不要に区間が広がる。
- scalar reference と SIMD backend の bitwise equivalence を得にくい。
- `kv` / `inari` 相当の tight endpoint を目標にできない。

誤差の符号を判定し、必要な場合にだけ隣接値へ移動する。

### 8.4 加算

finite operand で overflow がない場合、TwoSum により次を得る。

```text
s = roundNearest(x + y)
e = (x + y) - s
```

概念コード:

```csharp
var s = x + y;
var bVirtual = s - x;
var e = (x - (s - bVirtual)) + (y - bVirtual);
```

上向き丸め:

```text
e > 0 -> NextUp(s)
e <= 0 -> s
```

下向き丸め:

```text
e < 0 -> NextDown(s)
e >= 0 -> s
```

overflow 時:

```text
正の overflow:
  Up   -> +Infinity
  Down -> double.MaxValue

負の overflow:
  Up   -> -double.MaxValue
  Down -> -Infinity
```

### 8.5 減算

上向き減算は加算へ変換する。

```text
SubtractUp(x, y) = AddUp(x, -y)
```

下向きは次で得る。

```text
SubtractDown(x, y) = -SubtractUp(y, x)
```

### 8.6 乗算

通常領域では次を用いる。

```csharp
var product = x * y;
var error = Math.FusedMultiplyAdd(x, y, -product);
```

`Math.FusedMultiplyAdd` は `x * y - product` を 1 回の丸めで求めるため、誤差符号の判定に使用できる。

```text
error > 0 -> 上向き結果は NextUp(product)
error <= 0 -> 上向き結果は product
```

積が subnormal 領域に近い場合、誤差自体が underflow して 0 になり得る。そのため `kv` の no-hardware-rounding 実装を参照し、閾値以下では scaling を行う。

設計定数:

```text
SmallProductThreshold = 2^-969
ProductScale          = 2^537
```

概念処理:

1. infinity / zero と overflow を先に処理する。
2. 通常の product を計算する。
3. `abs(product) >= 2^-969` の場合は FMA residual の符号を使用する。
4. それ未満の場合は operand を `2^537` 倍した積を評価する。
5. scaled exact relation から元の product が真値より小さいかを判定する。
6. 必要な場合だけ `NextUp` / `NextDown` する。

production 実装で `BigInteger` は使用しない。

### 8.7 除算

通常の quotient を求める。

```text
q = roundNearest(x / y)
```

次に `q * y` と `x` の大小を、FMA を用いた exact-product decomposition で判定する。

```text
q * y < x -> q は真の商より小さいため Up は NextUp(q)
q * y >= x -> Up は q
```

`y < 0` の場合は両 operand の符号を反転して denominator を正に正規化してから比較する。

subnormal numerator については `kv` の no-hardware-rounding 実装を参照し、必要な scaling を行う。

設計定数:

```text
SmallNumeratorThreshold = 2^-969
LargeDenominatorLimit   = 2^918
DivisionScale           = 2^105
MinimumSubnormal        = 2^-1074
```

overflow 時は乗算と同じ方向別 saturation を行う。

### 8.8 参照実装移植とライセンス

方向付き丸めの具体実装で `kv` または `inari` のコードを翻案・移植する場合は、次を必須とする。

- 参照元 commit SHA を source comment に記録する。
- MIT License の copyright と permission notice を third-party notice に含める。
- 参照元テストケースを移植した場合も出典を記録する。

アルゴリズムを参照しただけであっても、設計書と実装 report に参照元を記録する。

## 9. 四則演算

以下では `RD` を下向き丸め、`RU` を上向き丸めとする。

全演算で、operand のどちらかが Empty の場合は Empty を返す。

### 9.1 加算

```text
[a, b] + [c, d]
= [RD(a + c), RU(b + d)]
```

内部表現:

```text
A = [-a, b]
B = [-c, d]

result = [RU(-a - c), RU(b + d)]
       = [-RD(a + c), RU(b + d)]
```

Phase 1 scalar 実装でも、この内部表現に沿って上向き primitive を使用する。

### 9.2 減算

```text
[a, b] - [c, d]
= [RD(a - d), RU(b - c)]
```

内部表現:

```text
result.negatedLower = RU((-a) + d)
result.upper        = RU(b + (-c))
```

右 operand の lane を交換したものとの加算に相当する。

### 9.3 単項マイナス

```text
-[a, b] = [-b, -a]
```

内部表現では lane 交換だけで得られる。

```text
[-a, b] -> [b, -a]
```

Empty は lane 交換後も Empty とする。

### 9.4 乗算の符号分類

区間 class:

```text
Z: [0, 0]
P: 0 <= lower かつ Zero ではない
N: upper <= 0 かつ Zero ではない
M: lower < 0 < upper
```

`A = [a, b]`, `B = [c, d]` とする。

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

Zero を先に処理することで `0 * Infinity` を endpoint primitive へ渡さない。

この表は Phase 1 と Phase 3 で共通の意味論とする。

### 9.5 除算: denominator が 0 を含まない場合

`A = [a, b]`, `B = [c, d]` とし、0 が B に含まれないものとする。

#### B が正

```text
0 < c <= d
```

| A | Lower | Upper |
|---|---|---|
| P | `RD(a/d)` | `RU(b/c)` |
| N | `RD(a/c)` | `RU(b/d)` |
| M | `RD(a/c)` | `RU(b/c)` |

#### B が負

```text
c <= d < 0
```

| A | Lower | Upper |
|---|---|---|
| P | `RD(b/d)` | `RU(a/c)` |
| N | `RD(b/c)` | `RU(a/d)` |
| M | `RD(b/d)` | `RU(a/d)` |

reciprocal interval を一度作って乗算する方式は採用しない。二段階の外向き丸めで不要に幅が増える可能性があるため、端点を直接除算する。

### 9.6 除算: denominator が 0 のみ

```text
B = [0, 0]
```

結果は numerator に関係なく Empty とする。

```text
A / [0, 0] -> Empty
```

### 9.7 除算: denominator が 0 に片側から接する場合

#### B = [0, d], d > 0

| A | Result |
|---|---|
| Z | `Zero` |
| P | `[RD(a/d), +Infinity]` |
| N | `[-Infinity, RU(b/d)]` |
| M | `Entire` |

#### B = [c, 0], c < 0

| A | Result |
|---|---|
| Z | `Zero` |
| P | `[-Infinity, RU(a/c)]` |
| N | `[RD(b/c), +Infinity]` |
| M | `Entire` |

P または N に zero endpoint が含まれる場合も表の式を使用する。zero endpoint は canonical zero へ正規化する。

### 9.8 除算: denominator が 0 を跨ぐ場合

```text
c < 0 < d
```

```text
Zero / B -> Zero
その他    -> Entire
```

nonzero numerator の実際の結果集合は 2 本の非連続な半直線になる場合がある。bare `Interval` は単一区間であるため、convex hull として Entire を返す。

2 区間を返す extended division は Phase 4 の別 API とする。

## 10. 等値性と Hash

### 10.1 等値性

集合として等しい区間を等しいとする。

- Empty 同士は等しい。
- 非空区間は正規化済み Lower / Upper が等しい場合に等しい。
- `+0.0` と `-0.0` の入力差は等値性へ影響しない。
- NaI は bare `Interval` に存在しない。

### 10.2 Hash

等しい区間は同じ Hash を返す。

- Empty は固定 Hash とする。
- zero endpoint は canonical bit pattern で Hash する。
- NaN payload を Hash へ直接使用しない。

### 10.3 順序比較

`<`, `<=`, `>`, `>=` は Phase 1 では定義しない。

区間には要素ごとの順序、strictly-before、subset 等の異なる関係があるため、通常数値の比較演算子へ割り当てない。

## 11. 正規化

### 11.1 constructor 入力

入力が zero の場合、外部 lower/upper の符号に関係なく次へ正規化する。

```text
lower zero -> -0.0
upper zero -> +0.0
```

内部では次になる。

```text
_negatedLower = +0.0
_upper        = +0.0
```

### 11.2 演算結果

raw result を生成する直前に同じ zero normalization を行う。

### 11.3 infinity

```text
lower = -Infinity -> _negatedLower = +Infinity
upper = +Infinity -> _upper = +Infinity
```

内部 lane に `-Infinity` を正規表現として保存しない。

## 12. TDD と検証

### 12.1 実装順序

Phase 1 は次の順で進める。

1. constructor / constants の失敗テスト
2. constructor / constants の実装
3. equality / zero normalization の失敗テスト
4. equality / zero normalization の実装
5. DirectedRounding 加算の失敗テスト
6. 加算・減算の実装
7. DirectedRounding 乗算の失敗テスト
8. 乗算の実装
9. DirectedRounding 除算の失敗テスト
10. 除算の実装
11. edge case と reference comparison の拡充

各論理単位を小さく commit / push する。

### 12.2 deterministic test

最低限次を含める。

#### construction

- finite singleton
- bounded interval
- lower `-Infinity`
- upper `+Infinity`
- Entire
- Empty
- lower > upper
- NaN endpoint
- lower `+Infinity`
- upper `-Infinity`
- signed zero normalization
- `default(Interval) == Interval.Zero`

#### arithmetic

- exact result
- inexact result
- positive / negative / mixed の全組合せ
- Empty propagation
- Entire との演算
- zero interval
- unbounded endpoint
- denominator の zero 非包含、片側接触、zero 跨ぎ、zero のみ

#### floating-point edge

- `double.Epsilon`
- 最大 subnormal
- 最小 normal
- `double.MaxValue`
- overflow の直前と直後
- cancellation
- 隣接 double
- `+0.0` / `-0.0`

### 12.3 independent exact oracle

production と同じ TwoSum / FMA algorithm だけを expected value の根拠にしない。

テスト専用に binary64 を exact rational として扱う oracle を作る。

有限 `double` は bit decomposition により次で正確に表現できる。

```text
significand * 2^exponent
```

加算・減算・乗算は `BigInteger` で exact value を生成する。除算は numerator / denominator の rational として保持する。

expected directed result の求め方:

1. BCL の nearest result `r` を取得する。
2. `r` 自体を exact rational へ変換する。
3. exact result と `r` を比較する。
4. 上向きで `r` が小さい場合だけ `BitIncrement(r)` する。
5. 下向きで `r` が大きい場合だけ `BitDecrement(r)` する。
6. overflow / infinity は exact rational と最大有限値を比較して決める。

この oracle は test project にのみ置き、production package には含めない。

### 12.4 reference implementation comparison

次を secondary oracle とする。

- `inari` の pinned commit
- `kv` の pinned commit

比較単位:

- Empty / Entire の意味
- endpoint の bit pattern
- zero sign は Devo6.Interval の canonical rule 適用後に比較
- NaN payload は比較しない

不一致時は、Devo6.Interval、exact rational oracle、`inari`、`kv` の全結果をログへ保存する。

### 12.5 property test

ランダムな有効区間に対して次を検証する。

- result が exact sampled value を包含する。
- `x + y == y + x`
- `x * y == y * x`
- `-(-x) == x`
- `x + Zero == x`
- `x * Zero == Zero` for nonempty x
- scalar backend の result invariant が保たれる。

分配法則の等号は依存性問題により一般には成立しないため、同一結果を要求しない。

### 12.6 cross-backend differential test

Phase 3 では同じ入力集合を scalar backend と SIMD backend の両方へ入力し、次を要求する。

- Empty / Entire が一致する。
- nonempty endpoint の canonical bit pattern が一致する。
- CPU 非対応時は scalar fallback と一致する。

単に SIMD result が scalar result を包含しているだけでは合格としない。四則演算については bitwise-equivalent endpoint を目標とする。

## 13. CI と failure diagnostics

Phase 1 で初めて executable project と test project を追加するとき、同じ PR で CI workflow を追加する。

CI は少なくとも次を保存する。

- test result (`.trx` 等)
- 標準出力
- 標準エラー
- test runner log
- 失敗した reference comparison の入力と全 backend result

artifact upload step はテスト失敗時にも実行されるよう `if: always()` 相当を使用する。

本書自体は documentation-only であり、現在のリポジトリには executable target と workflow がないため、本作業では workflow を追加しない。

## 14. API 確定ゲート

Phase 2 は「コードが動いた」だけでは完了としない。次を満たす必要がある。

### 14.1 利用性

- 代表的な計算例を operator だけで記述できる。
- Empty / Entire を明示的に判定できる。
- 不正 constructor と数学的に空になる演算結果の違いが明確である。
- signed zero が利用者に予期しない分岐を生じさせない。

### 14.2 正確性

- 四則演算の edge-case matrix が通る。
- exact rational oracle との差がない。
- `inari` / `kv` との差異が説明済みである。
- x64 と ARM64 scalar backend で同じ canonical result を返す。

### 14.3 API baseline

- public API 一覧を baseline file として保存する。
- 以後の PR で public API の差分を CI 検出する。
- breaking change が必要な場合は `doc/Design/BreakingChanges.md` に理由と移行方法を記録する。

### 14.4 性能 baseline

- allocation が 0 である。
- scalar operation の BenchmarkDotNet baseline を保存する。
- Debug 用 assertion が Release hot path に残らない。

性能値そのものを API 確定条件にはしない。Phase 3 の比較基準として保存する。

## 15. SIMD 詳細方針

### 15.1 public API 非依存

既存 scalar operator の署名と結果を変更しない。

SIMD は以下のいずれかとして追加する。

- operator の内部 backend 差し替え
- 追加の batch API

### 15.2 batch API 候補

Phase 3 で次を別途 API review する。

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

これは Phase 2 の基本 `Interval` API freeze とは別の additive API である。

長さ不一致、overlap、in-place 可否は Phase 3 の詳細設計で確定する。

### 15.3 AVX-512

AVX-512 backend は 4 区間を `Vector512<double>` の 8 lane に配置する。

加算の概念:

```text
left   = [-L0,U0,-L1,U1,-L2,U2,-L3,U3]
right  = [-R0,V0,-R1,V1,-R2,V2,-R3,V3]
result = Add(left, right, ToPositiveInfinity)
```

結果はそのまま 4 個の内部区間表現となる。

末尾が 4 区間未満の場合は scalar backend で処理する。

### 15.4 AVX2 / SSE2 / ARM64

embedded rounding を使用できない backend では、グローバル floating-point rounding mode を変更しない。

候補:

- vectorized TwoSum
- vectorized FMA residual
- residual sign による lane-wise next-up correction

MXCSR を native shim で変更する方式は、JIT の floating-point 最適化、例外時復元、thread-local state、async/thread migration の検証が必要であるため、初期 SIMD backend の対象外とする。

### 15.5 backend 選択

利用者へ backend 選択 option を最初から公開しない。

production は CPU support に基づき自動選択する。テストと benchmark だけが internal hook で backend を固定できるようにする。

### 15.6 SIMD 完了条件

- scalar backend と bitwise equivalent
- 非対応 CPU で scalar fallback
- Empty / zero / infinity / subnormal を含む差分試験成功
- 単一演算または batch のいずれかで測定可能な性能改善
- 性能改善がない経路は production dispatch に含めない

## 16. 四則演算以外の設計原則

### 16.1 追加順序

API が単純で正確性を証明しやすいものから追加する。

```text
set operations
  -> monotonic algebraic functions
  -> monotonic transcendental functions
  -> periodic / singular functions
  -> decorated / parsing / advanced models
```

### 16.2 IntervalMath

四則演算以外の数学関数は `Interval` の instance method ではなく、次の static class を第一候補とする。

```csharp
public static class IntervalMath
{
    public static Interval Sqrt(Interval value);
    public static Interval Exp(Interval value);
    public static Interval Log(Interval value);
    public static Interval Sin(Interval value);
}
```

これにより値型自体の責務を抑え、関数追加を additive change にできる。

### 16.3 定義域

bare interval は関数の定義域と入力区間の共通部分を評価する。

例:

```text
Sqrt([-1, 4]) -> [0, 2]
Sqrt([-4, -1]) -> Empty
```

定義域の一部が欠けた情報は bare interval だけでは保持しない。将来 decorated interval で decoration を付加する。

## 17. 性能設計

Phase 1 から次を守る。

- `readonly struct`
- heap allocation なし
- global rounding mode 変更なし
- production の `BigInteger` 利用なし
- hot path の virtual/interface/delegate dispatch なし
- 無条件 1 ULP 拡張なし
- reciprocal を経由した二重丸めなし
- internal raw constructor による validation 重複回避

`MethodImplOptions.AggressiveInlining` は benchmark で有効性を確認した method に限定する。

## 18. Thread safety / AOT / trimming

`Interval` は immutable value type とする。

Phase 1 の方向付き丸めは process/thread の floating-point environment を変更しないため、複数 thread から安全に使用できる。

reflection、runtime code generation、dynamic assembly、native resolver を使用しない。NativeAOT と trimming を妨げない構造とする。

## 19. 「同等の計算結果」の定義

既存ライブラリと同等であることを、単に「真値を含む」こととは定義しない。

四則演算では次を目標とする。

1. IEEE 1788.1 の set-based な結果集合を使用する。
2. その集合の convex hull を返す。
3. 各有限端点を、指定方向へ正しく丸めた最も内側の binary64 とする。
4. signed zero は Devo6.Interval の canonical rule に正規化する。
5. Empty の NaN payload は比較対象外とする。

この条件を満たす場合、四則演算の nonempty endpoint は原則として一意であり、`kv` / `inari` と bitwise に一致することを期待できる。

不一致がある場合は「既存ライブラリに合わせる」ことを先にせず、exact rational oracle と IEEE 1788.1 意味論を優先して原因を調査する。

## 20. 未確定事項

Phase 1 開始を妨げないが、Phase 2 または後続フェーズで決定する。

- namespace を `Devo6.Numerics` で確定するか
- `Interval(double, double)` と factory-only API のどちらを正式採用するか
- scalar double overload / conversion
- generic math operator interface
- `ToString` の正式 format
- public batch API の名称と overlap 規則
- `Vector128<double>` を物理フィールドとして保持するか
- AVX2 / ARM64 SIMD backend の採否
- parsing / serialization format
- decorated interval と extended division の公開時期

## 21. 参照

### 21.1 基本設計

- `doc/Design/basic/IntervalArithmetic.md`

### 21.2 inari

- Repository: `unageek/inari`
- Pinned commit: `18b83a571d7681c76067bc38d90a74e8be29f545`
- 参照点:
  - `[-Lower, Upper]` SIMD 表現
  - Empty の NaN 表現
  - bare / decorated interval の分離
  - IEEE 1788.1-oriented semantics
- License: MIT

### 21.3 kv

- Repository: `mskashi/kv`
- Pinned commit: `c7f8f2324a0e403cca6b39f46088a22843d440db`
- 参照点:
  - TwoSum / TwoProduct
  - hardware rounding を使わない方向付き丸め
  - overflow / underflow / subnormal 処理
  - AVX-512 embedded rounding
- License: MIT

### 21.4 .NET

- .NET releases and support
  - <https://learn.microsoft.com/dotnet/core/releases-and-support>
- `Math.FusedMultiplyAdd`
  - <https://learn.microsoft.com/dotnet/api/system.math.fusedmultiplyadd>
- `Avx512F`
  - <https://learn.microsoft.com/dotnet/api/system.runtime.intrinsics.x86.avx512f>

## 22. 実装開始条件

Phase 1 は本書が承認された後、次の単位で開始する。

1. solution / project / CI 基盤
2. `Interval` 生成・状態・正規化
3. managed directed rounding
4. 加算・減算
5. 乗算
6. 除算
7. reference oracle / differential test
8. sample / API 評価 report

各単位は TDD で実装し、レビュー可能な小さな commit とする。
