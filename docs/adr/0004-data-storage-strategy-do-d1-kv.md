# データストレージ戦略（Durable Objects + D1 + KV）

- 日付: YYYY-MM-DD
- ステータス: Accepted
- 関連: ADR-0003（ランタイム選定）, 設計書 01/02/04, 仕様書 01

## 背景 (Context)
- MVP はルーム内の整合性と順序保証が最重要。外部永続化は必須ではない。
- 将来的にユーザー/試合履歴/ランキングを扱う見込みがある。

## 決定 (Decision)
- ルーム状態は Durable Objects（DO）に保持し、MVP は外部DB無しで運用する。
- 将来の永続化は Cloudflare D1（SQLite）を採用し、読みやすい集計/照会を行う。
- 軽量なキー値用途（デイリーシード/トークン/レート制限等）は Cloudflare KV を任意で併用する。

## 根拠 (Rationale)
- DO: "ルーム=1オブジェクト" による直列処理と揮発的保存で対戦の一貫性を担保。
- D1: SQL による履歴/ランキング/プロフィールの集計・検索が容易。
- KV: 低レイテンシかつコスト効率の良いキャッシュ/メタデータ保管。

## 影響 (Consequences)
- ルーム終了と同時に DO 内の状態は破棄（必要に応じスナップショットをD1へ書き出し）。
- D1 は users/matches/match_players を中心に最小スキーマで開始、段階的に拡張。
- KV は導入/運用が容易だが、強整合性は不要な用途に限定。

## 将来スキーマ（D1, 概要）
- users(id, google_sub?, name, created_at)
- matches(id, room_id, mode, width, height, mines, seed, started_at, ended_at, winner_user_id?)
- match_players(match_id, user_id, score, cleared, mistakes, time_spent_ms, result)

## 参考 (References)
- Cloudflare Durable Objects / D1 / KV 公式ドキュメント
