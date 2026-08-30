# 詳細設計

## 読取順序

区間演算の詳細設計は、次の順序で読む。

1. `IntervalArithmetic.md`
2. `IntervalArithmetic.Revision3.md`
3. `IntervalNonArithmetic.Roadmap.md`
4. `IntervalSetAndNumeric.md`
5. `IntervalMathFunctions.md`
6. `IntervalAdvancedFeatures.md`

## 四則演算設計のprecedence

`IntervalArithmetic.Revision3.md`は、PR #3の四則演算詳細設計に対するfix verificationで残った`F-PR3-004`および`F-PR3-009`に対応する規範的revisionである。

次の事項について`IntervalArithmetic.md`と矛盾する場合、`IntervalArithmetic.Revision3.md`を優先する。

- `Interval.Empty`の公開`Lower` / `Upper`
- exact-rational oracleのfinite overflow変換
- IEEE 1788.1 Phase 1 conformance source mapping
- repository-defined `IsSingleton` matrix
- conformance adaptationとacceptance gate
- overflow fixtureのoracle経路
- review finding closure

それ以外の四則演算事項は`IntervalArithmetic.md`設計版2を継続して適用する。

## 四則演算以外の文書境界

- `IntervalNonArithmetic.Roadmap.md`
  - Phase 4A～4Eの順序、責務分割、正確性レベル、共通ゲート
- `IntervalSetAndNumeric.md`
  - 集合演算、関係判定、overlap、数値的属性、整数値関数
- `IntervalMathFunctions.md`
  - reciprocal、square、sqrt、integer/general power、FMA、初等関数
- `IntervalAdvancedFeatures.md`
  - 2成分union、extended/reverse division、decoration、parse/format、interchange、split

これら3つの機能別詳細設計がロードマップと矛盾する場合、より具体的な機能別詳細設計を優先する。ただし、既存四則演算の意味論やRevision 3の規範を暗黙に変更してはならない。

## review状態

`IntervalArithmetic.md`と`IntervalArithmetic.Revision3.md`の既存範囲は、reviewed design HEAD `13cf07cfcdf01205ab4466a99abd380fd1f1d103`でfix verification PASSとなった。

`IntervalNonArithmetic.Roadmap.md`以降の新規文書は、そのreviewed HEADより後に追加されたため、既存PASS verdictの対象外である。

四則演算以外の実装を開始する前に、新規文書を含むimmutable PR HEADを対象とした独立設計reviewを実施する。

## 将来の統合

将来、Revisionまたは複数文書を統合する場合は、統合内容を別のreviewable changeとして扱い、次を確認する。

- Revision 3の規定が失われていない。
- 機能別文書の意味論・API・検証ゲートが失われていない。
- 既存review verdictを統合後HEADへ自動的に引き継いでいない。
