# STEP 10 — アプリケーションサービス

## ゴール

**アプリケーション層の薄さ**を体で覚える。
ここがやるのは「読む → ドメインに渡す → 保存する」だけ。
ビジネス判断を 1 つでもここに書いたら、それはドメイン貧血症の始まり。

## 要件

`lib/application/` にユースケースを 4 本。全て `Future` を返す。

| ユースケース | 入力 | 処理 |
|---|---|---|
| `SubmitShiftRequestUseCase` | staffId / period / slots / deadline / now | `ShiftRequest.submit` → save |
| `AssignStaffUseCase` | scheduleId / date / slot / staffId / now | schedule を読む → `assign` → save |
| `CheckShiftScheduleUseCase` | scheduleId | profiles と requests を集めて `check` → `List<Violation>` を返す |
| `ConfirmShiftScheduleUseCase` | scheduleId / now | 同上を集めて `confirm` → save |

### この層の責務

1. リポジトリから集約を読む。無ければアプリ層の例外（`NotFoundException` など）。
2. **`Staff` → `StaffProfile` への変換**。ここが唯一の変換地点。
   `isMinorAt` にどの日付を渡すか（期間開始日か割当日か）を決めて、理由をコメントに残す。
3. ドメインのメソッドを呼ぶ。
4. `pullEvents()` でイベントを回収し、**戻り値に含める**（配送はまだしない）。
5. 保存する。

### 禁止的に重要な点

- ルールの判定、違反の解釈、状態の分岐を書かない。
- `if (violations.isNotEmpty) throw` を**アプリ層に書かない**。R-11 はドメインの責務。
- `DateTime.now()` はこの層より外から渡す（ユースケースの引数）。

### 入出力

- 入力は各ユースケース専用の `Command` 値オブジェクト、出力は `Result` 値オブジェクト。
  プリミティブの羅列で引数を並べない。

## 禁止事項

- ドメインロジックのアプリ層への流出（上記）。
- DI コンテナ。コンストラクタ注入を手で書く。
- CLI / UI。STEP 11。
- イベントの配送。

## 完了条件

- `submit_shift_request_use_case_test.dart`
  - `saves the request when submitted before deadline`
  - `returns submitted event`
  - `does not save when deadline has passed`
- `assign_staff_use_case_test.dart`
  - `saves the schedule with the new assignment`
  - `fails when schedule does not exist`
  - `does not save when assignment is rejected`
- `check_shift_schedule_use_case_test.dart`
  - `returns empty violations for a valid schedule`
  - `returns violations converted from staff profiles`
  - `does not throw even when violations exist`
- `confirm_shift_schedule_use_case_test.dart`
  - `confirms and saves when there is no violation`
  - `fails and does not save when violations exist`
  - `returns confirmed event on success`
