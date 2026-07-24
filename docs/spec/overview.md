# 仕様: 横断定義（用語・データモデル）

> 全機能で共有する **用語・データ名称・データモデル** の集約。
> 機能別の詳細仕様は `docs/spec/<keyword>.md` を参照。

> ステータス記法: **[実装済み]** = 現コードに存在  /  **[目標]** = 計画（未実装。関連 ADR/タスクを併記）。

## 1. 用語（概要）

- **ポイント残高** `PointBalance` … 子供ごとのポイント現在値。**日替わりポイント**＋**タスクポイント**の2種で構成。
  - **日替わりポイント** `DailyPoint` … 毎朝に付与。その日のうちに消費しなければ消える。
  - **タスクポイント** `TaskPoint` … タスク達成で付与。日替わりポイントとは異なり繰り越され日毎にリセットされない。
- **ポイントの記録** `PointRecord` … タスクでのポイントの獲得・遊びでのポイントの消費など本アプリ上での行動の記録。
- **計測された記録** `MeasuredRecord` … iPad / Switch 等の実測（スクショ由来）の記録。
- **履歴** `History` … 現状の表示用の取引履歴（部分的: redeem / auto / approve）。「ポイントの記録」の新規イベントモデルで**廃止・置き換え**（データ移行なし。ADR-0003）。
- **メンバー** `Member` … 親・子のユーザー。ポイント残高・PIN・アイコン等を持つ。
- **タスク** `Task` … ポイント獲得のための行為（お手伝い・宿題等）。親承認が要るものと、承認不要で即時加算されるものがある。
- **ごほうび** `Reward` … ポイントを消費して得る遊び（ゲーム / 動画 / マンガ等。時間とコストを持つ）。
- **申請** `Request` … タスク達成の親承認待ち。
- **タイマー** `Timer` … 遊びの残り時間管理（メンバーごと）。

## 2. 現状のデータモデル **[実装済み]**

すべて `localStorage`（キー `family-point-app`）の単一オブジェクトにまとめて保存する。残高は **直接保持・直接更新**（イベントからの導出はしていない）。`history` は表示用の取引履歴を**部分的に**持つ。各データ:

- **Member**（子供・親, 複数） = `{ id, name, role: 'parent'|'child', icon, pin, points, dailyPoints, dailyDate }`
  - `points` = タスクポイント / `dailyPoints` = 日替わりポイント / `dailyDate` = 日替わりの最終付与日
- **Task**（複数, 既定 `DEFAULT_TASKS`） = `{ id, name, points, icon, autoApprove? }`（`autoApprove`: 承認不要の即時加算）
- **Reward**（複数, 既定 `DEFAULT_REWARDS`） = `{ id, name, cost, minutes, icon }`
- **Request**（承認待ち, 複数） = `{ id, taskId, taskName, taskIcon, points, childId, childName, date, status:'pending' }`
- **HistoryEntry**（表示用の取引履歴・部分的, type 別）
  - `redeem`  = `{ type:'redeem', childId, childName, rewardName, cost, usedDaily, usedTask, minutes, date }`
  - `auto`    = `{ type:'auto', taskId, taskName, taskIcon, points, childId, childName, date }`
  - `approve` = `{ type:'approve', ...request, approvedBy, date }`
- **Timer**（`memberId` ごと） = `{ timerEnd, timerLabel, totalDur }`
- **設定 / フラグ**: `setup`（初期設定済みか） / `dailyAmount`（日替わり付与量, 既定 60）

要点:
- **残高は2種**: `dailyPoints`（日替わり）＋ `points`（タスク）。合計 = 両者の和。**消費順: 日替わり → タスク**。
- **日替わり付与は `history` に記録されない**（`dailyDate` が変わったら `dailyPoints` を再セットするのみ）。
- したがって現状は「**残高が真実の源・履歴は表示用で不完全**」。残高は保持したままイベント記録を整備するのが T_001（ADR-0003）。

## 3. ポイントの獲得 **[実装済み]**

- **日替わりポイント**: 毎朝5時（アプリ上の日付境界）に既定 60pt（`dailyAmount`）へ**リセット付与**（加算でなく上書き。未使用分は繰り越さない）。
- **タスク（即時 / `autoApprove`）**: 子がボタンを押すと即タスクポイントを加算。各タスク1日1回まで。
- **タスク（承認制）**: 子が申請（`Request`）→ 親が承認でタスクポイント加算（却下なら付与なし）。

## 4. ポイントの利用 **[実装済み]**

- ごほうび（`Reward`）を選ぶと `cost` 分を消費し、`minutes` 分のタイマーを起動する。
- **消費順: 日替わりポイント → タスクポイント**。
- **残高不足時は利用不可**（ポイント残高 ≥ `cost` のときのみ。負残高にはならない）。

## 5. ポイントの記録 — イベント記録（振り返り用） **[目標 / ADR-0003・T_001]**

獲得/使用を追記する append-only のイベント列。既存 `history` は流用せず**新規モデルで置き換える**（データ移行なし。ADR-0003）。残高（現在値）は §2 のとおり別に保持し、ここからは**導出しない**。残高更新とイベント追記は1か所の関数に集約する。

記録対象（ADR-0011。日替わりの失効は記録しない）:
- **獲得**: `daily_grant`（日替わり付与） / `task_earn`（タスク即時） / `task_approve`（タスク承認） / `manual_adjust`（親の手動調整）
- **使用**: `play`（あそび。ごほうびの「項目名」と「利用時間」= ポイントを利用して実行した時間量、を保持する）
- 将来拡張の type 例: `reconcile`（自己申告の遡及消費） / `repay`（借金返済）

補足:
- 「利用時間」は購入時点の時間量。将来タイマーの一時停止・途中キャンセルを実装した場合、実際に遊んだ時間と乖離しうる前提で設計する（補正イベントの追記等で対応。※「実測時間」という旧称は外部計測（§6）と紛らわしいため廃止）。
- 保持方針: 生イベントは直近 N 日、古いものは日次集計へロールアップ（localStorage 約5MB 対策。見直し条件は ADR-0004）。

## 6. 計測された記録 — Activity（取り込み契約） **[目標]**

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

## 7. 機能インデックス

- [活動突き合わせ機能](activity-reconciliation.md) — Keyword `activity-reconciliation`
