# 区間演算 基本設計

## 1. 目的

Devo6.Interval は、C# から利用できる高速な区間演算ライブラリを提供する。

本ライブラリでは、単に浮動小数点の上下限を計算するだけでなく、次を基本要件とする。

- 真の計算結果を必ず包含する区間を返す。
- IEEE 754 binary64 (`double`) の性質を前提として、外向き丸めを行う。
- IEEE 1788.1-2017 の区間演算意味論へ可能な限り寄せる。
- 公開 API と内部表現を分離し、SIMD に適した内部表現を採用する。
- 高速化のために SIMD、方向付き丸め、符号分類等を利用できる構造とする。
- 依存性問題は区間演算そのものの性質として扱い、基本 `Interval` 型が変数相関を暗黙に保持する設計にはしない。

初期の基本型は IEEE 1788.1 が対象とする binary64 に合わせて `double` を対象とする。

## 2. 参照仕様・参照実装

### 2.1 意味論の基準

区間の意味、Empty、Entire、非有界区間、定義域外を含む演算等の基本仕様は IEEE 1788.1-2017 を基準とする。

- IEEE 1788.1-2017: IEEE Standard for Interval Arithmetic (Simplified)
- binary64 の端点を持つ区間を対象とする。

### 2.2 主な参照実装

実装上は Rust の `inari` を主要な参照実装とする。

`inari` は IEEE 1788.1-2017 に準拠し、非空区間 `[a, b]` を SIMD ベクトル上で `[-a, b]` として保持している。この表現は本ライブラリが採用する内部表現方針と一致する。

性能設計の参考として、以下も参照する。

- `kv`: 方向付き丸め、丸めモード切替削減、AVX-512 embedded rounding 等
- Boost.Interval: rounding policy、丸め環境管理
- `inari`: SIMD に適した `[-Lower, Upper]` 表現と符号分類

## 3. 区間の意味

数学的な区間は次のように表す。

```text
[Lower, Upper]
```

非空区間については次を満たす。

```text
Lower <= Upper
Lower != +Infinity
Upper != -Infinity
```

`-Infinity` は下限として、`+Infinity` は上限として使用できる。これらは区間内の実数要素ではなく、区間がその方向へ非有界であることを表す端点として扱う。

### 3.1 Entire

全実数を表す区間を `Entire` とする。

```text
[-Infinity, +Infinity]
```

### 3.2 Empty

空集合を表す `Empty` を正式な区間値として持つ。

通常の `Lower > Upper` をそのまま通常区間として保持するのではなく、空集合は専用状態として扱う。

内部表現は `inari` と同様に `[NaN, NaN]` を利用する方式を第一候補とする。これにより通常区間の `[-Lower, Upper]` 表現と区別できる。

### 3.3 NaN / NaI

`NaN` を通常区間の端点として受け入れない。

```text
[NaN, 1.0]
[0.0, NaN]
```

は通常の `Interval` として不正入力とする。

IEEE 1788 の NaI (Not an Interval) は decorated interval の概念であるため、初期の bare `Interval` には混在させない。将来 decorated interval を追加する場合は別型で扱う。

### 3.4 +0.0 / -0.0

IEEE 754 では `+0.0` と `-0.0` は異なるビット表現を持つが、区間の集合としては同じ実数 0 として扱う。

内部表現では符号反転や SIMD 演算によりどちらの zero 表現も生じ得るため、内部の zero の符号自体を区間の意味として利用しない。比較、Hash、文字列表現等で正規化が必要な箇所は IEEE 1788.1 / `inari` の振る舞いに合わせる。

## 4. 内部表現

### 4.1 基本方針

外部から見える区間

```text
[Lower, Upper]
```

に対して、内部では次の順序で保持する。

```text
[-Lower, Upper]
```

したがって区間 `[l, u]` は内部では次の値になる。

```text
[-l, u]
```

この表現を本設計では negated-lower representation と呼ぶ。

### 4.2 生成と取得

区間生成時に下限を一度だけ符号反転する。

概念上は次の処理となる。

```csharp
internalLower = -lower;
internalUpper = upper;
```

利用者が下限を取得するときだけ符号を戻す。

```csharp
Lower => -internalLower;
Upper => internalUpper;
```

演算のたびに通常の `[Lower, Upper]` へ戻さず、区間オブジェクトの寿命中は原則として `[-Lower, Upper]` のまま処理する。

### 4.3 物理格納形式

論理的な格納順は `[-Lower, Upper]` に固定する。

実際の物理型は SIMD バックエンド設計で確定するが、2 個の `double` をそのまま SIMD レーンとして扱える配置とする。`Vector128<double>` 等を直接保持する方式、および同等のレイアウトを候補とする。

公開 API から内部レイアウトへ依存できないようにする。

## 5. 外向き丸め

### 5.1 必要性

通常の `double` 演算は原則として最近接丸めを行う。しかし区間演算では、丸め誤差により真値を区間の外へ追い出してはならない。

したがって下限は真値以下へ、上限は真値以上へ丸める。

```text
Lower: round toward -Infinity
Upper: round toward +Infinity
```

この処理を外向き丸め (outward rounding) とする。

### 5.2 数値例

binary64 の `0.1` と `0.2` を正確な実数として加えた結果は、隣接する `double` の間に位置する。

概念的には次の関係になる。

```text
0.29999999999999998889...   // 下側の double
        <
0.30000000000000001665...   // binary64 の 0.1 と 0.2 の正確な加算結果
        <
0.30000000000000004440...   // 上側の double
```

通常の最近接丸めだけで点区間を作ると、真値を包含できない場合がある。

区間加算では次のように外向きへ丸める。

```text
[roundDown(la + lb), roundUp(ua + ub)]
```

## 6. negated-lower representation と上向き丸め

下向き丸めには次の恒等関係を利用できる。

```text
roundDown(x) = -roundUp(-x)
```

加算の場合、内部表現を利用すると次の計算になる。

入力:

```text
A = [-la, ua]
B = [-lb, ub]
```

両レーンを上向き丸めで加算する。

```text
roundUp(A + B)
=
[roundUp(-la - lb), roundUp(ua + ub)]
```

第 1 レーンは次と等価である。

```text
roundUp(-(la + lb))
= -roundDown(la + lb)
```

したがって結果は、そのまま本ライブラリの内部表現となる。

```text
[-roundDown(la + lb), roundUp(ua + ub)]
```

演算後に下限の符号を戻す必要はない。`Lower` を外部へ公開するときだけ符号を戻す。

## 7. SIMD 方針

### 7.1 1 区間内の SIMD

`[-Lower, Upper]` を 2 レーンとして扱うことで、加算等では下限と上限を同時に処理できる。

```text
lane 0: -Lower
lane 1:  Upper
```

両レーンで上向き丸めを使用できるため、下限用・上限用に CPU の丸めモードを切り替える必要を減らせる。

### 7.2 複数区間の SIMD

複数区間を一括処理する API では、次のようなレイアウトでより広い SIMD レジスタへ詰めることができる。

```text
[-L0, U0, -L1, U1, ...]
```

AVX-512 の 512 bit packed double であれば、1 ベクトルに 4 区間分の 8 個の `double` を配置できる。

### 7.3 AVX-512 embedded rounding

対応 CPU / .NET ランタイムでは、AVX-512 の embedded rounding を利用し、対応する浮動小数点命令を `round toward +Infinity` 付きで実行するバックエンドを検討する。

これにより MXCSR のグローバル丸めモードを書き換えず、命令単位で方向付き丸めを行える。

AVX-512 非対応環境については、以下を候補として詳細設計で比較する。

- MXCSR / floating-point environment を利用した方向付き丸め
- TwoSum / TwoProduct および FMA を利用するソフトウェア方式
- 必要に応じた安全側の ULP 拡張

バックエンドが異なっても、公開される区間の包含保証と意味論は同一にする。

## 8. 基本演算の方針

### 8.1 加算

外部表現:

```text
[la, ua] + [lb, ub]
= [la + lb, ua + ub]
```

外向き丸めを含めると次になる。

```text
[roundDown(la + lb), roundUp(ua + ub)]
```

内部表現では 2 レーンの上向き丸め加算で求める。

### 8.2 減算

外部表現:

```text
[la, ua] - [lb, ub]
= [la - ub, ua - lb]
```

内部表現では B の 2 レーンを入れ替えた値を A へ加算する形に変換できる。

```text
A          = [-la, ua]
swap(B)    = [ ub, -lb]
A+swap(B)  = [ub-la, ua-lb]
```

これは結果区間の内部表現

```text
[-(la-ub), ua-lb]
```

と一致する。

### 8.3 符号反転

```text
-[l, u] = [-u, -l]
```

内部表現では 2 レーンの入れ替えにより実現できる。

### 8.4 乗算

一般には次の 4 候補を考える必要がある。

```text
la * lb
la * ub
ua * lb
ua * ub
```

下限は候補の最小、上限は候補の最大となる。

ただし毎回 4 候補を無条件に計算せず、区間を正・負・0 跨ぎ等に分類し、符号の組み合わせから必要な積だけを計算する。`inari` の符号分類を主要な参照とする。

### 8.5 除算

除数区間が 0 を含むかどうかで処理を分ける。

IEEE 1788.1 の set-based な意味論を基準とする。

例:

```text
[1, 2] / [0, 0]   -> Empty
[0, 0] / [0, 0]   -> Empty
[1, 2] / [-1, 1]  -> Entire
```

最後の例の実際の集合は非連続だが、単一の `Interval` では表現できないため、その集合を包含する最小の連続区間として `Entire` を返す。

将来、非連続な結果を 2 区間として返す extended division API を別途設ける余地を残す。

## 9. 依存性問題

### 9.1 性質

通常の区間演算は、同一変数が式中で複数回使用されたという相関情報を保持しない。

例えば

```text
x = [1, 2]
```

について数学的には `x - x = 0` だが、基本区間演算だけでは

```text
[1, 2] - [1, 2] = [-1, 1]
```

となる。

これは実装不良ではなく、上下限だけを保持する通常の区間演算が持つ依存性問題である。

### 9.2 本ライブラリでの扱い

基本 `Interval` 型は変数 ID や相関を保持しない。

式変形、単調性解析、区間分割、Affine Arithmetic、Taylor Model 等による依存性問題の低減は、基本区間演算とは別レイヤーの機能として扱う。

特に次の 2 方式は将来の上位 API として有効である。

- 単調性を利用し、端点評価だけでより狭い区間を得る。
- 入力区間を分割して各部分区間を評価し、過大評価を減らす。

区間分割は同一演算を多数の区間へ適用するため、バッチ SIMD / 並列処理と相性がよい。

## 10. 公開 API と実装バックエンド

公開 API は本ライブラリ独自の `Interval` 型とし、利用者へ内部 SIMD レイアウトや特定の native ライブラリ型を公開しない。

概念上、最低限次の機能を持つ。

```text
Interval
  Lower
  Upper
  Empty
  Entire
  IsEmpty
  IsEntire
  + - * /
```

正確な C# API 名、コンストラクタ / factory の選択、例外型等は詳細設計で決定する。

### 10.1 Native ライブラリ利用について

`kv` 等の native ライブラリを C ABI の薄い shim 経由で利用する方式は実装候補として残す。

ただし `a + b` のような非常に軽量なスカラー演算ごとに P/Invoke 境界を越えると、interop overhead が演算本体を上回る可能性がある。

native backend を採用する場合でも、公開 API は独自型のままとし、native 呼び出しは `Span<Interval>` 等を対象にしたバッチ演算を主用途として検討する。

全面 C# 実装と native 併用の最終選択は、四則演算・方向付き丸め・超越関数のベンチマークおよび実装負荷を比較して決定する。

## 11. 超越関数

`sin`, `cos`, `exp`, `log` 等についても、最終的には真値を必ず包含する結果を返す必要がある。

通常の `Math.Sin` 等を呼び、無条件に 1 ULP だけ広げれば常に十分であるとは限らないため、正しい argument reduction、誤差境界、極値・特異点の処理が必要となる。

この領域は四則演算より実装負荷が大きいため、基本 `Interval` と四則演算を先に成立させ、超越関数の具体アルゴリズムは詳細設計で `kv` / `inari` 等を参照して決定する。

## 12. 現時点の決定事項

- 初期の区間端点型は `double` とする。
- 区間意味論は IEEE 1788.1-2017 を基準とする。
- 実装上の主要参照は `inari` とする。
- 外部表現は `[Lower, Upper]` とする。
- 内部表現は `[-Lower, Upper]` とする。
- 下限 getter で内部下限の符号を戻す。
- 演算中は原則として negated-lower representation を維持する。
- 真値包含のため外向き丸めを必須とする。
- SIMD では両レーンを上向き丸めで処理できる構造を活用する。
- `Empty` / `Entire` を正式な区間値として扱う。
- `NaN` は通常区間の端点として受け入れない。
- bare `Interval` に NaI / decoration を混在させない。
- 基本 `Interval` は依存性情報を保持しない。
- 公開 API は実装バックエンドから独立させる。

## 13. 詳細設計へ持ち越す事項

- `Interval` の具体的な C# API と例外設計
- `Empty` の最終的な内部ビット表現と正規化規則
- signed zero の public API / Hash / serialization における正規化規則
- SIMD 物理型とメモリレイアウト
- AVX-512、AVX2/SSE、ARM64 等のバックエンド選択
- AVX-512 非対応時の正確な方向付き丸め方式
- 乗算・除算の符号分類と SIMD 命令列
- FMA の利用方針
- `sqrt` の下向き丸め方式
- 超越関数の実装方式
- C# 完全 managed 実装と native backend の採否
- バッチ API の形
- decorated interval / NaI / extended division の追加範囲
- IEEE 1788.1 conformance test の導入方法
