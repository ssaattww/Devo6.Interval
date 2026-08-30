# 四則演算以外の機能 詳細設計レポート

## Metadata

- Repository: `ssaattww/Devo6.Interval`
- Pull request: `#3`
- Base branch: `main`
- Working branch: `docs/detailed-interval-arithmetic-design`
- PR base SHA: `ad5c058f8a4164b0c7d0763c65246914ea5d1c03`
- Starting PR HEAD: `f60f78e7e1637d587d53c4f926fb05cf4ce0f3b8`
- Design content HEAD before this report: `01f379255e8677c2cd78a1ea696bf2a925ff7f89`
- Date: 2026-08-30
- Work mode: documentation implementation

## Purpose

利用者の依頼「四則演算以外のところも設計してください」に基づき、既存の四則演算詳細設計を前提として、Phase 4の機能を実装可能な単位へ詳細化した。

既存設計の次の順序は維持した。

1. managed scalar四則演算pilot
2. basic API freeze
3. SIMD four-arithmetic backend
4. 四則演算以外

今回、項目4をさらにPhase 4A～4Eへ分割した。

## Authoritative inputs

### Repository design

- `doc/Design/basic/IntervalArithmetic.md`
- `doc/Design/detail/IntervalArithmetic.md`
- `doc/Design/detail/IntervalArithmetic.Revision3.md`
- `doc/Design/detail/README.md`

継続した重要契約:

- endpoint typeは`double`
- IEEE 1788.1-oriented set semantics
- internal `[-Lower,Upper]`
- tight outward rounding
- Empty / Entire
- Empty public endpoints `+Infinity / -Infinity`
- managed scalar implementationをreference backendとする
- backend間でcanonical endpointを一致させる

### Repository task state

`tasks/tasks-status.md`は空であり、今回のuser instructionをaccepted scopeとした。

### Pinned references

- `unageek/inari`
  - commit `18b83a571d7681c76067bc38d90a74e8be29f545`
  - set operations、relations、numeric functions、integer functions、elementary functions、two-output division
- `unageek/ITF1788`
  - commit `d8c2a64478ebdc9cbde6ccef33eaad3bed60ed81`
  - conformance fixture source
- `mskashi/kv`
  - commit `c7f8f2324a0e403cca6b39f46088a22843d440db`
  - no-hardware-rounding primitivesとsqrt補正

## Work-context resolution

### Scope

- Phase 4 roadmap
- set operations and relations
- numeric properties
- integer-valued interval functions
- reciprocal/square/sqrt/power/root/FMA
- exponential/logarithmic/hyperbolic/trigonometric functions
- general positive-base power
- two-component interval results
- reverse multiplication and cancellative operations
- decorated interval / NaI
- exact/outward parsing and formatting
- binary interchange
- interval splitting
- TDD, conformance, reference corpus, CI and failure artifact gates

### Non-goals

- production source implementation
- test implementation
- CI workflow implementation
- SIMD implementation
- native wrapper implementation
- Affine Arithmetic / Taylor Model / solver implementation
- merge

### Write boundary

Allowed:

- new detailed-design documents under `doc/Design/detail/`
- detailed work report and handoff under `reports/`
- PR #3 body and summary comment

Intentionally untouched:

- `src/**`
- `tests/**`
- `.github/workflows/**`
- `tasks/**`
- accepted four-arithmetic normative documents

## Added design documents

### `IntervalNonArithmetic.Roadmap.md`

Purpose:

- divide Phase 4 into reviewable implementation phases
- define public type responsibilities
- define Exact / Tight directed algebraic / Tight certified elementary accuracy levels
- prevent silent valid-but-wider fallback
- define backend and release gates

Phase split:

```text
4A set / relations / numeric / integer functions
4B algebraic functions and constants
4C monotonic elementary functions
4D periodic, singular and multivariate functions
4E union, decorated, parsing, interchange and splitting
```

### `IntervalSetAndNumeric.md`

Defined:

- `Contains`
- `Intersect` / `ConvexHull`
- subset / interior / disjoint / precedes
- endpoint-wise weak/strict less
- 16-state `IntervalOverlap`
- `IsBounded`
- `Width`, `Midpoint`, `Radius`, `Magnitude`, `Mignitude`
- `Abs`, `Sign`
- pointwise min/max
- floor, ceiling, truncate and configurable round
- scalar/SIMD implementation formulas
- deterministic and property tests

Notable decisions:

- infinity endpoints are not real members; `Contains(±Infinity)=false`
- relation operators `< <= > >=` are not overloaded
- hull/intersection and pointwise min/max use distinct names
- Empty relation semantics are explicitly defined
- overlap relation includes three Empty states plus thirteen interval-position states
- Width uses upward rounding
- Midpoint is a representative scalar, not an enclosure endpoint

### `IntervalMathFunctions.md`

Defined:

- public `IntervalMath` and `IntervalConstants` candidates
- interval-extension layer versus certified scalar endpoint kernel
- backend independence and shipping rules
- tight interval constants
- reciprocal, square and square root
- integer power including negative exponent and zero crossing
- positive integer root
- fused multiply-add as one mathematical operation
- exponential/logarithmic/hyperbolic/inverse functions
- sine/cosine critical-point detection
- tangent pole detection
- `Atan2` rectangle/quadrant/branch-cut handling
- positive-base general interval power
- MPFR-based reference corpus
- managed/native implementation decision gate

Notable decisions:

- no public elementary function is shipped until a correctly directed endpoint kernel exists
- `Math.Sin`等に根拠なく固定ULPを加える方式は正式backendにしない
- inariが使用するMPFR RNDD/RNDUをprimary reference corpus generatorとする
- squareは`X*X`へ委譲せず依存性による拡大を避ける
- sqrtはkv方式のscaled exact-product comparisonを具体化
- integer powerとgeneral interval exponent powerを別APIとして段階導入する
- trigonometric range reductionは巨大binary64でも象限を誤らないfixed-point方式を要求する
- general powerはnegative baseを含めず、negative baseの整数乗はinteger overloadへ分離する

### `IntervalAdvancedFeatures.md`

Defined:

- allocation-free `IntervalUnion2`
- connected-component canonicalization
- exact two-component extended division
- reciprocal-to-union
- reverse multiplication / two-output division
- cancellative add/subtract
- `Decoration` and `DecoratedInterval`
- C# value equality versus IEEE semantic equality
- exact/outward decimal parsing
- exact hexadecimal round-trip format
- versioned binary interchange independent of private struct layout
- `TrySplitAt` and bounded `TryBisect`
- parser resource limits and security requirements

Notable decisions:

- nonconnected result, decoration and split metadata are not embedded into`Interval`
- `IntervalUnion2`has0～2 canonical components
- ordinary division equals the convex hull of extended division
- reverse multiplication is not conflated with ordinary division when zero is involved
- `default(DecoratedInterval)`uses`Ill`and becomes NaI
- C#`Equals`remains reflexive; IEEE semantic equality is a separate method
- decimal parsing operates on exact decimal rational values before binary64 rounding
- wire format encodes external canonical endpoints rather than raw private fields
- bisection children share the split point to avoid a real-number coverage gap

### `doc/Design/detail/README.md`

Updated to:

- list the complete reading order
- preserve Revision 3 precedence
- explain the boundary between roadmap and feature-specific documents
- state that the previous PASS verdict does not cover the newly added documents
- require a fresh independent review before Phase 4 implementation

## Reference evidence applied

### set and relations

The pinned inari implementation was inspected for:

- intersection / convex hull and Empty behavior
- `Contains` rejection of NaN and infinities
- subset / interior / disjoint / precedes / weak and strict relations
- 16 overlap states

The formulas were adapted to C#-oriented method names without assigning ambiguous interval relations to normal comparison operators.

### numeric and integer functions

The pinned inari implementation was inspected for:

- `inf(Empty)=+Infinity`, `sup(Empty)=-Infinity`
- magnitude, mignitude, midpoint, radius and width
- absolute value and pointwise min/max
- floor, ceiling, truncate, round and sign

### algebraic functions

The pinned inari and kv implementations were inspected for:

- reciprocal sign classes
- square and square-root domain clipping
- `mul_add`
- integer positive/negative powers
- sqrt scaling constants and exact product comparison

The sqrt design uses:

```text
threshold 2^-969
input scale 2^106
result scale 2^53
```

### elementary functions

The pinned inari implementation uses MPFR RNDD/RNDU wrappers for elementary scalar endpoints. This informed the separation between interval-extension semantics and certified endpoint kernels.

The designs for log, atanh, cosh, sin, cos, tan, atan2 and general positive-base pow were compared with the pinned sign/domain classification structure.

## Commit structure before report

The design change was published as five reviewable commits.

| Commit | Purpose |
|---|---|
| `6cc15315bc24c1be30a25eea7286ad2f793acb41` | Phase 4 roadmap |
| `a226fe92064d2702ce2e2cd2259d86b8ac5d007a` | set/relation/numeric design |
| `b36957a83d20714f6401714f4dbffc83b030efe7` | algebraic/elementary design |
| `f9f744ae8ec9ea5970fefa6b496fdcef2923eb09` | advanced-feature design |
| `01f379255e8677c2cd78a1ea696bf2a925ff7f89` | detailed-design index/update |

Comparison against starting HEAD `f60f78e...`:

- ahead by 5 commits
- behind by 0
- added 4 design files
- modified 1 design index
- no source/test/workflow change

## Validation

### Performed

- existing normative design and Revision 3 precedence inspected
- current PR review state inspected
- workflow directory inspected
- task list inspected
- pinned inari set/boolean/numeric/integer/elementary/basic implementations inspected
- pinned kv directed-rounding/sqrt implementation inspected
- internal consistency of phase boundaries, public type responsibilities and reference gates reviewed
- GitHub compare used to confirm the five-commit documentation-only delta

### Not performed

- build
- unit tests
- benchmark
- executable conformance test
- native/reference adapter execution

Reason:

The repository still has no executable project, test target or pull-request workflow. The current change is documentation-only.

No successful build/test/CI result is claimed.

## Review lifecycle

The prior fix-verification PASS reviewed design HEAD:

```text
13cf07cfcdf01205ab4466a99abd380fd1f1d103
```

The PR HEAD after prior review-artifact publication was:

```text
f60f78e7e1637d587d53c4f926fb05cf4ce0f3b8
```

The new Phase 4 design is added after those SHAs. Therefore:

- the prior PASS remains valid only for its reviewed four-arithmetic scope
- it is not transferred to the expanded PR HEAD
- a fresh independent design review is required for the new files
- Phase 4 implementation must not begin from an assumed PASS

## Risks and deferred decisions

### Public API names

The documents define concrete candidates, but final names remain subject to Phase-specific API review before baseline publication.

### Midpoint policy

The design currently selects a documented midpoint rounding policy rather than treating reference implementation behavior as implicit. Implementation must provide deterministic cross-architecture evidence and record any approved difference from a secondary reference.

### Elementary production backend

The public semantics and acceptance gate are defined, but a production endpoint backend is not selected yet.

Candidates:

- certified managed approximation/table implementation
- correctly-rounded managed port
- optional native MPFR/CRlibm class implementation

No elementary method is considered implementation-ready until a backend meets the tight endpoint and deployment gates.

### General power / Atan2

These functions have large classification surfaces. The design defines domain and algorithm structure, but implementation must freeze generated classification matrices and complete independent review before publication.

### Parsing resource limits

The required limits are identified; actual numeric limit values must be selected during parser implementation design and benchmark/security review.

## Failure diagnostics requirements

Future implementation workflows must preserve at least:

- test result
- stdout/stderr
- diagnostic runner log
- architecture/runtime/CPU capability
- operation/function and endpoint bits
- domain clipping and sign/quadrant classification
- directed-kernel backend and correction decision
- periodic reduction/critical-point metadata
- exact/MPFR/inari/kv expected and actual results
- conformance adaptation/deviation
- parser token/exact rational/resource-limit metadata
- union/decoration/split state

Artifact upload must run on failure.

## Next action

1. Persist the handoff packet.
2. Update PR #3 body and post a concise implementation-side summary.
3. Record the final current HEAD and exact-head workflow state.
4. Request independent review of the new Phase 4 design scope.
5. Do not begin Phase 4 source implementation until review closure.

## Merge boundary

No merge is performed by this worker. Merge remains the repository owner's action.
