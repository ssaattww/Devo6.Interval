# 詳細設計

区間演算の詳細設計は、次の順序で読む。

1. `IntervalArithmetic.md`
2. `IntervalArithmetic.Revision3.md`

`IntervalArithmetic.Revision3.md`は、PR #3のfix verificationで残った`F-PR3-004`および`F-PR3-009`に対応する規範的revisionである。

次の事項について両文書が矛盾する場合、`IntervalArithmetic.Revision3.md`を優先する。

- `Interval.Empty`の公開`Lower` / `Upper`
- exact-rational oracleのfinite overflow変換
- IEEE 1788.1 Phase 1 conformance source mapping
- repository-defined `IsSingleton` matrix
- conformance adaptationとacceptance gate
- overflow fixtureのoracle経路
- review finding closure

それ以外の事項は`IntervalArithmetic.md`設計版2を継続して適用する。

将来、両文書を1ファイルへ統合する場合は、統合内容を別PRでreviewし、Revision 3の規定が失われていないことを確認する。
