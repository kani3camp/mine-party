# API 仕様書

- バージョン: 0.4 (ドラフト)
- 更新日: YYYY-MM-DD

## 1. 概要
- ルーム作成/参加などは HTTP、リアルタイム更新は WebSocket を利用。

## 2. 認証/認可
- ルーム内トークン: 参加時にサーバーが発行する短命トークン（WS 接続に使用）
- 権限: `host` / `player`（MVP）
- 未来の拡張（メモ）: Google アカウントログインを想定（MVP では未実装）

## 3. 共通仕様
- ベース URL: `https://api.example.com`
- タイムスタンプ: RFC 3339（UTC 固定, `Z` サフィックス。例: `2025-01-01T00:00:00Z` または `2025-01-01T00:00:00.123Z`）
- エラー形式: `{ code: string, message: string }`
 - 時間管理: サーバーがクロックのソース・オブ・トゥルース。クライアントは `clock:sync` を受けて表示更新。

## 4. HTTP エンドポイント（MVP）
- POST `/rooms`
  - 概要: ルームを作成
  - Body: `{ nickname: string, settings?: { width: number, height: number, mines: number, timeControl?: { initialMs: number, incrementMs: number } } }`
  - 200: `{ roomId: string, role: 'host', playerId: string, token: string, wsUrl: string }`
- POST `/rooms/{roomId}/join`
  - 概要: ルームに参加
  - Body: `{ nickname: string }`
  - 200: `{ roomId: string, role: 'player', playerId: string, token: string, wsUrl: string }`
- GET `/rooms/{roomId}`
  - 概要: ルーム概要（人数、設定、状態）
  - 200: `{ roomId, players: [...], settings, state: 'idle'|'playing'|'ended' }`
  

## 5. WebSocket（イベント駆動）
- 接続: `wss://ws.example.com?roomId=...&playerId=...&token=...`
- メッセージ形式: JSON、`{ type: string, data: object, opId?: string }`

### クライアント→サーバー（例）
- `room:join` `{ nickname }`（HTTP 参加後は不要）
- `game:start` `{ settings? }`（host のみ）
- `cell:reveal` `{ r: number, c: number }`
- `cell:flag` `{ r: number, c: number, flagged: boolean }`
- `cell:chord` `{ r: number, c: number }`
  - 備考: これらは自分のターンでのみ受理。`error.not_your_turn` を返す場合あり。

### サーバー→クライアント（例）
- `room:state` `{ players: [...], settings, hostId }`
- `game:started` `{ seed: string, width, height, mines, startedAt, timeControl: { initialMs, incrementMs } }`
- `turn:state` `{ currentPlayerId, turn: number }`
- `clock:sync` `{ players: Array<{ playerId, remainingMs }> }`
- `cell:revealed` `{ updates: Array<{ r, c, value: number | 'M' }> }` ※連鎖はバッチで配信（共有盤面のため全員へ）
- `cell:flagged` `{ r, c, flagged: boolean, by: playerId }`
- `score:update` `{ players: Array<{ playerId, score: number, cleared: number, flags: number, mistakes: number }>, chainBonusMs?: number }`
- `game:progress` `{ players: Array<{ playerId, cleared: number, flags: number, mistakes: number, timeMs: number }> }`
- `game:ended` `{ result: 'win'|'lose'|'timeout', winners: playerId[], elapsedMs }`
- `error` `{ code, message }`

### エラーコード例
- `not_your_turn`
- `illegal_move`（開けないセル、旗連打制限 超過 等）
- `room_not_found` / `unauthorized`

## 6. スキーマ定義（抜粋）
- `CellValue = 0..8 | 'M'`
- 盤面座標は 0 始まりの `r`(row), `c`(col)

## 7. レート制限 / 互換性方針
- 連投抑制: 1 クライアントの操作頻度にソフトリミット（例: 50 ops/10s）
- 互換性: `type` 名称の非互換変更はメジャーアップデートで実施

## 8. 共有フラグの扱い
- `cell:flag` はルーム全員にブロードキャストされ、全クライアントで同じセルが旗/解除される。
- `cell:revealed` は旗が立っているセルに対しては無視（先に解除が必要）。

## 9. 永続化（MVP/将来）
- MVP: ルーム状態は Durable Objects に保持し、外部DBには保存しない。
- 将来: 試合終了時に D1 へ試合結果を保存し、ランキング/履歴APIを追加予定。
