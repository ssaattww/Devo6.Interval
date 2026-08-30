# フェーズ一覧

## 1. 文書情報

- 対象: `ssaattww/Devo6.Interval`
- 基準設計: `doc/Design/detail/IntervalArithmetic.md` 設計版5
- 詳細タスク: `tasks/tasks-status.md`
- 状態値: `未着手` / `進行中` / `Blocked` / `完了`

本書は開発フェーズ単位の目的、依存関係、開始条件、完了条件を管理する。
個別実装の期待入力・期待結果・例外・fixture・artifactは `tasks/tasks-status.md` を正とする。

## 2. 判定規則

フェーズまたはタスクを `完了` とするには、該当チェック項目について次を満たす。

1. 「成功する」「一致する」は、対象test/commandの終了コード0に加えて、本文で指定した期待値またはcanonical endpoint bit列が一致することを意味する。
2. 数値比較は、NaN payloadをpublic contractにしない箇所を除き、`BitConverter.DoubleToInt64Bits`相当のbinary64 bit列で判定する。
3. `N/A`を許す条件は本文で明示した項目だけとし、reportへ `N/A` の理由を1行以上記録する。
4. CI証拠は、確認時点のPR current HEAD SHAとworkflow runの`head_sha`が一致するrunだけを採用する。一致runが0件なら `CI未実施` と記録する。
5. Phase 4A～4Eは、それぞれの先頭preflight taskが `完了` になるまで最初のsource Red commitを作成しない。

## 3. フェーズサマリー

| Phase | 名称 | 状態 | 依存 |
|---|---|---|---|
| 0 | 詳細設計・検証方針確定 | 進行中 | なし |
| 1 | managed scalar 四則演算パイロット | 未着手 | Phase 0 |
| 2 | 基本 `Interval` API 確定 | 未着手 | Phase 1 |
| 3 | SIMD backend | 未着手 | Phase 2 |
| 4A | 集合・関係・数値的属性・整数値関数 | 未着手 | Phase 3 + P4A-000 |
| 4B | 代数関数・区間定数 | 未着手 | Phase 4A + P4B-000 |
| 4C | 単調な初等関数 | 未着手 | Phase 4B + P4C-000 |
| 4D | 周期・特異点・多変数関数 | 未着手 | Phase 4C + P4D-000 |
| 4E | 非連結結果・decorated interval・I/O・分割 | 未着手 | Phase 4D + P4E-000 |

---

## 4. Phase 0: 詳細設計・検証方針確定

### 目的

Phase 1のsource実装開始前に、数値意味論、参照実装、検証方式、CI診断、タスク依存関係を固定する。

### 完了条件

- [ ] 詳細設計版5を対象にfix verificationを実施し、review reportへ `reviewed HEAD=<40桁SHA>`、`verdict=pass`、`unresolved findings=0` を記録する。
- [ ] `F-PR3-010`～`F-PR3-017`について、各findingのrequired actionと設計書sectionを1対1で追跡できるclosure表がreportにある。
- [ ] `Interval`、`IntervalUnion2`、`DecoratedInterval`の責務がそれぞれ「連結bare interval」「最大2 connected componentのtight closed enclosure」「interval+decoration/NaI」と記載され、同一状態型へ混在させない。
- [ ] reference lock候補として `inari@18b83a571d7681c76067bc38d90a74e8be29f545`、`kv@c7f8f2324a0e403cca6b39f46088a22843d440db`、`ITF1788@d8c2a64478ebdc9cbde6ccef33eaad3bed60ed81` が記録されている。
- [ ] 結果判定優先順位が `exact oracle > adopted IEEE 1788.1-oriented semantics > inari > compatible kv primitive` と記録されている。
- [ ] `tasks/tasks-status.md` にPhase 1～4Eの全必須task、依存関係、期待入力・期待結果を含む受け入れ条件が存在する。
- [ ] Phase 1最初のtask `P1-001` に、test result/stdout/stderr/diagnostic logと数値不一致詳細を保存するdiagnostic artifact workflow追加が必須条件として存在する。
- [ ] 現時点で実行可能projectが存在しないことをGitHub repository treeで確認し、Phase 0では空workflowを追加しない。

---

## 5. Phase 1: managed scalar 四則演算パイロット

### 目的

SIMD、global rounding mode変更、production native dependencyを使わず、pure-managedでbasic `Interval` とtightな四則演算を成立させる。

### 開始条件

- [ ] Phase 0が `完了` である。
- [ ] `P1-001`の最初のPRでproduction/test projectとdiagnostic workflowを同時追加する。

### 完了条件

- [ ] `net10.0` production projectとtest projectがLinux x64/ARM64でbuild/test終了コード0となる。
- [ ] CI matrixが `ubuntu-24.04` x64 と `ubuntu-24.04-arm` ARM64で、同一commit・同一test assembly・同一reference corpusを実行する。
- [ ] artifactにtest result、stdout、stderr、diagnostic log、runtime、OS、architecture、CPU feature、reference-lock、conformance summary、canonical-results.jsonlを保存する。
- [ ] 数値不一致entryごとに `caseId`, input bits, selected branch, exact result, Devo6 result bits, inari/kv/MPFR resultまたは`N/A`, expected-difference reasonをartifactから取得できる。
- [ ] `[1,2]`の内部論理stateが`[-1,2]`、`[-2,-1]`が`[+2,-1]`であり、Emptyは両lane canonical qNaN、片側NaN stateは生成不可またはinternal validation failureとなる。
- [ ] `default(Interval) == Interval.Zero`、`Empty.Lower=+Infinity`、`Empty.Upper=-Infinity`、nonempty lower zero=`-0.0`、upper zero=`+0.0`をbit fixtureで固定する。
- [ ] constructor invalid matrix `(NaN,1)`, `(0,NaN)`, `(1,-1)`, `(+Inf,+Inf)`, `(-Inf,-Inf)` は `ArgumentException`、`TryCreate`は`false/out=Empty`となる。
- [ ] directed add/sub/mul/divはexact oracleとcanonical bitsが一致し、exact caseでは不要な`NextUp/NextDown`を行わない。
- [ ] multiplicationの通常residual/scaled path、divisionのnormal residual/large denominator early-return、positive/negative finite overflowを固定witnessで通す。
- [ ] interval add/sub/mul/divはEmpty伝播、Z/P/N/M sign-class、zero-only/one-sided/crossing denominator matrixの期待結果と一致する。
- [ ] Phase 1 conformance manifestのrequired caseが1件も0件抽出にならず、全required caseがpassする。
- [ ] x64/ARM64の`canonical-results.jsonl`をcaseId順でbyte-for-byte比較しSHA-256が一致する。
- [ ] production packageのruntime dependencyはBCLのみで、global rounding-mode変更とproduction hot pathの`BigInteger`を使用しない。
- [ ] basic `+`, `-`, `*`, `/` のBenchmarkDotNet allocation columnが各々 `0 B` である。
- [ ] operator `+`, `-`, `*`, `/` のJIT/AOT hot pathにinterface/delegate/virtual indirect dispatchがなく、reflection、runtime codegen、dynamic assembly、native resolverをproduction assemblyから使用しない。
- [ ] Linux x64/ARM64でNativeAOT publishしたsampleが `new Interval(1,2)+Interval.Point(3) -> [4,5]` を出力し終了コード0となる。
- [ ] trimming有効publishしたsampleでも同じbasic API smokeが成功する。
- [ ] operator result constructionがpublic validating constructorを再度呼ばないことをJIT/AOT disassemblyまたは同等のcall graph evidenceで確認する。

---

## 6. Phase 2: 基本 `Interval` API 確定

### 目的

Phase 1で成立した数値意味論を維持したまま、basic public APIをbaselineとしてfreezeする。

### 完了条件

- [ ] assembly/package名、namespace、public type名、constructor/factory、properties、operators、equality/hash、exception、signed-zero、format方針を1つのdecision recordへ単一値として記録し、`TBD`/候補併記を残さない。
- [ ] public-only compile sampleで `new Interval(1,2)+Interval.Point(3)` が `[4,5]`、`new Interval(1,2)/new Interval(-1,1)` が `Entire`、`new Interval(2,1)` が `ArgumentException` となる。
- [ ] 上記sampleからinternal backend型、raw constructor、SIMD型、reference adapterを参照しない。
- [ ] scalar conversion/overload、generic math interface、正式formatについて各項目を `Adopt` または `Reject` と記録し、Adoptの場合はexact signatureとfixture、Rejectの場合はpublic API baselineにsignatureが存在しないことを確認する。
- [ ] public API baseline fileをrepositoryへ保存し、未承認public API差分がある検証runは終了コード非0となる。
- [ ] Phase 1のconformance/canonical corpusを再実行し、API freeze commitでcanonical result SHA-256がPhase 1 finalと一致する。
- [ ] basic operatorsのallocation `0 B` とNativeAOT/trimming smokeを再実行してpassする。
- [ ] Phase 2完了後にbaselineを破壊する変更は、`doc/Design/BreakingChanges.md`へ変更前signature、変更後signature、理由、移行例を追加しない限り受け入れない。

---

## 7. Phase 3: SIMD backend

### 目的

Phase 2 public APIとscalar canonical resultを変更せず、correctnessと事前固定benchmark gateを通ったSIMD kernelだけをproduction dispatchへ追加する。

### 完了条件

- [ ] capability判定を `Avx512F`, `Avx2`, `Avx`, `Fma`, `Sse2`, `AdvSimd.Arm64` ごとに独立fixtureで偽/真を切り替え、FMAをAVX2/SSE2へ暗黙従属させない。
- [ ] batch layout `[-L0,U0,-L1,U1,...]` のload/store round-tripで入力external endpoint bitsが全件復元される。
- [ ] batch add/subおよび採用候補mul/divはscalar referenceとのdifferential corpusでcanonical endpoint bitsが全件一致する。
- [ ] AVX-512の4 interval batchは末尾1～3 intervalをscalar fallbackし、N=1,2,3,4,5,7,8のfixtureでscalar結果と一致する。
- [ ] unsupported ISA/FMA組合せでは必ずscalarまたはqualified lower-ISA backendへfallbackし、`PlatformNotSupportedException`をbasic operatorから送出しない。
- [ ] correctness未達candidateはbenchmark結果にかかわらずproduction dispatchへ登録しない。
- [ ] benchmark policyはcandidate性能結果を採取するcommitより前のcommitで固定し、少なくともruntime/job、CPU affinity、scalar baseline、input corpus seed、batch size `4/32/256/4096`、Add/Sub/Mul/Div、metric=`median ns/interval`、allocation、採用閾値を記録する。
- [ ] production採用閾値は、対象architectureのN>=256全workloadでscalar比medianの幾何平均 `<=0.95`、各workload `<=1.02`、allocation増加0 Bとする。いずれかを満たさないkernelはdispatchへ登録しない。
- [ ] benchmark corpusはscalar/candidateで同一input bitsを使用し、benchmark result artifactへcommit SHA、CPU model、runtime version、candidate backend名を保存する。
- [ ] SIMD追加後もpublic API baseline diffが0件である。

---

## 8. Phase 4 共通開始ゲート

Phase 4A～4Eは各subphaseごとに `P4?-000` preflightを置く。

各preflightは次をすべて満たす。

- [ ] 直前Phaseの必須taskが全件 `完了` である。
- [ ] Phase 2 basic API freezeが維持され、未承認public API baseline差分が0件である。
- [ ] 対象設計sectionをnormal reviewし、review reportに `reviewed HEAD=<40桁SHA>`、`verdict=pass`、`unresolved findings=0` を記録する。
- [ ] review report pathとreviewed HEADをpreflight task reportから追跡できる。
- [ ] `P1-001`で追加したdiagnostic artifact workflowがrepositoryに存在する。
- [ ] 対象subphaseの最初のsmoke fixtureを既存test/conformance/reference harnessから実行し終了コード0となる。
- [ ] preflight taskのstatusが `完了` になる前に最初のsource Red commitを作成しない。

---

## 9. Phase 4A: 集合・関係・数値的属性・整数値関数

### Preflight smoke

`Entire.Contains(+Infinity) == false`、`Entire.Contains(0.0) == true` を既存test harnessで実行する。

### 完了条件

- [ ] `Contains`はEmpty/NaN/±Infinityでfalse、Entireは任意のfinite doubleでtrue、Zeroは±0.0でtrueとなる。
- [ ] `Intersect([1,3],[2,4])=[2,3]`、`Intersect([1,2],[3,4])=Empty`、`ConvexHull(Empty,[1,2])=[1,2]`となる。
- [ ] subset/interior/disjoint/precedes/strict-precedes/weak-less/strict-lessのEmpty matrixとendpoint条件をtask fixtureで固定し、通常の`< <= > >=` operatorをpublic APIへ追加しない。
- [ ] `IntervalOverlap` 16 enum stateを各1fixture以上で生成し、`state(X,Y)`と`inverse(state(Y,X))`が全16 stateで一致する。
- [ ] `Width(Empty)=NaN`, `Width([1,1])=+0`, `Width(Entire)=+Infinity`, `Midpoint(Entire)=+0`, lower-unbounded midpoint=`double.MinValue`, upper-unbounded midpoint=`double.MaxValue`を固定する。
- [ ] `Abs([-2,1])=[-0.0,2]`、`Sign([-2,3])=[-1,1]`、pointwise min/max、Floor/Ceiling/Truncate/Roundの決定的fixtureがpassする。
- [ ] intersection/hullのcommutative/idempotent、intersection subset、hull superset、`Mignitude<=Magnitude`のproperty testが固定seedでpassする。
- [ ] Midpoint tie policyとpublic namingをpreflight reviewで単一値へ確定し、task implementationに`TBD`を残さない。
- [ ] x64/ARM64 canonical result SHA-256一致、backend間bitwise一致、API baseline updateが完了する。

---

## 10. Phase 4B: 代数関数・区間定数

### Preflight smoke

`Square([-2,1])=[-0.0,4]` のexpected matrixをtest harnessへ読み込めることを確認する。

### 完了条件

- [ ] `Pi`, `HalfPi`, `TwoPi`, `E`, `Ln2`, `Ln10`, `Sqrt2`はMPFR RNDD/RNDU生成bitをrepositoryへ固定し、lower<=真値<=upperをgenerator testで確認する。
- [ ] `Reciprocal(Zero)=Empty`, `Reciprocal([-1,1])=Entire`, `Reciprocal([0,2])=[RD(1/2),+Infinity]`, `Reciprocal([-2,0])=[-Infinity,RU(-1/2)]`となる。
- [ ] `Square([-2,1])=[-0.0,4]` かつ `Square(X)` が同じXに対する `X*X` のsubset-or-equalとなる固定/property testがpassする。
- [ ] `Sqrt([-1,4])=[-0.0,2]`, `Sqrt([-4,-1])=Empty`、`2^-969`前後のscaled/normal branchでMPFR directed bitsと一致する。
- [ ] integer Powは指数0、正奇数、正偶数、負指数、zero crossing、`int.MinValue`を、Rootはdegree<=0例外、degree=1 identity、odd/even domainを固定matrixでpassする。
- [ ] `FusedMultiplyAdd` endpointがexact `x*y+z`を1回丸めし、同一入力で `(X*Y)+Z` のsubset-or-equalとなるfixtureをpassする。
- [ ] MPFR reference corpusにはMPFR version、RNDD/RNDU、generator hash、corpus SHA-256をlockする。
- [ ] managed-only backend採用時はreportへ `native_backend_gate=N/A` と理由を記録する。
- [ ] native backendを採用する場合はinterop/copy/dispatch込みbenchmarkがPhase 3と同じ事前固定performance gateをpassし、x64/ARM64配布asset、ABI compatibility、concurrent-call test、NativeAOT publish、trimming publish、license/NOTICE、binary redistribution条件をすべてpass/保存する。
- [ ] native backend有無でpublic `Interval` API baseline diffが0件、canonical endpoint bitsが一致する。

---

## 11. Phase 4C: 単調な初等関数

### Preflight smoke

`Log([-1,1])=[-Infinity,+0.0]` をMPFR/reference harnessと比較できることを確認する。

### 完了条件

- [ ] Exp/Exp2/Exp10はEmpty、±Infinity、overflow/underflow/subnormal endpointのMPFR RNDD/RNDU bitsと一致する。
- [ ] Log/Log2/Log10は`b<=0 -> Empty`、`a<=0<b -> lower=-Infinity`、`b=+Infinity -> upper=+Infinity`を固定fixtureでpassする。
- [ ] Sinh/Tanh/Asinh/Atanはmonotonic endpoint式、Coshはnegative/positive/zero-crossing 3分岐のexpected endpointをpassする。
- [ ] Acoshはdomain `[1,+Infinity)`、Asin/Acosは`[-1,1]`、Atanhは`(-1,1)`でclipし、domain内点が0件ならEmpty、open boundary接触時はlimit ±Infinityを返す。
- [ ] endpoint backendはcertified managed、検証済みcorrectly-rounded port、またはqualified directed native backendのいずれかで、BCL `Math.*`単独をcorrectness根拠にしない。
- [ ] MPFR corpus全required caseでcanonical bits一致し、承認差異がある場合だけcaseId、expected、actual、reason、approvalをmanifestへ保存する。
- [ ] support platformで通常入力を`PlatformNotSupportedException`へ送らない。
- [ ] x64/ARM64 SHA-256一致、backend間bitwise一致、API baseline updateが完了する。

---

## 12. Phase 4D: 周期・特異点・多変数関数

### Preflight smoke

`Atan2(Zero,[-2,-1]) -> Pi` と `Sin([0,HalfPi]) -> [0,1]` のreference fixtureを読み込めることを確認する。

### 完了条件

- [ ] periodic reducerは`Math.PI`通常除算や`% (2*Math.PI)`だけでquadrant/pole判定せず、固定high-precision `2/pi`, `pi/2` tableを使用する。
- [ ] Sin/Cosはcritical latticeを含む区間でexact extrema `-1/+1`を返し、非有界または必要critical lattice双方を含む場合`[-1,1]`を返す。
- [ ] Tanはpoleなし1 branchでdirected endpoints、poleへdomain内から接近可能ならEntire、poleしかdomainに残らない入力はEmptyとなる。
- [ ] Atan2 negative-x branch cutの6 classを全件固定し、`Atan2([-1,0],[-2,-1])=[-pi,+pi]`, `Atan2(Zero,[-2,-1])=Pi`, `Atan2([0,1],[-2,-1])=QII..pi`, `Atan2([-1,1],[-2,-1])=[-pi,+pi]`をpassする。
- [ ] general positive-base Powは`Pow([0,0.5],[0,1])=[0,1]`, `Pow([0,0.5],[-1,0])=[1,+Infinity]`, `Pow([0,2],[0,1])=[0,2]`, `Pow([0,0],[-1,0])=Empty`, `Pow([0,0],[0,1])=Zero`をpassする。
- [ ] `PowDown/Up(0,0)`と`PowDown/Up(0,negative)`が通常scalar kernelから呼ばれないことをtest hookで確認する。
- [ ] MPFR/reference corpus、x64/ARM64、全qualified backendでcanonical endpoint bitsが一致する。
- [ ] API baseline updateが完了する。

---

## 13. Phase 4E: 非連結結果・decorated interval・I/O・分割

### Preflight smoke

`DivideToUnion([1,2],Entire)` のexpected component count=2をexisting harnessへ読み込めることを確認する。

### 完了条件

- [ ] `IntervalUnion2`はCount=0/1/2 canonical stateを満たし、zero-touching 2 componentを`First.Upper==Second.Lower`だけでmergeしない。
- [ ] `DivideToUnion([1,2],Entire)`、`ReciprocalToUnion(Entire)`、`ReverseMultiply([1,2],Entire)`がCount=2の`[-Infinity,-0.0]`と`[+0.0,+Infinity]`を保持する。
- [ ] 全extended division fixtureで `ordinary division == DivideToUnion(...).ConvexHull` が成立する。
- [ ] cancellative operationのEmpty/common/unbounded 3x3 matrixとexact-width comparisonが全件passする。
- [ ] `Decoration : byte` の値が `Ill=0`, `Trv=4`, `Def=8`, `Dac=12`, `Com=16` と一致する。
- [ ] `default(DecoratedInterval).IsNaI==true`, `NaI==NaI==true`, `NaI.SemanticallyEquals(any)==false`、result-state decoration capを固定fixtureでpassする。
- [ ] parserは`[-Infinity,1]`, `[1,+Infinity]`, `[0.1]`, `0x1p+0`系exact hex fixtureをaccepted matrixとして、`[+Infinity,1]`, `[1,-Infinity]`, `[Infinity]`, `[NaN,1]`をrejected matrixとしてpassする。
- [ ] parserはInvariantCulture固定、recursionなし、preflightで確定したmax input/significand/exponent digit limitのlimit値とlimit+1 fixtureをpassする。
- [ ] binary interchange version 1は18 byte固定で、byte0=version、byte1=state、byte2..9=external Lower bits little-endian、byte10..17=external Upper bits little-endianとする。
- [ ] binary decoderは17 byte/19 byteをendpoint decode前にrejectし、unknown version/state、NaN endpoint、reversed endpointをrejectする。
- [ ] Zero binary round-tripはLower=`-0.0` external bits、Upper=`+0.0` external bitsを保持する。
- [ ] `TrySplitAt`と`TryBisect`はchildrenが元intervalをcoverし、各childが元interval subset、隣接endpointでstrict interior binary64がない場合falseとなる。
- [ ] parser/binary decoder security review、x64/ARM64 canonical一致、backend間bitwise一致、API baseline updateが完了する。

---

## 14. Phase完了時の更新規則

- Phase内の必須taskがすべて `完了` かつ本書の完了条件が全件 `[x]` になった時だけPhaseを `完了` にする。
- Phase 4のpreflight review対象設計HEADが後から変わった場合、該当 `P4?-000` を再openし、新HEADのreview passまでsource実装を再開しない。
- Phase 2 API freeze後にbasic API差分が出た場合、API baseline gateをfailさせ、breaking-change recordまたは差分撤回まで後続Phaseを進めない。
- CI完了判定は必ず対象PR current HEADとrun `head_sha`のexact matchで行い、matching runが0件なら `CI未実施` とする。
- mergeは利用者が行うため、本タスク群のworkerはmergeしない。
