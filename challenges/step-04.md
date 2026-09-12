# STEP 04 — 仕様パターンと単一割当ルール

## ゴール

ビジネスルールを**オブジェクトとして分離**する（仕様パターン）。
そして、集約をまたぐ情報を **スナップショットの値オブジェクト** として
引数で受け取る技法を身につける。

## 要件

### `Violation`（値オブジェクト）

`lib/domain/rule/violation.dart`

- 属性: `ViolationType type` / `ShiftDate? date` / `ShiftSlot? slot` / `StaffId? staffId` / `String message`。
- `ViolationType` は enum。`r01MinorAtNight` / `r02WeeklyHoursExceeded` / ... の形で
  R-01〜R-08 分を先に定義してよい（実装は各 STEP）。
- 値で等価。

### `StaffProfile`（値オブジェクト）

`lib/domain/rule/staff_profile.dart`

- 属性: `StaffId staffId` / `bool isMinor` / `bool isRegisterCertified`。
- **判定に必要な述語だけ** を持つ。氏名も生年月日も持たない。
- `Staff` からの変換はここに書かない。アプリケーション層の責務（STEP 10）。

### `ShiftRule`（インターフェース）

`lib/domain/rule/shift_rule.dart`

```dart
abstract interface class ShiftRule {
  List<Violation> check(RuleContext context);
}
```

`RuleContext` は検査に必要な入力をまとめた値オブジェクト。

- `SchedulePeriod period`
- `List<ShiftAssignment> assignments`
- `Map<StaffId, StaffProfile> profiles`
- （STEP 06 で `requests` を追加する）

`profiles` に無い `StaffId` が割当に現れた場合の扱いを決め、テストで固定すること。

### 実装するルール

| ID | クラス | 内容 |
|---|---|---|
| R-01 | `MinorNightShiftRule` | 18 歳未満を深夜番に割り当てていないか |
| R-04 | `NightShiftIntervalRule` | 深夜番の翌日、同一スタッフが早番・中番に入っていないか |

どちらも割当 1 件、または 2 件の関係だけで判定できる。

## 禁止事項

- `ShiftRuleChecker`（ルールを束ねるもの）。STEP 07 で行う。
- 期間全体を走査しないと判定できないルール（R-02 R-03 R-05 R-06）。STEP 05。
- `ShiftSchedule` に `check()` を生やすこと。この STEP ではルール単体をテストする。
- `Staff` を `ShiftRule` から参照すること。`StaffProfile` 経由のみ。

## 完了条件

- `minor_night_shift_rule_test.dart`
  - `returns no violation when adult works night shift`
  - `returns violation when minor works night shift`
  - `returns violation per minor night assignment`
  - `returns no violation when minor works daytime`
- `night_shift_interval_rule_test.dart`
  - `returns violation when early shift follows night shift`
  - `returns violation when middle shift follows night shift`
  - `returns no violation when late shift follows night shift`
  - `returns no violation for different staff`
  - `returns no violation when next day is outside period`
- `staff_profile_test.dart`
  - `profiles with same values are equal`
