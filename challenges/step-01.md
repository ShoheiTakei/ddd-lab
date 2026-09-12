# STEP 01 — 値オブジェクト

## ゴール

**値オブジェクト**を理解する。不変であること、値で等価判定されること、
自分自身を検証すること、そして振る舞いを持つこと。
`int` や `String` をそのまま引き回す「原始型執着」を潰す。

## 要件

`lib/domain/model/` に以下を実装する。全て不変（`final class`、フィールドは `final`）。

### 識別子

`StaffId` / `StoreId` / `ScheduleId`

- 内部は `String`（UUID 文字列）。
- 空文字は拒否（`ArgumentError`）。
- `==` / `hashCode` を値ベースで実装する。
- 型が違えば等しくない（`StaffId('x') != StoreId('x')` がコンパイルで保証される）。

### `ShiftSlot`

`enum`。早番 / 中番 / 遅番 / 深夜番。

| 枠 | 時間帯 | 労働時間 | 深夜 |
|---|---|---|---|
| early | 06:00-12:00 | 6h | no |
| middle | 12:00-18:00 | 6h | no |
| late | 18:00-22:00 | 4h | no |
| night | 22:00-06:00(翌) | 8h | yes |

- `WorkingHours workingHours()` を持つ。
- `bool isLateNight()` を持つ。
- `bool isMorningOrDaytime()`（R-04 用。early / middle が true）。

「深夜かどうか」の判断を呼び出し側に時刻比較させない。知識は枠自身に閉じる。

### `ShiftDate`

- 暦日のみを表す。時刻成分を持たない（`DateTime` をそのまま公開しない）。
- `ShiftDate.of(2026, 10, 1)` で生成。
- `ShiftDate nextDay()` / `ShiftDate previousDay()`。
- `int get weekdayNumber`（月曜=1）。
- `Comparable<ShiftDate>` を実装する。

### `SchedulePeriod`

- 開始日と終了日を持つ。
- **半月のみ許可**する。`1日〜15日` または `16日〜その月の末日`。
  それ以外はコンストラクタで `ArgumentError`。
- `bool contains(ShiftDate date)`。
- `List<ShiftDate> dates()`。
- `List<List<ShiftDate>> weeks()` — 月曜起点で週に分割する。
  期間の端の週は欠けたままでよい（S-05 の既知の制限）。

### `WorkingHours`

- 時間数（`int`、負数は拒否）。
- `WorkingHours operator +(WorkingHours other)`。
- `bool exceeds(WorkingHours limit)`。
- `==` / `hashCode` / `toString`。

## 禁止事項

- エンティティ、集約、リポジトリ、ルール検査。
- ミュータブルなフィールド、`setter`。
- `DateTime` を公開 API のシグネチャに出すこと（`ShiftDate` で包む）。

## 完了条件

`test/domain/model/` に以下が存在し、通ること。

- `staff_id_test.dart`
  - `same value is equal`
  - `different value is not equal`
  - `empty string is rejected`
- `shift_slot_test.dart`
  - `night slot is late night`
  - `night slot has 8 working hours`
  - `early and middle are morning or daytime`
- `shift_date_test.dart`
  - `nextDay crosses month boundary`
  - `is comparable by calendar order`
- `schedule_period_test.dart`
  - `accepts first half of month`
  - `accepts second half of month ending on last day`
  - `rejects period spanning two months`
  - `rejects period not aligned to half month`
  - `contains returns true for date inside period`
  - `weeks splits by monday`
- `working_hours_test.dart`
  - `addition accumulates hours`
  - `exceeds returns true over limit`
  - `rejects negative hours`
