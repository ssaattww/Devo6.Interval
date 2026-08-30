# PR #3 追加設計レビュー 指摘対応レポート

## Metadata

- Repository: `ssaattww/Devo6.Interval`
- Pull request: `#3`
- Branch: `docs/detailed-interval-arithmetic-design`
- Source review: `reports/2026-08-30-pr-3-additional-design-review.md`
- Reviewed source technical HEAD: `8e6e7499204fccf0643da82f274d1485dc0e3272`
- Review-artifact/current-head before fix: `a37fc5bcd43f1aab1221731a7c42dab5e3e93865`
- Technical fix commit: `2fb636f9f5322f4918dfc10680c14ba59147a25e`
- Design file: `doc/Design/detail/IntervalArithmetic.md`
- Design version after fix: `5`
- Date: 2026-08-30
- Work type: documentation design fix

## Purpose

統合版Phase 4A～4Eに対する追加設計レビューで起票された8件を、finding identityを維持して修正した。

前回review済みの四則演算coreは再設計せず、追加設計と統合欠落に限定して変更した。

## Active findings addressed

### F-PR3-010 High — IntervalUnion2 zero-touch component loss

対応:

- `IntervalUnion2`をexact open-set型ではなく、各connected componentの**tight closed enclosure**を最大2個保持する型として明文化した。
- Count=2のorderingを`First.Upper <= Second.Lower`へ変更した。
- `First.Upper == Second.Lower`を理由にmergeする規則を廃止した。
- 0で接する2 enclosureはdistinct componentとして保持する。
- strict overlapはinternal construction invariant violationとし、黙ってmergeしない。
- open/closed topologyを保持しないため、初版では`Contains(double)`を`IntervalUnion2`へ提供しない。

固定case:

```text
DivideToUnion([1,2], Entire)
ReciprocalToUnion(Entire)
ReverseMultiply([1,2], Entire)
```

はいずれもCount=2で:

```text
[-Infinity,-0.0]
[+0.0,+Infinity]
```

を保持する。

### F-PR3-011 High — Atan2 negative-x branch-cut contact

対応:

strictly negative x intervalについてyを6classへ分離した。

- strictly negative
- nonpositive touching zero
- Zero
- nonnegative touching zero
- strictly positive
- crossing zero

主要規則:

```text
x<0, y=[negative,0] -> [-π,+π]
x<0, y=Zero         -> Pi
x<0, y=[0,positive] -> QII lower .. +π
x<0, y crosses zero -> [-π,+π]
```

signed zeroは同じ実数0として扱い、`-0.0`をbranch cut下側として解釈しない。

### F-PR3-012 Medium — general Pow zero-boundary

対応:

通常`PowDown/Up(x,y)` kernelのpreconditionを`x>0`とした。

zero-base境界はpoint valueではなくrectangle closure/value candidateとして:

```text
x -> 0+, y < 0 : +Infinity
x > 0,  y = 0 : 1
x = 0,  y > 0 : 0
```

をinterval extension層で注入する。

`a==0 && b>0`についてnegative / zero / positive exponentの全classを明示分岐した。

固定case:

```text
Pow([0,0.5],[0,1])  -> [0,1]
Pow([0,0.5],[-1,0]) -> [1,+Infinity]
Pow([0,2],[0,1])    -> [0,2]
```

### F-PR3-013 Medium — one-sided-zero extended division

対応:

統合時に脱落した`Y=[0,d]`および`Y=[c,0]`のZ/P/N/M表をsole normative documentへ復元した。

各caseで:

```text
DivideToUnion(X,Y).ConvexHull == X/Y
```

をrequired propertyとした。

### F-PR3-014 Medium — cancellative Empty semantics

対応:

`CancelSubtract`のEmpty / bounded-common / unbounded 3x3 matrixを固定した。

```text
                term
             Empty   Common   Unbounded
total Empty  Empty   Empty    Entire
      Common Entire  width    Entire
      Unbound Entire Entire   Entire
```

最低限:

```text
CancelSubtract(Empty,Empty)   -> Empty
CancelSubtract(Empty,bounded) -> Empty
```

を保証する。`CancelAdd`も同じmatrixを継承する。

### F-PR3-015 Medium — value equality API

対応:

`IntervalUnion2`へ次を追加した。

- `IEquatable<IntervalUnion2>`
- typed/object `Equals`
- `GetHashCode`
- `==` / `!=`
- indexer invalid access = `ArgumentOutOfRangeException`

`DecoratedInterval`へも同じC# value equality surfaceを追加した。

- NaI value equalityはreflexive: `NaI == NaI`
- NaIはfixed Hash
- non-NaIはinterval + decoration
- IEEE semantic equalityは`SemanticallyEquals`として分離

### F-PR3-016 Medium — result-state decoration cap

対応:

operation固有の`opDec`だけでなくresult interval自身のcapを導入した。

```text
maxForResult =
  Trv if Empty
  Dac if unbounded nonempty
  Com if bounded nonempty

resultDec = min(inputDec, opDec, maxForResult)
```

canonical constructorに集約し、IllはNaIに限定する。

fixture:

```text
Com [MaxValue,MaxValue] + Com [MaxValue,MaxValue]
 -> unbounded bare result
 -> decoration <= Dac
```

### F-PR3-017 Low — relation edge contracts

対応:

strict endpoint-wise-lessにEmpty truth tableを復元した。

```text
Empty vs Empty    -> true
Empty vs nonempty -> false
nonempty vs Empty -> false
```

`IntervalOverlap` inverseへ:

```text
BothEmpty <-> BothEmpty
```

を復元し、16状態全件のinverse fixtureを要求した。

## Consolidation record correction

過去のconsolidation report / handoffには、分割文書の規範を全て保持した旨の記録があった。しかし追加reviewにより、少なくとも次が統合時に脱落していたことが判明した。

- one-sided-zero extended division
- union value-equality surface
- strict-less Empty table
- `BothEmpty` inverse

過去reportは履歴として改変しない。本レポートと設計版5を訂正記録とし、過去の「全規範を保持した」という記述は現在のauthorityとして使用しない。

## Files changed

Technical fix:

- `doc/Design/detail/IntervalArithmetic.md`
  - version 4 -> version 5
  - sole normative detail fileのまま維持

This report:

- `reports/2026-08-30-pr-3-additional-design-review-follow-up.md`

Handoff is persisted separately after this report.

## Validation performed

- latest PR review chainを確認
- detailed review reportの8 active findingsをfinding identity単位で確認
- pinned `inari` `basic.rs`のtwo-output division / cancellative matrixを再確認
- sole normative designをversion 5へ更新
- 四則演算coreの主要規範を維持
- Phase 4 regression fixtureを追加

## Executable validation

未実施。

理由:

- repositoryに実行可能projectがまだない
- test targetがない
- pull-request workflowがない
- 今回はdocumentation-only change

build/test成功は主張しない。

## Review state

実装側では`F-PR3-010`～`F-PR3-017`を**addressed / fix verification pending**として扱う。

technical fix HEAD:

```text
2fb636f9f5322f4918dfc10680c14ba59147a25e
```

本report/handoff publication後のcurrent HEADを、新しいfix verification対象として確定する必要がある。

## Next action

1. handoff packetを保存する。
2. final current HEADを取得する。
3. current HEADと`head_sha`が一致するworkflow runだけを確認する。
4. PR本文と簡易コメントを更新する。
5. 同じnormal reviewerが`F-PR3-010`～`017`をfinding identity単位でfix verificationする。
6. mergeしない。
