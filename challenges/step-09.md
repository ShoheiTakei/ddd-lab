# STEP 09 — リポジトリ

## ゴール

**リポジトリ**を、集約の永続化境界として理解する。
抽象はドメイン層、実装はインフラ層（依存性逆転）。
保存単位は集約であって、その内部エンティティではない。

## 要件

### 抽象（`lib/domain/repository/`）

```dart
abstract interface class ShiftScheduleRepository {
  Future<ShiftSchedule?> findById(ScheduleId id);
  Future<ShiftSchedule?> findByStoreAndPeriod(StoreId storeId, SchedulePeriod period);
  Future<void> save(ShiftSchedule schedule);
}

abstract interface class ShiftRequestRepository {
  Future<ShiftRequest?> findByStaffAndPeriod(StaffId staffId, SchedulePeriod period);
  Future<List<ShiftRequest>> findByPeriod(SchedulePeriod period);
  Future<void> save(ShiftRequest request);
}

abstract interface class StaffRepository {
  Future<Staff?> findById(StaffId id);
  Future<List<Staff>> findByIds(Iterable<StaffId> ids);
  Future<List<Staff>> findActiveByStore(StoreId storeId);
}
```

- `ShiftAssignment` 専用のリポジトリは**作らない**。集約の内部だから。
- 検索条件を増やしたくなっても、この STEP では増やさない。

### 実装（`lib/infrastructure/`）

インメモリ実装を 3 本。`Map` で持つだけ。

- `save` は集約まるごとの置き換え。
- **`save` した集約と、`findById` で返る集約が同一インスタンスにならないこと。**
  外で書き換えた結果が勝手に「保存済み」にならないようにする（コピーして保持する）。
  そのためのコピー手段として `ShiftSchedule.reconstruct` を使う。
- 見つからなければ `null`。例外にしない。

### アーキテクチャテストの追加

STEP 00 のテストに 1 件足す。`lib/domain/repository/**` が
`lib/infrastructure/` を import していないこと。

## 禁止事項

- 実 DB、ファイル永続化、外部パッケージ。
- ユニットオブワーク、トランザクション境界の明示的な実装。
  集約 = トランザクション境界という前提だけで進む。
- 遅延読み込み、キャッシュ。
- `findAll()` のような何でも取れる操作。

## 完了条件

- `in_memory_shift_schedule_repository_test.dart`
  - `saved schedule can be found by id`
  - `saved schedule can be found by store and period`
  - `returns null when not found`
  - `save overwrites the existing schedule`
  - `returned schedule is not the same instance as the saved one`
  - `mutating the returned schedule does not affect the stored one`
- `in_memory_shift_request_repository_test.dart`
  - `saved request can be found by staff and period`
  - `find by period returns requests of all staff`
  - `returns null when not found`
- `in_memory_staff_repository_test.dart`
  - `find by ids returns only existing staff`
  - `find active by store excludes inactive staff`
- `architecture_test.dart`
  - `domain repository interfaces do not import infrastructure`
