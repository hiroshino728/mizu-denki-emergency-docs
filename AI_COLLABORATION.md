# AI共同作業ガイド

このガイドは、「水とでんきの救急センター」Phase 1の設計・開発に、Claude・Codex・ChatGPT・その他のAIツールが関わる際の共通ルールをまとめたものです。複数のAIが別々のセッション・別々の前提で作業すると、設計判断が食い違ったり、同じ議論を繰り返したりするリスクがあります。このガイドはそれを防ぐための最低限の合意事項です。

## 1. 正本リポジトリ（Single Source of Truth）

**GitHubが正本（Single Source of Truth）です。** ローカルの作業フォルダやAIセッション内の記憶を正本とみなさないでください。

- 親リポジトリ（public）: https://github.com/hiroshino728/water-denki-emergency-mvp
- docsリポジトリ（public）: https://github.com/hiroshino728/mizu-denki-emergency-docs

上記は2026-08-18時点でGitHub APIにより確認した現在値です。Repository visibilityの変更はCEO Decision事項であり、AIエージェントは独断で変更してはいけません。作業開始時にはGitHubの現在値を再確認してください。

作業を始める前に、必ずGitHubの最新の`main`ブランチを取得してから着手してください。ローカルコミットを作っただけで作業完了とせず、pushしてGitHubへ反映するところまでを一つの作業単位とします。

## 2. 親リポジトリとdocsサブモジュールの関係

- 親リポジトリには実装コード・画像アセット・秘密値を含まない設定を置きます。APIキー、token、password、secret付きWebhook URL等の秘密値は、GitHub Actions Secrets等の承認済みsecret storeで管理し、リポジトリ、Issue、PR、Actionsログへ書き込みません。
- 設計書・ADR・意思決定（Decision）・検証記録は、**このdocsリポジトリへ置きます。** 親リポジトリだけに設計Decisionを置くことはしません。
- 親リポジトリでは、このdocsリポジトリが`docs/`ディレクトリとしてGit submoduleでマウントされています。**docsと親は別のGit履歴です。** docs側の変更を先にdocsリポジトリのmainへマージし、その後で親リポジトリのsubmodule参照コミットを更新する、という順序を必ず守ってください（6章参照）。

## 3. Public repositoryの公開境界

- **親リポジトリとdocsリポジトリは、どちらも公開（public）です。** コード、設計書、Issue本文・コメント、PR、commit、公開Actionsのlog・summary、および明示的にuploadしたartifactは、第三者に閲覧され得るものとして扱います。
- 公開してよいのは、公開済みの設計・実装情報、秘密値を除いた設定、合成または十分に匿名化されたテストデータ、公開を前提に作成した検証記録です。
- Secrets、API key、token、password、secret付きWebhook URL、LINE User IDの実値、顧客・加盟店の個人情報、料金戦略、加盟店交渉方針、実加盟店ヒアリング結果、未公開の契約条件、非公開事業戦略、これらを含み得るAPI生レスポンスは、両リポジトリのcommit・Issue・コメント・PR・Actions log/summary/artifact・スクリーンショットへ一切含めません。
- 親リポジトリのAI discussion pipelineは、公開可能と確認済みの議題または合成テスト議題に限定します。機密情報を含む、または含む可能性を否定できない議題では実行を禁止します。内容をGitHubへ転記せず、必要最小限の非機密情報だけでCEOへrouting判断を求めます。今回、新しいprivate repositoryやconfidential control planeは作りません。
- Repository visibilityを将来変更するか、confidential routing先を設けるかはCEO Decisionとして別途判断します。現在Publicである事実を、AIが運用上の推測で上書きしてはいけません。

## 4. 作業開始時の取得手順

サブモジュール込みで最新のmainを取得してから作業を始めてください。

```bash
git switch main
git pull --ff-only origin main
git submodule sync --recursive
git submodule update --init --recursive
git -C docs switch main
git -C docs pull --ff-only origin main
```

意図不明の変更や未追跡ファイルが見つかった場合、削除・上書きせず、内容をユーザーへ報告してください。

## 5. Repository visibilityとアクセス失敗の扱い

両リポジトリは2026-08-18時点でPublicです。作業開始時は次のようにGitHubの現在値を確認してください。

```bash
gh auth status
gh repo view hiroshino728/water-denki-emergency-mvp
```

上記でアクセスできない場合、「privateだから404」「存在しない」と推測しないでください。ネットワーク、認証、repository名変更、またはCEO Decisionによるvisibility変更の可能性を区別し、GitHub API等の一次情報で確認できなかった事実をそのまま報告してください。

公開docsリポジトリの資料は、GitHubのtreeナビゲーション（ディレクトリを辿るUI）に依存せず、可能な限り対象ファイルの直接URL（blobリンク）を使ってください。ナビゲーションに頼ると、古いキャッシュや誤ったパスを参照してしまうリスクがあります。README.mdの「主要ドキュメント直接リンク」を参照してください。

## 6. docs → 親のコミット・push・マージ順

親リポジトリのsubmodule参照は、docsリポジトリの特定のコミットSHAを指しています。そのため、**docs側の変更を必ず先にdocsリポジトリのmainへマージし、それが完了してから**親リポジトリのsubmodule参照を更新してください。

1. docsリポジトリでブランチを作成し、変更をコミット・push
2. docsリポジトリでPRを作成し、レビュー後にmainへマージ（squash mergeは使わない。親のsubmodule参照コミットがdocsのmain履歴から到達可能である必要があるため、merge commitまたはfast-forwardを使う）
3. `git merge-base --is-ancestor <docsコミット> origin/main` で、そのコミットがdocsのmainから到達可能なことを確認
4. 確認できた場合のみ、親リポジトリでブランチを作成し、`git submodule update` でdocsを対象コミットへ進め、submodule参照の変更をコミット・push・PR作成
5. 親側PRをマージ

この順序を逆にする（先に親側のsubmodule参照だけを更新する）と、参照先のdocsコミットがdocsリポジトリのmainから辿れない「宙に浮いた」状態になり得るため避けてください。

## 7. 設計書の優先順位・ADRを優先する原則

設計判断について複数の文書が食い違って見える場合、以下の優先順位で判断してください。

1. **ADR（`adr/`配下）** — 正式に決定（Accepted）された意思決定記録。最も優先度が高い。
2. **各設計書（`data_model_phase1.md`、`business_workflow.md`、`line_channel_design_phase1.md`等）** — ADRの決定を反映した実装レベルの詳細。ADRと矛盾する記述が残っている場合はADRを正とし、設計書側の記述漏れとして報告する。
3. **`vision.md` / `assumptions.md`** — 事業レベルの前提・仮説。個別のデータモデル・フロー判断より抽象度が高い。
4. **検証記録（`poc/`配下）** — 実機で確認した事実（Fact）の記録。設計の妥当性を裏付けるが、検証記録自体が設計のDecisionを覆すことはない（Decisionを見直す場合は改めてADR/設計書を更新する）。

新しいADRを作成する前に、必ず`docs/adr/`配下の既存ファイル一覧を確認し、番号の衝突を避けてください。

```bash
find docs/adr -maxdepth 1 -type f -name 'ADR-*.md' -print | sort
```

## 8. Assumption／Decision／実機Factの区別

文書やコミットメッセージを読む際、以下の3種類を区別してください。混同すると、まだ検証していない仮説を確定事項として扱ってしまうリスクがあります。

- **Assumption（仮説）**: `assumptions.md`に記載される、まだ検証されていない前提。「〜だろう」「〜と想定する」という書き方。
- **Decision（決定）**: ADRや設計書に「決定」「採用」として明記された、プロダクトオーナーが下した意思決定。AIが提案はできるが、決定するのは常にプロダクトオーナー。
- **実機Fact（検証済み事実）**: `poc/`配下の検証ログのように、実際に動かして確認した結果。TC-01のように受け入れ基準に対するPASS/FAIL等で記録される。

## 9. 個人情報・User ID・トークンの禁止事項

親リポジトリとこのdocsリポジトリは、どちらも公開リポジトリです。以下を、コミット・PR本文・Issue・コメント・Actions log/summary/artifact・検証ログ・スクリーンショットのいずれにも含めないでください。

- LINE User IDの実値
- IDトークン、アクセストークン、Channel Secret、APIキー
- 顧客・加盟店の氏名・電話番号・住所等の個人情報の実データ
- LINE APIの生レスポンス（トークンや個人情報を含み得るため）
- 料金戦略、加盟店交渉方針、実加盟店ヒアリング結果、未公開の契約条件、非公開事業戦略等の商用機密
- 機密情報・個人情報を含む、または公開可能と確認できていないOpenAI / Anthropic等のAPI生レスポンス

検証記録には「実値は未記録」「実機申告」のように、確認した事実の種類だけを記録し、値そのものは書かないでください。

## 10. AIが読めない資料を根拠に推測しないこと

リポジトリや特定のファイルを読めない場合、その内容やvisibilityを推測して設計判断の根拠にしないでください。読めなかった事実をそのままユーザーへ報告し、必要であれば認証・ネットワーク状態の確認や直接URLの提供を依頼してください。「おそらくこうなっているはず」という推測に基づいて設計書を書き換えることは避けてください。

## 11. 関連ドキュメント

- [README.md](README.md) — 主要ドキュメント一覧・直接リンク
- [adr/ADR-004-line-channel-responsibility.md](adr/ADR-004-line-channel-responsibility.md) — LINE連携の責任分界
- [adr/ADR-005-line-identity-verification.md](adr/ADR-005-line-identity-verification.md) — LINE本人性検証方式
