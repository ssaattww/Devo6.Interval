# PR #5 初回レビュー報告

## 1. レビュー対象

- Repository: `ssaattww/Devo6.Interval`
- PR: #5 `docs: define implementation phases and task acceptance criteria`
- Review mode: `initial_review`
- Base: `main` (`f71466de50e6eca4942b614b253ecbc96464fa17`)
- Reviewed implementation HEAD: `b20fa7d7c31326770fac925a4e0c1b35cf72de6b`
- Changed files at reviewed HEAD:
  - `tasks/phases-status.md`
  - `tasks/tasks-status.md`
  - `reports/2026-08-30-task-and-phase-plan.md`
  - `reports/2026-08-30-task-and-phase-plan-handoff.yaml`
- Authoritative design: `doc/Design/detail/IntervalArithmetic.md` 設計版5
- Reviewer: current ChatGPT normal-review chat
- Reviewer continuity: initial review; this chat did not implement PR #5 or implement review fixes

今回の追加要求として、受け入れ条件は可能な限り具体化し、実装者が詳細設計本文を再解釈しなくても、タスク本文だけから合否を判定できることを重視した。

## 2. 検証状態

- PR current HEAD確認時点: `b20fa7d7c31326770fac925a4e0c1b35cf72de6b`
- 同HEADに一致するGitHub Actions workflow run: **0件**
- CI判定: **未実施**
- 別SHAのrun代用: なし
- `.github/workflows` は reviewed HEAD 時点で `.gitkeep` のみ
- production/test projectは未作成で、本PRはdocumentation-only

したがってsource build/testは対象外とし、設計版5、PR差分、task/phase/report/handoff間の整合性をレビュー証拠とした。

## 3. Verdict

**FAIL — 修正必須 finding あり。**

フェーズ順・TDD順・主要な数値仕様の転記は概ね詳細設計版5に沿っている。一方、今回の主成果物である受け入れ条件には、設計上の開始条件や必須診断情報が脱落している箇所と、「fixtureを置く」「改善を示す」だけで期待結果・判定閾値が確定していない箇所がある。このままでは、詳細設計に反する実装でもtask上は完了扱いにできる。

## 4. Findings

### F-PR5-001 — High — Phase 4の実装開始条件がtask graphに存在しない

**Origin:** introduced_by_change  
**Location:** `tasks/tasks-status.md` の `P4A-001`, `P4B-001`, `P4C-001`, `P4D-001`, `P4E-001` および依存関係

**Description**

詳細設計 §52.2 は、Phase 4A以降のsource実装開始前に少なくとも次を要求している。

- Phase 1～3の必要成果物完了
- Phase 2 basic API freeze完了
- **対象Phase 4 sectionのreview完了**
- test/conformance/reference基盤との統合
- diagnostic artifact workflowの存在

しかしtask graphでは、例えば `P4A-001` は `P3-006` のみ、`P4B-001` は `P4A-008` のみを依存先としており、「対象sectionのreview完了」を開始前に確認するtask/gateがない。各Phase末尾のAPI/conformance closeは実装後なので、開始前reviewの代わりにならない。

**Impact**

詳細設計が未reviewのままPhase 4 source TDDを開始しても、現在のtask依存関係上は違反にならない。§52.2の実装開始条件をtask管理が保証できない。

**Required action**

各Phase 4先頭にpreflight taskを追加するか、先頭source taskの開始条件としてreview gateを明示する。

**具体例**

`P4A-000 Phase 4A implementation preflight` を追加する場合、受け入れ条件は例えば次のように判定可能にする。

- [ ] `doc/Design/detail/IntervalArithmetic.md` のreview対象full SHAをreportに記録する。
- [ ] §19～24およびPhase 4A API候補に対するreview verdictが`pass`または修正後closure済みである。
- [ ] review report pathとreviewed HEADがtask reportから追跡できる。
- [ ] `P1-001`で追加されたdiagnostic artifact workflowがrepositoryに存在する。
- [ ] Phase 4A fixtureが既存conformance/reference harnessから実行できることを1件のsmoke case（例: `Entire.Contains(+Infinity) == false`）で確認する。
- [ ] 上記を満たすまで `P4A-001` のRed commitを作成しない。

同じgateを4B～4Eにも適用する。

---

### F-PR5-002 — Medium — `P1-001` のdiagnostic artifact条件が§16.4を満たしていない

**Origin:** introduced_by_change  
**Location:** `tasks/tasks-status.md` `P1-001`, `reports/2026-08-30-task-and-phase-plan.md` §4

**Description**

詳細設計 §16.4 は成功/失敗にかかわらず、基本artifactに加えて少なくとも次を保存する規範を持つ。

- mismatch input
- exact result
- Devo6 result
- inari/kv/MPFR result または N/A
- expected-difference reason

`P1-001` は test result/stdout/stderr/diagnostic log、runtime/architecture/reference-lock/conformance/canonical corpus、caseId/inputまでは要求するが、上記の数値比較情報をworkflowの受け入れ条件として列挙していない。共通完了条件は「原因調査可能」としているだけで、workflow自体が各フィールドをartifact化することを保証しない。

**Impact**

数値不一致のCI failureで、入力は分かってもexact/reference/実装結果のどこがずれたかartifactだけでは判断できないworkflowが `P1-001` 完了になり得る。

**Required action**

`P1-001` に§16.4の必須フィールドをすべて列挙し、未導入referenceは明示的に`N/A`を保存することまで受け入れ条件にする。

**具体例**

失敗case `mul-scaled-tie-001` のdiagnostic entryが最低限次を持つことをfixtureで確認する。

```text
caseId: mul-scaled-tie-001
inputBits: x=..., y=...
branch: MultiplyUp/scaled/t==s/s2>0
exactResult: <exact rational or encoded exact value>
devo6Bits: ...
reference:
  exactOracle: ...
  kv: ... or N/A
  inari: ... or N/A
  mpfr: N/A
expectedDifferenceReason: null
```

---

### F-PR5-003 — Medium — `P1-002` が詳細設計の内部canonical stateを受け入れ条件にしていない

**Origin:** coverage_miss  
**Location:** `tasks/tasks-status.md` `P1-002`

**Description**

詳細設計 §1/§6 はPhase 1の規範として、通常区間の内部表現 `[-Lower, Upper]`、Emptyのcanonical qNaN 2 lane、片側NaN禁止、raw constructor呼出側でcanonical state保証、layout非公開を定義している。

`P1-002` は公開endpoint、constructor、equality/hash等は具体化しているが、内部表現については「public APIからprivate layoutやNaN payloadを観測できない」だけである。そのため `[Lower,Upper]` をそのまま持つ実装や、Emptyを片側NaNだけで表す実装でも他の公開fixtureを通せばtask上は完了できる。

またPhase 1 API対象に含まれるdiagnostic `ToString()`について、Phase 1内の明示的受け入れ条件がない。

**Impact**

Phase 3 SIMDのnegated-lower layout前提とずれ、後から内部表現を作り直す可能性がある。Phase 1公開surfaceも設計どおり揃ったか判定できない。

**Required action**

`P1-002` にinternal invariantとdiagnostic `ToString`存在確認を追加する。

**具体例**

- [ ] test hook/internal assertionで `[1,2]` が内部 `[-1,2]`、`[-2,-1]` が内部 `[+2,-1]` として保持されることを確認する。
- [ ] Emptyは両laneがcanonical qNaNであり、片側だけNaNのraw stateを生成できない/validationで失敗する。
- [ ] raw constructor経由でZero/Entire/Empty/nonemptyを生成した後もpublic invariantが成立する。
- [ ] `ToString()` はEmpty/Entire/finite intervalで例外なくdiagnostic表現を返し、永続化formatの契約とはしない。

---

### F-PR5-004 — Medium — branch fixtureの「期待動作」が受け入れ条件に書かれていない

**Origin:** introduced_by_change  
**Location:** `tasks/tasks-status.md` `P1-006`, `P1-008`（同種の記述を含むtask全般）

**Description**

`P1-006` はscaled multiplyの `t<s`, `t>s`, `t==s && s2>0`, `t==s && s2<0`, exactを「固定witnessで検証」としている。`P1-008` もdivisionの `r==xn && r2>0`, `r==xn && r2<0`, `r<xn`, `r>xn`, exactを「固定witnessで検証」としている。

しかし各条件で `Up`/`Down` が `NextUp`/`NextDown`/無補正のどれになるべきかをtask本文に書いていない。今回の成果物は受け入れ条件なので、「caseが存在する」だけでなく期待結果まで固定する必要がある。

**Impact**

例えばdivision residualの符号条件を逆実装した場合でも、「各caseのfixtureが存在する」という文言自体は満たせる。詳細設計を再読しないとtask単体で合否判定できない。

**Required action**

branch tableをtask受け入れ条件へ転記し、各fixed witnessにexpected branch/expected bitsを持たせる。

**具体例: division**

- `DivideUp`: `r < xn` または `r == xn && r2 < 0` -> `NextUp(q)`、それ以外 -> `q`
- `DivideDown`: `r > xn` または `r == xn && r2 > 0` -> `NextDown(q)`、それ以外 -> `q`
- exact case -> Up/Downとも`q`

**具体例: multiply scaled path**

- Up: `t<s` または `t==s && s2>0` -> `NextUp(product)`
- Down: `t>s` または `t==s && s2<0` -> `NextDown(product)`
- `t==s && s2==0` -> 無補正

fixtureはcaseId、input bits、expected output bitsまで固定する。

---

### F-PR5-005 — Medium — API usability / benchmark gateが客観判定できない

**Origin:** introduced_by_change  
**Location:** `tasks/tasks-status.md` `P1-012`, `P2-001`, `P3-004`, `P3-006`; `tasks/phases-status.md` Phase 2/3

**Description**

次の語が合否基準として未定義である。

- `representative sample`
- `過度なfactory呼出し`
- `operator中心で自然に記述`
- `realistic workload`
- `benchmark改善が測定可能`
- `性能改善を示せないkernelは採用しない`

特にPhase 3は、測定ノイズ内の0.1%差でも「改善」と主張でき、production dispatch採用gateがレビュー者ごとに変わる。

**Impact**

同じ実装・同じbenchmark結果でも、担当者ごとに完了/未完了判定が変わる。

**Required action**

API sampleの具体シナリオと、benchmarkのworkload・baseline・統計的判定ルールをtask内または事前に固定するdecision recordへ明記する。

**具体例: API sample**

最低3scenarioを固定する。

1. `new Interval(1, 2) + Interval.Point(3)` -> `[4,5]`
2. `new Interval(1, 2) / new Interval(-1, 1)` -> `Entire`、例外なし
3. `new Interval(2, 1)` -> `ArgumentException`、数学的Emptyとは別経路

このsampleがpublic APIだけで記述でき、backend型/internal factoryが露出しないことを確認する。

**具体例: benchmark gate**

設計段階で特定の割合を勝手に決めるのではなく、`P3-006`開始前に例えば次を固定する。

- BenchmarkDotNet job/runtime/CPU affinity
- scalar baselineとcandidateが同一input corpusを使用
- warmup/iteration条件
- allocation差
- 採用判定に使うmetricと閾値
- ノイズ判定規則

例として「median ratio <= 0.95 かつ95% CI上限 < 1.0」のような判定式を採用するなら、その式をtask/decision recordへ固定してからcandidateを評価する。

---

### F-PR5-006 — Medium — parser受け入れ条件が§40/§43の入力surfaceを網羅していない

**Origin:** coverage_miss  
**Location:** `tasks/tasks-status.md` `P4E-009`

**Description**

詳細設計 §40.1 はendpoint tokenとして decimal / integer / `±Infinity` / exact hexadecimal binary literalを候補としている。また§43はculture曖昧性排除、recursionなしをsecurity規範としている。

`P4E-009` は`Empty`/`Entire`/`[a,b]`/`[a]`とdecimal exact parsingは要求するが、一般のunbounded intervalを作る`[-Infinity,1]`・`[1,+Infinity]`、exact hexadecimal binary literal、culture曖昧性排除、recursionなしが受け入れ条件にない。

**Impact**

`Entire`以外のunbounded intervalやhex exact tokenを拒否するparserでもtask完了になり得る。culture依存parserもsecurity gateをすり抜けられる。

**Required action**

採用するendpoint grammarをP4E-009で列挙し、accepted/rejected fixtureを具体化する。詳細設計で最終判断対象になっているsyntax細部は、実装前decisionとして固定したうえでtaskへ反映する。

**具体例**

Accepted:

- `[-Infinity,1]` -> `[-Infinity,1]`
- `[1,+Infinity]` -> `[1,+Infinity]`
- `[0.1]` -> exact decimal 0.1 の `[RoundDown,RoundUp]`
- 採用hex syntaxの最小/最大subnormal・signed zero exact literal

Rejected:

- `[+Infinity,1]`
- `[1,-Infinity]`
- `[Infinity]`
- `[NaN,1]`
- locale依存の`[1,5]`を「1.5」の意味として解釈する入力

---

### F-PR5-007 — Medium — binary interchangeのwire contract/decoder順序が受け入れ条件に不足

**Origin:** coverage_miss  
**Location:** `tasks/tasks-status.md` `P4E-010`

**Description**

詳細設計 §41 はversion 1候補として18 byteの固定layoutとlittle-endian external endpoint bitsを定義している。§43はdecoderがlength/stateを先に検証することを要求する。

`P4E-010` は「version/state/external Lower bits/external Upper bitsのversioned format」とround-tripまでは要求するが、byte offset、little-endian、固定長、length-first validationが受け入れ条件にない。

**Impact**

big-endianや可変長layout、短い入力をendpoint decodeしてから失敗する実装でもtask本文上は合格可能で、設計とwire互換性がずれる。

**Required action**

version 1を採用する時点で、固定長/byte位置/endianness/reject orderをacceptanceへ明記する。

**具体例**

- [ ] version 1 encoded lengthは18 byteである。
- [ ] byte 0=version、byte 1=state、byte 2..9=external `Lower` bits LE、byte 10..17=external `Upper` bits LE。
- [ ] ZeroのLowerは外部`-0.0` bits、Upperは`+0.0` bitsとしてround-tripする。
- [ ] 17 byte/19 byte入力はendpoint解釈前にrejectする。
- [ ] unknown version/state、NaN endpoint、reversed endpointをrejectし、内部片側NaN stateを生成しない。

---

### F-PR5-008 — Medium — Phase 1のhot-path / NativeAOT / trimming条件がtask・phase gateから脱落している

**Origin:** coverage_miss  
**Location:** `tasks/phases-status.md` Phase 1, `tasks/tasks-status.md` Phase 1 tasks

**Description**

詳細設計 §47 はPhase 1から次を要求している。

- hot pathにvirtual/interface/delegate dispatchなし
- reflection/runtime codegen/dynamic assembly/native resolverなし
- NativeAOT/trimmingを妨げない
- raw constructorでvalidation重複回避

現状Phase 1の受け入れ条件はallocation 0、BCL以外runtime dependencyなし、global rounding-mode変更なし、production hot path `BigInteger`なし等は拾っているが、上記項目はtask/phaseの完了条件にない。

**Impact**

Phase 1完了後にAOT不適合やhot-path dispatchが判明し、Phase 2 API freeze後に内部構造を大きく変更するリスクがある。

**Required action**

Phase 1完了gateまたは該当taskに検証可能な条件を追加する。

**具体例**

- [ ] basic `+/-/*//` hot pathがinterface/delegate/virtual dispatchを含まないことをdisassemblyまたは明示的code review checklistで確認する。
- [ ] production assemblyがreflection/runtime codegen/dynamic assembly/native resolverを参照しない。
- [ ] sampleをNativeAOT publishし、基本四則演算を実行するsmoke testが成功する。
- [ ] trimming有効publishで基本APIが欠落しない。
- [ ] internal raw result constructionがpublic constructor validationを重複実行しないことをbenchmark/disassemblyまたはfocused testで確認する。

---

### F-PR5-009 — Low — `Decoration` のpublic numeric valueが固定されていない

**Origin:** coverage_miss  
**Location:** `tasks/tasks-status.md` `P4E-006`

**Description**

詳細設計 §38.1 は `Decoration : byte` として `Ill=0, Trv=4, Def=8, Dac=12, Com=16` を定義している。`P4E-006` は品質順 `Ill < Trv < Def < Dac < Com` のみを要求するため、`0,1,2,3,4` でもtask上は合格できる。

**Impact**

public enumの数値値が設計と異なり、将来のserialization/interchangeやAPI compatibilityに影響する。

**Required action**

exact underlying type/valueを受け入れ条件に追加する。

**具体例**

```text
(typeof(Decoration).GetEnumUnderlyingType() == typeof(byte))
(byte)Decoration.Ill == 0
(byte)Decoration.Trv == 4
(byte)Decoration.Def == 8
(byte)Decoration.Dac == 12
(byte)Decoration.Com == 16
```

---

### F-PR5-010 — Medium — native elementary backend採用時の§18 qualificationがtask化されていない

**Origin:** coverage_miss  
**Location:** `tasks/tasks-status.md` `P4B-007`

**Description**

`P4B-007` はendpoint backend候補のcorrectness、supported function/platform、distribution/licenseを要求するが、詳細設計 §18 がnative backend採用時に要求する次を明示していない。

- interop/copy/dispatch込みの実workload改善
- ABI/thread safety
- NativeAOT/trimming影響
- x64/ARM64/deployment配布可能性
- license/NOTICE
- public `Interval`非変更

**Impact**

correctnessだけを満たすnative backendを採用し、配布/AOT/thread safety/interop overhead問題が後から判明する可能性がある。

**Required action**

`P4B-007` を「managedのみならN/A、native採用時は§18 gate必須」というconditional acceptanceにする。

**具体例**

- [ ] managed-only採用の場合 `native_backend_gate=N/A (reason=...)` をreportへ記録する。
- [ ] native candidateを採用する場合、interop/copy/dispatch込みbenchmarkが事前固定した採用閾値を満たす。
- [ ] x64/ARM64配布asset、ABI、concurrent call、NativeAOT、trimming smoke testが成功する。
- [ ] license/NOTICE/binary redistribution条件をrepositoryに保存する。
- [ ] native有無でpublic `Interval` API baselineが変化しない。

## 5. Coverage dispositions

| Criterion | Disposition | Evidence |
|---|---|---|
| requirement/design conformance | checked_finding | F-PR5-001, 002, 003, 006, 007, 008, 009, 010 |
| correctness/edge-case acceptance | checked_finding | F-PR5-004 |
| scope discipline | checked_no_finding | reviewed HEADの変更はtasks/reports 4ファイルのみ |
| task dependency/order | checked_finding | F-PR5-001 |
| public API acceptance | checked_finding | F-PR5-003, 005, 009 |
| configuration/workflow | checked_finding | F-PR5-002; workflow未実装方針自体は§16.1と一致 |
| error/failure diagnostics | checked_finding | F-PR5-002 |
| security/resource handling | checked_finding | F-PR5-006, 007 |
| tests/validation adequacy | checked_finding | F-PR5-004, 005, 008 |
| current-HEAD CI evidence | held | reviewed HEADに一致するrun 0件。CI未実施 |
| reports/handoff accuracy | checked_no_finding | reviewed HEADとCI未実施の説明はPRコメントと整合 |
| regression/maintainability | checked_finding | F-PR5-001, 003, 008, 010 |

## 6. Held / unexplored

### Held

- Exact-head CI: reviewed HEAD `b20fa7d7c31326770fac925a4e0c1b35cf72de6b` にworkflow runが存在しないため未実施。documentation-onlyかつworkflow未作成であることは設計 §16.1と整合するため、findingではなくheld evidenceとする。

### Unexplored

- なし。変更4ファイル、詳細設計版5の対応section、PR metadata/comment、workflow有無を確認した。

## 7. Fix verificationで必要な証拠

各findingについて、次をfix verification前に揃える。

| Finding | Required action | Production/document path | Actual composition fixture/evidence | Focused evidence |
|---|---|---|---|---|
| F-PR5-001 | Phase 4 preflight/start gate追加 | `tasks/tasks-status.md` / `tasks/phases-status.md` | 4A～4E各先頭taskの依存/開始条件 | 設計§52.2との対応表 |
| F-PR5-002 | §16.4 artifact field列挙 | `P1-001` | failure diagnostic example | §16.4全項目チェック |
| F-PR5-003 | internal canonical state/ToString追加 | `P1-002` | normal/Empty/raw state例 | §6/§5.10対応 |
| F-PR5-004 | branch expected action明記 | `P1-006`, `P1-008` 等 | branch table + witness caseId | §9.3/§10.4対応 |
| F-PR5-005 | sample/benchmark判定固定 | `P1-012`, `P2-001`, `P3-006` | sample 3scenario + benchmark decision rule | 合否が一意になる記録 |
| F-PR5-006 | parser grammar/security補完 | `P4E-009` | accepted/rejected input matrix | §40/§43対応 |
| F-PR5-007 | binary layout/reject order補完 | `P4E-010` | 18-byte/LE/invalid-length fixture | §41/§43対応 |
| F-PR5-008 | AOT/trimming/hot-path gate追加 | Phase 1 gate/task | AOT/trimming/disassembly等の検証方法 | §47対応 |
| F-PR5-009 | Decoration exact values追加 | `P4E-006` | enum underlying/value assertions | §38.1対応 |
| F-PR5-010 | conditional native gate追加 | `P4B-007` | managed N/A またはnative qualification matrix | §18対応 |

## 8. Remaining risks

- 受け入れ条件が詳細設計への参照だけで期待値を省略すると、実装chatごとに解釈差が出る。
- 性能gateの判定式を後付けすると、candidate結果を見てから採用基準を動かせる。
- Phase 4開始前review gateがtask graphにない状態で実装を進めると、設計変更による手戻りが大きくなる。

## 9. Next action / handoff

- Verdict: `fail`
- Next action: implementation/documentation fix
- Reviewed HEAD: `b20fa7d7c31326770fac925a4e0c1b35cf72de6b`
- 修正対象: F-PR5-001 ～ F-PR5-010
- 同じnormal review chatでfix verificationを行う。
- 修正後は新しいPR current HEADを取得し、そのHEADに対するdiffと各finding completeness matrixを確認する。
- CIは新HEADとrun `head_sha`が一致するものだけを採用し、一致runがなければCI未実施とする。
- mergeは行わない。
