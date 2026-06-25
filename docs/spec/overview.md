# 仕様: 横断定義（用語・データモデル）

> 全機能で共有する **用語・データ名称・データモデル** の集約。
> 機能別の詳細仕様は `docs/spec/<keyword>.md` を参照。

> ステータス記法: **[実装済み]** = 現コードに存在  /  **[目標]** = 計画（未実装。関連 ADR/タスクを併記）。

## 1. 用語
記録名はドメイン/表示名、レコード単位と実装方式は別レイヤ。

| 記録（ドメイン名 / コード） | レコード単位 | 実装方式 |
|---|---|---|
| **ポイントの記録** `pointRecord` | Event | 現在値（残高）を保持＋イベントは振り返り用に分離（残高は導出しない） **[目標 / ADR-0003・T_001]** |
| **計測された記録** `measuredRecord` | Activity | スクショ由来データの取り込み **[目標]** |

- 「ポイントの記録」= アプリ経由の遊び・タスク・日替わり付与（子供が申告した記録）。
- 「計測された記録」= iPad ScreenTime / みまもりSwitch のスクショ由来（実際に使った記録）。
- イベントは**振り返り用の追記記録**で、残高（現在値）の算出には使わない（event sourcing ではない。ADR-0003）。

## 2. 現状のデータモデル（`state`） **[実装済み]**
`localStorage`（キー `family-point-app`）に保存。残高は **直接保持・直接更新**（イベントからの導出はしていない）。`history` は表示用の取引履歴を**部分的に**持つ。

```
state = {
  setup: boolean,
  members: Member[],
  tasks: Task[],            // 既定: DEFAULT_TASKS
  rewards: Reward[],        // 既定: DEFAULT_REWARDS
  requests: Request[],      // 承認待ち
  history: HistoryEntry[],  // 表示用の取引履歴（部分的）
  timers: { [memberId]: Timer },
  dailyAmount: number,      // 日替わり付与量（既定 60）
}

Member  = { id, name, role: 'parent'|'child', icon, pin, points, dailyPoints, dailyDate }
Task    = { id, name, points, icon, autoApprove? }       // autoApprove: 承認不要の即時加算
Reward  = { id, name, cost, minutes, icon }
Request = { id, taskId, taskName, taskIcon, points, childId, childName, date, status:'pending' }
Timer   = { timerEnd, timerLabel, totalDur }

HistoryEntry（type 別）:
  redeem  = { type:'redeem',  childId, childName, rewardName, cost, usedDaily, usedTask, minutes, date }
  auto    = { type:'auto',    taskId, taskName, taskIcon, points, childId, childName, date }
  approve = { type:'approve', ...request, approvedBy, date }
```

要点:
- **残高は2種**: `dailyPoints`（日替わり）＋ `points`（タスク）。合計 = 両者の和。**消費順: 日替わり → タスク**。
- **日替わり付与は `history` に記録されない**（`dailyDate` が変わったら `dailyPoints` を再セットするのみ）。
- したがって現状は「**残高が真実の源・履歴は表示用で不完全**」。残高は保持したままイベント記録を整備するのが T_001（ADR-0003）。

## 3. ポイントの記録 — イベント記録（振り返り用） **[目標 / ADR-0003・T_001]**
獲得/消費/日替わり付与を追記する append-only のイベント列。残高（現在値）は §2 のとおり別に保持し、ここからは**導出しない**。
- type 例: `daily_grant` / `task_earn` / `task_approve` / `play_start` / `play_end` / `manual_adjust` / `reconcile` / `repay`
- 遊びイベントは「項目名」と「実測（タイマー）時間」を保持する。
- 保持方針: 生イベントは直近 N 日、古いものは日次集計へロールアップ（localStorage 約5MB 対策。見直し条件は ADR-0004）。

## 4. 計測された記録 — Activity（取り込み契約） **[目標]**
解析くん（外部AI）とアプリの契約スキーマ（ADR-0004）。
```json
{
  "source": "switch | screentime",
  "childId": "taro",
  "date": "2026-06-14",
  "items": [
    { "app": "Splatoon 3", "minutes": 75, "category": "game",      "chargeable": true  },
    { "app": "YouTube",    "minutes": 40, "category": "video",     "chargeable": true  },
    { "app": "Duolingo",   "minutes": 15, "category": "education", "chargeable": false }
  ]
}
```
- 1スクショ = 1エクスポート（`source × childId × date`）。
- `app / minutes / category / chargeable` は解析側（AI）が埋める。
- インポートは `(childId, date, source, app)` で upsert（冪等。再取り込みで二重計上しない）。

## 5. 機能インデックス
- [活動突き合わせ機能](activity-reconciliation.md) — Keyword `activity-reconciliation`
