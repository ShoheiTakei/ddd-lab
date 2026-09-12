# STEP 06 — 別集約の参照

## ゴール

**集約をまたぐ参照**を、オブジェクト参照ではなく ID と引数注入で解く。
そして `ShiftRequest` という 2 つ目の集約ルートを作り、
集約ごとに不変条件が閉じている状態を体験する。

## 要件

### `ShiftRequest`（集約ルート）

`lib/domain/model/shift_request.dart`

- 同一性: `StaffId` + `SchedulePeriod`。
- 属性: `staffId` / `period` / `Set<DateSlot> _desiredSlots` / `DateTime submittedAt`。
- `DateSlot` は値オブジェクト（`ShiftDate` + `ShiftSlot`）。`lib/domain/model/date_slot.dart`。
- `ShiftRequest.submit({staffId, period, slots, deadline, now})`
  - **R-09**: `now` が `deadline` を過ぎていれば `DomainException`。
  - `slots` に `period` 外の日が含まれていれば拒否。
  - `slots` が空でも成立する（「1 枠も入れない」という申告）。
- `bool covers(ShiftDate date, ShiftSlot slot)`
- `Set<DateSlot> get desiredSlots` は不変ビューで返す。

> `now` を引数で受け取ること。`DateTime.now()` をドメイン内で呼ばない。
> 時刻はドメインにとって外部入力である。テスト容易性の話だけではない。

### R-08 の実装

`lib/domain/rule/desired_slot_rule.dart` — `DesiredSlotRule`

- 希望を出していない枠への割当を違反とする。
- `RuleContext` に `Map<StaffId, ShiftRequest> requests` を追加する。
- 希望そのものが未提出のスタッフに割当がある場合も違反（理由をメッセージに書く）。

> `ShiftSchedule` が `ShiftRequestRepository` を呼んで希望を読む、という実装は禁止。
> 集約は他集約を取りに行かない。呼び出し側が読んで `RuleContext` に詰める。

## 禁止事項

- `ShiftSchedule` から `ShiftRequest` をオブジェクト参照で保持すること。
- ドメイン層での `DateTime.now()`。
- リポジトリの実装（インターフェースも）。STEP 09。

## 完了条件

- `shift_request_test.dart`
  - `submit before deadline succeeds`
  - `submit after deadline is rejected`
  - `submit exactly at deadline succeeds`
  - `submit with slot outside period is rejected`
  - `submit with empty slots succeeds`
  - `desired slots view is unmodifiable`
  - `covers returns true for submitted slot`
  - `requests with same staff and period are equal`
- `desired_slot_rule_test.dart`
  - `returns no violation when assignment matches request`
  - `returns violation when assignment is not in request`
  - `returns violation when staff has no request at all`
