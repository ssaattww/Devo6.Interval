# PR #5 指摘対応報告

## 1. 対象

- Repository: `ssaattww/Devo6.Interval`
- PR: #5 `docs: define implementation phases and task acceptance criteria`
- Mode: review follow-up / documentation fix
- Review finding source: `reports/2026-08-31-pr5-initial-review.md`
- Reviewed implementation HEAD: `b20fa7d7c31326770fac925a4e0c1b35cf72de6b`
- Review-report-added HEAD at follow-up start: `c46ba0ca7af476d1ed2857532f5efbe86f5cc0e2`
- Technical fix HEAD: `5c4c3c12490d16296867cdaf74015a6c7c8ab7ca`
- Branch: `docs/task-and-phase-plan`
- Base: `main`

## 2. 要求

ユーザー要求は、PR #5の指摘対応と、`tasks/tasks-status.md` にある受け入れ条件を具体的にすることである。

初回reviewは `F-PR5-001`～`F-PR5-010` を修正必須とした。特に、詳細設計を再読しないと合否を決定できない記述を減らし、入力、期待値、例外、bit、artifact、review gate、benchmark gateによって判定可能にすることが要求された。

## 3. 変更ファイル

### `tasks/phases-status.md`

- Phase単位の判定規則を追加した。
- Phase 4A～4Eにpreflight review gateを明記した。
- Phase 1へdiagnostic artifact、hot-path、NativeAOT、trimmingの具体的完了条件を追加した。
- Phase 3のbenchmark policyとproduction採用閾値を具体化した。
- Phase 4A～4Eの代表fixtureをフェーズ完了条件へ追加した。

### `tasks/tasks-status.md`

- source task共通のRed/Green証拠、bit比較、artifact、current-HEAD CI規則を具体化した。
- `P1-013 hot-path・NativeAOT・trimming final gate`を追加した。
- `P4A-000`, `P4B-000`, `P4C-000`, `P4D-000`, `P4E-000`の5つのimplementation preflightを追加した。
- 各taskへ入力例、期待結果、例外、branch条件、bitwise判定、fixture件数、artifact field等を追加した。
- 末尾に `F-PR5-001`～`F-PR5-010` の修正対応表を追加した。

## 4. Finding対応

| Finding | Disposition | 修正内容 | Evidence |
|---|---|---|---|
| F-PR5-001 High | addressed | Phase 4A～4Eの各先頭にpreflight taskを追加し、reviewed HEAD/verdict/report path/diagnostic workflow/smoke fixtureを必須化。preflight完了前のRed commitを禁止 | `tasks/tasks-status.md` P4A-000/P4B-000/P4C-000/P4D-000/P4E-000、`tasks/phases-status.md` Phase 4共通開始ゲート |
| F-PR5-002 Medium | addressed | P1-001 artifactにcaseId/inputBits/selectedBranch/exactResult/Devo6/inari/kv/MPFR/expectedDifferenceReasonを列挙 | `tasks/tasks-status.md` P1-001 |
| F-PR5-003 Medium | addressed | `[-Lower,Upper]`、Empty qNaN 2 lane、片側NaN禁止、raw result constructor、diagnostic ToStringを条件化 | P1-002 |
| F-PR5-004 Medium | addressed | multiply/divide branchごとにNextUp/NextDown/無補正条件と5 witnessを明示 | P1-006, P1-008 |
| F-PR5-005 Medium | addressed | API 3scenarioを固定。Phase 3 benchmarkはworkload/baseline/metric/採用閾値を明示 | P1-012, P2-001, P3-006 |
| F-PR5-006 Medium | addressed | parser accepted/rejected fixture、C99-style exact hex、InvariantCulture、recursion禁止、resource-limit gateを追加 | P4E-000, P4E-009 |
| F-PR5-007 Medium | addressed | binary v1=18 byte、byte offset、little-endian、17/19 byte length-first rejectを追加 | P4E-010 |
| F-PR5-008 Medium | addressed | allocation 0、disassembly、NativeAOT x64/ARM64、trimming、raw constructor、BigInteger/global rounding mode禁止をfinal gate化 | P1-013、Phase 1完了条件 |
| F-PR5-009 Low | addressed | `Decoration : byte`, `Ill=0,Trv=4,Def=8,Dac=12,Com=16`を固定 | P4E-006 |
| F-PR5-010 Medium | addressed | managed-only=N/A理由必須、native採用時はinterop込み性能、ABI/thread、AOT/trimming、x64/ARM64配布、license/NOTICEを必須化 | P4B-007 |

## 5. 受け入れ条件の具体化方針

今回の修正では、単に「fixtureを置く」「設計どおり」「改善する」と書かず、可能な箇所は次の形式へ変更した。

1. **固定入力→固定結果**
   - 例: `[1,2]/[0,0] -> Empty`
   - 例: `DivideToUnion([1,2],Entire)`はCount=2で`[-Infinity,-0.0]`, `[+0.0,+Infinity]`
2. **branch predicate→補正動作**
   - 例: DivideUpは`r<xn || (r==xn && r2<0)`で`BitIncrement(q)`
3. **API misuse→exact exception/result**
   - 例: reversed constructorは`ArgumentException`
4. **wire/parser→accepted/rejected byte/text fixture**
   - 例: binary v1は18 byte、17/19 byteはendpoint decode前にreject
5. **performance→事前固定metric/threshold**
   - N=`4/32/256/4096`, metric=`median ns/interval`, N>=256の幾何平均ratio<=0.95、個別<=1.02、allocation増加0 B
6. **process gate→証拠field**
   - review reportにreviewed full SHA、verdict、unresolved finding count、report pathを記録

## 6. Validation

### Repository compare

`c46ba0ca7af476d1ed2857532f5efbe86f5cc0e2..5c4c3c12490d16296867cdaf74015a6c7c8ab7ca` をGitHub compareで確認した。

- ahead_by: 2
- behind_by: 0
- modified: `tasks/phases-status.md`
- modified: `tasks/tasks-status.md`
- source/test/workflow変更: なし

### Content re-fetch

branch上の `tasks/tasks-status.md` を再取得し、少なくとも以下を確認した。

- Phase 4A～4E preflight dependencyがsummaryに存在する。
- P1-006のscaled multiply branchごとのNextUp/NextDown/無補正が存在する。
- P1-008のdivision residual branchごとのNextUp/NextDown/無補正が存在する。
- P1-013のNativeAOT/trimming/hot-path gateが存在する。
- P4E-009にparser accepted/rejected fixtureが存在する。
- P4E-010に18 byte/byte offset/little-endian/length-first rejectが存在する。
- F-PR5-001～010の対応表が存在する。

### Build/Test

本PRはdocumentation-onlyで、repositoryにはまだ実行可能production/test projectがないためbuild/testはnot applicableとした。

### CI

Technical fix HEAD `5c4c3c12490d16296867cdaf74015a6c7c8ab7ca`に紐づくpull_request workflow runを確認した結果、0件だった。

- CI status: `CI未実施`
- 別SHA run代用: なし

## 7. Intentionally untouched

- `doc/Design/**`: 数値意味論そのものは変更していない。
- `src/**`, `tests/**`: implementation未開始のため変更していない。
- `.github/workflows/**`: 詳細設計§16.1どおり、実行可能project/testを追加するP1-001で同時追加する。
- initial review report: 履歴証拠のため改変していない。

## 8. Remaining state

- P0-001 詳細設計版5 fix verification: 進行中
- P0-002 フェーズ・タスク管理基盤: fix verification待ちのため進行中
- Phase 1以降: 未着手
- PR #5の今回修正はnormal reviewerによるfix verificationが必要
- mergeは実施しない

## 9. Next action

同じnormal reviewで、`F-PR5-001`～`F-PR5-010`について `required action -> document path -> fixture/evidence -> focused evidence` のcompleteness matrixを用いてfix verificationを行う。

CI確認時は、その時点のPR current HEAD SHAとrun `head_sha`が一致するrunだけを使用し、一致runがなければ `CI未実施` とする。
