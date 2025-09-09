# ランタイム/ホスティング選定（Cloudflare Workers + Durable Objects）

- 日付: YYYY-MM-DD
- ステータス: Accepted
- 関連: 仕様書 01_requirements.md, 04_api_spec.md / 設計書 01_high_level_design.md

## 背景 (Context)
- 個人開発・低運用コスト・スマホ中心・リアルタイム対戦の要件。
- ルーム単位の順序保証/状態管理が必要（対戦ロジック/再接続復元）。

## 決定 (Decision)
- バックエンド: Cloudflare Workers + Durable Objects（DO）
- フレームワーク: Hono（TypeScript）
- ローカル/デプロイ: Wrangler（Miniflare）
- フロント: Vite + React を Cloudflare Pages にデプロイ
- 型/検証: TypeScript + Zod
- 通信: WebSocket（JSON ペイロード）

## 根拠 (Rationale)
- 運用: サーバ運用不要、無料〜低額で開始可能。
- リアルタイム: DO によるルーム=1オブジェクトでの直列化/状態保持が容易。
- レイテンシ: エッジ実行で地域近接、スマホでも快適。
- AI開発適性: Hono/Workers はボイラープレートが少なく、生成/保守がしやすい。

## 影響 (Consequences)
- 各ルームに対応する DO を作成し、接続と状態を一元管理。
- 水平スケールはルームキーの自動シャーディングに依存、共有ストア（KV/D1）は将来検討。
- ローカル実行は Miniflare を利用、WS/DO を統合して動作確認。

## リスク/制約
- 一部 Node ネイティブ API 非対応（Workers ランタイム準拠が必要）。
- 永続性要件が増えた場合は D1/KV/R2 等の採用を追加検討。

## 実施計画 (Implementation)
- HTTP エンドポイント: ルーム作成/参加/ヘルスチェック（Hono）
- DO: ルーム単位で作成。WS 接続管理/順序付け/スナップショット/進捗集計。
- フロント: Pages にビルド成果を配置、環境変数で API/WS エンドポイントを注入。

## 参考
- Cloudflare Workers / Durable Objects / Hono / Wrangler / Miniflare ドキュメント
