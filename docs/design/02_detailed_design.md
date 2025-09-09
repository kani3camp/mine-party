# 詳細設計

- バージョン: 0.6 (ドラフト)
- 更新日: YYYY-MM-DD

- ## 1. モジュール/パッケージ構成（案）
- `RoomManager`: ルーム生成/参加/退出、ホスト管理、準備状態、ブロードキャスト
- `GameEngine`: 盤面生成（シード/初手セーフ）、操作処理（開く/旗/チョード）、勝敗/スコア判定
- `TurnManager`: 手番制御、1手制約、旗連打制限（3連続禁止）
- `ChessClock`: 残り時間管理（初期60s、手番後 +2s）、timeout 検出
- `StateStore`: ルーム/ゲーム状態の保管（DO）、スナップショット
- `RealtimeGateway`: WS 接続管理、認証、流量制御、順序付け
- `Protocol`: イベント型/バリデーション（Zod）

## 2. インターフェース仕様（抜粋）
- `RoomManager.create(settings) -> { roomId, hostId }`
- `RoomManager.join(roomId, nickname) -> { playerId, token }`
- `RoomManager.listPlayers(roomId) -> Player[]`
- `GameEngine.start(roomId, settings) -> { seed, snapshot }`
- `TurnManager.current(roomId) -> { playerId, turn }`
- `TurnManager.commit(roomId, playerId, action) -> CommitResult` （自分のターンでのみ成功）
- `GameEngine.reveal(roomId, playerId, r, c) -> RevealResult{ updates[], ended? }`
- `GameEngine.flag(roomId, playerId, r, c, flagged) -> { r, c, flagged }`
- `GameEngine.chord(roomId, playerId, r, c) -> RevealResult`
- `ChessClock.tick(roomId, now) -> void`（サーバー側で定期/イベント駆動）
- `ChessClock.applyIncrement(roomId, playerId, ms) -> void`
- `GameEngine.progress(roomId) -> Progress`

## 3. データモデル（MVP）
- `Room`: `{ id, hostId, players: Player[], settings, state: 'idle'|'playing'|'ended', mode: 'turn-1v1' }`
- `Player`: `{ id, name, joinedAt }`
- `GameState`: `{ seed, width, height, mines, startedAt, boardHidden, boardPublic, flagsShared, scores, clocks, turn, turnHistory, mistakes }`
  - `boardHidden`: 地雷配置と数字値（サーバーのみ）
  - `boardPublic`: 共有盤面の公開状態
  - `flagsShared`: 共有フラグ集合（例: Set<coord> もしくは 2D boolean）
  - `scores`: `playerId -> { score, cleared, flags }`
  - `clocks`: `playerId -> remainingMs`, `timeControl: { initialMs, incrementMs }`
  - `turn`: `{ currentPlayerId, number, consecutiveFlagOnlyCountByPlayer }`
  - `turnHistory`: 直近の手のログ（行為/座標/時刻/所要/連鎖規模 など）
  - `mistakes`: `playerId -> count`（誤旗/誤チョード等の計数）

## 4. 状態管理
- ルーム単位で Durable Objects に保持（MVP）。
- スナップショット: 接続時/再接続時に `boardPublic` とメタ情報（scores/clocks/turn）を送信。
- 永続化: MVP は不要。将来、終了時サマリを D1 に書き出し（非同期）。

## 5. アルゴリズム/ロジック（要点）
- 盤面生成: `seed` に基づく擬似乱数で配置。初手が地雷の場合は地雷を移動
- 連鎖開放: 0 の BFS/DFS 展開、更新セルはバッチで通知
- チョード: 周囲旗数==数字で一括開放。誤旗混入時は即敗北（地雷ヒット扱い）
- 判定: 盤面解了時はスコア比較で勝利。その他は地雷ヒット/時間切れで決着
- 冪等性: `opId` を受付キューで重複排除
- 共有フラグ: `flagsShared` を単一ソースとし、全プレイヤーの入力で更新。`boardPublic` は共有盤面
- ターン制: `TurnManager` が 1手を原子処理→`ChessClock` に増分を適用→`turn` を交代
- 旗連打制限: `consecutiveFlagOnlyCountByPlayer[p]` が 3 到達時は `flag` を `illegal_move` として拒否
- 連鎖ボーナス: 連鎖規模に応じた `bonusMs` を `ChessClock` に適用（上限 1000ms）
- 終了判定: 地雷ヒット即終了 / `remainingMs<=0` で時間切れ終了 / 盤面解了時はスコア比較→タイブレーク

## 8. 永続化戦略（将来）
- D1（SQL）:
  - users(id, google_sub?, name, created_at)
  - matches(id, room_id, mode, width, height, mines, seed, started_at, ended_at, winner_user_id?)
  - match_players(match_id, user_id, score, cleared, mistakes, time_spent_ms, result)
- KV（任意）:
  - デイリーシード、参加トークン、軽いレート制限

## 6. エッジケース/例外
- 同時操作: ターン外の操作は `not_your_turn`。有効ターン内は直列処理
- 再接続: 無効トークン/期限切れは再参加を要求
- 違反: 非法手（opened/flagged 不整合, flag-stall）には `illegal_move` を返却しターンは維持

## 7. テスト観点
- 盤面生成の決定性（同一 `seed` → 同一配置）
- 初手セーフ性、連鎖開放の正確さ、チョード条件
- 同時操作時の整合性、冪等性、再送耐性
 - ターン制のクロック消費/増分/同期の正確性、勝敗確定の公平性
