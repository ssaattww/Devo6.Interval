# PR #5 指摘対応報告

## 1. 対象

- Repository: `ssaattww/Devo6.Interval`
- PR: #5 `docs: define implementation phases and task acceptance criteria`
- Mode: review follow-up / documentation fix
- Review finding source: `reports/2026-08-31-pr5-initial-review.md`
- Reviewed implementation HEAD: `b20fa7d7c31326770fac925a4e0c1b35cf72de6b`
- Follow-up start HEAD: `c46ba0ca7af476d1ed2857532f5efbe86f5cc0e2`
- Final technical-content HEAD: `5d87d2be94e6e2c48b292f90119e10cf621153b0`
- Branch: `docs/task-and-phase-plan`
- Base: `main`

## 2. 要求

ユーザー要求は、PR #5のレビュー指摘へ対応し、特に `tasks/tasks-status.md` の受け入れ条件を具体化することである。

初回reviewでは `F-PR5-001`～`F-PR5-010` が修正必須となった。修正後は、詳細設計本文を再解釈しなくても、task本文から入力、期待値、例外、branch補正、binary64 bits、artifact field、review gate、benchmark gateによって合否を判定できることを目標とした。

## 3. 変更概要

### `tasks/phases-status.md`

- Phase単位の判定規則を追加した。
- Phase 4A～4Eの各subphase開始前にpreflight review gateを要求した。
- Phase 1にdiagnostic artifact、hot-path、NativeAOT、trimmingの具体的完了条件を追加した。
- Phase 3に事前固定benchmark policyとproduction採用閾値を追加した。
- Phase 4A～4Eに代表的な固定入力・期待結果を追加した。

### `tasks/tasks-status.md`

- task数を67へ再編した。
- source task共通条件へRed/Green証拠、bit比較、artifact、exact-head CI規則を追加した。
- `P1-013 hot-path・NativeAOT・trimming final gate`を追加した。
- `P4A-000`, `P4B-000`, `P4C-000`, `P4D-000`, `P4E-000`の5 preflight taskを追加した。
- 四則演算、SIMD、集合演算、elementary functions、extended operations、parser、binary interchange等の各taskへ固定入力、期待式、期待例外、bitwise条件を追加した。
- 最終具体化commitでは、残っていた「相当」「候補」「十分広い」「matrixを検証」等の曖昧表現をさらに式・caseへ置換した。
- `F-PR5-001`～`F-PR5-010`の対応表を追加した。

## 4. Finding対応

| Finding | Disposition | 修正内容 | Evidence |
|---|---|---|---|
| `F-PR5-001` High | addressed | Phase 4A～4Eの先頭にpreflight taskを追加。reviewed HEAD、verdict、report path、diagnostic workflow、smoke fixture、Red開始禁止を条件化 | `P4A-000`, `P4B-000`, `P4C-000`, `P4D-000`, `P4E-000` |
| `F-PR5-002` Medium | addressed | failure artifactへcaseId/inputBits/selectedBranch/exactResult/Devo6/inari/kv/MPFR/expectedDifferenceReasonを必須化 | `P1-001` |
| `F-PR5-003` Medium | addressed | `[-Lower,Upper]`、Empty qNaN 2 lane、片側NaN禁止、raw result construction、diagnostic `ToString`を条件化 | `P1-002` |
| `F-PR5-004` Medium | addressed | multiply/divide residual branchの`BitIncrement`/`BitDecrement`/無補正条件とfixed witnessを明記 | `P1-006`, `P1-008` |
| `F-PR5-005` Medium | addressed | API 3scenarioを固定し、benchmark workload/metric/採用閾値をcandidate結果確認前に固定する条件を追加 | `P1-012`, `P2-001`, `P3-006` |
| `F-PR5-006` Medium | addressed | parser accepted/rejected fixture、exact hex、InvariantCulture、recursion禁止、resource-limit decisionを追加 | `P4E-000`, `P4E-009` |
| `F-PR5-007` Medium | addressed | binary v1=18 byte、offset、little-endian、17/19 byte length-first rejectを追加 | `P4E-010` |
| `F-PR5-008` Medium | addressed | allocation 0、disassembly、NativeAOT x64/ARM64、trimming、raw constructor、BigInteger/global rounding mode禁止をfinal gate化 | `P1-013`, Phase 1 gate |
| `F-PR5-009` Low | addressed | `Decoration : byte` と `0/4/8/12/16`を固定 | `P4E-006` |
| `F-PR5-010` Medium | addressed | managed-only=N/A理由、native採用時のinterop性能/ABI/thread/AOT/trimming/distribution/license gateを必須化 | `P4B-007` |

## 5. 受け入れ条件の具体化例

### 固定入力 → 固定結果

- `[1,2] + Interval.Point(3) -> [4,5]`
- `[1,2] / [-1,1] -> Entire`
- `new Interval(2,1) -> ArgumentException`
- `DivideToUnion([1,2],Entire)`はCount=2で `[-Infinity,-0.0]` と `[+0.0,+Infinity]`
- `Decoration : byte` は `Ill=0, Trv=4, Def=8, Dac=12, Com=16`

### branch predicate → 補正動作

- `MultiplyUp` scaled path: `t<s || (t==s && s2>0)`なら`BitIncrement(product)`、exactなら無補正。
- `MultiplyDown` scaled path: `t>s || (t==s && s2<0)`なら`BitDecrement(product)`、exactなら無補正。
- `DivideUp`: `r<xn || (r==xn && r2<0)`なら`BitIncrement(q)`。
- `DivideDown`: `r>xn || (r==xn && r2>0)`なら`BitDecrement(q)`。

### parser / wire contract

- `[-Infinity,1]`, `[1,+Infinity]`, `[0x1p+0]`等をaccepted fixture化。
- `[+Infinity,1]`, `[1,-Infinity]`, `[Infinity]`, `[NaN,1]`等をrejected fixture化。
- binary v1は18 byte固定: byte0=version, byte1=state, byte2..9=Lower LE, byte10..17=Upper LE。
- 17/19 byteはendpoint decode前にrejectする。

### performance gate

- BenchmarkDotNet/.NET 10。
- batch N=`4,32,256,4096`。
- operations=`Add,Sub,Mul,Div`。
- metric=`median ns/interval`。
- production採用条件: N>=256全workloadのscalar比median幾何平均`<=0.95`、各workload`<=1.02`、allocation増加`0 B`。
- benchmark policy commitはcandidate結果測定commitより前に存在することを要求する。

## 6. Validation

### Repository compare

follow-up start `c46ba0ca7af476d1ed2857532f5efbe86f5cc0e2` からfinal technical-content HEAD `5d87d2be94e6e2c48b292f90119e10cf621153b0` までGitHub compareを確認した。

- status: ahead
- ahead_by: 5
- behind_by: 0
- source/test/workflow変更: なし
- task/phase文書とfollow-up report/handoffのみ変更

technical contentを構成する主要commit:

- `cbe2f3c66bcb201bd1e05b6362b90302d5682bd6` — phase gate具体化
- `5c4c3c12490d16296867cdaf74015a6c7c8ab7ca` — task acceptance criteria全面具体化
- `5d87d2be94e6e2c48b292f90119e10cf621153b0` — 残存曖昧表現を式・固定caseへ置換

### Content re-fetch

branch上の `tasks/tasks-status.md` を再取得し、以下を確認した。

- Phase 4A～4Eのpreflight依存がtask summaryに存在する。
- P1-006/P1-008にbranch predicateと補正結果が存在する。
- P1-009にpositive/negative denominator × P/N/Mの6 endpoint式が存在する。
- P1-013にNativeAOT/trimming/hot-path gateが存在する。
- P3-006にbenchmark workload/metric/閾値が存在する。
- P4D-005にzero-touch general Powの6class式が存在する。
- P4E-002にone-sided/cross-zero extended division式が存在する。
- P4E-005にcancellative 3x3 matrixの期待結果が存在する。
- P4E-009/P4E-010にparser/binaryのaccepted/rejected contractが存在する。
- `F-PR5-001`～`F-PR5-010`の対応表が存在する。

### Build/Test

本PRはdocumentation-onlyで、repositoryにはまだ実行可能production/test projectがないためbuild/testはnot applicableである。

### CI

final technical-content HEAD `5d87d2be94e6e2c48b292f90119e10cf621153b0`時点ではpull_request workflow runが存在しない。

- preliminary status: `CI未実施`
- 別SHA run代用: なし
- 最終administrative HEADのexact-head CI確認結果は、handoff保存後にPRコメントへ記録する。

## 7. Intentionally untouched

- `doc/Design/**`: 数値意味論そのものは変更していない。
- `src/**`, `tests/**`: implementation未開始のため変更していない。
- `.github/workflows/**`: 詳細設計§16.1どおり、実行可能project/testを追加する`P1-001`で同時追加する。
- `reports/2026-08-31-pr5-initial-review.md`: immutable review historyとして改変していない。

## 8. Remaining state

- `P0-001` 詳細設計版5 fix verification: 進行中
- `P0-002` フェーズ・タスク管理基盤: PR #5 fix verification待ち
- Phase 1以降: 未着手
- `F-PR5-001`～`F-PR5-010`: implementation-worker dispositionはaddressed、normal reviewer fix verification待ち
- merge: 未実施

## 9. Next action

同じnormal reviewで、新しいPR current HEADについて `F-PR5-001`～`F-PR5-010`を `required action -> document path -> actual fixture/evidence -> focused evidence` のcompleteness matrixでfix verificationする。

CI確認は、その時点のPR current HEAD SHAとrun `head_sha`が一致するrunだけを使用し、一致runがなければ `CI未実施` とする。
