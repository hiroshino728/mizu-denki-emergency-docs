# LINE-04 Bubbleテスト手順書

## 1. 目的と制約

本手順書は、`bubble_line04_change_procedure.md`に従ってBubble Development環境へ実装するCustomer / ChannelIdentity、IDトークン検証、Identity resolver、Privacy Rulesの受入テストを定める。

> **未実施:** 本文書作成時点でBubble環境の変更とテスト実行は行っていない。すべてDevelopment環境で実施し、Production構造を変更しない。

## 2. テスト前提

- 対象がBubble **Development** versionであることを2名または画面証跡で確認する。
- 個人情報を含まないテストCustomerを使う。
- LINE User ID、IDトークン、アクセストークン、Channel Secretの実値をIssue、テスト証跡、スクリーンショット、URL、ログに残さない。
- 不正系はDevelopment専用のモックまたは非公開テストエンドポイントから実行する。検証を省略して任意の`sub`を本番用resolverに渡せる公開経路は作らない。
- 各テストの前後でCustomer / ChannelIdentity件数を記録する。記録は件数とマスク済みテスト識別子だけとする。

## 3. 判定記号

- `PASS`: 期待結果をすべて満たした
- `FAIL`: 期待結果と異なる、または機密情報が露出した
- `BLOCKED`: 必要な承認、テスト環境、外部APIが用意できず未実施

`FAIL`が1件でもある場合はProduction反映候補としない。

## 4. 受入テスト

### 4.1 Data typeとリレーション

| ID | 操作 | 期待結果 |
|---|---|---|
| TC-DM-01 | ChannelIdentityのField定義を確認 | `customer_id: Customer`、`channel: text`、`channel_user_id: text`、`identity_key: text`が非Listで存在する |
| TC-DM-02 | テストCustomer 1件に異なるテストChannelIdentity 2件を紐付ける | 2件とも同じCustomerを参照でき、1対多として取得できる |
| TC-DM-03 | CustomerのFieldと関連Workflowを確認 | Customer直下の`line_user_id`に新たな依存がなく、LINE識別はChannelIdentityに分離されている |
| TC-DM-04 | ChannelIdentityのCreated Date / Modified Dateを確認 | 設計上の`created_at` / `updated_at`として追跡できる |

### 4.2 IDトークン検証

| ID | 入力・状況 | 期待結果 |
|---|---|---|
| TC-AUTH-01 | Developmentの認証済みテストアカウントから有効なIDトークンを送信 | LINE検証APIがHTTP 200を返し、`iss / aud / exp / sub`検査後の`sub`だけがresolverへ渡る |
| TC-AUTH-02 | IDトークンの1文字を改ざん | 拒否され、Customer / ChannelIdentityは作成されない |
| TC-AUTH-03 | 期限切れ応答をDevelopment専用テスト経路で再現 | `exp`検査で拒否され、DB書き込みがない |
| TC-AUTH-04 | `aud`不一致をDevelopment専用テスト経路で再現 | 拒否され、DB書き込みがない |
| TC-AUTH-05 | `sub`欠落・空をDevelopment専用テスト経路で再現 | 拒否され、DB書き込みがない |
| TC-AUTH-06 | LINE APIタイムアウトまたは5xxを再現 | 安全な一般エラーのみを返し、再試行可能。DB書き込みはない |
| TC-AUTH-07 | クライアントからUser ID文字列だけを送信 | Identityの根拠として受理されず、Customer / ChannelIdentityは作成されない |

### 4.3 Identity resolverと重複防止

| ID | 操作 | 期待結果 |
|---|---|---|
| TC-ID-01 | 未登録の検証済みテストIdentityを、テストフラグ付きで処理 | Customer 1件と、それを参照するChannelIdentity 1件が作成される |
| TC-ID-02 | TC-ID-01と同じIdentityをもう1回処理 | 既存ChannelIdentity / Customerが返り、件数は増えない |
| TC-ID-03 | 同じIdentityを通常の二重実行（連続送信） | 作成前検索と直前再検索によりCustomer / ChannelIdentityは各1件のまま |
| TC-ID-04 | 異なる検証済みテストIdentityを処理 | 別のCustomer / ChannelIdentityが各1件作成され、電話番号で自動名寄せされない |
| TC-ID-05 | 同じIdentityを可能な限り同時に2回実行 | 同一`identity_key`が2件以上ないか重複検知を実行。重複が出た場合は既知制約として記録し、手動統合手順を確認する。原子性は未検証のまま安全と判定しない |
| TC-ID-06 | ChannelIdentity作成を意図的に失敗させる | 孤立テストCustomerが回収対象として検出されるか、補償処理で削除される |

### 4.4 Privacy Rulesと情報漏えい防止

| ID | 操作 | 期待結果 |
|---|---|---|
| TC-PRIV-01 | 未認証セッションでChannelIdentityの一覧・検索を試行 | 0件または拒否となり、レコードと件数を取得できない |
| TC-PRIV-02 | 未認証または他顧客のセッションでCustomerを参照 | 他顧客のCustomerを取得できない |
| TC-PRIV-03 | 認証成功・失敗のブラウザ応答を確認 | `channel_user_id`、`identity_key`、IDトークン、LINE API生レスポンスを含まない |
| TC-PRIV-04 | Bubble logs / debugger / server logsを確認 | トークン、User ID、Channel Secret、LINE API生レスポンスの実値がない |
| TC-PRIV-05 | App data APIからChannelIdentityへの読み書きを試行 | 公開されておらず、検索・作成・更新できない |

### 4.5 エラー表示と実ユーザー非作成

| ID | 操作 | 期待結果 |
|---|---|---|
| TC-ERR-01 | TC-AUTH-02〜06の各失敗を画面から確認 | 利用者向けの一般メッセージだけが表示され、内部詳細を含まない |
| TC-ERR-02 | LIFFを開いただけ、または友だち追加のみのフローを確認 | Customer / ChannelIdentityが作成されない |
| TC-ERR-03 | LINE-04テスト全体のデータを監査 | 実ユーザーのProduction Customer / ChannelIdentityが作成されていない |

## 5. 既存Customer移行の机上テスト

Bubbleのデータは更新せず、次を手順書と設計書に照らしてトレースする。

1. 旧`line_user_id`に値があるCustomerを想定する。
2. 旧値の由来が検証済み`sub`と証明できない場合、自動コピーしないことを確認する。
3. 次回の正規なIDトークン検証後にも、電話番号のみで既存Customerと自動名寄せしないことを確認する。
4. 既存Customerへの紐付けが必要な場合は、別Issue、対象リスト、根拠、作業者レビューが必要であることを確認する。
5. 旧Field削除は参照置換、テスト、ロールバック期間、本番変更承認の後であることを確認する。

すべて説明できれば机上テストは`PASS`。一つでも自動移行またはProduction変更を前提とする場合は`FAIL`とする。

## 6. テスト後の後処理

1. Developmentで作成したテストCustomer / ChannelIdentityだけを対象リストで照合して削除する。
2. 削除後の件数を確認し、他データを変更していないことを記録する。
3. Development専用のモック・テストエンドポイントを無効化または削除し、Productionへデプロイ対象に含まれないことを確認する。
4. ProductionのData type、Field、Privacy Rule、Workflowが変更されていないことを確認する。

## 7. 結果記録テンプレート

```text
Test date/time (ISO 8601):
Executor:
Bubble version: Development
Implementation Issue / commit:

TC-ID: PASS / FAIL / BLOCKED
Evidence: 実値を含まない説明またはマスク済み画像参照
Notes:

Customer count before/after:
ChannelIdentity count before/after:
Test data cleanup: PASS / FAIL
Production unchanged: PASS / FAIL
Open defects / follow-up Issue:
```

## 8. 最終受入判定

次のすべてを満たした場合のみ、Development環境のLINE-04実装テストを合格とする。

- [ ] 有効なIDトークンをLINE検証APIで検証できた
- [ ] 検証済み`sub`だけをChannelIdentityに利用した
- [ ] 改ざん・期限切れ・不正トークンを拒否した
- [ ] トークンとUser IDの実値がログ・画面・証跡にない
- [ ] 通常の二重実行でCustomer / ChannelIdentityを重複作成しない
- [ ] 同一`identity_key`の重複を検知できる
- [ ] 未認証クライアントからChannelIdentityを検索できない
- [ ] 他顧客のCustomerを参照できない
- [ ] エラー表示に機密情報と内部詳細がない
- [ ] 実ユーザーのProduction Customer / ChannelIdentityを作成していない
- [ ] Developmentテストデータを削除した
- [ ] Production環境の構造が未変更である

未実施の項目を推測でPASSにしない。Production反映は本手順書の合格とは別に、別Issueと篠さんの明示承認を必須とする。
