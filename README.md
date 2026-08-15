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
| [`vision.md`](vision.md) | 事業の目的・MVPで検証すること・KPI・やらないこと |
| [`assumptions.md`](assumptions.md) | 検証すべき仮説一覧(週次で更新される) |
| [`data_model_phase1.md`](data_model_phase1.md) | ドメインモデル・エンティティ設計 |
| [`business_workflow.md`](business_workflow.md) | 顧客・加盟店・運営・例外の業務フロー |
| [`line_channel_design_phase1.md`](line_channel_design_phase1.md) | LINEチャネル設計(責任分界・Customer設計・本人性検証・LINE-04方針) |
| [`partner_acquisition.md`](partner_acquisition.md) | 加盟店開拓プロセス(候補抽出・ヒアリング項目) |
| [`poc/LINE-03-liff-bubble-poc.md`](poc/LINE-03-liff-bubble-poc.md) | LINE-03: LIFF + Bubble PoC 検証手順・受け入れ基準・検証ログ |
| [`adr/`](adr/) | 技術選定・アーキテクチャの意思決定記録(ADR) |
| [`AI_COLLABORATION.md`](AI_COLLABORATION.md) | Claude・Codex・ChatGPT等が共同作業する際のルール(正本・境界・取得手順・ADR優先原則等) |

### ADR一覧

| ADR | タイトル |
|---|---|
| [ADR-001](adr/ADR-001-bubble.md) | プラットフォーム本体にBubbleを採用する |
| [ADR-003](adr/ADR-003-phase1-service-area.md) | Phase1のサービスエリアを千葉県柏市とする |
| [ADR-004](adr/ADR-004-line-channel-responsibility.md) | LINE連携の責任分界 |
| [ADR-005](adr/ADR-005-line-identity-verification.md) | LINE本人性検証はIDトークン検証APIを用いる |

(ADR-002は決済モデルに関する意思決定として他ADR内で言及されているが、本リポジトリには未作成)

## 本体リポジトリとの関係

本体(非公開)リポジトリでは、このリポジトリが `docs/` としてgit submoduleでマウントされています。設計書の実体はこのリポジトリのみに存在し、本体側には複製を持ちません。

## 複数AIでの共同作業について

Claude・ChatGPT・Codexなど複数のAIツールがこのプロジェクトに関わる場合は、[`AI_COLLABORATION.md`](AI_COLLABORATION.md) を必ず参照してください。要点は以下の通りです。

- **GitHubをSingle Source of Truthとする。** ローカルの作業フォルダやAIセッションの記憶を正本としない。ローカルコミットだけで作業完了にせず、pushして反映する。
- **設計書・ADR・Decision・検証記録は、この公開docsリポジトリへ置く。** private親リポジトリだけに設計Decisionを置かない。
- **private親リポジトリには実装コード・画像資産・秘密情報を含み得る設定を置く。** このdocsリポジトリには実装コード・機密情報を含めない。
- **AIレビューを依頼する際は、GitHubのtreeナビゲーションに依存せず、対象ファイルの直接URL(blobリンク)を渡す。** 上記「ドキュメント一覧」「ADR一覧」の直接リンクを利用すること。
- **private親リポジトリのコードレビューには認証済みGitHub接続(GitHub CLIまたは認証済みGit)が必要。** 未認証のWebアクセスでは404になるが、これは「存在しない」ことを意味しない。アクセスできない場合はその旨を報告し、存在しないと判断しない。
- **意思決定はADRを優先する。** ADRと設計書の記述が食い違う場合はADRを正とする。
