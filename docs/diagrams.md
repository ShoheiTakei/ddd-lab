# コンビニ シフト管理 — 図面集

用語・ルールの定義は [design.md](./design.md) を参照。

---

## 1. ドメインモデル図

```mermaid
classDiagram
    class ShiftSchedule {
        <<AggregateRoot>>
        +ScheduleId id
        +StoreId storeId
        +SchedulePeriod period
        +ScheduleStatus status
        -List~ShiftAssignment~ assignments
        +assign(date, slot, staffId)
        +unassign(date, slot, staffId)
        +check(profiles, requests) List~Violation~
        +confirm(profiles, requests)
        +unconfirm(reason)
    }

    class ShiftAssignment {
        <<Entity>>
        +ShiftDate date
        +ShiftSlot slot
        +StaffId staffId
        +workingHours() WorkingHours
    }

    class ShiftRequest {
        <<AggregateRoot>>
        +StaffId staffId
        +SchedulePeriod period
        -Set~DateSlot~ desiredSlots
        +DateTime submittedAt
        +submit(slots, deadline, now)
        +covers(date, slot) bool
    }

    class Staff {
        <<AggregateRoot>>
        +StaffId id
        +String name
        +Date birthDate
        +bool registerCertified
        +bool active
        +toProfile(asOf) StaffProfile
    }

    class ShiftSlot {
        <<ValueObject>>
        EARLY
        MIDDLE
        LATE
        NIGHT
        +workingHours() WorkingHours
        +isLateNight() bool
    }

    class SchedulePeriod {
        <<ValueObject>>
        +Date from
        +Date to
        +contains(date) bool
        +weeks() List~DateRange~
    }

    class StaffProfile {
        <<ValueObject>>
        +StaffId staffId
        +bool isMinor
        +bool isRegisterCertified
    }

    class Violation {
        <<ValueObject>>
        +RuleId ruleId
        +ShiftDate date
        +ShiftSlot slot
        +StaffId staffId
        +String message
    }

    class WorkingHours {
        <<ValueObject>>
        +Decimal hours
        +plus(other) WorkingHours
        +exceeds(limit) bool
    }

    class ShiftRuleChecker {
        <<DomainService>>
        -List~ShiftRule~ rules
        +check(schedule, profiles, requests) List~Violation~
    }

    class ShiftRule {
        <<Interface>>
        +apply(schedule, profiles, requests) List~Violation~
    }

    ShiftSchedule "1" *-- "0..*" ShiftAssignment : contains
    ShiftSchedule --> SchedulePeriod
    ShiftAssignment --> ShiftSlot
    ShiftAssignment ..> Staff : staffId only
    ShiftRequest --> SchedulePeriod
    ShiftRequest ..> Staff : staffId only
    ShiftSlot --> WorkingHours
    Staff ..> StaffProfile : creates
    ShiftRuleChecker --> ShiftRule : composes
    ShiftRuleChecker ..> ShiftSchedule : inspects
    ShiftRuleChecker ..> Violation : produces
    ShiftRuleChecker ..> StaffProfile : reads
    ShiftRuleChecker ..> ShiftRequest : reads
```

破線は ID 参照のみを表す。集約間でオブジェクト参照は持たない。

---

## 2. 集約境界図

```mermaid
flowchart TB
    subgraph BC["境界づけられたコンテキスト: シフト計画"]
        direction TB

        subgraph AGG1["集約: ShiftSchedule"]
            direction TB
            SS["ShiftSchedule (root)<br/>ScheduleId / StoreId / period / status"]
            SA1["ShiftAssignment<br/>date + slot + staffId"]
            SA2["ShiftAssignment"]
            SAN["... 最大約200件<br/>(15日 x 4枠 x 数名)"]
            SS --- SA1
            SS --- SA2
            SS --- SAN
        end

        subgraph AGG2["集約: ShiftRequest"]
            SR["ShiftRequest (root)<br/>staffId + period<br/>desiredSlots / submittedAt"]
        end

        subgraph AGG3["集約: Staff"]
            ST["Staff (root)<br/>StaffId / birthDate<br/>registerCertified / active"]
        end

        DS["ドメインサービス: ShiftRuleChecker<br/>集約をまたぐ判定をここに置く"]
    end

    subgraph OUT["スコープ外コンテキスト"]
        direction LR
        PAY["給与計算"]
        ATT["勤怠実績"]
        NTF["通知"]
    end

    AGG1 -. "staffId で参照" .-> AGG3
    AGG2 -. "staffId で参照" .-> AGG3
    DS -. "読み取り" .-> AGG1
    DS -. "StaffProfile で読み取り" .-> AGG3
    DS -. "読み取り" .-> AGG2
    AGG1 == "ShiftScheduleConfirmed" ==> OUT

    style AGG1 fill:#e8f4ff,stroke:#2b6cb0,stroke-width:3px
    style AGG2 fill:#fff4e8,stroke:#b06c2b,stroke-width:3px
    style AGG3 fill:#f0e8ff,stroke:#6c2bb0,stroke-width:3px
    style OUT fill:#f5f5f5,stroke:#999,stroke-dasharray: 5 5
```

**境界の根拠**

| ルール | 判定に必要な範囲 | 集約内で完結するか |
|---|---|---|
| R-01 深夜×未成年 | 割当1件 + StaffProfile | 引数で受け取れば可 |
| R-02 週40時間 | 対象期間の全割当 | ShiftSchedule 内で完結 |
| R-03 連続勤務日数 | 対象期間の全割当 | ShiftSchedule 内で完結 |
| R-04 深夜翌日の早番禁止 | 連続2日の割当 | ShiftSchedule 内で完結 |
| R-05 最低2名 | 1日1枠の全割当 | ShiftSchedule 内で完結 |
| R-06 レジ研修者1名 | 1日1枠の全割当 + StaffProfile | 引数で受け取れば可 |
| R-07 重複割当 | 割当2件 | ShiftSchedule 内で完結 |
| R-08 希望外割当 | 割当 + ShiftRequest | 引数で受け取れば可 |

トランザクション境界の外に出るのは `ShiftRequest` と `Staff` の読み取りのみ。
書き込みは常に1集約に閉じる。

---

## 3. 状態遷移図

### ShiftSchedule

```mermaid
stateDiagram-v2
    [*] --> Draft : create(storeId, period)

    Draft --> Draft : assign / unassign
    Draft --> Draft : check() → 違反リスト返却

    Draft --> Confirmed : confirm()<br/>[違反0件]
    Draft --> Draft : confirm()<br/>[違反あり] → 拒否 (R-11)

    Confirmed --> Draft : unconfirm(reason)<br/>[店長権限]
    Confirmed --> Confirmed : assign / unassign<br/>→ 拒否 (R-10)

    Confirmed --> [*] : 対象期間の終了

    note right of Draft
        違反を保持したまま編集できる。
        まとめ編集の途中で一時的に
        違反するのを許すため。
    end note

    note right of Confirmed
        変更不可。修正するには
        一度 unconfirm する。
    end note
```

### ShiftRequest

```mermaid
stateDiagram-v2
    [*] --> NotSubmitted : 対象期間の開始

    NotSubmitted --> Submitted : submit()<br/>[now <= deadline]
    NotSubmitted --> NotSubmitted : submit()<br/>[now > deadline] → 拒否 (R-09)

    Submitted --> Submitted : submit()<br/>[now <= deadline] → 上書き
    Submitted --> Submitted : submit()<br/>[now > deadline] → 拒否 (R-09)

    Submitted --> [*] : シフト確定後
    NotSubmitted --> [*] : シフト確定後（希望なし扱い）
```

---

## 4. シーケンス図

### S-01 正常系: 割当から確定まで

```mermaid
sequenceDiagram
    actor M as 店長
    participant App as ShiftAppService
    participant SchR as ShiftScheduleRepository
    participant StfR as StaffRepository
    participant ReqR as ShiftRequestRepository
    participant Sch as ShiftSchedule
    participant Chk as ShiftRuleChecker
    participant Bus as EventBus

    M->>App: assign(scheduleId, date, slot, staffId)
    App->>SchR: findById(scheduleId)
    SchR-->>App: ShiftSchedule (Draft)
    App->>Sch: assign(date, slot, staffId)
    Note over Sch: R-10 確定済みでないか確認<br/>R-07 重複でないか確認
    Sch-->>App: ShiftAssignmentAdded
    App->>SchR: save(schedule)
    App-->>M: 追加完了

    M->>App: check(scheduleId)
    App->>SchR: findById(scheduleId)
    App->>StfR: findByIds(割当中の staffId 一覧)
    StfR-->>App: List~Staff~
    Note over App: Staff → StaffProfile へ変換<br/>集約の内部を渡さない
    App->>ReqR: findByPeriod(period)
    ReqR-->>App: List~ShiftRequest~
    App->>Chk: check(schedule, profiles, requests)
    loop 各 ShiftRule
        Chk->>Chk: rule.apply(...)
    end
    Chk-->>App: List~Violation~ (空)
    App-->>M: 違反なし

    M->>App: confirm(scheduleId)
    App->>Chk: check(schedule, profiles, requests)
    Chk-->>App: List~Violation~ (空)
    App->>Sch: confirm()
    Note over Sch: R-11 違反0件を確認<br/>status = Confirmed
    Sch-->>App: ShiftScheduleConfirmed
    App->>SchR: save(schedule)
    App->>Bus: publish(ShiftScheduleConfirmed)
    App-->>M: 確定完了
```

### S-03 違反あり: 18歳未満を深夜番に割当

```mermaid
sequenceDiagram
    actor M as 店長
    participant App as ShiftAppService
    participant Sch as ShiftSchedule
    participant Chk as ShiftRuleChecker
    participant R1 as MinorNightShiftRule

    M->>App: assign(date=10/03, slot=NIGHT, staffId=S17)
    App->>Sch: assign(...)
    Note over Sch: 割当自体は成功する。<br/>Draft は違反を保持できる。
    Sch-->>App: ShiftAssignmentAdded
    App-->>M: 追加完了

    M->>App: confirm(scheduleId)
    App->>Chk: check(schedule, profiles, requests)
    Chk->>R1: apply(schedule, profiles, requests)
    Note over R1: 深夜枠の割当を走査<br/>profile.isMinor を確認
    R1-->>Chk: [Violation(R-01, 10/03, NIGHT, S17)]
    Chk-->>App: 違反1件
    App--xM: 確定失敗<br/>「10/03 深夜番: S17 は18歳未満のため割当不可」
    Note over M: 割当を修正して再試行
```

### S-02 締切後の希望提出

```mermaid
sequenceDiagram
    actor S as スタッフ
    participant App as RequestAppService
    participant ReqR as ShiftRequestRepository
    participant Req as ShiftRequest

    S->>App: submitRequest(staffId, period, desiredSlots)
    App->>ReqR: findByStaffAndPeriod(staffId, period)
    ReqR-->>App: ShiftRequest (NotSubmitted)
    App->>Req: submit(slots, deadline, now)
    Note over Req: R-09 now > deadline
    Req--xApp: DeadlinePassedError
    App--xS: 提出失敗<br/>「提出締切を過ぎています」
    Note over ReqR: 保存されない。状態は変わらない。
```

---

## 5. フロー図

### 全体業務フロー

```mermaid
flowchart TD
    Start(["対象期間の準備開始"]) --> Create["店長: シフト表を作成<br/>status=Draft"]
    Create --> Open["希望提出期間を開始"]
    Open --> Submit{"スタッフ:<br/>希望を提出"}

    Submit -->|締切前| Accept["希望を受理・保存"]
    Submit -->|締切後| Reject["R-09 拒否"]
    Reject --> Submit
    Accept --> Deadline{"締切到来?"}
    Deadline -->|まだ| Submit
    Deadline -->|到来| Assign

    Assign["店長: 割当を追加・削除"] --> Check["検査を実行<br/>ShiftRuleChecker"]
    Check --> HasV{"違反あり?"}

    HasV -->|あり| Show["違反一覧を提示<br/>ルールID・日・枠・スタッフ"]
    Show --> Assign

    HasV -->|なし| Confirm["店長: 確定<br/>status=Confirmed"]
    Confirm --> Event["ShiftScheduleConfirmed 発行"]
    Event --> Notify[/"スコープ外:<br/>通知・勤怠・給与へ連携"/]
    Notify --> Done(["確定済みシフト"])

    Done --> NeedFix{"修正が必要?"}
    NeedFix -->|はい| Unconfirm["店長: 確定取消<br/>status=Draft"]
    Unconfirm --> Assign
    NeedFix -->|いいえ| End(["対象期間の終了"])

    style Reject fill:#ffe8e8,stroke:#c00
    style Show fill:#fff4e8,stroke:#c80
    style Confirm fill:#e8ffe8,stroke:#0a0
    style Notify fill:#f5f5f5,stroke:#999,stroke-dasharray: 5 5
```

### 検査処理の内部フロー

```mermaid
flowchart TD
    In(["check 開始"]) --> Load["割当中の staffId を収集"]
    Load --> Prof["StaffRepository から Staff を取得<br/>→ StaffProfile に変換"]
    Prof --> Req["ShiftRequestRepository から<br/>対象期間の希望を取得"]
    Req --> Loop["ShiftRuleChecker に渡す"]

    Loop --> R1["R-01 未成年の深夜割当"]
    R1 --> R2["R-02 週40時間上限"]
    R2 --> R3["R-03 連続勤務6日上限"]
    R3 --> R4["R-04 深夜翌日の早番・中番禁止"]
    R4 --> R5["R-05 各枠 最低2名"]
    R5 --> R6["R-06 各枠 レジ研修者1名以上"]
    R6 --> R7["R-07 同一枠の重複割当"]
    R7 --> R8["R-08 希望外の割当"]
    R8 --> Agg["全違反を集約"]
    Agg --> Out(["List~Violation~ を返却"])

    style Loop fill:#e8f4ff,stroke:#2b6cb0
```

各ルールは `ShiftRule` の実装として独立している。途中で打ち切らず全て適用し、
違反をまとめて返す。店長が1件ずつ直して再実行する手間を避けるため。

---

## 6. ER図（対象スコープのみ）

```mermaid
erDiagram
    STORE ||--o{ SHIFT_SCHEDULE : "has"
    STORE ||--o{ STAFF : "employs"
    SHIFT_SCHEDULE ||--o{ SHIFT_ASSIGNMENT : "contains"
    STAFF ||--o{ SHIFT_ASSIGNMENT : "assigned to"
    STAFF ||--o{ SHIFT_REQUEST : "submits"
    SHIFT_REQUEST ||--o{ SHIFT_REQUEST_SLOT : "contains"

    STORE {
        uuid store_id PK
        string name
    }

    STAFF {
        uuid staff_id PK
        uuid store_id FK
        string name
        date birth_date
        boolean register_certified
        boolean active
    }

    SHIFT_SCHEDULE {
        uuid schedule_id PK
        uuid store_id FK
        date period_from
        date period_to
        string status "Draft | Confirmed"
        timestamp request_deadline
        timestamp confirmed_at "nullable"
    }

    SHIFT_ASSIGNMENT {
        uuid schedule_id PK,FK
        date shift_date PK
        string shift_slot PK "EARLY|MIDDLE|LATE|NIGHT"
        uuid staff_id PK,FK
    }

    SHIFT_REQUEST {
        uuid staff_id PK,FK
        date period_from PK
        date period_to
        timestamp submitted_at "nullable"
    }

    SHIFT_REQUEST_SLOT {
        uuid staff_id PK,FK
        date period_from PK,FK
        date shift_date PK
        string shift_slot PK
    }
```

**永続化の方針**

- `SHIFT_ASSIGNMENT` の主キーは `(schedule_id, shift_date, shift_slot, staff_id)` の
  複合キー。R-07（重複割当禁止）はこの主キー制約でも担保される。ドメイン側の検査と
  二重になるが、DB制約は最後の砦として残す。
- `SHIFT_SCHEDULE` に `(store_id, period_from)` の一意制約を置く。
  同一店舗・同一期間のシフト表は1つだけ。
- 集約単位で保存する。`ShiftScheduleRepository.save()` は `SHIFT_SCHEDULE` と
  配下の `SHIFT_ASSIGNMENT` を1トランザクションで書き込む。
- `SHIFT_ASSIGNMENT` に代理キーを置かない。割当は集約内部のエンティティであり、
  外部から単独で参照されないため。

**スコープ外のためテーブルを持たないもの**

勤怠実績、給与、シフト交代申請、欠勤連絡、通知履歴。

---

## 7. シナリオパターン一覧

詳細は [design.md 第7章](./design.md#7-シナリオパターン) を参照。

```mermaid
flowchart LR
    subgraph Normal["正常系"]
        S1["S-01 希望提出→確定"]
        S8["S-08 確定取消→再編集"]
    end

    subgraph Rule["ルール違反系"]
        S3["S-03 未成年×深夜 (R-01)"]
        S4["S-04 最低人数不足 (R-05)"]
        S5["S-05 週40時間超過 (R-02)"]
        S6["S-06 希望外の割当 (R-08)"]
    end

    subgraph State["状態違反系"]
        S2["S-02 締切後の提出 (R-09)"]
        S7["S-07 確定済みの変更 (R-10)"]
    end

    Rule -->|"全て確定時に<br/>R-11 で阻止"| Gate["確定ゲート"]
    State -->|"操作時点で<br/>即座に拒否"| Err["例外"]
    Normal --> Ok["確定成功"]

    style Gate fill:#fff4e8,stroke:#c80
    style Err fill:#ffe8e8,stroke:#c00
    style Ok fill:#e8ffe8,stroke:#0a0
```

**2種類の拒否の使い分け**

| 種類 | 対象 | タイミング | 表現 |
|---|---|---|---|
| 状態違反 | R-09, R-10 | 操作の時点 | 例外を投げる。操作は成立しない |
| ルール違反 | R-01〜R-08 | 確定の時点 | `Violation` として蓄積。編集は続行できる |

前者は「その操作自体が無効」、後者は「その状態では確定できない」。
Draft が違反を保持できることが、この設計の核になっている。
