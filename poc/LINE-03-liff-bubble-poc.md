# LINE-03: LIFF + Bubble PoC

## 目的

LINEのリッチメニューからLIFFを起動し、BubbleがホストするページでLINE User Idを取得・画面表示できることを検証する。**案件登録機能は含めない**、最小スコープのPoC。

参照：`docs/line_channel_design_phase1.md` 3章、`docs/adr/ADR-004-line-channel-responsibility.md`

## Definition of Done

LINE-03は、**GitHubへのコード・手順書配置ではなく**、実機においてTC-01〜TC-07（P0）を成功させ、検証結果を本ファイル末尾の「検証ログ」に記録した時点で完了とする。

## 受け入れ基準

### P0（LINE-03の必須完了条件）

潰したい技術的不確実性はただ一つ：**Bubble上のLIFFからLINE User IDを取得できるか**。

| ID | 内容 |
| --- | --- |
| TC-01 | LINEアプリからLIFF URL（リッチメニュー経由）を開ける |
| TC-02 | LIFFが正常に初期化される（`liff.init()`がエラーを返さない） |
| TC-03 | `liff.getProfile()` からLINE User Idを取得できる |
| TC-04 | 取得したUser Idを、Bubble側の変数／Custom Stateに格納できる（画面表示だけでなく値として扱えることを確認） |
| TC-05 | 同一ユーザーで再実行しても同じUser Idが得られる |
| TC-06 | 未ログイン等の異常系での挙動を確認する |
| TC-07 | スマートフォンのLINEアプリ内（LIFFブラウザ）で実際に動作する |

P0がすべて満たされればLINE-03は完了とし、`docs/line_channel_design_phase1.md` の設計をそのまま実装フェーズ（LINE-04以降）に進める。失敗した場合のみ設計を見直す（v0.4）。

### P1（並行調査・LINE-03の完了条件には含めない）

| ID | 内容 |
| --- | --- |
| TC-08 | `openid`スコープ＋`liff.getIDToken()`経路でUser Id相当の値（`sub`）が取得できるか調査する |

本実装（LINE-05以降）ではID token検証方式への切り替えを想定しているため、時間があれば調査してよいが、**LINE-03の完了をこれで止めない**。ID tokenの署名検証・issuer/audience確認等の本人性検証方式は、本実装時にLINE公式ドキュメントを確認したうえで別途確定する。PoC段階での`getProfile()`によるUser Id取得を「唯一正しい認証方式」として固定しない。

**完了条件（再掲）**：GitHub上にファイルを作成しただけでは「LINE-03完了」としない。LINE Developers ConsoleおよびBubble側の設定を実際に行い、TC-01〜TC-07を実機で検証した結果を検証ログに記録した時点をもって完了とする。

**認証境界**：LINE Developers Console・Bubbleエディタへの初回ログイン・パスワード入力・MFA/認証コード・OAuth本人確認・重要な権限承認は篠さん本人が行う。それ以降の認証済みセッションでの画面操作・設定変更は、Web経由でClaude Codeが実施してよい。

## 手順

### A. LINE Developers Console

🔶 **着手前に確認**：以下の手順（チャネル構成・LIFFアプリ追加方法）は2026年8月時点の一般的な情報に基づく想定であり、最新のLINE公式ドキュメント（https://developers.line.biz/ja/docs/liff/ ）で現在の仕様と齟齬がないか確認してから進めること。仕様が変わっていた場合はこのファイルを更新すること。

1. https://developers.line.biz/console/ にログイン（初回ログインは篠さん本人が実施）
2. 既存のMessaging APIチャネルと同じプロバイダー配下に、LINEログインチャネルを新規作成（未作成の場合）
3. LINEログインチャネルの「LIFF」タブから新規LIFFアプリを追加
   - LIFFアプリ名：`l03-poc-userid`
   - サイズ：Full
   - エンドポイントURL：Bubble側で作成するPoCページのURL（Bで作成後に設定）
   - Scope：`profile`（必須）。時間があれば`openid`も追加し、P1調査（TC-08）に使ってよい
   - Bluetooth等の権限：不要
4. 発行済みの **LIFF ID** `2011043480-hLNRE7GE` を使用する

### B. Bubble

🔶 以下はToolboxプラグインを使う想定だが、これは候補の一つであり固定ではない。着手前にBubble公式ドキュメント・プラグインマーケットプレイスで、外部JS SDK（LIFF SDK）をページに読み込ませる最適な方法（HTML要素／Toolbox／他プラグイン）を確認し、必要なら手順を更新すること。

🔶 **既知の制約**：Claude Codeが利用するブラウザ拡張（Claude in Chrome）は `bubble.io` / `*.bubbleapps.io` へのナビゲーションがポリシーでブロックされ、自動操作できないことを確認済み（拡張の「サイトへのアクセス」は「すべてのサイト」で許可されていても拒否される）。そのためBのBubble側作業は篠さん本人の手作業で実施する。

1. Bubbleエディタで新規ページを作成：`liff-poc`
2. LIFF SDK本体の読み込み：App SettingsまたはページのSEO/metatags設定にある「Script/meta tags in header」に以下を追加
   ```html
   <script charset="utf-8" src="https://static.line-scdn.net/liff/edge/2/sdk.js"></script>
   ```
3. プラグイン「Toolbox」（Misha V製、無料）をインストール（Plugins → Add plugins → "Toolbox"で検索）
4. ページに以下の要素を配置
   - テキスト要素 × 2（`text_userid`, `text_displayname`、初期値は空でよい）
5. ページのCustom Stateを3つ作成：`userId`（text）, `displayName`（text）, `errorMsg`（text）
6. Workflowタブで「Page is loaded」イベントを作成し、以下のアクションを追加
   1. Toolbox「Run javascript」：`poc/line03-liff/liff-init.js` の内容を貼り付ける。実機構成のcallback名はUser Id用 `bubble_fn_userid`、displayName用 `bubble_fn_displayname`。エラー用 `bubble_fn_error` は任意で、未設定でもスクリプトがエラーにならない実装とする
   2. Element Actions「Set state」：Page / `userId` = 「Result of step 1's value 1」
   3. Element Actions「Set state」：Page / `displayName` = 「Result of step 1's value 2」
7. `text_userid` の Text フィールドに動的expressionで「Page's userId」を、`text_displayname` に「Page's displayName」を設定
8. Previewで一次確認（LIFFコンテキスト外なので `liff.login()` にリダイレクトされるのが正常。開発者コンソールで `[line03-liff]` ログを確認）
9. ページのURL（version-test環境等）が確定したら、手順A-3のLIFFエンドポイントURLに反映する（Claude Codeが対応可能）

### C. 動作確認

1. LINE公式アカウントのリッチメニューに、上記LIFFを開くボタンを暫定的に1つ設定
2. スマホのLINEアプリから友だち追加 → リッチメニューをタップ
3. `text_userid` / `text_displayname` に値が表示されることを確認
4. 受け入れ基準のチェックリストを埋める

## 注意事項（本実装時に対応、PoCでは不要）

- LINE公式ドキュメントによると、`liff.getProfile()` の結果をそのままサーバーに送るのはなりすまし等のリスクがあるため、本実装（LINE-05以降のフォーム送信）では `liff.getIDToken()` で取得したIDトークンをBubble側で検証する方式に切り替えることが推奨されている。PoC段階ではクライアント側表示のみで問題ない。

## 作業分担の実績（2026-08-10時点）

- **LINE Developers Console（プロバイダー作成、LINEログインチャネル作成、LIFFアプリ登録）**：Claude Codeがブラウザ操作で実施済み。詳細は下記検証ログ参照。
- **Bubble（PoCページ作成、Toolboxプラグイン設定、JavaScript貼り付け）**：篠さんが手作業で構築。2026-08-10にCodexが認証済みBubbleエディタで設定を照合し、LINE User IdとdisplayNameを出力していたコンソールログを削除した安全化版スクリプトへ更新。

## LINE Developers Console 作成物メモ

| 項目 | 値 |
| --- | --- |
| プロバイダー | 水とでんきの救急センター |
| LINEログインチャネル名 | 水電救急センターLogin |
| チャネルID | 2011043480 |
| LIFFアプリ名 | l03-poc-userid（`line03-poc`は予約語エラーのため変更） |
| LIFF ID | 2011043480-hLNRE7GE |
| LIFF URL | https://liff.line.me/2011043480-hLNRE7GE |
| Scope | profile, openid（openidはTC-08 P1調査用） |
| エンドポイントURL | https://shino0728.bubbleapps.io/version-test/liff-poc |

## 検証ログ

### 検証環境と構成

- 検証日：2026-08-10〜2026-08-11（Asia/Tokyo。時刻は未記録）
- Bubbleアプリ：`Shino0728`
- Bubbleページ：`liff-poc`
- Endpoint：`https://shino0728.bubbleapps.io/version-test/liff-poc`
- LIFF ID：`2011043480-hLNRE7GE`
- Bubbleプラグイン：Toolbox（Not Quite Unicorns）
- callback / Custom State：`bubble_fn_userid` → `userId`、`bubble_fn_displayname` → `displayName`
- LIFF SDKは`liff-poc`専用ではなく、Bubbleアプリ共通の「Script/meta tags in header」で読み込み
- LINE User Id実値、トークン、個人情報を検証記録に保存しない
- LINE公式アカウント：`水とでんきの救急センター`（2026-08-11作成。未認証アカウント）
- リッチメニュー：`LINE-03 LIFF PoC`、1ボタン全領域、LIFF URLへ遷移、デフォルト表示オン（表示期間：2026-08-11 01:05〜2026-08-17 23:59 JST）

### 受け入れテスト

| ID | 結果 | 確認内容 |
| --- | --- | --- |
| TC-01 | PASS（実機申告） | 2026-08-11、LINE公式アカウントのリッチメニューからLIFFを起動し、User IDとDisplay Nameが表示されることを確認。実値は未記録 |
| TC-02 | PASS（実機申告） | `liff.init()`成功後にBubbleへ値が渡ることを確認 |
| TC-03 | PASS（実機申告） | `liff.getProfile()`からLINE User Id取得を確認。実値は未記録 |
| TC-04 | PASS（実機申告＋設定照合） | `bubble_fn_userid` → `userId` Custom State → 画面表示を確認 |
| TC-05 | PASS（実機申告） | 2026-08-11、同一ユーザーがLINEアプリ内でLIFF URLを再実行し、同じUser IDが表示されることを確認。実値は未記録 |
| TC-06 | PASS | 外部ブラウザ未ログイン時に`liff.init()`が成功し、`liff.login()`経由でLINE Loginへ遷移することをCodexが確認 |
| TC-07 | PASS（実機申告） | 2026-08-11、スマートフォンのLINEアプリ内でLIFF URLを開き、User IDとDisplay Nameが表示されることを確認。実値は未記録 |

### 現在の判定

**完了**。TC-01〜TC-07のP0受け入れテストがすべてPASSし、個人識別子やトークンの実値をGitHubへ保存していない。LINE-04以降へ進行可能。

### 次工程への申し送り

- リッチメニューはLINE-03検証用の一時設定で、表示終了は2026-08-17 23:59 JST。LINE-05の修理依頼フォーム実装時に本番用メニューへ置き換える。
- LINE-03は`liff.getProfile()`による表示PoC。個人識別情報をBubble DBへ永続化する前に、LINE-05以降でIDトークン検証方式を確定する。
- LINE-04は`Customer` / `ChannelIdentity`の簡素化版実装に限定し、案件登録や修理依頼生成を含めない。
