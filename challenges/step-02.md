# STEP 02 — エンティティと同一性

## ゴール

**エンティティ**と値オブジェクトの違いを、等価性の実装を通して理解する。
エンティティは同一性で等しく、値オブジェクトは全属性で等しい。

## 要件

### `ShiftAssignment`（エンティティ）

`lib/domain/model/shift_assignment.dart`

- 属性: `ShiftDate date` / `ShiftSlot slot` / `StaffId staffId`。
- 同一性: **date + slot + staffId の 3 つ組**。
  この 3 つが一致すれば同じ割当である（R-07 の重複判定の土台）。
- `WorkingHours workingHours()` — `slot.workingHours()` に委譲する。
- `bool isLateNight()` — 同上。
- `ShiftDate get nextDate` — R-04 の判定で使う。

> 考えどころ: この 3 つ組は全属性でもある。ならば値オブジェクトではないのか。
> 「割当は差し替えられる対象として集約内で識別される」という立場を取り、
> エンティティとして扱う。この判断の根拠を PR の説明に一言書くこと。

### `Staff`（集約ルート、この STEP では単体）

`lib/domain/model/staff.dart`

- 属性: `StaffId id` / `String name` / `DateTime birthDate` / `bool isRegisterCertified` / `bool isActive`。
- 同一性: `StaffId` のみ。名前が変わっても同じスタッフ。
- `bool isMinorAt(ShiftDate date)` — その日時点で 18 歳未満か。
  誕生日当日は 18 歳になっている（未成年ではない）扱い。
- 名前が空文字なら拒否。

## 禁止事項

- `ShiftSchedule`（集約ルート）の実装。STEP 03 で行う。
- `StaffProfile`。STEP 04 で行う。
- ルール検査ロジック。`isMinorAt` は属性の計算であってルールではない。
- リポジトリ。

## 完了条件

- `shift_assignment_test.dart`
  - `assignments with same date slot and staff are equal`
  - `assignments differing in staff are not equal`
  - `working hours delegate to slot`
- `staff_test.dart`
  - `staff with same id is equal even if name differs`
  - `staff with different id is not equal`
  - `is minor before eighteenth birthday`
  - `is not minor on eighteenth birthday`
  - `rejects empty name`
