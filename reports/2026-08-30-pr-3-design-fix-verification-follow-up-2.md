# PR #3 詳細設計 Fix Verification 指摘対応レポート（第2回）

## Metadata

- Repository: `ssaattww/Devo6.Interval`
- Pull request: `#3` (`docs: add interval arithmetic detailed design`)
- Mode: `review follow-up`
- Base ref: `main`
- PR branch: `docs/detailed-interval-arithmetic-design`
- Source fix-verification reviewed HEAD: `c1826cc8dab3f070bfb5133ece44969a03727e97`
- Source fix-verification report: `reports/2026-08-30-pr-3-design-fix-verification.md`
- PR HEAD at follow-up start: `0c0bd720e7e32332fc1de9a136c954e50695ec1b`
- Design correction commit: `8c90ae3cc7bc11d26d85da81b57612197938b7e9`
- Detailed-design index commit: `430d2e852579451876e848d33f0d76dda1738f48`
- Date: 2026-08-30

## Purpose

PR #3のfix verificationで残った次の2件を、finding identity単位で修正した。

- `F-PR3-004` Medium: conformance corpus/source/adaptationの不整合
- `F-PR3-009` Medium: exact-rational primary oracleのfinite overflow処理欠落

前回までにresolvedと判定されたfindingは再設計せず、今回修正と直接関係する契約だけを変更した。

## Authoritative requirements

- 詳細設計はIEEE 1788.1-oriented semanticsと主要参照実装`inari`へ寄せる。
- Phase 1はmanaged scalar四則演算のpilotであり、後続SIMDのreferenceとなる。
- exact rational oracleはproduction algorithmから独立して期待値を生成する。
- current HEADとworkflow runの`head_sha`が一致するrunだけをCI evidenceに使用する。
- mergeは行わない。

## Changed files

### `doc/Design/detail/IntervalArithmetic.Revision3.md`

設計版2に対する規範的revisionを追加した。次の事項について設計版2を置き換える。

- Emptyの公開端点
- exact rationalのfinite overflow変換
- ITF1788 source mapping
- repository-defined `IsSingleton` matrix
- conformance adaptation、manifest、summary、gate
- overflow fixtureのoracle経路
- finding closure

### `doc/Design/detail/README.md`

詳細設計の読取順序と文書優先順位を明記した。

```text
IntervalArithmetic.md
  -> IntervalArithmetic.Revision3.md
```

Revision 3が置き換える対象を列挙し、設計版2単独で実装を開始しないようにした。

## Finding dispositions

### F-PR3-004 — Medium — addressed

#### 1. `IsSingleton` source mapping

固定ITF1788 SHAの`libieeep1788_bool.itl`には`isSingleton` caseが存在しないため、同fileから抽出する設計を撤回した。

`IsSingleton`はrepository-defined IEEE 1788.1-equivalent matrixとして、次を固定した。

- Empty
- Entire
- finite singleton
- negative singleton
- signed-zero singletonと入力zero variants
- bounded non-singleton
- lower-unbounded
- upper-unbounded

corpus上のsourceを`repository-defined-ieee1788.1-equivalent`として明記し、ITF1788由来と表示しない。

#### 2. numeric `numsToInterval` source

`itl/libieeep1788_class.itl`の`minimal_nums_to_interval_test`をrequired sourceへ追加した。

対象には次が含まれる。

- finite bounded interval
- lower-unbounded interval
- upper-unbounded interval
- Entire
- NaN endpoints
- reversed bounds
- `[-Infinity,-Infinity]`
- `[+Infinity,+Infinity]`

invalid ITF resultの`[empty] signal UndefinedOperation`は、C# constructorでは`ArgumentException`、`TryCreate`では`false`と`Empty` out値へ適応する。

`ieee1788-constructors.itl`はsupplemental sourceとして維持し、重複caseを無言で上書きしない規則を追加した。

#### 3. Empty `inf/sup`

API freeze前であるためapproved deviationは設けず、公開APIを標準・inariへ合わせた。

```text
Interval.Empty.Lower -> +Infinity
Interval.Empty.Upper -> -Infinity
```

内部Emptyは引き続き`[canonical-qNaN, canonical-qNaN]`であり、getterがEmptyをspecial-caseする。

conformance adapterはpublic getterを検証し、内部NaN payloadを観測・比較しない。

#### 4. conformance gate

次を別集計する設計へ変更した。

- required external cases
- required repository-defined cases
- passed / failed
- approved deviationsと一致件数
- deferred / excluded
- source extraction errors

存在しないoperationを宣言sourceから0件抽出した場合は成功扱いせず、source extraction errorとする。

本revision時点ではEmpty `inf/sup`にapproved deviationはない。

### F-PR3-009 — Medium — addressed

exact resultをBCL nearest resultへ変換する前に、`double.MaxValue`のexact rational表現と比較する分岐を追加した。

```text
R > +MaxFinite:
  Up   -> +Infinity
  Down -> +double.MaxValue

R < -MaxFinite:
  Up   -> -double.MaxValue
  Down -> -Infinity
```

`-MaxFinite <= R <= MaxFinite`の場合だけBCL nearest resultをfinite rationalへ変換し、exact valueとの大小により`BitIncrement` / `BitDecrement`を判断する。

Infinity operandはfinite overflowと分離し、exact rationalへ変換しない。

次の全primitiveについて正負のfinite overflow fixtureを固定した。

- addition
- subtraction
- multiplication
- division

各fixtureは`oraclePath: finite-exact-overflow`を通ること自体をassertする。Infinity operandのfixtureは`interval-special-value-semantics`として別管理する。

## Prior finding status

| Finding | Source severity | Current implementation-side status |
|---|---:|---|
| `F-PR3-001` | High | previous fix-verificationでresolved。変更なし |
| `F-PR3-002` | Medium | previous fix-verificationでresolved。変更なし |
| `F-PR3-004` | Medium | 今回addressed |
| `F-PR3-005` | Medium | previous fix-verificationでresolved。変更なし |
| `F-PR3-006` | Medium | previous fix-verificationでresolved。変更なし |
| `F-PR3-007` | Medium | previous fix-verificationでresolved。Revision 3のoverflow pathを追加 |
| `F-PR3-008` | Low | previous fix-verificationでresolved。変更なし |
| `F-PR3-009` | Medium | 今回addressed |
| `F-PR3-003` | withdrawn | actionなし |

この表は実装側の対応記録であり、独立review verdictを代替しない。

## Evidence inspected

- `reports/2026-08-30-pr-3-design-fix-verification.md`
- `doc/Design/detail/IntervalArithmetic.md`
- `doc/Design/basic/IntervalArithmetic.md`
- pinned `unageek/ITF1788` commit `d8c2a64478ebdc9cbde6ccef33eaad3bed60ed81`
  - `itl/libieeep1788_bool.itl`
  - `itl/libieeep1788_class.itl`
  - `itl/libieeep1788_num.itl`
- pinned `unageek/inari` commit `18b83a571d7681c76067bc38d90a74e8be29f545`
- current PR workflow directory

確認事項:

- pinned boolean ITLに`isSingleton`がない。
- pinned class ITLにbare numeric constructorのfinite/invalid matrixがある。
- pinned numeric ITLが`inf(Empty)=+Infinity`、`sup(Empty)=-Infinity`を要求する。
- current branchの`.github/workflows`は`.gitkeep`のみである。

## Validation

本作業はdocumentation-onlyである。

- production source: 変更なし
- tests: 変更なし
- workflows: 変更なし
- executable project: 存在しない

そのためbuild/test成功は主張しない。

文書検証として次を行った。

- active findingのrequired actionとRevision 3各節を対応付けた。
- ITF1788 source fileとoperationの実在性を確認した。
- Emptyの内部表現と公開端点の責務を分離した。
- finite overflowとInfinity operandのoracle経路を分離した。
- add/sub/mul/divすべてにfinite overflow fixtureを定義した。
- 詳細設計の文書優先順位をREADMEへ明記した。

## Intentionally untouched

- `doc/Design/detail/IntervalArithmetic.md`: 設計版2を履歴として保持し、Revision 3で対象節を規範的に置換した。
- 既存review report: 歴史的review evidenceであるため書き換えていない。
- 既存follow-up report/handoff: 当時の対応記録を改変せず、本レポートでpartial判定と追加対応を訂正した。
- `src/**`, `tests/**`, `.github/workflows/**`: documentation-only scopeのため未変更。
- `tasks/**`: accepted task entryがなく、PR #3 review follow-upが直接のscopeであるため未変更。

## Remaining risk

- 設計版2とRevision 3を併読する必要がある。読取漏れを防ぐため`doc/Design/detail/README.md`で優先順位を固定した。
- finding closureは同一reviewerによるfix verificationを要する。
- 実際のoracle、conformance generator、CI matrixはPhase 1のTDD実装で検証される。

## Next action

PR #3の新しいimmutable HEADを対象に、`F-PR3-004`および`F-PR3-009`のfix verificationを行う。

## Merge boundary

mergeは行っていない。mergeはrepository ownerが行う。
