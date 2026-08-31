# PR #5 2回目指摘対応報告

## 1. 対象

- Repository: `ssaattww/Devo6.Interval`
- PR: #5 `docs: define implementation phases and task acceptance criteria`
- Mode: review follow-up / documentation fix
- Review source: `reports/2026-08-31-pr5-fix-verification.md`
- Fix-verification target HEAD: `57676dd738cc6c9ed930879aa873e43cc128d72d`
- Review-report-added HEAD at follow-up start: `c4c0c37e1c12a7d4f00f9685e5bf7d95d7cfafb4`
- Technical fix HEAD: `bd40f0e35030cc4a2f583569de2d67ffdc7e6e33`
- Branch: `docs/task-and-phase-plan`
- Base: `main`

## 2. 修正対象

fix verificationでopen/newとなった次の5 findingだけを修正した。

| Finding | Severity | 状態 |
|---|---|---|
| F-PR5-001 | High | addressed; reviewer re-verification required |
| F-PR5-007 | Medium | addressed; reviewer re-verification required |
| F-PR5-010 | Medium | addressed; reviewer re-verification required |
| F-PR5-011 | Medium | addressed; reviewer re-verification required |
| F-PR5-012 | Medium | addressed; reviewer re-verification required |

fix verificationでclosure済みの `F-PR5-002/003/004/005/006/008/009` は変更対象にしていない。

## 3. 変更内容

### F-PR5-001: Phase 4 preflightとRedを分離

`tasks/phases-status.md` と `tasks/tasks-status.md` で、preflightの責務を次に限定した。

- 対象design sectionのreview PASS
- diagnostic workflow/reference基盤の存在
- fixture metadata (`caseId/input/expected/reference`) の登録
- production targetを呼び出さないmetadata列挙・parse commandの成功

production behaviorの失敗確認は各subphase最初のsource taskへ移した。

- 4A: `P4A-000` metadata -> `P4A-001 Contains` Red
- 4B: `P4B-000` Pi metadata -> `P4B-001 IntervalConstants` Red
- 4C: `P4C-000` `Exp([0,0])=[1,1]` metadata -> `P4C-001 Exp` Red
- 4D: `P4D-000` reducer zero metadata -> `P4D-001 reducer` Red
- 4E: `P4E-000` `default(IntervalUnion2)` metadata -> `P4E-001 IntervalUnion2` Red

これによりpreflightで未実装production APIのGreenを要求せず、§44のTDD順序とpreflight依存関係を両立させた。

### F-PR5-007: binary v1 state/canonical payload固定

P4E preflightでversion 1を次に固定した。

```text
byte0 = 0x01
byte1 = 0x00 Normal / 0x01 Empty
byte2..9 = external Lower bits LE
byte10..17 = external Upper bits LE
```

- Normalは全nonempty intervalを表す。
- Empty payloadは16 byte all-zeroのみcanonical。nonzero payloadはreject。
- NormalのNaN endpoint / lower>upper / lower=+Infinity / upper=-Infinityはreject。
- lower=+0.0、upper=-0.0はdecode時にcanonical signed-zeroへ補正。
- unknown version/stateはreject。

さらに `tasks/fixtures/binary-v1-fixtures.md` へ4個のcanonical 18-byte列を固定した。

```text
[1,2] : 01 00 00 00 00 00 00 00 f0 3f 00 00 00 00 00 00 00 40
Zero  : 01 00 00 00 00 00 00 00 00 80 00 00 00 00 00 00 00 00
Entire: 01 00 00 00 00 00 00 00 f0 ff 00 00 00 00 00 00 f0 7f
Empty : 01 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
```

### F-PR5-010: elementary native backend実workload benchmark

`P4B-007` はP3のAdd/Sub/Mul/Div operation setを流用しないよう変更した。

- production採用予定の `ExpDown/Up`, `LogDown/Up`, `SinDown/Up`, `CosDown/Up` 等の実entrypointをfunction単位で測定する。
- interop + marshalling/copy + dispatch + native call + return conversionを含める。
- baselineは同じproduction adapter contractのmanaged endpoint backend。
- batch N=`32/256/4096`, metric=`median ns/endpoint`。
- N>=256の幾何平均ratio `<=0.95`、各workload `<=1.02`、allocation増加`0 B`をfunction単位採用条件とする。
- function単位で条件未達ならそのfunctionをnative dispatchへ登録しない。

### F-PR5-011: Acos decreasing endpoint rule復元

`P4C-004`へ次を固定した。

```text
clip後 [l,u]
Acos(X) = [AcosDown(u), AcosUp(l)]
```

`Acos([0,1])=[AcosDown(1),AcosUp(0)]` をcaseId付きfixtureとし、expected endpoint bitsをMPFR RNDD/RNDUから固定する。

### F-PR5-012: exact text round-trip formatを必須化

P4E preflightで`R`を必須persistent/round-trip formatとして固定した。

- finite endpoint: exact C99-style hexadecimal binary literal
- Empty: `Empty`
- Entire: `Entire`
- unbounded endpoint: `-Infinity/+Infinity`
- `R`をReject/N/Aにしない

`P4E-009/P4E-012`ではNormal bounded、Zero、Empty、Entire、lower-unbounded、upper-unbounded、min-subnormal singletonのformat -> ParseExact/TryParseExact canonical bitwise round-tripを必須化した。x64/ARM64でtextも一致させる。

## 4. Validation

### Repository compare

GitHub compare `c4c0c37e1c12a7d4f00f9685e5bf7d95d7cfafb4..bd40f0e35030cc4a2f583569de2d67ffdc7e6e33`:

- ahead_by: 4
- behind_by: 0
- modified: `tasks/tasks-status.md`
- modified: `tasks/phases-status.md`
- added: `tasks/fixtures/binary-v1-fixtures.md`
- `src/**`, `tests/**`, `.github/workflows/**`, `doc/Design/**`: changeなし

### Content verification

GitHub connectorでbranch上のtask文書を再取得し、次を確認した。

- P4A～P4E preflightはproduction API成功を要求せずmetadata parseだけを要求する。
- 各preflight fixtureが当該subphase最初のsource taskと一致する。
- P4B-007がelementary endpoint production adapter pathを測定しAdd/Sub/Mul/Div代用を禁止する。
- P4C-004がAcos decreasing ruleを明示する。
- P4E-000/P4E-009が必須`R` formatを持つ。
- P4E-000/P4E-010がbinary state byteとcanonicalization/reject matrixを固定する。

### Build/Test

本PRはdocumentation-onlyで、repositoryにはまだ実行可能production/test projectがないためbuild/testはnot applicable。

### CI

Technical fix HEAD `bd40f0e35030cc4a2f583569de2d67ffdc7e6e33` に一致するpull_request workflow runは0件。

- CI: `CI未実施`
- 別SHA run代用: なし

最終report/handoff commit後はPR current HEADを再取得し、そのHEADに一致するrunだけを最終CI証拠として確認する。

## 5. Remaining state

- 初回review FAILおよびfix-verification FAILは履歴として変更しない。
- 本workerは上記5 findingを `addressed` と判断しただけで、review closureは行わない。
- 同じnormal reviewerによる再fix-verificationが必要。
- `P0-001`: 進行中。
- Phase 1以降: 未着手。
- mergeは実施しない。
