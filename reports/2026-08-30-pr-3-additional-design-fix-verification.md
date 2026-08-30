# PR #3 追加設計 Fix Verification Report

## Metadata

- Repository: `ssaattww/Devo6.Interval`
- Pull request: `#3` (`docs: add interval arithmetic detailed design`)
- Review mode: `fix_verification`
- Source review report: `reports/2026-08-30-pr-3-additional-design-review.md`
- Source reviewed technical HEAD: `8e6e7499204fccf0643da82f274d1485dc0e3272`
- Source review-artifact HEAD: `a37fc5bcd43f1aab1221731a7c42dab5e3e93865`
- Technical fix commit: `2fb636f9f5322f4918dfc10680c14ba59147a25e`
- PR HEAD at fix-verification start: `b88aed05640dcf1fb00328aff7ba621ce9e0ccda`
- Technical diff: `a37fc5bcd43f1aab1221731a7c42dab5e3e93865..2fb636f9f5322f4918dfc10680c14ba59147a25e`
- Administrative diff after technical fix: `2fb636f9f5322f4918dfc10680c14ba59147a25e..b88aed05640dcf1fb00328aff7ba621ce9e0ccda`
- Primary artifact: `doc/Design/detail/IntervalArithmetic.md` design version 5
- Reviewer role: same normal reviewer that issued the source additional-design review
- Date: 2026-08-30
- Verdict: **PASS**

## Review boundary

This review verifies only the eight active findings from the additional-design review and directly affected regressions.

The following finding identities and source severities are preserved:

- `F-PR3-010` High
- `F-PR3-011` High
- `F-PR3-012` Medium
- `F-PR3-013` Medium
- `F-PR3-014` Medium
- `F-PR3-015` Medium
- `F-PR3-016` Medium
- `F-PR3-017` Low

Previously resolved four-arithmetic findings were not re-reviewed. Because the version-5 technical fix rewrote the sole detailed-design file substantially, a small set of already-reviewed high-risk contracts was checked only for preservation:

- multiplication and division threshold / residual-tie rules
- exact-rational finite-overflow conversion
- corrected ITF1788 constructor source / repository-defined `IsSingleton`
- Linux x64 / ARM64 matrix
- exact-head-only CI rule

No prior finding was reopened.

## Repository delta assessment

`a37fc5b... -> 2fb636f...` is exactly one technical commit and changes only:

- `doc/Design/detail/IntervalArithmetic.md`

`2fb636f... -> b88aed0...` contains only:

- `reports/2026-08-30-pr-3-additional-design-review-follow-up.md`
- `reports/2026-08-30-pr-3-additional-design-review-follow-up-handoff.yaml`

Therefore the implementation-side report/handoff do not alter the technical design being verified.

## Finding verification summary

| Finding | Severity | Verification result |
|---|---:|---|
| `F-PR3-010` | High | **resolved** |
| `F-PR3-011` | High | **resolved** |
| `F-PR3-012` | Medium | **resolved** |
| `F-PR3-013` | Medium | **resolved** |
| `F-PR3-014` | Medium | **resolved** |
| `F-PR3-015` | Medium | **resolved** |
| `F-PR3-016` | Medium | **resolved** |
| `F-PR3-017` | Low | **resolved** |

New active findings: **0**.

## Per-finding verification

### F-PR3-010 — High — resolved

Source problem:

`IntervalUnion2` previously required a strict gap between Count=2 components and merged touching closed enclosures. That destroyed two-output information for results whose exact connected components are separated only by an excluded zero but whose tight closed enclosures touch at zero.

Verified correction:

- `IntervalUnion2` is explicitly defined as a representation of the **tight closed enclosure of each mathematical connected component**, not as a fully exact open/closed set representation.
- Count=2 permits:

```text
First.Upper <= Second.Lower
```

- equality at the enclosure boundary is not a merge condition.
- strict overlap remains an internal construction error.
- `Contains(double)` is deliberately not exposed because closure-enclosure storage cannot express excluded boundaries exactly.
- value equality compares the canonical component-enclosure sequence.

Required fixtures are now normative:

```text
DivideToUnion([1,2], Entire)
ReciprocalToUnion(Entire)
ReverseMultiply([1,2], Entire)
```

Each must retain Count=2 with:

```text
[-Infinity,-0.0]
[+0.0,+Infinity]
```

This also agrees with the pinned inari two-output behavior, which permits ordered component enclosures to meet at zero.

Disposition: `resolved`.

### F-PR3-011 — High — resolved

Source problem:

The former `Atan2` design handled negative-x branch-cut crossing but omitted lower-side contact such as `x=[-2,-1], y=[-1,0]`, which requires the bare closed hull `[-pi,+pi]`.

Verified correction:

For strictly negative x, y is explicitly split into six classes:

1. strictly negative
2. nonpositive touching zero
3. Zero
4. nonnegative touching zero
5. strictly positive
6. crossing zero

The design now fixes:

```text
x<0, y=[negative,0] -> [-pi,+pi]
x<0, y=Zero         -> Pi
x<0, y=[0,positive] -> QII lower .. +pi
x<0, y crosses zero -> [-pi,+pi]
```

The closed hull uses the tight Pi enclosure:

```text
lower = -IntervalConstants.Pi.Upper
upper = +IntervalConstants.Pi.Upper
```

Canonical signed zero is treated as the single real value zero, not as two branch-cut sides.

The QII/QIII corner formulas were checked against the monotonicity of `atan2(y,x)` in the corresponding quadrants and are consistent.

Required fixed cases now include the original witness and adjacent branch-cut classes.

Disposition: `resolved`.

### F-PR3-012 — Medium — resolved

Source problem:

The general positive-base power formulas could require forbidden scalar calls such as `PowUp(0,0)` or `PowUp(0,negative)`.

Verified correction:

- normal `PowDown/Up(x,y)` has explicit precondition `x>0`.
- zero-base values/limits are supplied by the interval-extension layer:

```text
x -> 0+, y < 0 : +Infinity
x > 0,  y = 0 : 1
x = 0,  y > 0 : 0
```

- zero-only, zero-touching, strictly-negative exponent, zero-only exponent, positive exponent, and crossing-zero exponent classes are separately defined.
- no branch requires `0^0` or `0^negative` as a scalar point evaluation.

The source witnesses are explicitly fixed:

```text
Pow([0,0.5],[0,1])  -> [0,1]
Pow([0,0.5],[-1,0]) -> [1,+Infinity]
Pow([0,2],[0,1])    -> [0,2]
```

Additional zero-only fixtures are present.

Disposition: `resolved`.

### F-PR3-013 — Medium — resolved

Source problem:

One-sided-zero denominator cases were dropped during detailed-design consolidation.

Verified correction:

`DivideToUnion` now has explicit tables for both:

```text
Y=[0,d], d>0
Y=[c,0], c<0
```

with Z/P/N/M numerator classes.

P/N intentionally include one-sided zero endpoints while excluding the Zero class itself, so `[0,b]` and `[a,0]` are covered correctly.

The required property is fixed for all cases:

```text
X / Y == DivideToUnion(X,Y).ConvexHull
```

Disposition: `resolved`.

### F-PR3-014 — Medium — resolved

Source problem:

The prior cancellative-operation wording returned `Entire` for Empty-total cases that the standard-style matrix treats specially.

Verified correction:

The sole normative document now contains the Empty / bounded-common / unbounded 3x3 matrix:

| total \ term | Empty | bounded/common | unbounded |
|---|---|---|---|
| Empty | Empty | Empty | Entire |
| bounded/common | Entire | width condition | Entire |
| unbounded | Entire | Entire | Entire |

For bounded/common operands, exact width comparison is required rather than comparing rounded `Width` properties.

At least the original witnesses are now fixed:

```text
CancelSubtract(Empty,Empty)   -> Empty
CancelSubtract(Empty,bounded) -> Empty
```

`CancelAdd` inherits the same matrix through `CancelSubtract(total,-term)`.

The matrix matches the pinned inari `cancel_minus` behavior.

Disposition: `resolved`.

### F-PR3-015 — Medium — resolved

Source problem:

The public API candidates described value equality in prose but did not expose a complete C# equality/hash surface.

Verified correction:

`IntervalUnion2` now declares:

- `IEquatable<IntervalUnion2>`
- typed `Equals`
- `Equals(object?)`
- `GetHashCode`
- `==` / `!=`

Its hash uses Count plus only valid canonical components and does not depend on unused fields or internal Empty NaN payloads.

The indexer contract is explicit:

```text
0 <= index < Count
```

otherwise `ArgumentOutOfRangeException`.

`DecoratedInterval` now declares the corresponding C# value-equality surface plus `SemanticallyEquals`.

C# value equality is reflexive:

```text
NaI == NaI -> true
```

while semantic equality retains:

```text
NaI.SemanticallyEquals(any) -> false
```

Non-NaI C# value equality compares interval part and decoration; semantic equality ignores decoration. NaI is canonical and has a fixed hash.

Disposition: `resolved`.

### F-PR3-016 — Medium — resolved

Source problem:

Decoration propagation considered only input decorations and operation-specific maximum decoration, allowing an impossible `Com` decoration on an unbounded or Empty result.

Verified correction:

The result-state cap is now normative:

```text
maxForResult =
  Trv if Empty
  Dac if unbounded nonempty
  Com if bounded nonempty

resultDec = min(inputDec, opDec, maxForResult)
```

A single canonical construction path enforces the cap and reserves `Ill` for NaI.

The overflow witness is fixed:

```text
Com [MaxValue,MaxValue] + Com [MaxValue,MaxValue]
 -> unbounded bare result
 -> decoration <= Dac
```

Empty result <= Trv is also required.

This matches the state restriction represented by pinned inari `DecInterval::set_dec`.

Disposition: `resolved`.

### F-PR3-017 — Low — resolved

Source problem:

Two edge contracts disappeared during consolidation.

Verified correction:

Strict endpoint-wise less explicitly defines:

```text
Empty vs Empty    -> true
Empty vs nonempty -> false
nonempty vs Empty -> false
```

`IntervalOverlap` inverse explicitly restores:

```text
BothEmpty <-> BothEmpty
```

The design requires all 16 overlap states to have inverse-consistency fixtures.

Disposition: `resolved`.

## Directly affected regression inspection

### Union component semantics

The correction deliberately distinguishes mathematical component identity from the closed endpoint enclosure used to store each component. This avoids the original zero-touch information loss while not pretending the type can answer exact membership at an excluded boundary.

No additional public membership contract is inferred.

### Extended/reverse division

The restored one-sided tables and strict zero-crossing formulas are consistent with the ordinary-division hull property. The representative `Entire` denominator/factor cases remain two-component results rather than collapsing to Entire in the union representation.

### Atan2 branch cut

The original lower-side contact witness is now explicitly handled. Signed zero is not used as hidden open-boundary metadata. The fixed six-class matrix covers the source finding and its adjacent strict/touch/cross cases.

### General power zero boundary

Zero-boundary limits/values are injected before the scalar kernel, and all `a==0` exponent classes are explicit. No undefined scalar endpoint call is required by the normative formulas.

### Cancellative / decorated value semantics

Cancellative Empty behavior now has a complete class matrix. Decorated value equality and semantic equality are intentionally separate and their Hash contracts are consistent with that distinction.

### Large version-5 rewrite preservation fingerprints

Although no prior four-arithmetic correctness review was repeated, the following already-reviewed contracts remain present after the version-5 rewrite:

- multiplication small-product threshold and scaled residual comparison
- division small-numerator / large-denominator thresholds and residual tie
- exact-rational finite-overflow conversion before nearest-result rationalization
- `libieeep1788_class.itl` constructor source and repository-defined `IsSingleton`
- Linux x64 + Linux ARM64 correctness matrix
- exact-head-only CI rule with CI未実施 when no matching run exists

No regression finding is raised for these checkpoints.

## Validation assessment

### Build / tests

Not executable for this PR.

The repository still has no executable project or test target, and the reviewed change is documentation-only. No build/test success is claimed.

### Exact-head CI at review start

PR current HEAD at fix-verification start:

```text
b88aed05640dcf1fb00328aff7ba621ce9e0ccda
```

Pull-request workflow runs whose `head_sha` exactly matches this SHA: `0`.

```text
CI status: 未実施
other-SHA substitution: false
```

This is not treated as successful CI.

## Coverage dispositions

| Criterion | Disposition | Evidence |
|---|---|---|
| source finding identity closure | `checked_no_finding` | all F-PR3-010..017 resolved |
| zero-touch union component preservation | `checked_no_finding` | component-closure model + Count2 equality boundary + fixtures |
| Atan2 branch-cut containment | `checked_no_finding` | six-class negative-x matrix + original witness |
| general Pow zero-boundary domain safety | `checked_no_finding` | scalar x>0 precondition + closure candidates + exhaustive a==0 classes |
| one-sided extended division | `checked_no_finding` | both denominator-side tables restored |
| cancellative Empty semantics | `checked_no_finding` | 3x3 matrix + exact width condition |
| value equality / Hash / indexer | `checked_no_finding` | complete union/decorated C# value surface |
| decoration result-state cap | `checked_no_finding` | Empty/Unbounded/Bounded cap + canonical constructor |
| relation/inverse consolidation edges | `checked_no_finding` | strict-less Empty table + BothEmpty inverse |
| direct regression around affected sections | `checked_no_finding` | sibling cases inspected |
| prior four-arithmetic preservation fingerprints | `checked_no_finding` | selected high-risk normative fingerprints retained |
| scope discipline | `checked_no_finding` | one technical design-file commit + administrative report/handoff commits |
| build/test | `not_applicable` | documentation-only; no executable target |
| exact-head CI | `checked_no_finding` | zero matching runs => CI未実施; no other SHA used |
| security/secrets | `not_applicable` | documentation-only review |

## Held / deferred decisions

The following remain intentional later gates and are not findings in this fix verification:

- Phase 4A final public naming
- Midpoint tie-policy final conformance review
- elementary-function production backend selection
- decorated operation final surface beyond the contracts fixed here
- parser syntax/resource-limit numeric values
- final binary-interchange reject/canonicalize details

## Verdict

**PASS**

All eight active additional-design findings are verified resolved. No new active finding was identified in the directly affected regression scope.

This PASS is a design-review result. It does not claim executable CI success.

## Next action

The additional Phase 4 design finding set is closed. Subsequent implementation remains subject to the phase gates in the detailed design, TDD, diagnostic-artifact workflow requirements, and exact-current-HEAD CI verification.

## Merge boundary

No merge is performed by the reviewer. Merge remains the repository owner's action.
