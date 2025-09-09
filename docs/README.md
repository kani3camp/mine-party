# ドキュメント

このフォルダは本プロジェクトの「仕様書」「設計書」および関連ドキュメントを管理します。すべて Markdown で記述し、レビュー可能な形で継続的に更新します。

## 構成

- `spec/`: 仕様書（要件定義、機能仕様、API、UI 等）
- `design/`: 設計書（基本設計、詳細設計、DB スキーマ、アーキテクチャ等）
- `adr/`: ADR (Architecture Decision Record)
- `glossary.md`: 用語集

## 運用方針

- 言語: 日本語（必要に応じて英語併記可）
- 形式: Markdown。図は Mermaid 記法や画像を利用
- 命名: 章立て順に `01_...`, `02_...` のように連番プレフィックスを推奨
- レビュー: PR ベースでレビューし、議論は ADR に記録

## まずはここから

1. `spec/01_requirements.md` に要件のたたき台を記載
2. `spec/02_functional_spec.md` に主要ユースケース・受け入れ基準を記載
3. `design/01_high_level_design.md` に全体構成・技術選定を記載
4. 意思決定は `adr/` に記録

