# PR #3 設計レビュー 再実行 report

## Metadata

- Repository: `ssaattww/Devo6.Interval`
- Pull request: `#3` (`docs: add interval arithmetic detailed design`)
- Review mode: `initial_review / exhaustive rerun`
- Base ref: `main`
- Base SHA: `ad5c058f8a4164b0c7d0763c65246914ea5d1c03`
- Reviewed implementation/design HEAD: `da6e2ae04d35b01acfb307953a093c81c15342b8`
- PR HEAD at rerun start: `9936d9f2d5bb270ba03938fdecb6dde8cc0541f1`
- Relationship: `9936d9f...` is exactly one administrative review-report commit after `da6e2ae...`; the detailed design itself is unchanged.
- Reviewer: ChatGPT GPT-5.6 Sol / normal reviewer
- Reviewer continuity: same normal-review role as the previous review; this rerun supersedes the previous finding set.
- Reviewer independence: the review did not implement the detailed design or any design fix. The only prior write by the reviewer was the previous review report.
- Date: 2026-08-29
- Verdict: **fail**

This report intentionally re-runs the review from zero rather than only extending the previous three findings. It supersedes `reports/2026-08-29-pr-3-design-review.md` as the authoritative review result for the unchanged technical HEAD `da6e2ae...`.

## Review objective

The objective is to determine whether PR #3 is sufficiently complete and internally consistent to start Phase 1 without rediscovering numerical semantics, validation requirements, or backend assumptions during implementation.

Coverage includes:

- user-specified phase order
- accepted basic design
- public API candidate and value semantics
- internal `[-Lower, Upper]` representation
- Empty / Entire / signed zero
- exact outward rounding for addition, subtraction, multiplication, and division
- overflow / underflow / subnormal paths
- multiplication and division sign classification
- reference implementation boundaries
- IEEE 1788.1 conformance verification
- TDD and deterministic/random/reference tests
- x64 / ARM64 acceptance evidence
- SIMD capability dispatch
- CI failure diagnostics
- report/handoff accuracy and review history
- basic-design requirement traceability

## Authoritative inputs inspected

### Repository

- `doc/Design/basic/IntervalArithmetic.md`
- `doc/Design/detail/IntervalArithmetic.md`
- `reports/2026-08-29-interval-arithmetic-detailed-design.md`
- `reports/2026-08-29-interval-arithmetic-detailed-design-handoff.yaml`
- `reports/2026-08-29-pr-3-design-review.md`
- `.github/workflows/` current state

### Pinned references

- `unageek/inari` commit `18b83a571d7681c76067bc38d90a74e8be29f545`
  - repository HEAD as of this review is the same commit
  - `src/arith.rs`
  - `src/basic.rs`
  - `src/interval.rs`
  - `src/_docs/conformance.md`
  - `.gitmodules` (`ITF1788` reference)
- `mskashi/kv` commit `c7f8f2324a0e403cca6b39f46088a22843d440db`
  - repository HEAD as of this review is the same commit
  - `kv/rdouble-nohwround.hpp`
  - `kv/interval.hpp`

### .NET / GitHub Actions

- .NET 10 `System.Runtime.Intrinsics.X86.Fma`
- .NET 10 `Avx512F` packed embedded-rounding APIs
- .NET 10 ISA-specific `IsSupported` model
- current GitHub-hosted x64 and ARM64 Linux runners

## Positive findings

The following areas were rechecked and no defect was found.

1. The phase order `managed scalar pilot -> API freeze -> SIMD -> non-four-arithmetic` is appropriate and matches the user instruction.
2. `[-Lower, Upper]` is consistently maintained as the internal representation.
3. `default(Interval) == Interval.Zero` is coherent with the proposed two-double Phase 1 representation.
4. Empty as a canonical two-lane NaN state is compatible with the accepted design and inari-style representation.
5. Signed-zero normalization is internally consistent with set equality and Hash requirements.
6. Multiplication sign-class formulas were rechecked against endpoint extrema and the pinned inari implementation; no formula error was found.
7. Division formulas for positive/negative denominator, `[0,0]`, one-sided zero contact, and zero crossing match the intended set-based hull semantics and the pinned inari behavior.
8. The AVX-512 concept of four intervals in one `Vector512<double>` is supported by .NET 10 packed `Add/Multiply/...` overloads with `FloatRoundingMode`; the design is correct not to treat the scalar `Vector128<double>` embedded-rounding forms as a two-lane packed replacement.
9. The exact-rational oracle as the primary mathematical oracle is a good separation from production TwoSum/FMA logic.
10. No production/source implementation is mixed into this documentation-only PR.

## Active findings

### F-PR3-001 — High — directed-rounding specification is still not implementation-complete

- Origin: `introduced_by_change`
- Location: `doc/Design/detail/IntervalArithmetic.md` §8.6–8.7
- Status: retained from the previous review and **expanded** by the exhaustive rerun.

#### Problem

The document defines the intended contract correctly, but does not fully define the finite binary64 decision procedure needed to satisfy it.

There are two independent gaps inside the same numerical component.

#### A. Multiplication subnormal/scaled path is not defined to a unique result

For `abs(product) < 2^-969`, the design says to scale operands by `2^537` and determine the result from the “scaled exact relation”. It does not define the actual comparison and tie condition.

The pinned `kv` algorithm contains a concrete relation. For upward rounding it computes a scaled product decomposition `(s, s2)`, compares `t = (r * c) * c`, and increments when:

```text
t < s || (t == s && s2 > 0)
```

The downward case uses the symmetric `>` / negative-residual condition.

The design currently lists the same threshold and scale but omits the relation that makes them correct.

#### B. Division comparison is under-specified even outside the subnormal path

The design says conceptually:

```text
q * y < x  -> NextUp(q)
q * y >= x -> q
```

while also saying the comparison uses FMA-based exact-product decomposition. A correct implementation must compare the high rounded product and its residual, not only the rounded `q*y` value.

After normalizing the denominator positive, the pinned `kv` upward rule is:

```text
(r < x) || (r == x && r2 < 0) -> successor(q)
```

where `(r, r2)` is an error-free product decomposition of `q*y`. Downward rounding uses:

```text
(r > x) || (r == x && r2 > 0) -> predecessor(q)
```

The residual tie is necessary. A concrete binary64 witness is:

```text
x = 0.025395755270045856
y = 9.884361488333468e-07
q = roundNearest(x / y) = 25692.86372217418

roundNearest(q * y) == x
exact(q * y) < exact(x)
```

Therefore `q` is below the exact quotient and `DivideUp(x, y)` must be the next binary64 value:

```text
25692.863722174185
```

A literal implementation of only `q*y >= x -> q` would return one ULP too low.

#### C. Division large-denominator underflow branch is missing

For the small-numerator branch, the design lists:

```text
SmallNumeratorThreshold = 2^-969
LargeDenominatorLimit   = 2^918
DivisionScale           = 2^105
MinimumSubnormal        = 2^-1074
```

but does not specify the branch that uses `LargeDenominatorLimit` and `MinimumSubnormal`.

The pinned `kv` logic, after normalizing the denominator positive, distinguishes:

```text
abs(x) < 2^-969 and abs(y) < 2^918
    -> scale x and y by 2^105 and continue

abs(x) < 2^-969 and abs(y) >= 2^918
    -> return a direction/sign-dependent 0 or minimum subnormal directly
```

For example, the upward path returns `+0` for a negative quotient and `+2^-1074` for a positive quotient; the downward path returns `-2^-1074` for a negative quotient and `+0` for a positive quotient.

#### Impact

Phase 1 is explicitly the scalar reference backend and Phase 3 requires bitwise-equivalent endpoints. Leaving these branches to implementation-time interpretation can make the reference itself wrong by one ULP or by zero/min-subnormal at underflow. That invalidates the later SIMD differential gate.

It also conflicts with the Phase 0 completion condition that Phase 1 can begin without additional basic numerical-design decisions.

#### Required action

Replace the conceptual §8.6–8.7 descriptions with implementation-complete pseudocode or an equivalent proven algorithm that defines:

- multiplication normal residual condition
- multiplication scaled comparison including equality/residual tie
- division denominator-sign normalization
- division product decomposition and high/residual tie comparison
- division small-numerator scaling branch
- division large-denominator direction/sign early-return branch
- positive and negative overflow saturation
- behavior at every threshold equality

The algorithm may differ from `kv`, but it must be sufficiently precise to derive a unique expected binary64 result for every allowed operand pair.

---

### F-PR3-002 — Medium — SIMD capability model conflates AVX2/SSE2 availability with FMA availability

- Origin: `introduced_by_change`
- Location: §4.2, §15.4–15.5
- Status: retained from previous review.

#### Problem

The design groups `AVX2 / SSE2 / ARM64` and lists `vectorized FMA residual` as a candidate implementation technique. On x86/x64, FMA is an independent instruction-set capability. .NET exposes `Fma.IsSupported` separately; `Fma` derives from `Avx`, and neither `Avx2.IsSupported` nor `Sse2.IsSupported` implies FMA support.

#### Impact

Without a capability matrix, Phase 3 can accidentally make multiplication/division dependent on FMA on a CPU where the chosen AVX2/SSE2 path is available but FMA is not.

#### Required action

Define backend requirements separately, at least:

```text
AVX-512 packed embedded rounding
AVX2 + FMA
AVX2 without FMA
SSE2 without FMA
ARM64 AdvSimd + applicable fused-operation capability
scalar fallback
```

For non-FMA vector paths, state whether the implementation uses vectorized error-free TwoProduct/Dekker logic, SIMD only for add/sub while mul/div fall back, or rejects that backend after benchmark evaluation.

---

### F-PR3-004 — Medium — IEEE 1788.1 conformance-test introduction was carried by the basic design but is missing from the detailed design

- Origin: `introduced_by_change`
- Location: basic design §13 versus detailed design §12 / §14 / §20

#### Problem

The accepted basic design explicitly left “IEEE 1788.1 conformance test の導入方法” for detailed design. PR #3 defines an exact-rational oracle, inari/kv differential comparison, random tests, and edge cases, but never defines a standard-conformance corpus or an explicit decision not to use one.

`inari`, the primary reference implementation, documents IEEE 1788.1 conformance and retains an `ITF1788` submodule/reference. This demonstrates that conformance verification is a separate concern from implementation-differential testing.

#### Impact

Exact-rational endpoint tests can prove individual arithmetic values, but they do not by themselves prove that all applicable IEEE operation semantics, constructor rules, Empty behavior, and required edge classifications are covered. The basic-design carryover is therefore unresolved.

#### Required action

Define the Phase 1 conformance strategy before implementation begins. At minimum state:

- which IEEE 1788.1 / ITF1788-compatible cases or corpus are pinned
- which Phase 1 bare-binary64 operations are in scope
- how the C# API and canonical signed-zero policy are adapted for comparison
- which cases are intentionally deferred to Phase 4
- how conformance results are represented in CI and failure artifacts

If a formal corpus is intentionally not adopted, document the equivalent conformance matrix and why it is sufficient.

---

### F-PR3-005 — Medium — reference-oracle responsibilities and execution mechanism are not defined reproducibly

- Origin: `introduced_by_change`
- Location: §12.4 and §14.2

#### Problem A: `kv` is assigned a semantic comparison role it does not implement compatibly

§12.4 says `inari` and `kv` are secondary oracles and lists `Empty / Entire の意味` among comparison units.

The pinned `kv/interval.hpp` is not an IEEE 1788 bare-interval semantic oracle for these cases. In particular its ordinary interval division throws `std::domain_error("interval: division by 0")` whenever the denominator contains zero, while Devo6.Interval intentionally returns Empty, a one-sided unbounded interval, or Entire depending on the set-based case.

`kv` is valuable here primarily for `rop<double>` directed-rounding primitives and compatible finite endpoint arithmetic, not for Devo6/inari Empty/Entire/zero-crossing semantics.

#### Problem B: there is no reproducible mechanism for obtaining the pinned external results

The design requires mismatches to log Devo6.Interval, exact-rational, inari, and kv results, but does not specify how C# tests obtain inari/kv results.

No decision is made between, for example:

- building small pinned reference executables in CI
- invoking test-only native/reference tools
- generating and committing a golden corpus from pinned SHAs
- maintaining a generator script with toolchain metadata

#### Impact

An implementation worker can satisfy the words “reference comparison” in mutually incompatible ways, and a naïve implementation can produce false failures by comparing Devo6 zero-crossing division semantics directly with `kv`.

The API-freeze condition “`inari` / `kv` との差異が説明済み” also lacks a reproducible evidence path.

#### Required action

Define oracle ownership and transport explicitly:

```text
exact rational oracle -> primary mathematical endpoint oracle
inari                  -> interval semantic/differential oracle
kv rop<double>          -> directed-rounding primitive oracle for compatible cases
```

Then define one reproducible reference-result mechanism, including pinned SHA, toolchain/dependency requirements, input/output format, generated artifact/golden-data policy, and how incompatible semantic cases are excluded or recorded as expected differences.

---

### F-PR3-006 — Medium — CI design does not implement the x64/ARM64 correctness gate declared by the design

- Origin: `introduced_by_change`
- Location: §4.2, §13, §14.2

#### Problem

Phase 1 declares x64 and ARM64 targets. The API-freeze correctness gate requires:

```text
x64 と ARM64 scalar backend で同じ canonical result を返す
```

However §13 specifies only diagnostic artifacts. It does not require an architecture matrix or any ARM64 execution.

As of this review, GitHub-hosted runners provide standard Linux x64 runners and Linux ARM64 labels such as `ubuntu-24.04-arm`, so this gate can be represented directly in the planned workflow.

#### Impact

A future Phase 1 PR could have a green x64 workflow without ever producing the ARM64 evidence required for Phase 2/API freeze. This is especially relevant because the scalar reference relies on floating-point edge behavior, FMA semantics, and subnormal handling.

#### Required action

Add a Phase 1 CI acceptance matrix that runs the same deterministic/oracle suite on at least:

- Linux x64
- Linux ARM64

Require both before the scalar pilot can become the API-freeze reference. Preserve per-architecture diagnostics. If direct cross-run bit-pattern comparison is desired, specify a deterministic result corpus/artifact that can be compared after both jobs complete.

---

### F-PR3-007 — Medium — deterministic test matrix omits the exact branch boundaries introduced by the rounding algorithm

- Origin: `introduced_by_change`
- Location: §12.2–12.5

#### Problem

The deterministic tests mention generic values such as `double.Epsilon`, max subnormal, min normal, `double.MaxValue`, cancellation, and adjacent doubles. They do not require cases at the internal algorithm thresholds or at residual ties.

These are precisely the rare branches most unlikely to be covered reliably by random testing.

#### Required action

Add deterministic tests for at least:

- immediately below / exactly at / immediately above `2^-969` for multiplication result magnitude
- immediately below / exactly at / immediately above the division small-numerator threshold
- immediately below / exactly at / immediately above `2^918` denominator magnitude
- all direction/sign combinations for the `0` versus `±2^-1074` underflow returns
- multiplication scaled comparison branches `<`, `>`, and equality with positive/negative residual
- division product-decomposition branches where rounded `q*y == x` but residual is positive or negative
- exact-result cases where no `BitIncrement`/`BitDecrement` is permitted
- positive/negative overflow for both Up and Down

The exact-rational oracle should provide expected endpoints for each fixture.

---

### F-PR3-008 — Low — basic-design native-backend decision is no longer traceable in the detailed phase plan

- Origin: `introduced_by_change`
- Location: basic design §10.1 / §13 versus detailed design §3 / §20

#### Problem

The basic design retained managed-only versus native-backend adoption as a future benchmark-driven decision. The detailed design correctly excludes native calls from Phase 1, but neither the later phase definitions nor §20’s remaining decisions say when that accepted basic-design question is closed.

This does not block the managed scalar pilot, and the user’s current preferred phase order does not require a native backend. The issue is traceability: a basic-design decision item silently disappears rather than being explicitly rejected or deferred.

#### Required action

Add one explicit decision, for example either:

- “Devo6.Interval will remain managed-only; native backend candidate from the basic design is rejected for the current roadmap”, or
- identify the later benchmark gate/phase where native batch or elementary-function acceleration may be reconsidered.

No native implementation is required in PR #3.

## Erratum to the previous review

### F-PR3-003 — WITHDRAWN — previous CI handoff-state finding was over-strong

The previous review asserted that the implementation handoff’s `ci.conclusion: not_applicable` / `head_sha: unknown` was itself a defect because no exact-head run existed.

That finding is withdrawn.

Reason:

- The handoff packet is committed before its own final commit SHA exists, so it cannot truthfully contain its own future SHA.
- The existing packet explicitly marks commit/push/CI publication as pending and states that final exact-head evidence is recorded externally after publication.
- The PR completion comment then records final HEAD `da6e2ae...`, matching workflow run count zero, and explicitly reports CI as unexecuted without substituting another SHA.

Therefore the terminal publication mechanics are auditable and consistent with the worker handoff model. The absence of a workflow is still a fact, but it is not a defect in that handoff packet for this documentation-only PR.

The historical previous report is not modified; this report is the corrective record.

## Required coverage dispositions

| Criterion | Disposition | Evidence / result |
|---|---|---|
| Requirement / design conformance | `checked_finding` | Phase order is correct; IEEE conformance and native-backend carryovers need closure (F-PR3-004, F-PR3-008). |
| Numerical correctness / edge cases | `checked_finding` | Sign tables pass; directed-rounding specification remains incomplete (F-PR3-001). |
| Scope discipline / unrelated changes | `checked_no_finding` | Technical PR content is documentation/report/handoff only. |
| Changed files / direct dependencies | `checked_finding` | All technical files, accepted basic design, pinned inari/kv, .NET surface, and previous review report inspected; reference boundary issue found (F-PR3-005). |
| Public API / compatibility | `checked_no_finding` | Pilot API is explicitly provisional until Phase 2; no contradictory signature requirement found. |
| Internal representation / value semantics | `checked_no_finding` | `[-Lower, Upper]`, Empty, Zero, equality and Hash design are coherent. |
| Multiplication / division classification | `checked_no_finding` | Rechecked against endpoint extrema and pinned inari. |
| SIMD backend / capability design | `checked_finding` | x86 FMA capability separation missing (F-PR3-002). |
| Error handling / domain semantics | `checked_no_finding` | Devo6 set-based zero-division behavior is coherent; kv must not be used as semantic oracle for incompatible cases. |
| Security / secrets | `not_applicable` | Documentation-only numerical library design; no credentials or security-sensitive execution path. |
| TDD / deterministic test adequacy | `checked_finding` | Algorithm-specific threshold/tie fixtures missing (F-PR3-007). |
| Independent oracle adequacy | `checked_finding` | Exact rational strategy is sound; external reference roles/harness need definition (F-PR3-005). |
| IEEE conformance validation | `checked_finding` | Basic-design carryover is absent (F-PR3-004). |
| x64 / ARM64 validation | `checked_finding` | Gate exists but CI matrix does not (F-PR3-006). |
| Failure diagnostics | `checked_no_finding` | Required test/stdout/stderr/runner/reference-mismatch artifacts are specified; Phase 1 workflow still needs to implement them. |
| Current-HEAD CI evidence | `checked_no_finding` | Technical reviewed HEAD and rerun-start administrative HEAD both have zero matching pull-request workflow runs; correctly treated as CI未実施. No other SHA used. |
| Report / handoff accuracy | `checked_finding` | Previous F-PR3-003 is corrected by this erratum; implementation handoff terminal mechanics are accepted. |
| Regression / maintainability risk | `checked_finding` | Reference ambiguity, architecture evidence, and threshold coverage would otherwise propagate into the frozen scalar reference. |

## Validation assessment

### Repository state

At rerun start:

- PR current HEAD: `9936d9f2d5bb270ba03938fdecb6dde8cc0541f1`
- technical design HEAD: `da6e2ae04d35b01acfb307953a093c81c15342b8`
- commits after technical HEAD before this rerun: one previous review-report commit only
- `.github/workflows/`: `.gitkeep` only
- matching pull-request workflow runs for `da6e2ae...`: 0
- matching pull-request workflow runs for `9936d9f...`: 0
- CI state: **CI未実施**

No run from another SHA was used.

### Reference freshness

As of 2026-08-29:

- pinned `unageek/inari` SHA `18b83a...` is the repository HEAD returned by GitHub
- pinned `mskashi/kv` SHA `c7f8f232...` is the repository HEAD returned by GitHub

No stale-pin finding is raised.

### Numerical witness validation

The division residual-tie witness in F-PR3-001 was checked using exact rational representations of the binary64 inputs and quotient. The rounded product equals `x`, while the exact product of the rounded quotient and denominator is below `x`; the next representable quotient is the tight upward result.

### Build / tests

No executable project exists in this PR, so there is no repository build/test to run. This review evaluates the design and reference algorithms rather than claiming executable validation.

## Verdict

**FAIL**

PR #3 should not be treated as Phase 0 complete yet.

The blocking numerical issue is F-PR3-001. The Medium findings should be resolved in the same design correction so Phase 1 does not begin with incomplete acceptance/evidence rules. F-PR3-008 is non-blocking by itself but should be closed while updating requirement traceability.

## Required closure set

A single correction pass should address all active findings together:

1. make multiplication/division directed-rounding pseudocode complete, including residual ties and subnormal/underflow branches
2. define FMA/ISA capability matrix
3. define IEEE 1788.1 conformance-test strategy
4. split exact-rational / inari / kv oracle responsibilities and define the reproducible comparison harness
5. define x64 + ARM64 CI matrix
6. add deterministic threshold/tie/underflow fixtures
7. explicitly close or defer the native-backend decision
8. update the design implementation report/handoff to reflect the corrected design

After those changes, a fix-verification review should check only the resolved finding set plus regression in directly affected sections.

## Merge boundary

No merge is performed by the reviewer.