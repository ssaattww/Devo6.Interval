# 区間演算 拡張機能 詳細設計

## 1. 文書情報

- 対象: Phase 4E
- 前提:
  - `IntervalArithmetic.md`
  - `IntervalArithmetic.Revision3.md`
  - `IntervalNonArithmetic.Roadmap.md`
  - `IntervalSetAndNumeric.md`
  - `IntervalMathFunctions.md`
- 主要参照:
  - `unageek/inari` commit `18b83a571d7681c76067bc38d90a74e8be29f545`
  - `unageek/ITF1788` commit `d8c2a64478ebdc9cbde6ccef33eaad3bed60ed81`
- 作成日: 2026-08-30
- 設計状態: review required

本書では、単一の連結bare intervalだけでは表現できない結果、定義状態、厳密な文字列変換、interchangeおよび区間分割を設計する。

## 2. 機能境界

### 2.1 core型を拡張し過ぎない

`Interval`は連結なbare intervalのまま維持する。

次を`Interval`の内部flagとして混在させない。

- 2本の非連結区間
- decoration
- NaI
- parser error
- split metadata

### 2.2 拡張型

```text
Interval          連結なbare interval
IntervalUnion2    0～2個の互いに分離した連結成分
DecoratedInterval Interval + Decoration、またはNaI
```

区間分割は集合unionではなく、境界点を共有する2子区間であるため、`IntervalUnion2`とは別APIにする。

## 3. `IntervalUnion2`

### 3.1 目的

zeroを跨ぐ除数による商等、真の結果が最大2本の連結成分になる演算で、`Entire`へhullする前の情報を保持する。

### 3.2 public API候補

```csharp
namespace Devo6.Numerics;

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

初版では`IEnumerable<Interval>`を実装しない。value-type enumeratorの追加は可能だが、boxingとallocationの有無をAPI reviewで確認する。

### 3.3 canonical representation

```text
Count = 0:
    First  = Empty
    Second = Empty

Count = 1:
    First  = nonempty
    Second = Empty

Count = 2:
    First,Second are nonempty
    First.Lower <= First.Upper < Second.Lower <= Second.Upper
```

2成分が接触またはoverlapする場合は`ConvexHull`へ統合し、Count=1とする。

```text
First.Upper == Second.Lower -> merge
```

閉区間同士が端点を共有するため、unionは連結だからである。

### 3.4 default

`default(IntervalUnion2)`はCount=0のempty unionとする。内部Countを明示的に保持し、defaultの`Interval.Zero` fieldを成分として解釈しない。

### 3.5 construction

public constructorは初版では提供しない。演算結果以外からunionを作る必要が確認された場合に、canonicalization付きfactoryを追加する。

internal factory:

```text
Create0()
Create1(interval)
Create2(first,second)
```

`Create2`はEmpty除去、sort、merge、signed-zero正規化を行う。

### 3.6 equality

canonical component列が同じ場合に等しい。Empty NaN payloadを比較しない。

## 4. Extended division

### 4.1 API

```csharp
public static partial class IntervalMath
{
    public static IntervalUnion2 DivideToUnion(
        Interval numerator,
        Interval denominator);

    public static IntervalUnion2 ReciprocalToUnion(
        Interval value);
}
```

`/` operatorは従来どおり単一`Interval`を返し、非連結結果をconvex hullへ変換する。`DivideToUnion`は最大2成分を保持する。

### 4.2 point-set semantics

```text
DivideToUnion(X,Y)
= { x/y | x∈X, y∈Y, y != 0 }
```

各成分endpointはtightに方向丸めする。

### 4.3 共通case

```text
X Empty or Y Empty -> Count 0
Y == Zero          -> Count 0
Y excludes zero    -> Count 1 containing X / Y
X == Zero and Y has nonzero member -> Count 1 containing Zero
```

### 4.4 denominatorがzeroに片側から接する場合

結果は連結なので、既存`/` operatorと同じ1成分を返す。

`Y=[0,d]`, `d>0`:

| X | Result |
|---|---|
| Zero | `[0,0]` |
| positive/nonnegative | `[RD(a/d),+Infinity]` |
| negative/nonpositive | `[-Infinity,RU(b/d)]` |
| mixed | Entire |

`Y=[c,0]`, `c<0`は符号を反転した対称caseとする。

### 4.5 denominatorがzeroを跨ぐ場合

`Y=[c,d]`, `c<0<d`。

#### numeratorがstrictly positive

`X=[a,b]`, `0<a<=b`。

```text
First  = [-Infinity, RU(a/c)]
Second = [RD(a/d), +Infinity]
```

#### numeratorがstrictly negative

`X=[a,b]`, `a<=b<0`。

```text
First  = [-Infinity, RU(b/d)]
Second = [RD(b/c), +Infinity]
```

#### numeratorがzeroを含む

```text
X == Zero -> Zero
otherwise -> Entire
```

nonzero幅でzeroを含むnumeratorは、zeroと正負の任意に大きい商を生成できるため、結果集合そのものがEntireとなる。

### 4.6 ordinary divisionとの関係

```text
(numerator / denominator)
== DivideToUnion(numerator,denominator).ConvexHull
```

全caseでproperty testする。

### 4.7 reciprocal union

```text
ReciprocalToUnion(X)
= DivideToUnion([1,1],X)
```

implementationは共通kernelを利用するが、不要なpublic interval生成を避ける内部entry pointを持てる。

## 5. Reverse multiplication / two-output division

### 5.1 API候補

solver寄りの機能であるため、`IntervalMath`ではなく`IntervalContractor`へ配置する。

```csharp
public static class IntervalContractor
{
    public static IntervalUnion2 ReverseMultiply(
        Interval product,
        Interval factor);
}
```

argument orderは意味が明確になるよう、`product`を先、既知の`factor`を後にする。

### 5.2 semantics

```text
ReverseMultiply(P,Y)
= { z ∈ R | exists y∈Y : z*y ∈ P }
```

通常除算との違いは`y=0`を候補から除外しない点である。

### 5.3 algorithm

```text
P Empty or Y Empty -> empty union

0 ∈ P and 0 ∈ Y -> Entire

otherwise -> DivideToUnion(P,Y)
```

理由:

- `0∈P`かつ`0∈Y`なら、任意の`z`について`z*0=0∈P`となる。
- それ以外では`y=0`は解を追加せず、nonzero factorによるquotient集合と一致する。

例:

```text
ReverseMultiply([1,2], Zero) = empty union
ReverseMultiply(Zero, Zero)  = Entire
ReverseMultiply([0,2], Zero) = Entire
ReverseMultiply([1,2], [-1,1])
  = [-Infinity,-1] union [1,+Infinity]
```

### 5.4 standard naming

IEEE系の名称`mulRevToPair`をXML documentationとconformance mappingへ記載する。C# public methodはargumentの意味が分かる`ReverseMultiply`を使用する。

## 6. Cancellative addition/subtraction

### 6.1 API候補

```csharp
public static class IntervalContractor
{
    public static Interval CancelSubtract(
        Interval total,
        Interval term);

    public static Interval CancelAdd(
        Interval total,
        Interval term);
}
```

### 6.2 `CancelSubtract`

`total=[a,b]`、`term=[c,d]`について、次を満たすtightest interval `z`を返す。

```text
term + z contains total
```

次の前提をすべて満たす場合:

- totalとtermがnonemptyかつbounded
- exact width(total) >= exact width(term)

結果:

```text
[RD(a-c), RU(b-d)]
```

前提を満たさない場合は`Entire`。

通常のsubtractionとは異なる。

```text
CancelSubtract(total,term) generally != total - term
CancelSubtract(total,term) subset of total - term
```

### 6.3 width比較

`Width` propertyのrounded resultだけを比較しない。2Sum expansion等を用いて、

```text
exact(b-a) >= exact(d-c)
```

を判定する。

比較を保証できない実装は安全側に`Entire`を返すが、正式scalar referenceは全binary64 bounded inputsでexact relationを決定できなければならない。

### 6.4 `CancelAdd`

```text
CancelAdd(total,term)
= CancelSubtract(total,-term)
```

public APIを採用するかはsolver利用例を確認して決定する。単なる利用者向け逆演算と誤認されないXML documentationを必須とする。

## 7. `DecoratedInterval`

### 7.1 目的

bare intervalでは失われる次の情報を保持する。

- 計算が入力全体で定義されていたか
- 連続性
- common intervalであるか
- 不正なinterval datumであるNaI

### 7.2 Decoration

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

数値順は品質順であり、複数入力の基本伝播はminimumを使用する。

- `Ill`: ill-formed / NaI
- `Trv`: trivial
- `Def`: defined
- `Dac`: defined and continuous
- `Com`: common

### 7.3 public API候補

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

    // operators and DecoratedIntervalMath are added by function group
}
```

### 7.4 default

`default(DecoratedInterval)`が偶然`Zero_Com`等にならないよう、representationを設計する。

候補は、`Decoration`のdefault値`Ill=0`を利用し、defaultをNaIとする。

```text
default(DecoratedInterval).IsNaI == true
```

内部`Interval` fieldのdefault Zeroは、Decoration Illの場合に観測しない。

### 7.5 `FromInterval`

```text
bounded nonempty -> Com
unbounded        -> Dac
Empty            -> Trv
```

### 7.6 NaI

- NaIはbare `Interval`ではない。
- `TryGetInterval`はfalseを返し、out値を`Interval.Empty`とする。
- NaI入力を受けたdecorated operationはNaIを返す。
- NaIの追加payloadは初版では持たない。

### 7.7 decoration propagation

operationごとに最大可能decoration`opDec`を決める。

```text
resultDec = min(input decorations, opDec)
```

例:

- 全入力上で定義・連続な加算は入力decorationのminimum。
- zeroを含む除数は`Trv`以下。
- `Sqrt([-1,4])`はbare result `[0,2]`だが、入力全体で定義されないため`Trv`。
- `Tan`がpoleを跨ぐ場合は`Trv`。

operation別policyを分散したif文として作らず、conformance sourceと対応する`DecorationPolicy`へ集約する。

### 7.8 C# equality adaptation

C#の`Equals`、`==`、Hashはreflexiveな値等値性とする。

```text
NaI == NaI -> true
```

- NaI同士は等しい。
- 非NaIはinterval partとdecorationの両方を比較する。

IEEEの`equal`相当は`SemanticallyEquals`で提供する。

```text
NaI.SemanticallyEquals(any) -> false
non-NaI -> decorationを無視してinterval partを比較
```

C# collection keyとしての安全性とIEEE semantic operationを混同しない。

### 7.9 decorated math API

`DecoratedIntervalMath`を別static classとする。

```csharp
public static class DecoratedIntervalMath
{
    public static DecoratedInterval Sqrt(DecoratedInterval value);
    public static DecoratedInterval Exp(DecoratedInterval value);
    public static DecoratedInterval Sin(DecoratedInterval value);
}
```

bare/decoration overloadを同じclassへ大量に混在させない。演算子は`DecoratedInterval`同士に限る。

## 8. 文字列入力

### 8.1 API候補

```csharp
public readonly partial struct Interval
{
    public static Interval Parse(ReadOnlySpan<char> text);
    public static bool TryParse(
        ReadOnlySpan<char> text,
        out Interval interval);

    public static Interval ParseExact(ReadOnlySpan<char> text);
    public static bool TryParseExact(
        ReadOnlySpan<char> text,
        out Interval interval);
}
```

`IParsable` / `ISpanParsable`の実装は、providerと標準interval syntaxの関係をAPI reviewした後に追加する。

### 8.2 Phase 4E初版syntax

```text
Empty
Entire
[a,b]
[a]
```

endpoint token:

- decimal floating literal
- decimal integer
- `Infinity`, `+Infinity`, `-Infinity`
- exact hexadecimal binary literal

uncertain form、rational token、decoration suffixは次のsubphaseへ分ける。

### 8.3 outward decimal parsing

`double.Parse`後に点区間を作らない。

10進tokenを次として正確に解析する。

```text
sign * integerSignificand * 10^decimalExponent
```

production parserは`BigInteger`等を使用してよい。parserはhot arithmetic pathではなく、正しい文字列変換が優先される。

`[x]`でxがbinary64に正確に表現できない場合:

```text
[RoundDown(x), RoundUp(x)]
```

例としてdecimal 0.1は点区間にしない。

`[a,b]`:

```text
lower = RoundDown(exact a)
upper = RoundUp(exact b)
```

入力のexact real relationとして`a <= b`であることを先に確認する。

### 8.4 range外の有限decimal

有限なexact decimalがbinary64 range外の場合も、可能な限りtight enclosureを返す。

```text
x > double.MaxValue:
    [double.MaxValue,+Infinity]

x < -double.MaxValue:
    [-Infinity,-double.MaxValue]
```

無限大tokenそのものと有限decimal overflowをcase metadataで区別する。

### 8.5 invalid input

- syntax error
- NaN endpoint
- exact lower > exact upper
- lower `+Infinity`
- upper `-Infinity`
- `[+Infinity,+Infinity]`
- `[-Infinity,-Infinity]`

`Parse`は`FormatException`を基本とし、`TryParse`はfalseとEmptyを返す。例外型の最終選択は.NET parsing conventionとのAPI reviewで固定する。

### 8.6 exact parsing

`ParseExact`は、指定されたendpointが要求されたbinary64 endpointへ正確に変換できる場合だけ成功する。

- exact hexadecimal literalは主要なround-trip形式とする。
- inexact decimal singletonは失敗する。
- intervalの両endpointが個別にexactであれば成功する。

## 9. 文字列出力

### 9.1 diagnostic `ToString`

既存Phase 1の`ToString()`はhuman-readable diagnostic用途のままとし、永続化契約へ昇格させない。

### 9.2 format候補

```text
G: human-readable valid enclosure
R: exact round-trip interval representation
X: exact hexadecimal endpoints
```

`R`と`X`は、対応するparseによってcanonical intervalへ完全に戻る。

### 9.3 exact representation

```text
Empty
[-0x0p+0,0x0p+0]
[0x1.0000000000000p+0,0x1.0000000000001p+0]
[-Infinity,+Infinity]
```

hex syntaxの詳細、指数表現、case、signed zeroを固定し、cultureに依存させない。

### 9.4 outward human-readable output

`G`で桁数を削る場合も、表示文字列を再parseした区間が元区間を包含することを保証する。単純なendpoint round-to-nearest formatで元区間を狭めない。

## 10. binary interchange

### 10.1 raw memoryを使用しない

`MemoryMarshal.AsBytes`等でprivate field layoutをそのまま永続化しない。内部`[-Lower,Upper]`表現と将来のSIMD layoutをwire contractへしない。

### 10.2 version 1 format候補

外部canonical stateを明示的にencodeする。

```text
byte 0: version = 1
byte 1: state   = 0 Empty, 1 Interval
byte 2..9:   external Lower IEEE 754 bits, little endian
byte 10..17: external Upper IEEE 754 bits, little endian
```

全長18 byte。

- Emptyではendpoint bytesを0に固定する。
- Entireはstate=1、Lower=-Infinity、Upper=+Infinity。
- zeroはLower=-0.0、Upper=+0.0。
- decoderはreserved state、noncanonical zero、NaN endpoint、不正順序を拒否またはcanonicalizeする規則をversionごとに固定する。

### 10.3 API候補

```csharp
public bool TryWriteLittleEndian(Span<byte> destination);

public static bool TryReadLittleEndian(
    ReadOnlySpan<byte> source,
    out Interval interval);
```

big endianが必要になった場合は別methodを追加し、platform native endianをwire formatにしない。

### 10.4 JSON

JSON converterはcore assemblyへ暗黙登録しない。別converter型またはintegration packageとし、number配列・object・stringのどれをcanonicalとするかを専用設計で決める。

## 11. 区間分割

### 11.1 unionとの違い

branch-and-bound用の分割は、境界点を両方の子へ含める。

```text
[a,m] and [m,b]
```

2成分はmで接触するため、`IntervalUnion2`ではcanonicalに1成分へmergeされる。したがって分割結果をunion型で返さない。

### 11.2 API候補

```csharp
public readonly partial struct Interval
{
    public bool TrySplitAt(
        double splitPoint,
        out Interval left,
        out Interval right);

    public bool TryBisect(
        out Interval left,
        out Interval right);
}
```

### 11.3 `TrySplitAt`

成功条件:

- intervalがnonempty
- splitPointがfinite
- `Lower < splitPoint < Upper`

結果:

```text
left  = [Lower, splitPoint]
right = [splitPoint, Upper]
```

- coverageにgapがない。
- 共有点以外のoverlapがない。
- 両方が元区間のproper subset。

失敗時はfalseを返し、left/rightをEmptyとする。

非有界区間も、finiteなstrict interior pointを指定すれば分割できる。

### 11.4 `TryBisect`

初版はbounded intervalだけを自動二分する。

失敗条件:

- Empty
- unbounded
- singleton
- lowerとupperの間にstrict interiorとなるbinary64が存在しない

手順:

1. overflow-safe midpoint candidateを求める。
2. candidateがLower以下なら`BitIncrement(Lower)`を試す。
3. candidateがUpper以上なら`BitDecrement(Upper)`を試す。
4. strict interiorを作れなければfalse。
5. `TrySplitAt(candidate)`を呼ぶ。

隣接binary64 endpointの区間は、binary64 endpoint表現におけるatomic intervalとして自動分割しない。

### 11.5 unbounded bisection

Entireや片側非有界区間の自動pivotには任意性があるため、初版`TryBisect`でheuristicを暗黙に採用しない。利用者が`TrySplitAt(0)`等で明示する。

将来solver packageで探索戦略に応じたpivot policyを提供する。

### 11.6 batch split

allocation-freeのwork queue APIはsolver側の責務とし、core `Interval`へcollection APIを追加しない。

## 12. parsingとdecorationの関係

- bare parserが不正datumを受けた場合は失敗する。
- decorated parserは、不正decorated datumをNaIへ変換するstandard operationと、C# parse failureを区別する。
- syntaxとして正しい`[nai]`は`DecoratedInterval.NaI`になり得る。
- bare `Interval.Parse("NaI")`は失敗する。

text-to-intervalのpossibly-undefined状態をどのC# result型で表すかは、uncertain/rational text subphaseで専用設計する。

## 13. 内部ファイル構成

```text
src/Devo6.Interval/
  Interval.Parsing.cs
  Interval.Formatting.cs
  Interval.Interchange.cs
  Interval.Splitting.cs
  IntervalUnion2.cs
  DecoratedInterval.cs
  Decoration.cs
  DecoratedInterval.Operators.cs
  DecoratedIntervalMath.cs
  IntervalContractor.cs
  Internal/
    IntervalUnion2Canonicalizer.cs
    ExtendedDivisionKernel.cs
    DecorationPolicy.cs
    ExactDecimalParser.cs
    Binary64TextConverter.cs
    IntervalBinaryCodec.cs
```

optional JSON integrationは別directoryまたは別projectとする。

## 14. TDD実装順

1. `IntervalUnion2` canonical state/equality
2. `DivideToUnion`
3. `ReciprocalToUnion`
4. `ReverseMultiply`
5. cancellative add/sub
6. `Decoration` / default NaI
7. `DecoratedInterval` construction/equality
8. decorated arithmetic
9. decorated elementary functions
10. exact decimal parser
11. outward interval parser
12. exact/human-readable formatter
13. binary interchange
14. `TrySplitAt`
15. `TryBisect`

各機能は失敗testから開始する。parser/formatterはround-trip、invalid syntax、range外finite decimalを先に固定する。

## 15. 決定的fixture

### 15.1 union/extended division

- Count 0/1/2
- reverse-order inputのsort
- overlap/touching component merge
- positive/negative numeratorとcross-zero denominator
- numerator Zero
- numeratorがzeroを含むnonzero区間
- one-sided zero denominator
- denominator Zero/Empty
- ordinary division hullとの一致

### 15.2 reverse/cancel

- `Zero` factorとproductの0包含/非包含
- cross-zero factorの2成分
- bounded equal-width cancellation
- widthが1 ULP差のcase
- unbounded/Empty precondition failure
- cancellation resultとordinary subtractionの差

### 15.3 decorated

- default NaI
- NaI propagation
- Empty/Entire/boundedのinitial decoration
- domain clippingによるTrv
- discontinuity/poleによるdegrade
- `Equals`と`SemanticallyEquals`の差
- Hashのreflexivity/consistency

### 15.4 parsing

- exact decimal/hex
- inexact singleton decimal 0.1
- endpoint range外finite decimal
- signed zero
- Empty/Entire
- NaN、reversed bound、不正Infinity singleton
- whitespace/case policy
- malformed exponent
- very long significand
- resource limit超過

parserは入力長、digit数、exponent絶対値に明示的なresource limitを持ち、過大入力で無制限に`BigInteger`を拡張しない。limit超過時の`Parse`/`TryParse`挙動を固定する。

### 15.5 interchange

- 全state
- each endian byte pattern
- invalid version/state/length
- NaN endpoint
- noncanonical zero
- round-trip
- internal field layoutを変えてもwire bytesが不変

### 15.6 splitting

- normal bounded
- odd/even number of representable endpoints
- adjacent endpoints
- subnormal range
- huge same-sign range
- signed zero crossing
- explicit split of unbounded interval
- invalid split point
- coverage/subset/progress property

## 16. Property test

```text
union.Count in [0,2]
Count=2 -> First.Upper < Second.Lower
union.ConvexHull contains every component
DivideToUnion(...).ConvexHull == ordinary division
ReverseMultiply result satisfies existential relation on sampled values
ParseExact(FormatExact(X)) == X
Parse(FormatRoundTrip(X)) == X
Decode(Encode(X)) == X
Split.Left union Split.Right covers original
Split.Left and Split.Right are subsets of original
```

existential/reverse relationはsampleだけをprimary proofにせず、sign-class matrixとITF1788 referenceを併用する。

## 17. CIとartifact

既存x64/ARM64 matrixへ次を追加する。

- extended result component count/order/bits
- decoration and NaI state
- parse token classificationとexact rational
- parser resource-limit metadata
- formatted textとreparsed bits
- wire bytes
- split pointとchild intervals

失敗時artifactにはexternal reference result、adaptation rule、expected-difference reasonを含める。

## 18. security/resource considerations

数値演算自体とは別に、parserはuntrusted textを受ける可能性がある。

- 最大入力長
- 最大significand digit数
- 最大exponent digit数
- recursionを使わないparser
- culture-dependent separatorの曖昧性排除
- exception messageへ入力全文を無制限に含めない
- JSON converterの深さ・token limit

を実装前に固定する。

binary decoderはlengthとstateを先に検証し、invalid datumから片側NaN等の内部不変状態を作らない。

## 19. 完了条件

- `IntervalUnion2`、`DecoratedInterval`、parsing、splittingの型境界がreview済み。
- 全canonical stateとdefault値が固定される。
- conformance/exact/reference fixtureがx64/ARM64で成功する。
- parse/format/interchangeのround-tripがbitwiseに成功する。
- parser resource limitが決定的に機能する。
- bare intervalとdecorated semantic differenceがdocumentationに明記される。
- API baselineとbreaking-change policyへ登録される。
- failure artifactからunion、decoration、parse、splitの分岐を追跡できる。
- current PR HEADと一致するCI runが成功する。
