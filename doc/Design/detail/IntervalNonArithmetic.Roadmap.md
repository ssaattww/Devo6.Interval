# 四則演算以外の機能 詳細設計ロードマップ

## 1. 文書情報

- 対象リポジトリ: `ssaattww/Devo6.Interval`
- 対象ライブラリ: `Devo6.Interval`
- 前提文書:
  - `doc/Design/basic/IntervalArithmetic.md`
  - `doc/Design/detail/IntervalArithmetic.md`
  - `doc/Design/detail/IntervalArithmetic.Revision3.md`
- 作成日: 2026-08-30
- 設計状態: review required

本書は、四則演算と基本 `Interval` APIの確定後に追加する機能を、実装順、公開API境界、正確性契約および検証ゲートまで分解する。

既存詳細設計の最終fix verificationは、既存の四則演算設計を対象としたものである。本書および本書から参照する新規詳細設計は、そのPASS verdictの対象外であり、実装開始前に新しいimmutable HEADを対象とした設計reviewを必要とする。

## 2. 設計対象

四則演算以外を次の5群へ分ける。

1. 集合演算、関係判定、数値的属性、整数値関数
2. 代数関数
3. 単調な初等関数
4. 周期関数、特異点を持つ関数、多変数関数
5. 非連結結果、decorated interval、文字列変換、区間分割等の拡張機能

詳細は次の文書で定義する。

- `IntervalSetAndNumeric.md`
- `IntervalMathFunctions.md`
- `IntervalAdvancedFeatures.md`

## 3. 非対象

本設計では次を実装しない。

- Phase 1のmanaged scalar四則演算
- Phase 3のSIMD backend
- generic endpoint型による`Interval<T>`
- Affine Arithmetic
- Taylor Model
- constraint solver全体
- root finding、global optimization、automatic differentiation
- 任意精度区間を公開型として提供すること

Affine Arithmetic、Taylor Modelおよびsolverは、基本`Interval`を利用する上位packageとして別途設計する。

## 4. 公開型の責務分割

### 4.1 `Interval`

`Interval`はimmutableなbare interval値型として維持する。

追加する責務は次に限定する。

- 集合としての状態・関係の問い合わせ
- 集合演算
- 区間自身の数値的属性

数学関数をinstance methodとして大量に追加しない。

### 4.2 `IntervalMath`

点関数の区間拡張は`IntervalMath`へ配置する。

```csharp
public static class IntervalMath
{
    public static Interval Abs(Interval value);
    public static Interval Square(Interval value);
    public static Interval Sqrt(Interval value);
    public static Interval Exp(Interval value);
    public static Interval Log(Interval value);
    public static Interval Sin(Interval value);
}
```

四則演算と同様に、公開methodはbackendを公開せず、scalar、SIMD、managedまたはnativeの選択を内部へ閉じ込める。

### 4.3 `IntervalConstants`

`π`等の実数定数を点の`double`として偽装せず、tightな区間定数として公開する。

```csharp
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

### 4.4 拡張型

非連結な最大2成分の結果は`IntervalUnion2`、定義状態を保持する区間は`DecoratedInterval`として、bare `Interval`と別型にする。

```text
Interval                 連結なbare interval
IntervalUnion2           0～2個の連結成分
DecoratedInterval        Interval + Decoration または NaI
```

## 5. 共通アーキテクチャ

四則演算以外の数学関数は、次の2層へ分ける。

```text
public interval extension
  - Empty伝播
  - 定義域との共通部分
  - 単調性・符号・象限の分類
  - 極値・特異点・周期点の検出
  - 単一区間または2成分への構成
             ↓
certified scalar endpoint kernel
  - FooDown(double)
  - FooUp(double)
  - 定数とrange reduction
  - overflow / underflow / subnormal
  - 正しい方向付きbinary64丸め
```

区間レベルの実装とスカラー関数の正しい丸めを混在させない。これにより、区間意味論を維持したままendpoint kernelだけを置換できる。

## 6. 正確性レベル

### 6.1 Exact

比較、集合演算、符号分類等、浮動小数点演算による丸めを必要としない機能は、集合として正確な結果を返す。

### 6.2 Tight directed algebraic

`square`、`sqrt`、integer power等は、各端点を指定方向へ正しく丸めた最も内側のbinary64を返す。

### 6.3 Tight certified elementary

`exp`、`log`、`sin`等の初等関数も、公開時にはtightな端点を必須とする。

通常の`Math.Sin`等を呼んだ後に、根拠なく一定ULPだけ広げる実装を正式backendとして採用しない。BCL関数は近似値のseedとして使用できるが、包含保証と正しい方向丸めは別の証明済み処理で確定する。

### 6.4 暗黙の精度低下を禁止する

同じ公開methodが環境により`Tight`と「真値を含むが広い結果」を暗黙に切り替えない。

正しく丸められるproduction kernelが用意できない関数は、未実装のまま保持する。性能または配布上の理由でvalid-only APIが必要になった場合は、tight APIと異なる名称・型・accuracy metadataを持つ別設計とする。

## 7. 実装フェーズ

### 7.1 Phase 4A: 集合・関係・数値的属性

対象:

- `Contains`
- `Intersect`
- `ConvexHull`
- subset、interior、disjoint、precedes等のnamed relation
- `IntervalOverlap`
- `IsBounded`
- `Width`、`Midpoint`、`Radius`、`Magnitude`、`Mignitude`
- pointwise min/max
- absolute value、sign
- floor、ceiling、truncate、round

特徴:

- 超越関数backendを必要としない。
- 大半が比較、lane-wise min/max、既存の加減算方向丸めで実装できる。
- `[-Lower, Upper]`表現とSIMDの効果を利用しやすい。

完了条件:

- IEEE 1788.1-orientedなEmpty semanticsを含む決定的testが成功する。
- x64 / ARM64でcanonical結果が一致する。
- relationへ`<`, `<=`, `>`, `>=`を割り当てず、named APIの意味が固定される。

### 7.2 Phase 4B: 代数関数

対象:

- reciprocal
- square
- square root
- integer power
- integer root
- fused multiply-add
- tight interval constants

実装順:

1. `Abs`, `Square`, `Reciprocal`
2. `Sqrt`
3. integer `Pow`
4. integer `Root`
5. `FusedMultiplyAdd`
6. constants

完了条件:

- endpointごとにexact rationalまたはMPFR oracleとの差がない。
- domain boundary、zero、Infinity、subnormal、overflowの固定fixtureが成功する。
-合成可能な関数でも、不要な二重丸めにより結果を広げない。

### 7.3 Phase 4C: 単調な初等関数

対象:

- `Exp`, `Exp2`, `Exp10`
- `Log`, `Log2`, `Log10`
- `Sinh`, `Tanh`
- `Asinh`, `Atan`
- `Acosh`, `Atanh`
- `Asin`, `Acos`
- `Cosh`の区分単調処理

実装順:

1. endpoint kernelとMPFR golden corpus基盤
2. 全実数上で単調増加する関数
3. 半直線または有限区間の定義域を持つ関数
4. 偶関数・減少関数

完了条件:

- scalar endpoint kernelの正しい方向丸めが証明または検証可能である。
- domain clippingとEmpty/非有界結果が標準matrixと一致する。
- managed/nativeの実装方式にかかわらず公開結果が同一である。

### 7.4 Phase 4D: 周期・特異点・多変数関数

対象:

- `Sin`, `Cos`, `Tan`
- `Atan2`
- 将来の一般`Pow`

必要要素:

- binary64全域を対象にした正確なrange reduction
- `π/2 + kπ`等の臨界点・極・branch cut検出
- rectangleの象限分類
- 非連結な真の像をbare intervalへ写すhull規則

一般`Pow(Interval, Interval)`は負の底と非整数指数の扱いが複雑であるため、integer powerとは同時に公開しない。専用の意味論・定義域reviewを完了してから追加する。

### 7.5 Phase 4E: 拡張機能

対象:

- `IntervalUnion2`
- 2成分を保持するextended division
- reverse multiplication / two-output division
- cancellative addition/subtraction
- `DecoratedInterval` / NaI
- exact/outward text parsing
- exact text/binary interchange
- interval splitting

完了条件:

- bare intervalとの型境界が明確である。
- 非連結結果を暗黙に`Entire`へ潰すAPIと、成分を保持するAPIを区別できる。
- parsingが`double.Parse`後の点区間化によって真値を失わない。
- decorated equalityをC#の値等値性とIEEEのsemantic equalityで混同しない。

## 8. Dependency graph

```text
Phase 1 scalar四則演算
       ↓
Phase 2 basic API freeze
       ↓
Phase 3 SIMD four arithmetic
       ↓
Phase 4A set / relation / numeric
       ↓
Phase 4B algebraic + constants
       ↓
Phase 4C monotonic elementary
       ↓
Phase 4D periodic / multivariate
       ↓
Phase 4E union / decorated / parsing / splitting
```

`IntervalUnion2`の内部型だけは、Phase 4Dの実装試験で必要ならpublic化前にtest/internal型として先行できる。公開APIはPhase 4Eでreviewする。

## 9. Backend選択

### 9.1 Core managed-only範囲

Phase 4Aと、既存方向付き四則演算だけでtightに構成できるPhase 4B機能はmanaged-onlyを原則とする。

### 9.2 初等関数

初等関数endpoint kernelは次を候補とする。

1. 誤差境界を証明したmanaged polynomial/table implementation
2. correctly-rounded libraryのmanaged移植
3. optional native MPFR/CRlibm系backend

採用は関数群ごとに行える。ただし同一公開関数の結果はbackend間でcanonical bitwise equivalentでなければならない。

`inari`はpinned implementationで初等関数の方向付き端点にMPFRを使用しているため、MPFRをreference oracleおよびproduction候補の一つとして扱う。production採用の有無は、配布、ABI、license、NativeAOT、性能を含むdecision reportで確定する。

## 10. API変更規則

Phase 2で固定した基本`Interval` APIは変更しない。

- 新しいrelation/propertyはadditive changeとする。
- 数学関数は`IntervalMath`へ追加する。
- 非連結結果やdecorationは別型とする。
- backend選択optionを初版public APIへ出さない。
- 同じ概念に複数の意味がある場合は、演算子ではなく明示的な名前を用いる。

## 11. TDDと検証

各Phase 4実装はTDDで進める。

1. semantics、domain、特殊値の失敗test
2. scalar endpoint kernelの失敗test
3. exact/MPFR oracleとの差分test
4. production実装
5. property test
6. x64 / ARM64差分test
7. SIMD/native backend差分test

random testだけに依存せず、次を固定fixtureとする。

- 0、signed zero、Empty、Entire
- 最小subnormal、最小normal、最大有限値
- 関数の定義域境界
- 単調性の切替点
- 三角関数の極値・極・branch cutの直前、一致、直後
- overflow / underflow
- endpointが隣接binary64となるhard case

## 12. CIと診断artifact

Phase 4の実装PRでは、確認時点のPR current HEADと`head_sha`が一致するrunだけをCI evidenceとする。

CIは既存のx64 / ARM64 matrixを拡張し、少なくとも次を保存する。

- test result
- standard output / standard error
- function名、入力endpoint bits、backend
- domain clipping結果
- critical point / pole判定
- expected/actual endpoint bits
- MPFRまたはexact oracle結果
- reference revision
- CPU featureとruntime情報
- cross-architecture corpus差分

artifact uploadは成功・失敗にかかわらず実行する。

## 13. 参照実装

- `unageek/inari`
  - pinned commit: `18b83a571d7681c76067bc38d90a74e8be29f545`
  - set operation、relation、numeric property、elementary function semantics
  -初等関数のMPFR方向付き丸め
- `unageek/ITF1788`
  - pinned commit: `d8c2a64478ebdc9cbde6ccef33eaad3bed60ed81`
  - test-only conformance corpus source
- `mskashi/kv`
  - pinned commit: `c7f8f2324a0e403cca6b39f46088a22843d440db`
  - directed algebraic endpoint primitiveの参照

参照コードを翻案・移植する場合は、既存詳細設計のlicense/notice規則を適用する。

## 14. 実装開始条件

Phase 4Aは次をすべて満たした後に開始する。

- Phase 1～3の対象成果物が完了している。
- 本書と3つの詳細設計がreview済みである。
- public API候補の名称と配置が承認されている。
- conformance/oracle corpusの追加方針が既存test基盤と統合されている。
- 診断artifactを保存するworkflowが存在する。

本設計文書の追加だけではPhase 4実装開始完了とはみなさない。