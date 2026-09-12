# STEP 08 — ドメインイベント

## ゴール

**ドメインイベント**を、発行と配送に分けて理解する。
集約はイベントを溜めるだけ。誰にどう届けるかは集約の関心ではない。
Dart の `sealed class` + `switch` で網羅性をコンパイラに検査させる。

## 要件

### イベント定義

`lib/domain/event/domain_event.dart`

```dart
sealed class DomainEvent {
  DateTime get occurredAt;
}
```

| イベント | 発生タイミング | 含む情報 |
|---|---|---|
| `ShiftRequestSubmitted` | 希望提出時 | staffId / period / desiredSlots |
| `ShiftAssignmentAdded` | 割当追加時 | scheduleId / date / slot / staffId |
| `ShiftAssignmentRemoved` | 割当削除時 | scheduleId / date / slot / staffId |
| `ShiftScheduleConfirmed` | 確定時 | scheduleId / storeId / period / confirmedAt |
| `ShiftScheduleUnconfirmed` | 確定取消時 | scheduleId / reason |

全て不変（`final class`、値で等価）。

### 集約側

- `ShiftSchedule` と `ShiftRequest` に private なイベントバッファを持たせる。
- `List<DomainEvent> pullEvents()` — 溜まったイベントを返し、**バッファを空にする**。
- 操作が例外で失敗した場合、イベントは積まれないこと。
- `occurredAt` はドメイン内で `DateTime.now()` を呼ばず、操作の引数として受け取る
  （STEP 06 の方針の継続）。

### 網羅性の確認

`switch` 式で `DomainEvent` を分岐し、全ケースを扱うテスト用の関数を 1 つ書く。
`sealed` なので `default` 無しで網羅できることを確認する。
新しいイベント型を足すとコンパイルエラーになる —— それが狙い。

## 禁止事項

- イベントバス / ディスパッチャ / 購読側の実装。この STEP は発行まで。
- 非同期処理（`Future` / `Stream`）。
- イベントの永続化。
- `pullEvents()` を呼ばずに済む「自動配送」の仕込み。

## 完了条件

- `domain_event_test.dart`
  - `events with same payload are equal`
  - `switch over sealed event covers all cases without default`
- `shift_schedule_event_test.dart`
  - `assign records assignment added event`
  - `unassign records assignment removed event`
  - `confirm records confirmed event`
  - `unconfirm records unconfirmed event`
  - `pull events clears the buffer`
  - `failed assign records no event`
  - `failed confirm records no event`
- `shift_request_event_test.dart`
  - `submit records request submitted event`
  - `rejected submit records no event`
