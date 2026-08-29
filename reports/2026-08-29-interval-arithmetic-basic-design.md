# 区間演算 基本設計 作業レポート

## Metadata

- Repository: `ssaattww/Devo6.Interval`
- Pull Request: `#2`
- Base branch: `main`
- Working branch: `docs/basic-interval-arithmetic-design`
- Base HEAD: `9e5a4840d4f83d0f9b215bd3394f3d4bb60453b7`
- Technical design HEAD: `04d68c02c3b000610449b509f6d0a1ea88f9061c`
- Date: 2026-08-29
- Verification capability: `remote_ci_only`

## Purpose

これまでの会話で整理した区間演算ライブラリの設計方針を、Devo6.Interval の基本設計としてリポジトリへ保存する。

## Scope

- 区間演算ライブラリの目的と設計原則を文書化する。
- IEEE 1788.1-2017 を意味論の基準とする方針を文書化する。
- Rust `inari` を主要な参照実装とする方針を文書化する。
- 外部表現 `[Lower, Upper]` と内部表現 `[-Lower, Upper]` の分離を文書化する。
- 外向き丸めと negated-lower representation の関係を文書化する。
- SIMD / AVX-512 embedded rounding の基本方針を文書化する。
- 加算、減算、符号反転、乗算、除算の基本方針を文書化する。
- Empty / Entire / NaN / NaI / signed zero / Infinity の扱いを基本設計レベルで整理する。
- 依存性問題の性質と、基本 `Interval` 型の責務範囲を文書化する。
- managed / native backend の境界と未決事項を整理する。

## Non-Goals

- C# 実装コードの追加。
- project / solution ファイルの追加。
- CI workflow の追加。
- TDD 実施。
- AVX-512 / AVX2 / ARM64 の具体的命令列確定。
- 超越関数アルゴリズムの確定。
- decorated interval / NaI の実装。
- managed 実装と native backend の最終選定。

## Authoritative Requirements

### User instruction

- ここまでの区間演算に関する会話を設計書へ起こす。
- 現在の内容は基本設計として扱う。
- 対象リポジトリ等は project instruction に従う。

### Project / Skill instruction

- GitHub リポジトリの参照・更新、PR 作成・コメントは GitHub connector を使用する。
- 変更はレビュー可能な論理単位で commit / push する。
- 完了時に詳細 report を repository へ保存する。
- PR に簡易 report をコメントする。
- merge は行わない。
- documentation-only 作業であるため TDD は適用しない。

## Inspected Evidence

- `tasks/tasks-status.md`: 空。既存タスクとの競合なし。
- `AGENTS.md`: 空。
- `.codex/`: instruction ファイルなし。
- `doc/Design/basic/`: `.gitkeep` のみで、既存の基本設計なし。
- `.github/workflows/`: `.gitkeep` のみで、実行可能な CI workflow なし。
- `reports/2026-08-29-repository-initialization.md`: repository 初期化時の方針を確認。
- `reports/2026-08-29-repository-initialization-handoff.yaml`: report / handoff の既存保存形式を確認。
- `unageek/inari` の現行 `src/interval.rs`: 非空区間を `[-a; b]`、Empty を `[NaN; NaN]` として SIMD 表現する実装を確認。
- IEEE 1788.1-2017: binary64 を対象とする simplified interval arithmetic standard であることを確認。

## Changed Files

### `doc/Design/basic/IntervalArithmetic.md`

新規基本設計書を追加した。

主な内容:

1. `double` を初期端点型とする。
2. IEEE 1788.1-2017 を意味論の基準とする。
3. `inari` を主要な実装参照とする。
4. 外部 `[Lower, Upper]` / 内部 `[-Lower, Upper]` を採用する。
5. Lower は getter 時に符号を戻し、演算中は内部表現を維持する。
6. 下限は下向き、上限は上向きの外向き丸めを必須とする。
7. `roundDown(x) = -roundUp(-x)` を利用し、両レーンを上向き丸めで扱う設計を採る。
8. SIMD レーン配置と AVX-512 embedded rounding の利用方針を整理する。
9. 基本四則演算の実装方針を整理する。
10. Empty / Entire / NaN / Infinity / signed zero の基本的な意味を整理する。
11. bare `Interval` に依存性情報や NaI / decoration を持たせない。
12. managed / native の選択を公開 API から隠蔽する。
13. 未確定事項を詳細設計へ明示的に持ち越す。

## Intentionally Untouched

- `doc/Design/README.md`: 現在空であり、今回の依頼は基本設計本体の作成であるため変更しない。
- `tasks/*`: 対応する既存タスクがなく、タスク登録の依頼もないため変更しない。
- `.github/workflows/*`: executable / test target がなく、今回が documentation-only 作業であるため変更しない。
- `src/*`, `tests/*`: 実装を開始していないため変更しない。

## Validation

### Repository diff

`main` の `9e5a4840d4f83d0f9b215bd3394f3d4bb60453b7` と technical design HEAD `04d68c02c3b000610449b509f6d0a1ea88f9061c` を GitHub compare で確認した。

結果:

- Status: `ahead`
- Ahead by: `1`
- Changed files: `1`
- Added: `doc/Design/basic/IntervalArithmetic.md`
- Additions: `447`
- Deletions: `0`

### Content verification

GitHub connector で branch 上の `doc/Design/basic/IntervalArithmetic.md` を再取得し、作成内容が保存されていることを確認した。

### Build / Test

実行対象なし。

repository には project / solution / executable test target がまだ存在しないため、build / unit test / E2E test は `not_applicable` とする。

### CI

technical design HEAD `04d68c02c3b000610449b509f6d0a1ea88f9061c` に紐づく workflow run を確認したが、該当 run は存在しなかった。

`.github/workflows/` に workflow 自体が存在しないため、別 SHA の run を代用せず、CI は未実装 / not applicable と記録する。

## Failure Diagnostics

テスト workflow が存在せず、今回の変更も documentation-only であるため、失敗診断 artifact の対象となるテスト実行はない。

## Remaining Risks / Detailed-Design Follow-ups

- `Interval` の具体的な C# API / exception contract。
- Empty と signed zero の最終的な内部正規化規則。
- `Vector128<double>` 等の具体的な物理表現。
- AVX-512 非対応環境での方向付き丸め方式。
- 乗算 / 除算の最終 SIMD アルゴリズム。
- `sqrt`、FMA、超越関数の実装方式。
- managed-only / native backend の最終選定。
- batch API の形。
- IEEE 1788.1 conformance test の導入方法。

## Merge Boundary

PR #2 を作成済み。merge は利用者が行うため、この worker は merge しない。
