# 水とでんきの救急センター - 設計ドキュメント(公開用)

このリポジトリは「水とでんきの救急センター」MVPの**設計ドキュメントのみ**を公開するものです。

## 目的

Claude・ChatGPTなど複数のAIツールが、同じ最新の前提条件(設計書)を参照した上でレビュー・提案を行えるようにするための、共有ドキュメント置き場です。

## 実装コード・機密情報について

このリポジトリには実装コード、環境変数(.env)、Bubble/Make/LINE等の設定値、APIキー、Webhook URLといった機密情報は一切含まれていません。これらは別の非公開リポジトリで管理しています。

## 開発憲章

**設計書が更新されていない状態では、Bubbleの構造(データモデル・業務フロー・技術選定)を変更しない。**

```
要件変更 → vision.md / data_model_phase1.md 更新 → 必要ならADR追加 → Bubble変更
```

AIは提案者であり、決定者ではありません。最終的な意思決定は常にプロダクトオーナーが行います。

## ドキュメント一覧

| ファイル | 役割 |
|---|---|
| [`docs/vision.md`](docs/vision.md) | 事業の目的・MVPで検証すること・KPI・やらないこと |
| [`docs/assumptions.md`](docs/assumptions.md) | 検証すべき仮説一覧(週次で更新される) |
| [`docs/data_model_phase1.md`](docs/data_model_phase1.md) | ドメインモデル・エンティティ設計 |
| [`docs/business_workflow.md`](docs/business_workflow.md) | 顧客・加盟店・運営・例外の業務フロー |
| [`docs/adr/`](docs/adr/) | 技術選定・アーキテクチャの意思決定記録(ADR) |
