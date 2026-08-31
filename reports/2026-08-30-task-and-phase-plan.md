# タスク・フェーズ計画整備レポート

## 1. 実施概要

- 対象リポジトリ: `ssaattww/Devo6.Interval`
- Base branch: `main`
- 作業branch: `docs/task-and-phase-plan`
- 作業開始時base HEAD: `f71466de50e6eca4942b614b253ecbc96464fa17`
- 計画文書確定時HEAD: `0ba19efe76bf4ceec264d49270065bf2af651baa`
- 基準設計: `doc/Design/detail/IntervalArithmetic.md` 設計版5
- 対象範囲: フェーズ一覧、タスク一覧、具体的受け入れ条件の作成
- 対象外: production source実装、test実装、CI workflow実装、既存詳細設計の仕様変更

詳細設計版5を実装可能な作業単位へ分解し、`tasks/phases-status.md` と `tasks/tasks-status.md` を作成した。

## 2. 参照した設計規範

主に次の詳細設計をタスク化の基準とした。

- §3: Phase 0 / 1 / 2 / 3 / 4A / 4B / 4C / 4D / 4E の開発順序と各Phase対象
- §13～15: exact-rational oracle、reference corpus、conformance、threshold/overflow fixture
- §16: x64/ARM64 CI、diagnostic artifact、exact-head CI
- §17～18: SIMD capability matrix、production採用gate、native backend判断
- §19～43: Phase 4の集合・関係・数学関数・union・decoration・I/O・split仕様
- §44～45: TDD順序、review-regression fixture、property/differential test
- §46: API確定・Phase完了gate
- §47～49: performance、thread safety、結果同等性、third-party license
- §50: 未確定事項
- §51: `F-PR3-010`～`F-PR3-017` のfix verification状態
- §52: Phase 1実装開始順序、Phase 4実装開始条件

## 3. 変更内容

### 3.1 `tasks/phases-status.md`

Phase単位で次を定義した。

- Phase 0: 詳細設計・検証方針確定
- Phase 1: managed scalar 四則演算パイロット
- Phase 2: 基本 `Interval` API確定
- Phase 3: SIMD backend
- Phase 4A: 集合・関係・数値的属性・整数値関数
- Phase 4B: 代数関数・区間定数
- Phase 4C: 単調な初等関数
- Phase 4D: 周期・特異点・多変数関数
- Phase 4E: 非連結結果・decorated interval・I/O・分割

各Phaseには目的、依存関係、対象または主成果物、判定可能な完了条件を記載した。

Phase完了条件は、単なる「実装済み」ではなく、以下を確認できる形にした。

- deterministic fixture成功
- exact oracle / MPFR / pinned referenceとの一致または承認済み差異
- Linux x64 / ARM64 canonical result一致
- 複数backend間のcanonical endpoint bitwise一致
- failure artifactによる原因追跡可能性
- public API baseline更新
- breaking change記録
- PR current HEAD SHAとworkflow run `head_sha`のexact match

### 3.2 `tasks/tasks-status.md`

詳細設計の実装順序を61タスクへ分解した。

内訳:

- Phase 0: 2タスク
- Phase 1: 12タスク
- Phase 2: 3タスク
- Phase 3: 6タスク
- Phase 4A: 8タスク
- Phase 4B: 7タスク
- Phase 4C: 5タスク
- Phase 4D: 6タスク
- Phase 4E: 12タスク

各タスクには設計参照と具体的な受け入れ条件を付与した。

代表例:

- `P1-001`: solution/project、Linux x64/ARM64 CI、diagnostic artifact基盤
- `P1-003`: exact-rational oracle、finite overflow判定、threshold corpus
- `P1-006`: multiplicationのFMA residual/scaled path境界と固定witness
- `P1-008`: divisionのsmall numerator/large denominator/residual tie fixture
- `P3-001`: ISA/FMA独立判定とscalar/SIMD differential基盤
- `P4A-004`: `IntervalOverlap` 16状態とinverse consistency
- `P4B-007`: MPFR RNDD/RNDU corpusとelementary backend qualification
- `P4D-004`: `Atan2` negative-x branch cut 6class matrix
- `P4E-002`: extended divisionのone-sided/strict zero-crossing matrixとConvexHull property
- `P4E-006`: Decoration result-state cap
- `P4E-009`: exact decimal parserとresource limit

### 3.3 TDD・commit運用

source実装を含む全タスクの共通完了条件に次を明記した。

1. 失敗testを先に追加する。
2. 仕様を理由にRedになることを確認する。
3. production implementationを追加する。
4. Red/Greenをレビュー可能な小さい論理単位でcommit/pushする。

infra/documentationのみのタスクにはTDDを要求しない。

## 4. Diagnostic artifact workflowの確認

作業開始時点で `.github/workflows` には `.gitkeep` のみで、実行可能なworkflowは存在しなかった。

一方、詳細設計 §16.1 は、現状repositoryに実行可能projectがないため設計PRではworkflowを追加せず、Phase 1でproject/testを追加する最初のPRにdiagnostic artifact workflowを同時追加すると明記している。

このため本作業ではworkflowを追加していない。
代わりに `P1-001` の必須受け入れ条件として、最初の実装PR内で以下をartifact化することを固定した。

- test result
- stdout
- stderr
- diagnostic log
- runtime / OS / architecture / CPU features
- reference-lock
- conformance summary
- canonical result corpus
- caseId / mismatchを原因調査できるdiagnostic情報

詳細設計 §16.4 に定義された追加診断情報についても、各後続タスクの共通完了条件で追跡可能性を要求している。

## 5. Status方針

現時点では次のstatusとした。

- Phase 0: `進行中`
- `P0-001 詳細設計版5 fix verification`: `進行中`
- `P0-002 フェーズ・タスク管理基盤作成`: `進行中`
- Phase 1以降: `未着手`

理由は、詳細設計版5自身が `fix verification required` であり、`F-PR3-010`～`F-PR3-017` が addressed ではあるもののverification pendingであるためである。

本PRがmergeされた事実だけでPhase 0全体を完了扱いにはしない。
`P0-002`についても、作業branch上で作成済みだが、repositoryの確定状態になるまでは進行中を維持する。

## 6. 検証

実施した検証:

- GitHub connectorから基本設計・詳細設計・既存tasks配下を取得し、対象機能と開発順序を照合した。
- `tasks/phases-status.md` をbranchから再取得し、Phase 0～4E、依存関係、完了条件が保存されていることを確認した。
- `tasks/tasks-status.md` をbranchから再取得し、61タスク、共通完了条件、Phase 1開始順序、Phase 4 TDD順序、review-regression項目が保存されていることを確認した。
- extended reciprocalの受け入れ条件について、設計 §35 の関係に合わせ `ReciprocalToUnion(value).ConvexHull == Reciprocal(value)` へ訂正した。
- 作業branch HEAD `0ba19efe76bf4ceec264d49270065bf2af651baa` をGitHub上で確認した。

未実施:

- source build/test: production/test projectがまだ存在せず、本変更はdocumentationのみのため対象外。
- CI: repositoryに実行可能workflowが存在しないため、この時点では未実施。

PR作成後は、その時点のcurrent HEAD SHAと完全一致するworkflow runのみを確認対象とする。matching runがなければCI未実施として報告する。

## 7. Commit

計画文書に関するcommit:

1. `c5393812294d490445569a03047a0ec45feb9757` — `docs: define development phases and acceptance gates`
2. `2921e1fd8e582c65b441e70ce0558a2427e93f2d` — `docs: define implementation tasks and acceptance criteria`
3. `0ba19efe76bf4ceec264d49270065bf2af651baa` — `docs: correct reciprocal union acceptance relation`

report/handoffの保存は管理成果物として後続commitに含める。

## 8. 残件・開始条件

Phase 1 source実装前の残件は `P0-001` である。

- 詳細設計版5に対するfix verificationをimmutable HEAD単位で完了する。
- `F-PR3-010`～`F-PR3-017`を含む未解決findingがないことを確認する。

Phase 1開始後の最初の作業は `P1-001` とし、project/testとdiagnostic artifact workflowを同じPR内で整備する。

## 9. 結論

詳細設計版5のPhase構成と実装順序を、9フェーズ・61タスクへ具体化した。
各タスクはtest、oracle/reference、architecture/backend一致、artifact、API baseline等で受け入れ可否を判断できる。

本作業では設計仕様そのものを変更していない。
また、repositoryの現状と詳細設計 §16.1に従い、実行可能projectがない段階で空のCI workflowを先行追加していない。
