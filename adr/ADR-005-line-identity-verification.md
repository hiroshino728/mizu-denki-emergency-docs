# ADR-005: LINE本人性検証はIDトークン検証APIを用いる

## ステータス

Accepted

## コンテキスト

ADR-004（LINE連携の責任分界）は、顧客の識別をChannelIdentity（`channel` / `channel_user_id`）としてCustomerと分離して持つことを決定済みである。しかし、その `channel_user_id` をどうやって「本人が確かに認証したLINEアカウントのID」として確定するかについては、ADR-004・`line_channel_design_phase1.md`（v0.1〜v0.3時点）ともに未確定のままだった。

LINE-03のPoCでは `liff.getProfile()` から得た `userId` を画面表示するところまでを検証したが、これは「表示できるか」の技術検証であり、「そのuserIdをサーバー側の永続化・業務データの根拠として信頼してよいか」という論点とは別である。`liff.getProfile()` はクライアントサイドで呼び出せるAPIであり、その戻り値をそのままBubbleへ送信させる方式では、クライアントが任意の文字列を送りつけた場合にサーバー側で検証する手段がない。

LINE-04（Customer / ChannelIdentity実装）に着手する前に、本人性検証の方式を正式に決定する必要がある。

## 決定

1. **本人性検証にはLINEのIDトークン検証API（`POST https://api.line.me/oauth2/v2.1/verify`）を用いる。** 検証フローは以下の通りとする。

   ```
   LIFF → liff.getIDToken()
        → Bubbleバックエンドへ送信
        → Bubble → POST https://api.line.me/oauth2/v2.1/verify （id_token + client_id）
        → HTTP 200 かつ iss / aud / exp / sub を防御的に確認
        → 検証成功レスポンスの sub を channel_user_id として使用
   ```

2. **`liff.getProfile()` の `userId` を、サーバー側永続化の根拠にしない。** 画面表示・UX目的でのクライアント内利用は妨げないが、Bubbleへ送ってChannelIdentityを作成・検索する用途には使わない。
3. **クライアントから送られてきたUser ID文字列そのものを信頼しない。** ChannelIdentityの作成・検索は、必ずサーバー側（Bubbleバックエンドワークフロー）でのIDトークン検証を経た `sub` を用いる。
4. **この検証方式ではChannel Secretは不要。** `POST /oauth2/v2.1/verify` は `id_token` と `client_id`（LINEログインチャネルID）のみで検証できるため、Channel Secretを検証処理に含めない。Channel Secretが必要になる別方式（サーバーサイドのOAuth code交換等）は採用しない。
5. **`client_id` にはLINEログインチャネルIDを使用する。**
6. **検証時はHTTP 200を確認したうえで、レスポンス内の `iss`（発行者）・`aud`（audience = client_id と一致）・`exp`（有効期限）・`sub`（LINE User Id）を防御的に確認する。** いずれかが不正・欠落・期限切れの場合は検証失敗として扱い、ChannelIdentityの作成・検索を行わない。
7. **Phase 1ではnonceを使用しない。** 現在の認可要求（LIFFの標準ログインフロー）ではnonceを発行していないため、独自のnonce管理を追加実装しない（過剰実装を避ける）。**将来、認可フローで独自にnonceを発行する場合は、検証API呼び出し時に同じnonceを渡して照合すること。**
8. **IDトークン・アクセストークンはDB・ログへ保存しない。** 検証に使った後は破棄する。保存するのは検証成功後に確定した `sub`（`ChannelIdentity.channel_user_id` として）のみ。
9. **エラー画面にトークンやLINE APIの生レスポンス・詳細を表示しない。** 利用者には安全な一般メッセージのみを表示する（詳細は `line_channel_design_phase1.md` 13章「エラー処理」参照）。

## 却下した代替案

- **`liff.getProfile()` の `userId` をそのままサーバーへ送り、Bubble側で無条件に信頼する。** クライアントが任意の値を送信できてしまい、なりすまし・データ汚染を防げないため却下。
- **LINE User Idをハッシュ化してのみ保存する。** Messaging APIのプッシュ通知にはUser IDの原値が必須であり、ハッシュ化のみでは通知要件を満たせないため却下（詳細は`line_channel_design_phase1.md` 13章）。
- **サーバーサイドOAuthのcode交換方式（Channel Secretを用いる方式）。** Phase 1のLIFFベースの構成には過剰であり、IDトークン検証APIのみで要件を満たせるため見送り。将来Messaging APIのユーザー認可フロー等でChannel Secretが必要になった場合は別途ADRを起こす。

## 結果

- ChannelIdentityの `channel_user_id` は、検証済みIDトークンの `sub` のみを起点とすることが確定した。データモデル上の定義は `data_model_phase1.md` の `ChannelIdentity` エンティティを参照。
- LINE-04の受入条件（改ざん・期限切れ・不正トークンの拒否、トークン非ログ化等）は本ADRの決定に基づいて `line_channel_design_phase1.md` 13章以降に具体化する。
- Channel Secretを本検証フローの実装に含める必要がないことが明確になり、秘密情報の取り扱い範囲を最小化できる。

## 参照

- LIFFでユーザーデータを安全に扱う方法: https://developers.line.biz/en/docs/liff/using-user-profile/
- LINE Login v2.1 API: https://developers.line.biz/en/reference/line-login/
- IDトークン検証: https://developers.line.biz/en/docs/line-login/verify-id-token/

非公式ブログ・まとめ記事を本ADRの主根拠にしていない。実装時に仕様変更がないか、上記公式ドキュメントで再確認すること。

## 関連ドキュメント

- ADR-004: LINE連携の責任分界
- `docs/data_model_phase1.md`（ChannelIdentityエンティティ）
- `docs/line_channel_design_phase1.md`（13章以降、LINE-04実装方針）
- `docs/poc/LINE-03-liff-bubble-poc.md`（LINE-03検証ログ）
