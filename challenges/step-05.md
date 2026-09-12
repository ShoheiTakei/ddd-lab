# STEP 05 — 集合ルール

## ゴール

「対象期間の割当**全体**を見ないと判定できないルール」を実装し、
**なぜ対象期間が集約境界なのか**を実感する。
設計書 4 章の主張が、コードとして正しいかを自分で確かめる STEP。

## 要件

`lib/domain/rule/` に `ShiftRule` の実装を追加する。

| ID | クラス | 内容 |
|---|---|---|
| R-02 | `WeeklyWorkingHoursRule` | スタッフ 1 名の週労働時間が 40 時間を超えない |
| R-03 | `ConsecutiveWorkDaysRule` | スタッフ 1 名の連続勤務日数が 6 日を超えない |
| R-05 | `MinimumStaffingRule` | 期間内の各日・各枠に最低 2 名 |
| R-06 | `RegisterCertifiedRule` | 期間内の各日・各枠にレジ研修修了者 1 名以上 |

### 実装上の決めごと

- **R-02**: 週は月曜起点。`SchedulePeriod.weeks()` を使う。
  期間の端で週が欠ける場合、**期間内の割当のみで計算する**（設計書 8 章の既知の制限）。
  この単純化をテスト名で明示すること。
- **R-03**: 同上。期間の先頭より前は見ない。
- **R-05 / R-06**: 「割当が 0 件の枠」も違反として検出する必要がある。
  割当のリストを走査するだけでは 0 件の枠は現れない。
  `period.dates()` × `ShiftSlot.values` を回して埋める。
- **R-06**: `profiles` を引く。研修修了者が 1 人もいない枠が違反。

### 違反の粒度

- R-02 / R-03 は **スタッフ単位**（1 スタッフ 1 週につき 1 件）。
- R-05 / R-06 は **日 × 枠単位**。
- `Violation.message` に、不足数や合計時間など判断材料を含める。

## 禁止事項

- `ShiftRuleChecker`。STEP 07。
- 前後期間の割当を参照すること（集約をまたぐ。既知の制限として受け入れる）。
- `ShiftRequest` の参照。STEP 06。

## 完了条件

- `weekly_working_hours_rule_test.dart`
  - `returns no violation at exactly forty hours`
  - `returns violation above forty hours in one week`
  - `counts hours per staff independently`
  - `counts hours per week independently`
  - `partial week at period boundary counts only assignments inside period`
- `consecutive_work_days_rule_test.dart`
  - `returns no violation for six consecutive days`
  - `returns violation for seven consecutive days`
  - `day off resets the streak`
  - `multiple slots on the same day count as one day`
- `minimum_staffing_rule_test.dart`
  - `returns no violation when every slot has two staff`
  - `returns violation when a slot has one staff`
  - `returns violation when a slot has no assignment at all`
  - `reports the shortage in the message`
- `register_certified_rule_test.dart`
  - `returns no violation when slot includes certified staff`
  - `returns violation when no staff in slot is certified`
  - `returns violation for empty slot`
