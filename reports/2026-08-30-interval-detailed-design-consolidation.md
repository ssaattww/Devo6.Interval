# 区間演算 詳細設計統合レポート

## Metadata

- Repository: `ssaattww/Devo6.Interval`
- Pull request: `#3`
- Branch: `docs/detailed-interval-arithmetic-design`
- Request: 詳細設計を可能な限り1ファイルへ統合
- Consolidation base HEAD: `8df7802b725738578cd93e27cdf64bd71b2b0ece`
- Consolidated technical HEAD before report: `fd4d12f0da9e4f883454281190726babb11479a2`
- Date: 2026-08-30

## Outcome

`doc/Design/detail/` 配下の詳細設計を `IntervalArithmetic.md` 1ファイルへ統合した。

統合後のdirectoryは次だけである。

```text
doc/Design/detail/
  IntervalArithmetic.md
```

`.gitkeep`も不要となったため削除した。

## Merged source documents

次の内容を `IntervalArithmetic.md` 設計版4へ吸収した。

- 旧 `IntervalArithmetic.md` 設計版2
- `IntervalArithmetic.Revision3.md`
- `IntervalNonArithmetic.Roadmap.md`
- `IntervalSetAndNumeric.md`
- `IntervalMathFunctions.md`
- `IntervalAdvancedFeatures.md`
- `README.md` の読取順序・precedence・review boundary

単純なファイル連結ではなく、重複を除去し、Revision 3の修正を本文へ直接反映した。

## Normative corrections retained

統合時に失わないことを明示した既存review対応:

- Empty内部表現はcanonical NaN 2 lane
- `Interval.Empty.Lower = +Infinity`
- `Interval.Empty.Upper = -Infinity`
- finite overflowをexact rational oracleでBCL nearest変換前に処理
- ITF1788 constructor sourceとして`libieeep1788_class.itl`を使用
- `IsSingleton`はrepository-defined equivalent matrix
- exact-head CI only
- x64 / ARM64 canonical result comparison
- ISA/FMA capabilityの独立判定
- multiplication/divisionのsubnormal threshold、scaled comparison、residual tie
- native backendは後続decision gateへdefer

## Consolidated structure

統合版は大きく次で構成した。

1. 文書情報・設計原則
2. Phase 0～Phase 4E
3. target framework / architecture
4. 基本公開API
5. `[-Lower,Upper]`内部表現
6. directed rounding
7. 四則演算kernel
8. equality / oracle / conformance / fixtures / CI
9. SIMD / native backend
10. Phase 4A set/relation/numeric
11. Phase 4B algebraic / constants
12. Phase 4C monotonic elementary
13. Phase 4D periodic / `Atan2` / general power
14. Phase 4E union / decorated / parsing / interchange / splitting
15. TDD / completion gates / performance / license / references
16. review history / implementation start gates

## Phase 4 details retained

### Phase 4A

- `Contains`
- `Intersect`, `ConvexHull`
- subset/interior/disjoint/precedes/less
- 16-state `IntervalOverlap`
- Width/Midpoint/Radius/Magnitude/Mignitude
- Abs/Sign/pointwise min-max
- integer-valued rounding functions

### Phase 4B

- reciprocal
- square
- sqrt with scaled exact-product correction
- integer power/root
- FMA as one-round mathematical operation
- tight interval constants

### Phase 4C / 4D

- certified scalar endpoint kernel boundary
- MPFR reference corpus
- exp/log/hyperbolic/inverse functions
- fixed-point periodic range reduction
- sin/cos extrema
- tan poles
- atan2 rectangle/branch-cut handling
- positive-base general power

### Phase 4E

- `IntervalUnion2`
- extended division
- reverse multiplication
- cancellative operations
- `DecoratedInterval` / NaI
- exact/outward decimal parsing
- exact hexadecimal round-trip formatting
- versioned binary interchange
- explicit split / bounded bisection

## File operations

### Updated

- `doc/Design/detail/IntervalArithmetic.md`
  - commit: `ec72a8d634d8b59ad1ce977c65a7f04a94c499c4`
  - content blob after update: `cff4324de49ad0070fa0d0cff404951b8956b1d0`

### Removed after merge

- `IntervalArithmetic.Revision3.md`
  - commit: `8e9aa42524c52e644b1b7bc9359dd656363f2a6d`
- `IntervalNonArithmetic.Roadmap.md`
  - commit: `fccaa7029f736cb34dcd358a3905e69d4035bfc4`
- `IntervalSetAndNumeric.md`
  - commit: `ab240c093aa135c4b5ebfa8518b8aa7cb205c7d9`
- `IntervalMathFunctions.md`
  - commit: `f2bc5fa3bffb208ef629d07905ed63b2a41d4ce1`
- `IntervalAdvancedFeatures.md`
  - commit: `46878b24bc5e67a106cb4a731745f5f4c973489a`
- `README.md`
  - commit: `8b1cc2c11fc3df6599fc9c994fe606b5aa4ffb3c`
- `.gitkeep`
  - commit: `fd4d12f0da9e4f883454281190726babb11479a2`

## Validation

### Directory validation

GitHub contents APIで統合後の`doc/Design/detail/`を確認し、次の1ファイルだけであることを確認した。

```text
IntervalArithmetic.md
```

### Diff validation

`8df7802b...` から `fd4d12f0...` のcompare結果:

- ahead by 8 commits
- behind by 0
- 1 design file modified
- 6 superseded design/index files removed
- `.gitkeep` removed
- source/test/workflow changeなし

### Executable validation

未実施。

理由:

- documentation-only change
- repositoryに実行可能project/test targetがまだ存在しない
- pull-request workflowも存在しない

build/test成功は主張しない。

## Review boundary

過去のreview PASSは過去のimmutable reviewed HEADに対するものだった。

今回、文書内容を1ファイルへ再構成しPhase 4の内容も同じ規範文書へ統合したため、統合版は新しいreview対象とする。

実装開始前に、最終current HEADの`doc/Design/detail/IntervalArithmetic.md`を単一のauthoritative detailed-design artifactとして独立reviewする。

## CI policy

最終handoff commit後にPR current HEADを取得し、そのSHAとworkflow runの`head_sha`が完全一致するrunだけを確認する。

一致runがなければCIは`未実施`とし、別SHAのrunを代用しない。

## Merge

mergeは行わない。repository ownerが行う。
