# STEP 03 — 集約ルート

## ゴール

**集約**を理解する。集約ルートが内部コレクションを外から壊されないよう守り、
不変条件を自分の内側で維持する。外部は集約ルート経由でしか内部に触れない。

## 要件

`lib/domain/model/shift_schedule.dart`

### 属性

| 名前 | 型 |
|---|---|
| `id` | `ScheduleId` |
| `storeId` | `StoreId` |
| `period` | `SchedulePeriod` |
| `status` | `ScheduleStatus`（enum: `draft` / `confirmed`） |
| `_assignments` | `List<ShiftAssignment>`（private） |

### 公開 API

- `ShiftSchedule.create({required id, required storeId, required period})`
  → `status` は `draft`、割当は空。
- `List<ShiftAssignment> get assignments`
  → `UnmodifiableListView` で返す。呼び出し側が `add` しても例外になること。
- `void assign(ShiftDate date, ShiftSlot slot, StaffId staffId)`
- `void unassign(ShiftDate date, ShiftSlot slot, StaffId staffId)`

### この STEP で実装する不変条件

| ID | ルール | 違反時の挙動 |
|---|---|---|
| — | 割当日は `period` の範囲内であること | 例外 |
| R-07 | 同一日・同一枠・同一スタッフの重複禁止 | 例外 |
| R-10 | `confirmed` のシフト表は `assign` / `unassign` 不可 | 例外 |

例外は `lib/domain/model/domain_exception.dart` に定義した
`DomainException`（またはそのサブクラス）を投げる。`ArgumentError` は使わない。

> なぜ R-07 と R-10 だけ例外で、他のルールは例外にしないのか。
> 設計書 S-03 を読むこと。「その操作が成立しない」ものは例外、
> 「一時的に成立していないだけ」のものは違反として保持する。R-07 は
> 同じ割当を 2 つ持つこと自体が表現として不整合なので例外側。

- `unassign` で存在しない割当を指定した場合も例外。

## 禁止事項

- ルール検査（`check`）、`Violation`、`confirm` / `unconfirm`。STEP 04〜07 で行う。
- `Staff` や `ShiftRequest` への参照（オブジェクト参照もインポートも）。
- `_assignments` を `get` で直接返すこと。

## 完了条件

- `shift_schedule_test.dart`
  - `newly created schedule is draft and empty`
  - `assign adds an assignment`
  - `assignments list is unmodifiable`
  - `assign outside period is rejected`
  - `assign duplicate date slot staff is rejected`
  - `unassign removes an assignment`
  - `unassign of missing assignment is rejected`
  - `assign on confirmed schedule is rejected`
  - `unassign on confirmed schedule is rejected`

> `confirmed` 状態のテストは `confirm()` がまだ無い。テスト用に
> `ShiftSchedule.reconstruct(...)` を用意して状態を直接与えてよい
> （永続化からの復元用として STEP 09 でも使う）。
