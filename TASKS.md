# TASKS

プロジェクト全体のタスク台帳

## ルール

### 情報単位

- マイルストーン（Milestone）
- タスク（Task）

タスクはマイルストーンに紐付く事ができる。（紐付かなくてもよい）

### 属性情報

Status：
  Todo / Doing / Done
Blocked：
  指定した Task が終わらないと進行できない場合に付く
Milestone：
  Task が紐付く Milestone。Milestone 見出しでまとめている場合は記載不要
Keyword：
  Keyword で引けるドキュメントが有る場合は Keyword を設定する

### 参考

Document Rule: ./CLAUDE.md h2.ドキュメント構造 を参照

## Milestone

* M_001 「ポイントの記録」を Day や Week にまとめて表示する
  - Status: ToDo
  - Blocked: None
  - Keyword: `activity-reconciliation`
  - 「ポイントの記録」は「遊んだ」「宿題をした」などの子供の行動の記録
* M_002 申告と実測を突き合わせ、子供が自分で申告できる
  - Status: ToDo
  - Blocked: None
  - Keyword: `activity-reconciliation`
* M_003 スクリーンタイム解析くん（スクショ → 構造化データ, PoC 先行）
  - Status: ToDo
  - Blocked: None
  - Keyword: `activity-reconciliation`

## Task

### M_001 「ポイントの記録」を Day や Week にまとめて表示する

* T_001 ポイントの記録への移行（イベントログ化）
  - Status: ToDo
  - Blocked: None
  - 残高中心 `state` を追記型の「ポイントの記録」（イベントログ / event sourcing）へ移行。残高は導出（ADR-0003）。全機能の基盤。
* T_002 遊びセッション拡張
  - Status: ToDo
  - Blocked: T_001
  - 遊びイベントに「項目名」「実測（タイマー）時間」を記録。
* T_003 Day/Week ビューの追加
  - Status: ToDo
  - Blocked: T_001, T_002
  - 記録（努力/遊び/外部計測）を種別フィルタで単体表示できる Day/Week 集計 + タイムライン（spec 4.2 振り返りビュー）。比較ビューの基盤も兼ねるが、単体表示だけで価値が出る。

### M_XXX 「計測された記録」の導入と表示
### M_XXX 「ポイントの記録」と「計測された記録」を突き合わせて確認する
子供が自分で申告できる

* T_004 借金モデル（アプリ化）
  - Status: ToDo
  - Blocked: None
  - 負残高 = 借金、返済、段階返済（1倍 / 1倍 / 2倍）。現状は紙運用、後段でアプリ化。
* T_005 Activity インポートインターフェース
  - Status: ToDo
  - Blocked: None
  - Activity スキーマ + 取り込み UI（貼付 / 読込, 冪等 upsert）。初期は手入力で開始。
* T_007 比較ビュー
  - Status: ToDo
  - Blocked: T_003, T_005
  - 選んだ2系列を子供 × 日 で並置（spec 4.3）。例: 遊び申告 vs 実測 / 遊び vs 努力。システムは差を強調・判定しない（ADR-0009）。気づきは子供に委ねる。
* T_008 自己申告導線
  - Status: ToDo
  - Blocked: T_001, T_004, T_007
  - 比較ビューから遡及消費。申告事実を可視化。
* T_009 2倍ルール連動
  - Status: ToDo
  - Blocked: T_004, T_007
  - 未申告かつ親確認の隠しを2倍へ。

### M_003 スクリーンタイム解析くん（スクショ → 構造化データ, PoC 先行）

* T_006 解析くん（スクショ → AI 構造化）
  - Status: ToDo
  - Blocked: None
  - スクショから T_005 のスキーマ JSON を出力する外部ツール。PoC 先行・アプリ非依存。