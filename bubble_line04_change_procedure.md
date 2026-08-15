# LINE-04 Bubble変更手順書

## 1. 目的と実行境界

本手順書は、LINE-04で確定したCustomer / ChannelIdentity設計をBubbleに反映するための作業手順を定める。設計の正本は以下とする。

- `data_model_phase1.md` 2.1節「Customer」、2.2節「ChannelIdentity」
- `adr/ADR-004-line-channel-responsibility.md`
- `adr/ADR-005-line-identity-verification.md`
- `business_workflow.md`の「LINEチャネルにおけるCustomer作成タイミング」
- `line_channel_design_phase1.md` 13章

> **実行禁止:** 本手順書の作成・レビュー段階では、Bubble Production環境のData type、Field、Privacy Rule、Workflowを変更しない。実装は承認後の別IssueでDevelopment環境から行い、Productionへの反映には篠さんの明示承認を必須とする。

## 2. 変更の対象と対象外

### 対象

- Customerの現行定義の確認
- 新規Data type `ChannelIdentity`
- CustomerとChannelIdentityのリレーション
- IDトークン検証用バックエンド処理
- 検証済み`sub`からChannelIdentity / Customerを検索・作成するIdentity resolver
- 作成前検索、重複検知、Privacy Rules、安全なエラー処理
- Development環境のテストデータによる確認

### 対象外

- Production環境の構造変更とデプロイ
- 実ユーザーのCustomer / ChannelIdentity作成
- LINE-05以降の修理依頼フォーム、Repair Request、写真、緊急度判定、Make.com連携、本番LINE通知
- 電話番号による自動名寄せ

## 3. 作業前チェック

1. 対象Issueと設計正本がレビュー済みであることを確認する。
2. Bubbleで対象アプリの **Development** versionを開き、Productionでないことを画面上で再確認する。
3. 現在のData types、Privacy Rules、Backend workflowsをスクリーンショットまたは構成メモで記録する。User ID、トークン、個人情報の実値は記録しない。
4. Customerに`line_user_id`または同等の旧Fieldがあるか、そのFieldを参照するWorkflow・Page・APIがあるかを検索する。存在してもこの時点で削除しない。
5. 変更対象とロールバック対象をIssueに記録する。

## 4. Data typeとFieldの作成

### 4.1 Customerの確認

Customer直下に新しいLINE固有IDのFieldを追加しない。Customerに必要な業務データは`data_model_phase1.md` 2.1節に従う。

### 4.2 ChannelIdentityの作成

Bubble EditorのData > Data typesで`ChannelIdentity`を新規作成し、次のFieldを追加する。

| Bubble Field | Bubble type | List | 必須条件・用途 |
|---|---|---:|---|
| `customer_id` | Customer | No | 対応するCustomerへの参照。作成時に必ず設定 |
| `channel` | text | No | Phase 1は`LINE`のみ。保存値の大文字小文字を統一 |
| `channel_user_id` | text | No | LINE検証APIで確認済みの`sub`のみ。入力画面や公開APIから直接設定しない |
| `identity_key` | text | No | `line:` + 検証済み`sub`。作成前検索と重複検知に使用 |

`created_at` / `updated_at`はBubble標準の`Created Date` / `Modified Date`を設計上の該当項目として使う。同じ意味のカスタムFieldは重複作成しない。

### 4.3 リレーションの設定

- ChannelIdentityからCustomerへの参照は`customer_id` 1本とする。
- 1 Customerに複数ChannelIdentityを紐付けられる1対多とする。
- Customer側にChannelIdentityのList Fieldを重複作成しない。必要時は`Search for ChannelIdentities` with `customer_id = Customer`で取得する。
- Bubble標準DBに厳密なunique constraintはないため、`identity_key`をuniqueとみなすのはワークフロー運用上の保証であり、DBレベルの原子性を前提としない。

## 5. Privacy Rulesの設定

Development環境でChannelIdentityのPrivacy Rulesを作成する。

1. 未認証ユーザーに`Find this in searches`、`View all fields`、API経由の読み取りを許可しない。
2. クライアント画面から`channel_user_id`と`identity_key`を読み取れるRuleを作らない。
3. Customerは未認証ユーザーの一覧検索を許可せず、他顧客のレコードを参照できないRuleにする。
4. IDトークン検証、ChannelIdentity検索・作成はバックエンドワークフローに限定し、公開データAPIからCreate/Modifyさせない。
5. App data APIでChannelIdentityを公開しない。すでに公開設定がある場合は、依存調査後にDevelopment環境で無効化する。

Privacy Ruleを緩和してテストを通さない。バックエンドに必要な権限と、ブラウザへのデータ開示は分けて検証する。複数のPrivacy Ruleが同時に成立する場合、いずれかのRuleが許可すればデータにアクセスできるため、ChannelIdentityに広すぎる別Ruleがないことも確認する。

## 6. IDトークン検証ワークフロー

### 6.1 受付入力

- LIFF側は`liff.getIDToken()`の戻り値をBubbleバックエンドに送る。
- `liff.getProfile().userId`や、ブラウザが送信したUser ID文字列をIdentityの根拠として受け取らない。
- IDトークンはWorkflowの一時入力とし、ThingのField、URL、画面、ログに保存しない。
- Bubble API Workflowは原則としてPrivacy Rulesに従う。ChannelIdentity / Customerの検索・作成でPrivacy Rulesを越える必要がある場合に限り、必要なWorkflowだけで`Ignore privacy rules when running the workflow`を有効にする。
- `Ignore privacy rules when running the workflow`を有効にした場合でも、IDトークン検証の成功前にCustomer / ChannelIdentityの検索・作成・変更を一切行わない。
- Workflow入力はIDトークンなどの必要最小限に限定し、検証に失敗した場合はCustomer / ChannelIdentityへアクセスせず即時終了する。
- `Ignore privacy rules when running the workflow`は強力な設定のため、LINE-04で上記の必要性を確認したWorkflow以外には付与しない。

### 6.2 LINE検証API呼び出し

Bubbleサーバー側から次を呼び出す。具体的な認証値はBubbleの秘密設定で管理し、手順書・Issue・ログに転記しない。

- Method: `POST`
- URL: `https://api.line.me/oauth2/v2.1/verify`
- Content-Type: `application/x-www-form-urlencoded`
- Body: `id_token=<ID token>&client_id=<LINE Login channel ID>`

Channel Secretは送信しない。初期化用の実トークンやLINE User IDをAPI Connectorの共有値・履歴・スクリーンショットに残さない。

### 6.3 検証結果の判定

後続処理に進むのは、次をすべて満たす場合だけとする。

- HTTP statusが200
- `iss` がLINEの想定発行者
- `aud` が設定したLINE Login channel IDと一致
- `exp` が現在時刻より後
- `sub` が存在し、空でない

不正、欠落、期限切れ、通信失敗の場合はCustomer / ChannelIdentityを検索・作成せず、利用者には「認証を確認できませんでした。時間をおいてもう一度お試しください。」等の一般メッセージのみを返す。生レスポンス、トークン、User ID、内部エラーは返さない。

## 7. Identity resolverと重複防止

1. 検証成功後の`sub`を一時変数とし、`identity_key = "line:" + sub`をサーバー側で生成する。
2. `Search for ChannelIdentities:first item`を`identity_key = generated identity_key`で実行する。
3. 既存ChannelIdentityが見つかった場合は、新規レコードを作らず、紐付くCustomerを後続処理に渡す。
4. 見つからない場合の作成は、LINE-05フォームで個人情報利用への同意と送信が完了した後に限る。LINE-04の開発テストでは明示的なテストフラグとテストデータだけを使う。
5. 新規時はCustomerを先に作成し、そのCustomerを`customer_id`に設定したChannelIdentityを1件作成する。
6. 作成直前に同じ`identity_key`で再検索し、既存が見つかった場合はCreateを実行しない。
7. 同時実行の完全な原子性は前提とせず、定期または管理者用の重複検知で同一`identity_key`の2件以上を検出できるようにする。

バックエンド処理の途中でChannelIdentity作成に失敗した場合、孤立Customerを放置しない。開発実装時に、エラーフラグで回収対象として検出するか、後続が失敗した場合にテストCustomerを削除する補償手順を選ぶ。選択した方式は実装Issueに記録する。

## 8. 既存Customerレコードの移行方針

1. Development / ProductionそれぞれでCustomer件数、旧`line_user_id` Fieldの有無、値あり件数、参照Workflowを棚卸しする。実値はIssueやログへ転記しない。Productionは読み取り確認の承認範囲を超えて変更しない。
2. 旧`line_user_id`は検証済み`sub`である証拠がない限り、ChannelIdentityへ自動コピーしない。
3. 既存Customerと検証済み`sub`を紐付ける必要がある場合は、別Issueで根拠と対象を確定し、手動移行とレビューを行う。電話番号だけで自動紐付けしない。
4. 旧Fieldの参照をすべて新resolverへ切り替え、テストを通過し、ロールバック期間を終えるまで旧Fieldを削除しない。
5. Productionの旧Field削除、一括移行、デプロイは、対象件数とロールバック手順を示したうえで、篠さんの明示承認後に別Issueで行う。

## 9. Development環境での実装順序

1. 作業前チェックと現行構成記録
2. ChannelIdentity Data type / Field
3. Privacy Rules
4. LINE IDトークン検証処理
5. Identity resolverと作成前再検索
6. 重複検知と安全なエラー処理
7. `bubble_line04_test_procedure.md`によるDevelopmentテスト
8. テストデータの削除と証跡記録
9. Production反映差分とロールバック手順のレビュー
10. **停止：人間の明示承認が得られるまでProductionへ反映しない**

## 10. ロールバック計画

Development環境で問題が出た場合は、新規受付フローを旧ワークフローへ戻し、ChannelIdentityへの新規書き込みを停止する。新Data typeとFieldは、参照がないこととテスト証跡の保全を確認するまで即時削除しない。Productionのロールバックは、必ず本番反映用の別Issueで詳細化する。

## 11. 完了判定

手順書の作成完了とBubble実装完了を区別する。本手順書がレビュー済みになっても、Bubble変更とテストが未実施である限りLINE-04実装は完了としない。

## 12. Bubble公式参照

- [Data tab](https://manual.bubble.io/core-resources/bubbles-interface/data-tab)
- [Privacy](https://manual.bubble.io/core-resources/data/privacy)
- [API workflows](https://manual.bubble.io/help-guides/integrations/api/the-bubble-api/the-workflow-api/api-workflows)
- [API Connector security](https://manual.bubble.io/help-guides/security/api-security/api-connector-security)
