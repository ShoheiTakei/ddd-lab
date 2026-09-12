# STEP 07 — ドメインサービスと確定

## ゴール

**ドメインサービス**の正しい薄さを理解する。ロジックを持つのはルール側であって、
サービスは束ねるだけ。そして集約の**状態遷移**を確定というゲートで表現する。

## 要件

### `ShiftRuleChecker`（ドメインサービス）

`lib/domain/rule/shift_rule_checker.dart`

- コンストラクタで `List<ShiftRule>` を受け取る。
- `List<Violation> check(RuleContext context)` — 全ルールを順に適用して結果を連結する。
- **それ以外のロジックを持たない**。分岐も条件も書かない。
- `ShiftRuleChecker.standard()` で R-01〜R-08 の全ルールを組んだ既定構成を返す。

### `ShiftSchedule` への追加

- `List<Violation> check(ShiftRuleChecker checker, Map<StaffId, StaffProfile> profiles, Map<StaffId, ShiftRequest> requests)`
  - 自身の `period` と `assignments` から `RuleContext` を組んで `checker` に渡す。
- `void confirm({checker, profiles, requests})`
  - **R-11**: 違反が 1 件でもあれば `DomainException`。違反リストを例外に含める。
  - すでに `confirmed` なら例外。
  - 成功時に `status` を `confirmed` にする。
- `void unconfirm(String reason)`
  - `draft` のときに呼べば例外。
  - `reason` が空文字なら例外。
  - 成功時に `status` を `draft` に戻す。

### 状態遷移

```
draft ──confirm()（違反0件のみ）──> confirmed
draft <──unconfirm(reason)────────── confirmed
```

`draft` でのみ `assign` / `unassign` 可（STEP 03 で実装済み）。

## 禁止事項

- `ShiftRuleChecker` に個別ルールの知識を書くこと（`if (rule is MinorNightShiftRule)` の類）。
- ドメインイベントの発行。STEP 08。
- 確定時に割当を自動修正すること。検査して通すか弾くかだけ。

## 完了条件

- `shift_rule_checker_test.dart`
  - `returns empty list when all rules pass`
  - `concatenates violations from multiple rules`
  - `standard checker includes all eight rules`
- `shift_schedule_confirm_test.dart`
  - `confirm succeeds when no violation`
  - `confirm fails when violation exists`
  - `confirm failure keeps status draft`
  - `confirm on already confirmed schedule is rejected`
  - `unconfirm returns schedule to draft`
  - `unconfirm on draft schedule is rejected`
  - `unconfirm with empty reason is rejected`
  - `assign succeeds again after unconfirm`
