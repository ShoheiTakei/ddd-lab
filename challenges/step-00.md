# STEP 00 — 土台とレイヤ分離

## ゴール

レイヤ化アーキテクチャの器を作り、**依存の向き**を仕組みで固定する。
ドメイン層が他層を知らない状態を、意志ではなく lint で担保する。

## 要件

1. `dart create -t package ddd_lab` 相当のパッケージを作る（このリポジトリ直下）。
2. ディレクトリを作る。

   ```
   lib/
     domain/          ドメインモデル。他層を import しない
     application/     ユースケース。domain のみ import 可
     infrastructure/  永続化実装。domain / application を import 可
   test/
     domain/
     application/
   bin/
     ddd_lab.dart     STEP 11 で使う。今は空でよい
   ```

3. `analysis_options.yaml` を厳格化する。

   - `include: package:lints/recommended.yaml`
   - `language: strict-casts / strict-inference / strict-raw-types` を全て `true`
   - `errors:` で `invalid_use_of_visible_for_testing_member` などを `error` に昇格

4. 依存方向を守るテストを 1 本書く。
   `lib/domain/**/*.dart` を読み、`import 'package:ddd_lab/application/` または
   `.../infrastructure/` を含む行があれば失敗させる。
   （`dart:io` でファイルを走査するだけの素朴なテストでよい）

5. `dart analyze` が警告 0 で通ること。

## 禁止事項

- ドメインモデルの実装（値オブジェクト、エンティティ、集約）。この STEP は器だけ。
- 外部パッケージの追加（`test` 以外）。
- `build_runner` / コード生成。

## 完了条件

- `dart analyze` → `No issues found.`
- `dart test` が通る。以下のテストが存在する。
  - `architecture_test.dart`
    - `domain layer does not import application layer`
    - `domain layer does not import infrastructure layer`
