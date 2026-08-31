# PR #5 fix verification report

## 1. Review target

- Repository: `ssaattww/Devo6.Interval`
- PR: #5 `docs: define implementation phases and task acceptance criteria`
- Review mode: `fix_verification`
- Base: `main` (`f71466de50e6eca4942b614b253ecbc96464fa17`)
- Initial reviewed implementation HEAD: `b20fa7d7c31326770fac925a4e0c1b35cf72de6b`
- Fix technical-content HEAD: `5d87d2be94e6e2c48b292f90119e10cf621153b0`
- Fix-verification target PR HEAD: `57676dd738cc6c9ed930879aa873e43cc128d72d`
- Fix range: `c46ba0ca7af476d1ed2857532f5efbe86f5cc0e2..57676dd738cc6c9ed930879aa873e43cc128d72d`
- Reviewer: same ChatGPT normal-review chat as the initial review
- Reviewer continuity: maintained; this chat performed the initial review and did not implement the fixes
- Authoritative design: `doc/Design/detail/IntervalArithmetic.md` design version 5
- Previous findings: `F-PR5-001` through `F-PR5-010`

The fix range is 8 commits ahead of the initial review-report commit. The technical acceptance-criteria changes end at `5d87d2be94e6e2c48b292f90119e10cf621153b0`; the later commits only update the follow-up report and handoff.

## 2. Validation / CI state

- Repository changes remain documentation/report only. No `src/**`, `tests/**`, or executable workflow was added.
- No executable production/test project exists yet, therefore source build/test is `not_applicable` for this PR.
- PR current HEAD at verification time: `57676dd738cc6c9ed930879aa873e43cc128d72d`.
- Matching pull-request workflow runs for that exact SHA: **0**.
- CI status: **CI未実施**.
- No workflow run from another SHA was substituted.

## 3. Verdict

**FAIL — fix verification did not close all required findings.**

Seven of the ten initial findings are verified as addressed. Three initial findings remain open/partial, and two regressions/new omissions were found in the modified acceptance criteria.

Open carried findings:

- `F-PR5-001` High
- `F-PR5-007` Medium
- `F-PR5-010` Medium

New findings:

- `F-PR5-011` Medium
- `F-PR5-012` Medium

## 4. Finding completeness matrix

| Finding | Source severity | Required action | Document path | Actual composition fixture/evidence | Focused verification | Disposition |
|---|---|---|---|---|---|---|
| F-PR5-001 | High | Add Phase 4A–4E preflight/start gates that complete before the first source Red commit | `tasks/phases-status.md`, `P4A-000`–`P4E-000` | Preflight tasks and dependencies are present | Gate ordering is contradictory: common gate requires the first new-function smoke to exit 0 before Red; `P4A-000` requires `Contains` to pass even though `Contains` is implemented by dependent `P4A-001`; `P4B-000` requires a Square Red smoke while the common gate requires smoke success | **partial / open** |
| F-PR5-002 | Medium | Enumerate §16.4 diagnostic fields | `P1-001` | `caseId/inputBits/selectedBranch/exactResult/devo6ResultBits/inari/kv/mpfr/expectedDifferenceReason`; `N/A` rule | All requested diagnostic fields are explicit | **addressed** |
| F-PR5-003 | Medium | Fix internal canonical state and diagnostic `ToString` acceptance | `P1-002` | `[-Lower,Upper]`, canonical qNaN two lanes, one-sided NaN rejection, raw result construction, ToString cases | Required internal/public-state evidence is explicit | **addressed** |
| F-PR5-004 | Medium | Specify branch predicate → increment/decrement/no-correction behavior | `P1-006`, `P1-008` | Multiply and divide five-case witness sets with expected output bits | Predicates match design §§9.3/10.4 | **addressed** |
| F-PR5-005 | Medium | Make API and benchmark acceptance objective | `P1-012`, `P2-001`, `P3-006`, phase 3 gate | Three fixed API scenarios; fixed BDN runtime/corpus/batch/operations/metric/threshold; policy commit must precede measurement | Completion rule is objectively checkable before candidate adoption | **addressed** |
| F-PR5-006 | Medium | Complete parser grammar/security fixtures | `P4E-000`, `P4E-009` | unbounded/decimal/C99 hex accepted cases, invalid cases, InvariantCulture, no recursion, numeric resource-limit decision | Requested parser-surface/security gaps are covered | **addressed** |
| F-PR5-007 | Medium | Fix binary v1 layout and reject-order contract | `P4E-000`, `P4E-010` | 18-byte length, offsets, little-endian, 17/19-byte length-first rejection | Physical layout was fixed, but byte-1 state-code values and per-state canonical payload/reject-vs-canonicalize rules remain unspecified, so two incompatible v1 encoders can satisfy the task | **partial / open** |
| F-PR5-008 | Medium | Add Phase 1 hot-path/AOT/trimming gate | `P1-013`, Phase 1 gate | allocation 0, disassembly, reflection/codegen/native-resolver prohibition, NativeAOT x64/ARM64, trimming, raw construction | §47 items are explicitly testable | **addressed** |
| F-PR5-009 | Low | Fix Decoration underlying type/numeric values | `P4E-006` | `byte`, `0/4/8/12/16` | Exact public enum representation is fixed | **addressed** |
| F-PR5-010 | Medium | Native backend: N/A or §18 qualification with interop-inclusive real workload | `P4B-007` | Conditional native gate now exists | Performance condition says to use `P3-006`'s same benchmark policy, whose operation set is `Add/Sub/Mul/Div`; that does not exercise an elementary endpoint backend used for Exp/Log/Sin/etc., so the native backend's interop/copy/dispatch cost and benefit are not actually measured | **partial / open** |

## 5. Remaining carried findings

### F-PR5-001 — High — Phase 4 preflight now exists but its smoke gate is circular/contradictory

**Origin:** introduced_by_fix / carried finding not closed  
**Location:** `tasks/phases-status.md` §8; `tasks/tasks-status.md` `P4A-000`–`P4E-000`

The initial finding required each Phase 4 subphase review preflight to complete before the first source Red commit. That dependency is now present. However, the common Phase 4 gate additionally requires:

> the subphase's first smoke fixture to execute from the existing harness with exit code 0

before the preflight is complete and before the first source Red commit.

`P4A-000` then makes the smoke behavior `Entire.Contains(+Infinity)==false` and `Entire.Contains(0.0)==true`. `Contains` itself is the production behavior implemented in `P4A-001`, which directly depends on `P4A-000`.

Therefore the stated order is impossible for a fresh implementation:

1. `P4A-000` must be complete.
2. To complete it, `Contains` smoke must pass.
3. `Contains` production implementation cannot start until `P4A-000` is complete and `P4A-001` Red is allowed.

`P4B-000` has the inverse contradiction: it explicitly asks for a **Square smoke Red test**, while the phase-common gate requires the first smoke fixture to exit 0. Similar wording differences exist for 4C–4E (`compareable`, `registered`, or `readable` versus the common `exit code 0`).

**Impact**

The task graph either blocks Phase 4A entirely or forces an implementer to violate the specified TDD/preflight order. Different workers can also interpret “smoke” differently and claim incompatible completion evidence.

**Required action**

Separate **preflight harness/reference readiness** from the **first production Red test**.

Concrete acceptance example:

- `P4A-000`: verify that the fixture definition `input=Entire, member=+Infinity, expected=false` can be loaded by the existing harness/reference layer without executing a not-yet-implemented `Contains` production path.
- `P4A-000`: verify design review/report/workflow/reference availability and then mark preflight complete.
- `P4A-001` Red commit: execute the actual public/internal `Contains` test; it must fail for the intended unimplemented behavior.
- `P4A-001` Green commit: implement `Contains`; the same focused test must exit 0.

Apply the same distinction to P4B–P4E and change the Phase 4 common gate so it does not require unimplemented subphase production behavior to pass before Red.

### F-PR5-007 — Medium — v1 wire format still leaves state-byte/canonical-payload interoperability undefined

**Origin:** coverage_miss / carried finding not fully closed  
**Location:** `P4E-000`, `P4E-010`

The fix correctly pins 18 bytes, byte offsets, little-endian endpoints, and length-first rejection. It still does not fix:

- the exact numeric byte values representing each valid state;
- the canonical bytes required in endpoint payload positions for special states, especially Empty;
- whether noncanonical special-state endpoint payloads are rejected or canonicalized/ignored;
- the complete state-to-endpoint validation matrix.

Design §50 explicitly leaves final version-1 reject/canonicalize details to the Phase 4E subphase, so the task needs a decision gate for those details.

**Impact**

Two implementations can both satisfy the current acceptance conditions yet emit incompatible byte streams for the same interval state. That defeats a versioned interchange contract.

**Required action**

In `P4E-000`, require a version-1 state table to be fixed before `P4E-010` Red begins. The table must contain at least:

- state symbolic name;
- exact byte-1 numeric code;
- allowed/required Lower payload bits;
- allowed/required Upper payload bits;
- decoder action for a noncanonical payload (`reject` or `canonicalize`, exactly one);
- expected decoded public state.

Then `P4E-010` must include fixed 18-byte hex fixtures for Normal, Zero, Entire, and Empty plus invalid-state/noncanonical-payload negative fixtures.

### F-PR5-010 — Medium — native elementary backend benchmark measures the wrong operation class

**Origin:** introduced_by_fix / carried finding not closed  
**Location:** `P4B-007`, `P3-006`

`P4B-007` now correctly makes native qualification conditional, covers ABI/thread/AOT/trimming/distribution/license, and requires an interop-inclusive benchmark. However it says to measure with the **same P3-006 policy**.

`P3-006` fixes its operation set to `Add, Sub, Mul, Div`. Those are Phase 1/3 arithmetic kernels. A Phase 4 elementary endpoint native backend is selected to supply operations such as Exp/Log/Sin/etc. Measuring Add/Sub/Mul/Div cannot demonstrate the native elementary backend's interop/copy/dispatch overhead or its benefit on the path that would actually be shipped.

**Impact**

A native endpoint backend may be accepted or rejected using unrelated measurements. If no managed certified elementary backend exists, this can block Phase 4C for the wrong reason; if arithmetic happens to benchmark well, it can also falsely qualify an elementary backend whose actual calls regress performance.

**Required action**

Reuse only the **measurement mechanics** of P3-006, not its operation list. Before native elementary measurement, fix a workload that exercises every native endpoint function proposed for production (or a documented representative production subset plus per-function no-regression checks), including the real interop/copy/dispatch path.

Example policy fields:

- fixed function list, e.g. the candidate's production Exp/Log/Sin/... entry points;
- fixed binary64 input corpus and SHA-256;
- scalar/managed baseline used for the same public operation;
- call shape (scalar calls and any supported batch shape);
- median metric, adoption threshold, per-function no-regression limit, and allocation/copy metric;
- same CPU/runtime pinning rules as P3-006.

## 6. New findings introduced/exposed by the fix

### F-PR5-011 — Medium — `P4C-004` dropped the Acos decreasing-endpoint acceptance rule

**Origin:** introduced_by_fix  
**Location:** `tasks/tasks-status.md` `P4C-004`

The design defines a generic monotonic-decreasing interval rule, and Acos is the Phase 4C decreasing function. The pre-fix task explicitly required that Acos endpoint order not be reversed. The rewritten `P4C-004` now only states that Asin/Acos clip to `[-1,1]`; it does not state the Acos lower/upper construction.

A worker reading only the task can no longer distinguish the correct rule:

```text
Acos([a,b] clipped to [l,u]) = [AcosDown(u), AcosUp(l)]
```

from an incorrect increasing-order implementation.

**Impact**

This is exactly the class of ambiguity PR #5 is intended to remove. Incorrect endpoint ordering can create invalid or non-enclosing results near the Acos domain boundaries.

**Required action**

Restore an explicit decreasing formula and at least one fixed example. For example:

- clip to `[l,u]=X∩[-1,1]`; Empty intersection -> Empty;
- `Acos(X)=[AcosDown(u),AcosUp(l)]`;
- `Acos([0,1])=[AcosDown(1),AcosUp(0)]`, with expected endpoint bits fixed by the MPFR corpus;
- include a clipped case such as `Acos([-2,0])=[AcosDown(0),AcosUp(-1)]`.

### F-PR5-012 — Medium — exact text formatting became optional although Phase 4E includes exact text interchange

**Origin:** introduced_by_fix  
**Location:** `tasks/tasks-status.md` `P4E-009`, `P4E-012`; `tasks/phases-status.md` Phase 4E

The design's Phase 4E scope includes exact text/binary interchange. The pre-fix `P4E-009` required exact/round-trip formatting to parse back to identical canonical endpoint bits. The rewritten condition now says:

> exact round-trip format **if adopted**

No preflight task requires at least one exact persistent text format to be adopted, and `P4E-012` does not restore that requirement. Therefore the project can reject every formal round-trip format and still mark P4E-009/P4E-012 complete.

**Impact**

Phase 4E can be declared complete without one of its designed outputs: exact persistent text interchange. It also weakens the API freeze because later introduction of a required exact format would add a public contract after the phase gate.

**Required action**

Make the formatting decision mandatory in `P4E-000`/`P4E-009`:

- select at least one exact persistent/round-trip format and fix its syntax;
- keep diagnostic `ToString` separate;
- require `formatExact -> parseExact` canonical endpoint bits to match for Normal, Zero, Empty, Entire, bounded/unbounded, subnormal, and signed-zero cases;
- `P4E-012` must require those round-trip fixtures to pass before Phase 4E completion.

## 7. Closed initial findings

The following initial findings are verified closed for this HEAD:

- `F-PR5-002` Medium — diagnostic artifact fields
- `F-PR5-003` Medium — internal canonical state / `ToString`
- `F-PR5-004` Medium — directed mul/div branch expectations
- `F-PR5-005` Medium — API scenarios / objective benchmark gate
- `F-PR5-006` Medium — parser input surface/security
- `F-PR5-008` Medium — hot path / NativeAOT / trimming
- `F-PR5-009` Low — Decoration exact representation

Do not edit the initial review report to change its historical verdict. Closure is recorded by this fix-verification report.

## 8. Coverage dispositions

| Criterion | Disposition | Evidence |
|---|---|---|
| requirement/design conformance | checked_finding | F-PR5-001, F-PR5-010, F-PR5-011, F-PR5-012 |
| correctness / edge-case acceptance | checked_finding | F-PR5-011 |
| scope discipline | checked_no_finding | fix range changes task/phase/report/handoff only |
| task dependency/order | checked_finding | F-PR5-001 |
| API/data compatibility | checked_finding | F-PR5-007, F-PR5-012 |
| configuration/workflow | checked_no_finding | workflow deferral remains aligned with design §16.1 |
| error/failure diagnostics | checked_no_finding | F-PR5-002 verified closed |
| security/resource handling | checked_no_finding | parser limits/culture/recursion and binary length-first rejection are present; wire interoperability still tracked by F-PR5-007 |
| tests/validation adequacy | checked_finding | F-PR5-001, F-PR5-010 |
| current-HEAD CI evidence | held | exact current HEAD has 0 workflow runs; CI未実施 |
| reports/handoff accuracy | checked_finding | follow-up artifacts correctly say normal fix verification is still required, but their implementation-worker `addressed` dispositions are not reviewer closure; this report records the reviewer result |
| regression/maintainability | checked_finding | F-PR5-011 and F-PR5-012 are regressions from the pre-fix acceptance text |

## 9. Held / unexplored

### Held

- Exact-head CI: current verification target `57676dd738cc6c9ed930879aa873e43cc128d72d` has zero matching workflow runs. This is recorded as `CI未実施`, not success.

### Unexplored

- None for the documentation fix scope. All seven PR-changed files were considered; the fix delta and direct design sections for the carried findings were inspected.

## 10. Next action

- Verdict: `fail`
- Return to the implementation/documentation-fix worker.
- Fix `F-PR5-001`, `F-PR5-007`, `F-PR5-010`, `F-PR5-011`, `F-PR5-012`.
- Preserve the closed status of `F-PR5-002/003/004/005/006/008/009` unless a later change touches the same contract.
- After fixes, use this same normal-review chat for another fix verification against the new immutable PR HEAD.
- CI evidence must match the new PR current HEAD exactly; no other SHA may be substituted.
- Do not merge.
