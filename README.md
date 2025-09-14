# mine-party

Initial repository.

## ドキュメント

- 仕様書/設計書は `docs/` 配下に整理しています。
  - 一覧: docs/README.md

## 開発（MVP スキャフォールド）

- ランタイム: Cloudflare Workers + Durable Objects（Hono + Wrangler）
- フロント: Vite + React（Cloudflare Pages）
- 共有: `packages/protocol` に Zod スキーマ（WS メッセージ）

### セットアップ（ローカル）

1. Node 18+ と pnpm を用意
2. 依存関係インストール（ネット接続が必要）
   - `pnpm install`
3. Workers 側を起動
   - `pnpm dev:workers`
4. Web 側（別ターミナル）
   - `pnpm dev:web`

補足
- DO のバインドは `workers/wrangler.toml` を参照（初回は migraions 自動）
- `POST /rooms` で部屋作成、`/ws/:roomId` へ接続（現状はスタブ）
