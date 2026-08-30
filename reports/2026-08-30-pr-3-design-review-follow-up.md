# PR #3 詳細設計 指摘対応レポート

## Metadata

- Repository: `ssaattww/Devo6.Interval`
- Pull request: `#3` (`docs: add interval arithmetic detailed design`)
- Mode: review follow-up
- Base ref: `main`
- Base SHA: `ad5c058f8a4164b0c7d0763c65246914ea5d1c03`
- Working branch: `docs/detailed-interval-arithmetic-design`
- Reviewed technical HEAD: `da6e2ae04d35b01acfb307953a093c81c15342b8`
- Authoritative review report: `reports/2026-08-29-pr-3-design-review-exhaustive.md`
- PR current HEAD before design fix: `4e9cfc7787ee885b98ea7253f93bc48e876d3086`
- Design fix commit: `3b0c103dd5fb523b17b90ff9026d7384c0cf3ad4`
- Date: 2026-08-30

## Purpose

PR #3のexhaustive design reviewで示された有効指摘7件を、Phase 1実装を開始できる粒度で一括して解消した。

本作業は詳細設計の修正に限定し、production source、test、workflowは変更していない。

## Authoritative findings

有効指摘:

- `F-PR3-001` High: directed roundingのsubnormal、residual tie、overflow分岐が実装可能な粒度に達していない
- `F-PR3-002` Medium: AVX2/SSE2とx86 FMA capabilityが分離されていない
- `F-PR3-004` Medium: IEEE 1788.1 conformance test導入設計がない
- `F-PR3-005` Medium: exact rational / inari / kvの責務と実行方式が再現可能でない
- `F-PR3-006` Medium: x64 / ARM64 correctness gateをCI設計が実装していない
- `F-PR3-007` Medium: algorithm branch boundaryとresidual tieの決定的fixtureがない
- `F-PR3-008` Low: native backend判断が基本設計から追跡できない

`F-PR3-003`はexhaustive reviewで`withdrawn_erratum`とされた。今回の有効な実装指摘には含めず、設計書にもwithdrawnであることを記録した。

## Changed file

### `doc/Design/detail/IntervalArithmetic.md`

設計版を2へ更新し、次を追加・確定した。

- review finding traceability
- multiplication / divisionの実装完全な方向付き丸め疑似コード
- threshold equality rule
- scaled product comparisonとresidual tie
- division large-denominator early return
- division `q*y` high-part equality時のresidual判定
- exact rational / inari / kvのoracle責務
- pinned reference adapterとgolden corpus生成方式
- IEEE 1788.1 Phase 1 conformance matrix
- ITF1788固定SHAとadaptation rule
- Linux x64 / Linux ARM64 CI matrix
- architecture間canonical result比較
- threshold / tie / overflow固定fixture
- x86 FMAを独立判定するSIMD capability matrix
- native backendの後続decision gate

## Finding closure

### F-PR3-001 — High — addressed

#### 原因

初版は`kv`と同じ閾値・scale定数を列挙したが、値をどのように比較して`NextUp` / `NextDown`するかを確定していなかった。

特に次が欠けていた。

- multiplication scaled pathの`t`、`s`、`s2`比較
- `t == s`時のresidual tie
- divisionのsmall numerator / large denominator early return
- `r == xn`時の`r2`によるtie判定
- threshold一致時の経路
- finite overflowとexact Infinityの区別

#### 対応

詳細設計§9、§10で次を固定した。

Multiplication Up:

```text
t < s || (t == s && s2 > 0) -> NextUp(r)
```

Multiplication Down:

```text
t > s || (t == s && s2 < 0) -> NextDown(r)
```

Division Up:

```text
r < xn || (r == xn && r2 < 0) -> NextUp(q)
```

Division Down:

```text
r > xn || (r == xn && r2 > 0) -> NextDown(q)
```

small numeratorかつlarge denominatorでは次を直接返す。

```text
positive quotient:
  Up   -> +2^-1074
  Down -> +0.0

negative quotient:
  Up   -> +0.0
  Down -> -2^-1074
```

境界はstrict / inclusiveを明記した。

```text
abs(product) >= 2^-969 -> normal residual path
abs(product) <  2^-969 -> scaled path

abs(xn) < 2^-969 and abs(yn) <  2^918 -> scale
abs(xn) < 2^-969 and abs(yn) >= 2^918 -> early return
```

#### Evidence path

- `doc/Design/detail/IntervalArithmetic.md` §7–§10
- 同 §15

### F-PR3-002 — Medium — addressed

#### 原因

初版はAVX2 / SSE2とvector FMA residualを同じ候補群として記述し、`Fma.IsSupported`が独立capabilityであることをdispatch契約へ反映していなかった。

#### 対応

詳細設計§19で次を独立rowとして定義した。

- x64 AVX-512F
- x64 AVX2 + FMA
- x64 AVX2 without FMA
- x64 AVX + FMA without AVX2
- x64 SSE2 without FMA
- ARM64 AdvSimd
- scalar fallback

AVX2 without FMAとSSE2では、初期mul/divをscalar fallbackとする。vectorized Dekker / TwoProductは別benchmark gateへ延期した。

ARM64もx86 FMAと同一視せず、使用intrinsicのexactnessとscalar differential testを通過するまでmul/div production dispatchへ入れない。

#### Evidence path

- `doc/Design/detail/IntervalArithmetic.md` §3.2
- 同 §19

### F-PR3-004 — Medium — addressed

#### 原因

初版にはexact-rational testとreference differential testはあったが、基本設計から持ち越されたIEEE 1788.1 conformance testの対象operation、corpus、adaptation、evidenceがなかった。

#### 対応

詳細設計§14でPhase 1 required matrixを定義した。

```text
empty / entire / numsToInterval
inf / sup
isEmpty / isEntire / isSingleton / equal
neg / add / sub / mul / div
```

ITF1788を次のSHAへ固定した。

```text
d8c2a64478ebdc9cbde6ccef33eaad3bed60ed81
```

Phase 1で利用するITL source、signed-zero / Empty / constructor adaptation、Phase 4へ延期するoperation、conformance artifactを明記した。

#### Evidence path

- `doc/Design/detail/IntervalArithmetic.md` §14
- 同 §17

### F-PR3-005 — Medium — addressed

#### 原因

初版はinariとkvを同列のsecondary oracleとしていたが、kvの通常interval divisionはzero-containing denominatorに対する意味論が互換でない。また、C# testが固定reference結果を取得する方式がなかった。

#### 対応

責務を次で固定した。

1. exact rational: primary mathematical oracle
2. IEEE 1788.1 matrix: required interval semantics
3. inari: interval semantic / endpoint differential oracle
4. kv: directed-rounding primitive oracle

kvのzero-containing interval divisionは`expected-difference`として除外する。

さらに次を定義した。

- Rust inari CLI adapter
- C++ kv CLI adapter
- JSON Lines入出力
- 16桁hex endpoint bit pattern
- `reference-lock.json`
- pinned SHA、toolchain、generator command、corpus SHA-256
- commit済みgolden corpus
- reference-integrity regeneration job

#### Evidence path

- `doc/Design/detail/IntervalArithmetic.md` §13

### F-PR3-006 — Medium — addressed

#### 原因

API freeze gateはx64 / ARM64のcanonical result一致を要求していたが、CI sectionにarchitecture matrixと比較方式がなかった。

#### 対応

詳細設計§17で次を必須とした。

```text
Linux x64:   ubuntu-24.04
Linux ARM64: ubuntu-24.04-arm
```

両jobで同一suiteと同一corpusを実行し、caseId順の`canonical-results.jsonl`を生成する。後続jobでbyte-for-byte比較し、SHA-256と全差分を保存する。

各architectureのdiagnostic artifactを独立保存する。

#### Evidence path

- `doc/Design/detail/IntervalArithmetic.md` §17
- 同 §18.2

### F-PR3-007 — Medium — addressed

#### 原因

初版のedge testは一般的なsubnormal / min-normal / overflowを挙げるだけで、production algorithmの分岐を必ず通すfixtureになっていなかった。

#### 対応

詳細設計§15へ次を固定した。

- `2^-969`のprevious / equal / next bit pattern
- `2^918`のprevious / equal / next bit pattern
- `2^-1074`のbit pattern
- large-denominatorの4方向・符号結果
- multiplication scaled pathの`t<s`、`t>s`、`t==s && s2>0`、`t==s && s2<0`、exact
- divisionの`r==xn && r2>0`、`r==xn && r2<0`
- exact resultで補正しないcase
- 正負overflowのUp / Down

scaled multiplicationとdivision tieには固定operand bit patternを記録した。

#### Evidence path

- `doc/Design/detail/IntervalArithmetic.md` §15

### F-PR3-008 — Low — addressed

#### 原因

初版はPhase 1でnativeを使わないことだけを記載し、基本設計が残した将来のmanaged/native比較判断を追跡できなかった。

#### 対応

詳細設計§20で、Phase 1–3初期実装をmanaged-onlyとした上で、native backendを次へ延期した。

- Phase 3完了後のlarge-batch benchmark
- Phase 4の超越関数実装方式決定

scalar operatorごとのP/Invokeは採用しない。採用条件として性能、bitwise semantics、配布、ABI、thread safety、AOT、licenseを定義した。

#### Evidence path

- `doc/Design/detail/IntervalArithmetic.md` §20

### F-PR3-003 — withdrawn_erratum — no implementation action

exhaustive reviewに従い、有効指摘として扱わなかった。最終publication後のexact-head CI確認はrepository ruleどおり別途行う。

## Sibling-case inspection

指摘箇所だけでなく次の隣接条件も同時に確認・設計へ反映した。

- multiplication / divisionの正負両方向overflow
- exact Infinityとfinite overflowの区別
- threshold equality
- signed zero correction
- AVX + FMA without AVX2
- ARM64 fused operationとx86 FMAの分離
- ITF1788とIEEE 1788.1の適用範囲差
- kv interval semanticsとprimitive semanticsの分離
- reference corpusのtoolchain再現性
- architecture metadataとresult corpusの分離

追加のactive findingは認識していない。独立したfix verificationは別工程で行う。

## Validation

### Repository validation

- design fileの既存blob SHAを確認してから更新した。
- PR #3の既存branchを更新し、新しいPRは作成していない。
- design fixは1個の論理commitとしてpushした。
- production source、test、workflowは変更していない。

### Numerical design validation

- multiplication / divisionの分岐をpinned `kv/rdouble-nohwround.hpp`と照合した。
- inari、kv、ITF1788の参照SHAを固定した。
- threshold equality、scaled relation、residual tie、large-denominator returnを一意に定義した。
- fixed operand bit patternを用いる決定的fixtureを設計へ追加した。

本PRには実行可能projectとtest targetが存在しないため、repository build / testは実行していない。これはtest成功を意味しない。

## CI state

このreport作成時点では、reportおよびhandoffのpublication後に最終HEADが変わるため、exact-head CI確認は未完了である。

最終commit後にPR current HEADを取得し、そのSHAとworkflow runの`head_sha`が一致するrunだけを確認する。一致するrunがない場合はCI未実施と報告し、別SHAを代用しない。

## Files

Changed in this follow-up:

- `doc/Design/detail/IntervalArithmetic.md`
- `reports/2026-08-30-pr-3-design-review-follow-up.md`
- `reports/2026-08-30-pr-3-design-review-follow-up-handoff.yaml`

Intentionally untouched:

- `src/**`
- `tests/**`
- `.github/workflows/**`
- `doc/Design/basic/IntervalArithmetic.md`
- prior review reports

## Remaining risks

- 独立reviewerによるfix verificationは未実施。
- Phase 1のproduction algorithm、oracle、adapter、CIはまだ実装されていない。
- API候補はPhase 2までpilot扱いである。
- SIMD kernelとnative backendの採否は後続benchmark gateに従う。

## Next action

PR #3の新しいimmutable HEADを対象に、authoritative exhaustive reviewの7件が閉じたことをfix verificationする。

## Merge boundary

本workerはmergeしない。mergeはrepository ownerが行う。