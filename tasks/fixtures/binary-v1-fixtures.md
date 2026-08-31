# Binary interchange v1 canonical fixtures

`tasks/tasks-status.md` の `P4E-000` / `P4E-010` で固定したversion 1 wire contractのcanonical byte fixtureである。

共通layout:

```text
byte 0      version = 0x01
byte 1      state   = 0x00 Normal / 0x01 Empty
byte 2..9   external Lower bits, little-endian
byte 10..17 external Upper bits, little-endian
```

## Canonical 18-byte fixtures

| caseId | Semantic state | 18-byte expected hex |
|---|---|---|
| `binary-v1-normal-1-2` | Normal `[1,2]` | `01 00 00 00 00 00 00 00 f0 3f 00 00 00 00 00 00 00 40` |
| `binary-v1-zero` | Normal Zero (`Lower=-0.0`, `Upper=+0.0`) | `01 00 00 00 00 00 00 00 00 80 00 00 00 00 00 00 00 00` |
| `binary-v1-entire` | Normal Entire | `01 00 00 00 00 00 00 00 f0 ff 00 00 00 00 00 00 f0 7f` |
| `binary-v1-empty` | Empty, all-zero payload | `01 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00` |

## Decode normalization / rejection

- Normal stateのLower bitsが`+0.0` (`0x0000000000000000`)ならdecode後は`-0.0` (`0x8000000000000000`)へcanonicalizeする。
- Normal stateのUpper bitsが`-0.0`ならdecode後は`+0.0`へcanonicalizeする。
- Empty stateではbyte 2..17がすべて`0x00`の入力だけをcanonicalとして受理し、1 bitでも非zeroならrejectする。
- versionが`0x01`以外、stateが`0x00/0x01`以外、Normal stateのNaN endpoint、`Lower > Upper`、`Lower=+Infinity`、`Upper=-Infinity`はrejectする。
- length 17/19 byteはversion/state/endpointをdecodeする前にrejectする。
