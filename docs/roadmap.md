# DDD 学習ロードマップ — コンビニ シフト管理 (Dart)

対象設計: [design.md](./design.md) / [diagrams.md](./diagrams.md)

## 前提

- 言語: Dart（Flutter 不要）。単体パッケージ `ddd_lab`。
- 依存: `package:test` のみ。外部 DI・ORM・コード生成は使わない。
- 進め方: 1 STEP = 1 ブランチ = 1 PR。テストを先に書く（TDD）。
- 各 STEP には **禁止事項** がある。先回り実装はロードマップの学習効果を壊すため守る。

## 課題ファイルの形式

`challenges/step-NN.md` に以下を記載する。

| 節 | 内容 |
|---|---|
| ゴール | この STEP で理解すべき DDD 概念 |
| 要件 | 実装する対象 |
| 禁止事項 | この STEP でやってはいけないこと |
| 完了条件 | 通るべきテスト名の一覧 |

## STEP 一覧

| STEP | テーマ | 主な成果物 |
|---|---|---|
| [00](../challenges/step-00.md) | 土台とレイヤ分離 | パッケージ、`analysis_options.yaml`、層ディレクトリ |
| [01](../challenges/step-01.md) | 値オブジェクト | `StaffId` `ShiftSlot` `ShiftDate` `SchedulePeriod` `WorkingHours` |
| [02](../challenges/step-02.md) | エンティティと同一性 | `ShiftAssignment` `Staff` |
| [03](../challenges/step-03.md) | 集約ルート | `ShiftSchedule` (assign / unassign) |
| [04](../challenges/step-04.md) | 仕様パターン・単一割当ルール | `ShiftRule` `Violation` `StaffProfile` R-01 R-04 |
| [05](../challenges/step-05.md) | 集合ルール | R-02 R-03 R-05 R-06 |
| [06](../challenges/step-06.md) | 別集約参照 | `ShiftRequest` R-09 R-08 |
| [07](../challenges/step-07.md) | ドメインサービスと確定 | `ShiftRuleChecker` `confirm` `unconfirm` R-11 R-10 |
| [08](../challenges/step-08.md) | ドメインイベント | 5 イベント + `pullEvents` |
| [09](../challenges/step-09.md) | リポジトリ | 3 抽象 + インメモリ実装 |
| [10](../challenges/step-10.md) | アプリケーションサービス | ユースケース 4 本 |
| [11](../challenges/step-11.md) | 統合シナリオ | S-01〜S-08 受入テスト + CLI |
| [12](../challenges/step-12.md) | 拡張性の検証（任意） | 新ルール R-12 を既存変更なしで追加 |

## 依存関係

```
00 → 01 → 02 → 03 → 04 → 05 → 06 → 07 → 08 → 09 → 10 → 11 → 12
```

直列。前 STEP のテストが全て緑であることが次の STEP の開始条件。

## ルール ID との対応

| ルール | 実装 STEP |
|---|---|
| R-01 深夜番・18歳未満禁止 | 04 |
| R-02 週40時間 | 05 |
| R-03 連続勤務6日 | 05 |
| R-04 深夜番の翌日 | 04 |
| R-05 最低2名 | 05 |
| R-06 レジ研修修了者1名以上 | 05 |
| R-07 同一日同一枠の重複 | 03 |
| R-08 希望外割当 | 06 |
| R-09 締切後の希望提出 | 06 |
| R-10 確定済みの変更禁止 | 03（拒否）/ 07（状態遷移全体） |
| R-11 違反ありは確定不可 | 07 |
