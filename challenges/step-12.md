# STEP 12 — 拡張性の検証（任意）

## ゴール

STEP 04 で導入した仕様パターンが**本当に効いているか**を実地で測る。
開放閉鎖原則は主張ではなく、検証できる性質である。

## 課題

新しいビジネスルールを追加する。

> **R-12**: 1 人のスタッフを同一日に 3 枠以上割り当ててはならない（2 枠まで）。

### 制約

**既存ファイルの変更を、以下の 2 箇所だけに収めること。**

1. `ViolationType` への enum 値の追加。
2. `ShiftRuleChecker.standard()` のルール一覧への 1 行追加。

これ以外の既存ファイルに差分が出たら、STEP 04 の設計に問題がある。
`git diff --stat` で確認する。

### 手順

1. 先に `daily_slot_limit_rule_test.dart` を書く（Red）。
2. `DailySlotLimitRule` を新規ファイルで実装する（Green）。
3. `git diff --stat` を取り、変更ファイルを数える。
4. 制約を超えていたら、**超えた原因を特定して STEP 04〜07 を直す**。
   ルールを通すことではなく、設計を直すことがこの STEP の目的。

## 追加課題（さらに任意）

R-12 の「2 枠」を店舗ごとに変えたくなったとする。

- ルールを設定値でパラメータ化するとき、その設定はどこに置くか。
  ドメインか、アプリケーションか、インフラか。
- `MinimumStaffingRule` の「2 名」も同じ問題を持つ。両者を同じ扱いにできるか。

結論をコードにする必要はない。`docs/retrospective.md` に考察を追記する。

## 禁止事項

- 制約を守るために既存ルールのインターフェースを歪めること。
  差分が広がるなら、それは発見であって失敗ではない。記録して直す。

## 完了条件

- `daily_slot_limit_rule_test.dart`
  - `returns no violation for two slots on the same day`
  - `returns violation for three slots on the same day`
  - `counts slots per staff independently`
  - `counts slots per day independently`
- `shift_rule_checker_test.dart` の `standard checker includes all eight rules` を
  9 ルールに更新（この 1 行はカウント対象外）。
- `git diff --stat` の結果を `docs/retrospective.md` に貼り、制約内であることを示す。
