# PR #3 詳細設計 Fix Verification Report

## Metadata

- Repository: `ssaattww/Devo6.Interval`
- Pull request: `#3` (`docs: add interval arithmetic detailed design`)
- Review mode: `fix_verification`
- Source exhaustive review technical HEAD: `da6e2ae04d35b01acfb307953a093c81c15342b8`
- Source exhaustive review report: `reports/2026-08-29-pr-3-design-review-exhaustive.md`
- Source exhaustive review handoff: `reports/2026-08-29-pr-3-design-review-exhaustive-handoff.yaml`
- PR HEAD at fix-verification start: `c1826cc8dab3f070bfb5133ece44969a03727e97`
- Technical design-fix commit: `3b0c103dd5fb523b17b90ff9026d7384c0cf3ad4`
- Fix report commit parent: `4aa86f2f7e357c3b5a0b05dd8aabce4874f22291`
- Fix handoff/current reviewed HEAD: `c1826cc8dab3f070bfb5133ece44969a03727e97`
- Reviewer role: same normal reviewer performing closure verification
- Date: 2026-08-30
- Verdict: **FAIL**

本レポートは、前回exhaustive reviewの有効指摘7件をfinding identity単位で検証し、修正差分が新たに変更したconformance/oracle設計を含めて再確認した結果である。

## Scope and evidence

確認対象:

- `4e9cfc7787ee885b98ea7253f93bc48e876d3086..c1826cc8dab3f070bfb5133ece44969a03727e97`
- `doc/Design/detail/IntervalArithmetic.md`
- `reports/2026-08-30-pr-3-design-review-follow-up.md`
- `reports/2026-08-30-pr-3-design-review-follow-up-handoff.yaml`
- accepted basic design
- pinned `mskashi/kv` rounding source
- pinned `unageek/inari` arithmetic/numeric semantics
- pinned `unageek/ITF1788` tree and selected ITL files
- current PR exact-head workflow state

修正前review reportのみを読んでclosure判定せず、設計本文、参照元、境界fixture、sibling casesを個別に確認した。

## Overall result

前回のactive findings 7件について:

| Finding | Source severity | Verification result |
|---|---:|---|
| `F-PR3-001` | High | **resolved** |
| `F-PR3-002` | Medium | **resolved** |
| `F-PR3-004` | Medium | **partial / still active** |
| `F-PR3-005` | Medium | **resolved** |
| `F-PR3-006` | Medium | **resolved** |
| `F-PR3-007` | Medium | **resolved** |
| `F-PR3-008` | Low | **resolved** |
| `F-PR3-003` | withdrawn | remains withdrawn; no action |

新規 finding:

- `F-PR3-009` Medium — exact-rational primary oracleのfinite overflow処理が設計版2で欠落した。

したがってactive findingは2件である。

## Finding verification

### F-PR3-001 — High — resolved

Source finding: directed-rounding specification is not implementation-complete.

確認結果:

- multiplication threshold `abs(r) >= 2^-969` / scaled path `< 2^-969` を明示。
- scaled multiplicationで `t`, `s`, `s2` を定義。
- Up条件 `t < s || (t == s && s2 > 0)` を固定。
- Down条件 `t > s || (t == s && s2 < 0)` を固定。
- division denominatorを正へ正規化。
- `abs(xn) < 2^-969 && abs(yn) < 2^918` のscaleを固定。
- `abs(yn) >= 2^918` の方向・符号別 zero/min-subnormal early returnを固定。
- divisionの `r == xn` に対するFMA residual tie条件を固定。
- positive/negative finite overflowのUp/Down結果を固定。
- threshold equalityの所属経路を固定。

さらに§15の固定bit-pattern fixtureを独立再計算し、multiplication scaled比較4分岐とdivision residual tie 2分岐が記載どおりの関係になることを確認した。

Disposition: `resolved`。

### F-PR3-002 — Medium — resolved

Source finding: AVX2/SSE2とx86 FMA capabilityの混同。

確認結果:

§3.2 / §19で次を独立capabilityとして扱っている。

- AVX-512F
- AVX2 + FMA
- AVX2 without FMA
- AVX + FMA without AVX2
- SSE2 without FMA
- ARM64 AdvSimd
- scalar fallback

非FMA x86のinitial mul/divはscalar fallbackであり、FMAの暗黙前提は除去された。ARM64 fused pathもx86 FMAと同一視していない。

Disposition: `resolved`。

### F-PR3-004 — Medium — partial / still active

Source finding: IEEE 1788.1 conformance-test導入設計がない。

改善を確認した点:

- pinned ITF1788 SHA `d8c2a64478ebdc9cbde6ccef33eaad3bed60ed81`を定義。
- Phase 1 operation matrixを定義。
- Phase 4 deferred operationsを定義。
- adaptation ruleとartifact要件を定義。
- ITF1788は補助入力源であり標準そのものの代替ではないと明記。

しかし、fixed corpusの具体的な抽出設計に3つの不整合が残る。

#### A. `isSingleton` のsource mappingが存在しない

§14.4は次を宣言している。

```text
itl/libieeep1788_bool.itl の equal/isEmpty/isEntire/isSingleton
```

pinned ITF1788 SHAを検索した結果、`isSingleton` はrepository内に0件であり、`libieeep1788_bool.itl`にも存在しない。

したがって現設計どおりのgeneratorは、Phase 1 matrixで`required`とした`isSingleton` corpusを指定sourceから生成できない。

#### B. numeric `numsToInterval` の選択sourceが実質的に不足している

§14.4は`itl/ieee1788-constructors.itl`をconstructor sourceとしている。同fileのbare numeric constructor caseは主に次の1件である。

```text
b-numsToInterval -infinity infinity = [entire]
```

一方、同じpinned ITF1788の`itl/libieeep1788_class.itl`には、有限、非有界、NaN、reversed bounds、±Infinity singleton等のbare `b-numsToInterval` casesが存在するが、現設計の抽出sourceに含まれていない。

Phase 1 matrixでnumeric constructorをrequired conformance itemとするなら、`libieeep1788_class.itl`を取り込むか、同等のrepository-defined standards matrixを明示する必要がある。

#### C. Emptyの`inf/sup` semanticsが現在APIと矛盾する

現在のDevo6.Interval設計:

```text
Interval.Empty.Lower -> NaN
Interval.Empty.Upper -> NaN
```

一方、pinned ITF1788 `libieeep1788_num.itl`は次を要求する。

```text
inf [empty] = +infinity
sup [empty] = -infinity
```

主要参照実装inariも同じく:

```text
Interval::EMPTY.inf() -> +Infinity
Interval::EMPTY.sup() -> -Infinity
```

ところが§14.2は`inf -> Lower`, `sup -> Upper`をPhase 1 `required`とし、§14.5 adaptation ruleにはEmptyでのこの差異が記録されていない。さらに§18.2は「Phase 1 conformance matrixが全件pass」をgateにしている。

現状では、API semanticsを変更しない限り、このrequired matrixは全件passできない。

これはAPIを必ず変更せよという指摘ではない。次のどちらかを設計上決定する必要がある。

1. pilot API段階で`Empty.Lower/Upper`をstandard/inariの`+Infinity/-Infinity`へ合わせる。
2. `NaN/NaN` public endpoint semanticsを意図的deviationとして維持し、conformance manifestへ`approved-deviation`等で明示し、`all pass` gateをdeviation-awareな判定へ変更する。

#### Required action for F-PR3-004 closure

- `isSingleton`について、実在するsourceまたはrepository-defined equivalent matrixを指定する。
- numeric `numsToInterval` required casesのsourceを`libieeep1788_class.itl`等へ補正する、または同等matrixを定義する。
- Empty `inf/sup`差異をAPI変更または明示的standards deviationとして処理する。
- conformance summary/gateを実際のpolicyと一致させる。
- follow-up report/handoffの「F-PR3-004 addressed」記録も更新する。

Disposition: `partial`, severity stays `Medium`。

### F-PR3-005 — Medium — resolved

Source finding: reference-oracle責務とexecution mechanismが再現可能でない。

確認結果:

- exact rational = primary mathematical oracle
- standards matrix = interval semantics requirement
- inari = interval semantic / endpoint differential oracle
- kv = compatible directed-rounding primitive oracle
- kv zero-containing interval divisionは`expected-difference`として除外
- pinned adapter paths / JSONL bit representation / `reference-lock.json`
- reference SHA、toolchain、target、generator、corpus hashをlock
- ordinary CIはcommitted golden corpusを読む
- integrity jobがpinned toolsからbyte-for-byte regeneration

元findingで求めた役割分担と再現可能性は満たしている。

Disposition: `resolved`。

### F-PR3-006 — Medium — resolved

Source finding: x64/ARM64 correctness gateをCI設計が実装していない。

確認結果:

- Linux x64 `ubuntu-24.04`
- Linux ARM64 `ubuntu-24.04-arm`
- same deterministic/oracle/conformance/golden suite
- per-architecture diagnostic artifact
- sorted `canonical-results.jsonl`
- cross-architecture byte-for-byte comparison + SHA-256 + full differences

Disposition: `resolved`。

### F-PR3-007 — Medium — resolved

Source finding: deterministic testsがalgorithm branch boundaries/residual tiesを固定していない。

確認結果:

- `2^-969` previous/equal/next
- `2^918` previous/equal/next
- `2^-1074`
- large-denominator 4 direction/sign outcomes
- multiplication scaled `t<s`, `t>s`, equality positive/negative residual, exact
- division `r==xn && r2>0/<0`, plus `<`, `>`, exact
- positive/negative overflow Up/Down
- exact no-correction

fixture bit patternsを独立再計算して主要branch witnessが成立することも確認した。

Disposition: `resolved`。

### F-PR3-008 — Low — resolved

Source finding: native-backend decisionがbasic designからtraceできない。

確認結果:

§20でPhase1/2/initial Phase3をmanaged-onlyとし、native backendを:

- Phase3後のlarge-batch benchmark
- Phase4 transcendental implementation decision

へ明示的にdeferしている。adoption criteriaとreport requirementも定義されている。

Disposition: `resolved`。

### F-PR3-003 — withdrawn erratum

前回exhaustive reviewでwithdrawn済み。今回もimplementation defectとして扱わない。

## New finding

### F-PR3-009 — Medium — exact-rational primary oracleのfinite overflow処理が修正版で欠落した

- Origin: `introduced_by_fix`
- Location: `doc/Design/detail/IntervalArithmetic.md` §13.2, §15.5, §18.2

#### Problem

設計版2のexact rational oracleは次の流れを記述している。

```text
BCL nearest resultをexact rationalへ変換し、真値との大小でUp/Down補正を決める
```

しかしfinite operandの正確な実数結果がbinary64有限範囲を超える場合、BCL nearest resultは`+Infinity`または`-Infinity`となる。Infinityは「有限binary64をexact rationalへ分解する」という§13.2の処理へ入れられない。

具体例:

```text
double.MaxValue * 2.0
```

- exact real resultはfiniteだがbinary64 range外。
- nearest binary64 resultは`+Infinity`。
- directed expectedは Up=`+Infinity`, Down=`double.MaxValue`。

同じ問題は`double.MaxValue + double.MaxValue`、巨大division等でも生じる。

修正前設計§12.3には次の明示的stepが存在していた。

```text
overflow / infinity は exact rational と最大有限値を比較して決める
```

設計版2の§13.2ではこのbranchが落ちている一方、§15.5はfinite overflowを必須fixtureにし、§18.2はexact rational oracleとの差がないことをacceptance gateとしている。

したがってprimary oracleのexpected conversion procedureがoverflow fixtureを一意に処理できない。

#### Required action

exact oracleにproduction algorithmから独立したoverflow conversionを明記すること。少なくともfinite operandsについて:

```text
exact > +double.MaxValue:
  Up   -> +Infinity
  Down -> +double.MaxValue

exact < -double.MaxValue:
  Up   -> -double.MaxValue
  Down -> -Infinity
```

また、operand自体がInfinityの場合のset/special-value semanticsと、finite exact resultのoverflowを区別すること。

addition / subtraction / multiplication / divisionのfinite overflow fixtureが、このoracle pathを通ることを固定すること。

Disposition: `active`, severity `Medium`。

## Sibling-case and regression inspection

次を追加確認した。

- FMA scaled multiplicationのtie witness bit patterns
- division rounded-high-product tie witness bit patterns
- threshold strict/equality paths
- positive/negative finite overflow
- exact Infinityとfinite overflowの区別
- zero-containing interval divisionとkv primitive責務の分離
- x64/ARM64 result corpus determinism
- ITF1788 selected-file existence and operation presence
- inari Empty `inf/sup` behavior
- follow-up report/handoff claims against actual design

上記以外の新しいactive numerical/API findingは認識していない。

## Validation assessment

### Build / tests

Repositoryには現在も実行可能project/test targetがなく、このPRはdocumentation-onlyである。build/test成功は主張しない。

### Exact-head CI

Fix-verification対象HEAD:

```text
c1826cc8dab3f070bfb5133ece44969a03727e97
```

このSHAを`head_sha`に持つpull_request workflow runは0件。

```text
CI状態: 未実施
other-SHA substitution: false
```

### Scope

Fix implementation after prior review changed:

- detailed design
- follow-up report
- follow-up handoff

production source/test/workflowは変更されていない。

## Coverage dispositions

| Criterion | Disposition | Evidence |
|---|---|---|
| prior finding identity closure | `checked_finding` | F-PR3-004 partial; others resolved |
| numerical directed rounding | `checked_no_finding` | F-PR3-001 resolved and fixture witnesses rechecked |
| SIMD capability dispatch | `checked_no_finding` | F-PR3-002 resolved |
| IEEE conformance design | `checked_finding` | F-PR3-004 remains partial |
| reference responsibility/reproducibility | `checked_no_finding` | F-PR3-005 resolved |
| primary exact oracle completeness | `checked_finding` | new F-PR3-009 |
| x64/ARM64 CI design | `checked_no_finding` | F-PR3-006 resolved |
| deterministic branch fixtures | `checked_no_finding` | F-PR3-007 resolved |
| native backend traceability | `checked_no_finding` | F-PR3-008 resolved |
| public API/value semantics | `checked_finding` | Empty getter semantics conflict is part of F-PR3-004 conformance closure |
| scope discipline | `checked_no_finding` | no production change |
| failure diagnostics | `checked_no_finding` | per-arch artifacts defined |
| exact-head CI | `checked_no_finding` | zero matching runs, correctly reported as CI未実施 |
| report/handoff accuracy | `checked_finding` | current follow-up claims F-PR3-004 fully addressed; must be corrected |
| security/secrets | `not_applicable` | documentation-only |

## Verdict

**FAIL**

Blocking active findings before Phase 1 start:

1. `F-PR3-004` Medium — partial: conformance corpus/source/adaptation does not match the pinned data and current Empty endpoint semantics.
2. `F-PR3-009` Medium — new: exact-rational primary oracle lacks finite-overflow expected-result conversion.

`F-PR3-001`, `002`, `005`, `006`, `007`, `008` are verified resolved. `F-PR3-003` remains withdrawn.

## Required next action

Address `F-PR3-004` and `F-PR3-009` only, plus update the follow-up report/handoff closure claims. Then run another same-reviewer fix verification on the new immutable PR HEAD.

## Merge boundary

Do not merge. Merge remains repository-owner responsibility.
