# AI協働運用ルール v2.1

**対象**: 水とでんきの救急センター MVP開発
**目的**: Claude Codeが利用制限・障害・セッション終了等で停止しても、Codexが追加説明なしにGitHub上の情報(設計書＋Issue)だけを読み、同一基準で作業を継続できる状態を作る。Claude CodeとCodexは対等なエージェントとして扱い、本ルールに両者とも同じように従う。

**このドキュメントの位置づけ**: このファイルは「GitHub Issueの運用ルール(SSOTの分離・ラベル設計・Bubble実行方針・引き継ぎ手順)」を定める。サブモジュール(このリポジトリ)と親リポジトリの関係・公開境界・ADR優先順位・Assumption/Decision/Factの区別といった「リポジトリ運用そのもの」のルールは[`AI_COLLABORATION.md`](AI_COLLABORATION.md)を参照する。親リポジトリ側のブランチ保護・PR運用・Claude Code固有の補足は`AGENTS.md`/`CLAUDE.md`(親リポジトリ直下)を参照する。三者は役割を分けており、内容を重複させない。

---

## 1. Single Source of Truth(SSOT)の分離

情報源を2種類に固定する。これ以外の進捗管理ファイル(`WORK_LOG.md`、`implementation_status.md`等)は作らない。

| 情報の種類 | SSOT | 更新タイミング |
|---|---|---|
| 設計仕様(何を作るか、なぜそう決めたか) | `mizu-denki-emergency-docs`の`vision.md` / `assumptions.md` / `data_model_phase1.md` / `business_workflow.md` / `adr/*` | 仕様変更が発生した時点で、Bubble変更に**先立って**更新 |
| 実行状態(どこまでやったか、次に何をするか) | GitHub Issue(本文＋ラベル) | 作業の開始時・中断時・完了時に必ず更新 |

Claude Code / Codexは、作業開始時に「対象Issue本文」＋「関連設計書」の両方を読んでから着手する。Issue本文と設計書が矛盾する場合、原則として設計書を正とし、Issueの記述を修正する(Issueは実行ログであり仕様ではない)。

ただし、Issue本文またはコメントに、既存設計を明示的に変更する新しいCEO Decisionが記録されている場合は、そのDecisionをIssueだけから直接実装してはいけない。まずDecisionの主体・対象・変更内容が明示されていることを確認し、影響する設計書・ADR・運用文書をブランチとPRで更新してレビュー・マージし、SSOTとIssueを整合させてから実装を再開する。記述の新しさだけで優先順位を決めず、Authority / Decision / SSOTの整合を優先する。Decisionの範囲または権限が不明な場合は実装を止め、非機密の範囲でCEOへ確認する。

---

## 2. Issueの機械判定可能フィールド — LabelとIssue本文の二重管理を避ける

同じ状態をGitHub LabelとIssue本文の両方で管理しない。以下のように役割を分離する。

**GitHub LabelをSSOTとする項目(機械判定用・状態遷移が主目的)**

- `status:todo` / `status:in_progress` / `status:blocked` / `status:hold` / `status:review` / `status:done`
- `ready:true` / `ready:false`
- `owner:claude-code` / `owner:codex` / `owner:human` / `owner:unassigned`
- `execution:ai-direct` / `execution:ai-semi-auto` / `execution:human-bubble` / `execution:tbd`

**`status:hold`について:** 経営・Gate判断により意図的に停止中であることを表す。再開条件を満たしても自動的に`status:todo`へは戻さず、再評価・承認を必要とする。外部依存の解消によって自動再開しうる`status:blocked`とは明確に区別する。`status:hold`が付いたIssueは、`ready:true`が残っていてもAIの自律着手対象としない(第7節の着手条件は`status:todo`かつ`ready:true`かつ`blocked-by`解消がそろって初めて成立するものとし、`status:hold`はこれに優先する)。`status:hold`を解除できるのは、Hold判断を行った人間(篠さん)の承認、またはHold理由となった条件の解消を確認した上での再評価のみとする。

これらはGitHub上でLabelとして付け替える。Issue本文中に重複して同じ値を書かない(本文中の該当フィールドはテンプレート上「Labelを参照」とだけ記す)。

**Issue本文をSSOTとする項目(文章・文脈が必要なもの)**

- `blocked-by`: 依存するIssue番号(Issue参照はLabelでは表現しづらいため本文管理)
- `objective`: このIssueで何を達成するか(1〜2文)
- `scope`: 対象ワークフロー・データ型・ファイル
- `acceptance_criteria`: 完了とみなす具体的条件(チェックリスト形式)
- `test_cases`: 検証手順(手動実行含む)
- `current_state`: 現時点の状態(作業中に更新)
- `completed`: 完了した項目
- `remaining`: 残っている項目
- `next_action`: 次に誰が何をするか
- `latest_handoff`: 直近の作業者が次の作業者(別エージェントの可能性あり)に向けて書く「再開サマリー」。日時・作業者・要点を必ず記載する。ただし引き継ぎ時はこれだけを読めばよいわけではない(詳細は7節)。

Labelの状態遷移(例: `status:in_progress`→`status:done`)は、対応するIssue本文の更新と必ず同時に行う。片方だけ更新した状態を残さない。

---

## 3. Bubble操作の実行方針(3段階優先順位)

Bubble操作を最初から人間作業と決めつけない。以下の優先順位で判定する。

**Priority 1: AI直接実行(`execution: ai-direct`)**

Claude Code / Codexがブラウザ操作・computer-use・MCP・CLI・API等、利用可能な手段で安全かつ再現性高くBubble Editorを操作できる場合、AIが直接実行する。(実際の利用可否は「Bubble AI操作能力検証」Issueで個別に確認する。)

**Priority 2: AI半自動実行(`execution: ai-semi-auto`)**

完全自動化が困難でも、AIが可能な範囲まで実行し、人間の操作を最小限に絞る(例: AIがBubble上の該当画面まで遷移させ、最後の1クリックだけ人間に依頼する等)。

**Priority 3: Human-in-the-loop(`execution: human-bubble`)**

AIによる操作が技術的に不可能・不安定・危険、または人間の確認が必須な場合のみ、人間へフォールバックする。この場合、Issueに`execution: human-bubble`を明示し、操作指示は以下の粒度まで具体化する。

```
Step 1: ...
Step 2: ...
Only when: ...
Expected result: ...
```

人間(篠さん)が操作結果を報告した後、AIがAcceptance Criteriaと照合し、Issueを更新する。**この判定順位は固定ではなく、「Bubble AI操作能力検証」Issueの結果に応じて随時見直す。**以後に作成するIssueの`execution`初期値は、検証結果に基づくルールに従う。

**個別Issueでの試行順序**: 各Issueの`execution`は着手前に決め打ちしない。Data Type/Fieldの確認のような低リスクな読み取り操作は特に、まず`ai-direct`を試行し、失敗した場合のみ`ai-semi-auto`、それも困難な場合に初めて`human-bubble`へフォールバックする。この試行と結果は、それ自体が「Bubble AI操作能力検証」Issueへのフィードバックになるため、簡潔に記録する。

**owner と execution の責任分離**: `owner`はそのIssueを完遂させる責任を持つAIエージェント(claude-code または codex)を表し、`execution`は実際の操作手段を表す。両者は独立した軸であり、`execution: human-bubble`になったからといって`owner`を`human`に変更しない。Bubble操作が人間へのフォールバックになった場合でも、Issue管理・検証・Handoff作成・Close判断の責任はownerであるAIエージェントが持ち続ける。`owner: human`は、AI側での管理自体が不可能な例外的状況に限定する。

**親リポジトリ側との関係**: `execution: ai-direct`/`ai-semi-auto`であっても、Bubble **Production環境**の構造変更(データモデル変更、Workflow変更、既存ページの破壊的変更等)を伴うIssueをCloseする際は、親リポジトリ`AGENTS.md`のBubble運用ルールに従い、人間の明示的な承認を経ていることをAcceptance Criteriaで確認する。本ルールの3段階優先順位は「AIがどこまで操作を代行できるか」の判定基準であり、「Production構造変更にあたって人間の承認が要るかどうか」を無効化するものではない。

---

## 4. レースコンディション・排他制御についての方針

Bubbleのバックエンドワークフローにおける`Only when`条件は、**MVPにおける楽観的ガードとして採用する。Bubble上で厳密なatomicity(同時実行時の排他制御)が保証されることを前提としない。**この方針はbusiness_workflow.mdまたは該当ADRに明記すること(Bubble変更より先に文書化)。

この前提のもと、Bubble変更を伴うIssueで状態遷移に関わるものは、Acceptance Criteria / Test Casesに以下の競合テストを必ず含める。

- Case A: partner Aがaccept→matched。その後check_timeout_t2が走ってもcancelledに上書きされないこと。
- Case B: check_timeout_t2でcancelled。その後partner Aがacceptしてもmatchedに変更されないこと。
- Case C: partner Aとpartner Bがほぼ同時にaccept。二重matched・matched_partnerの不整合が発生しないこと。

Case Cで問題が確認された場合、楽観的ガードでは不十分と判断し、当該Issueをcloseせず、排他制御方式の再設計を別Issueとして起票する。

---

## 5. T1/T2テスト時間の扱い

開発・E2Eテスト時、T1/T2は1〜2分程度に短縮してよい。**T1/T2はPhase 1の暫定値であり、確定仕様ではない。加盟店ヒアリングおよびPilot Matching結果を踏まえて再評価する**(2026-08-15 Gate Check Round 3で再確認)。これに関わるすべてのIssueのAcceptance Criteriaに、

> テスト終了後、その時点の暫定運用値へ復元されていることを確認した

を必須項目として含める。将来的にテスト値／本番値を設定用データ(例: Config的なデータタイプの1レコード)として分離し、手動でのハードコード書き換えを不要にする案は、UI構築(最小内部確認用UI以降)と合わせて検討する。

---

## 6. IssueのClose(Done)条件

以下すべてを満たすまで、Bubble実装を伴うIssueを`done`にしない。AIが手順書や設計書だけを作った段階、またはAI自身がBubbleを直接操作した場合も同様に、実際の変更確認なしにcloseしない。

- [ ] Bubble上の変更が実際に反映されている(human-bubbleの場合は人間からの報告を確認済み)
- [ ] Acceptance Criteriaの全項目が満たされている
- [ ] Test Casesが実行され、結果が記録されている
- [ ] 設計書との整合性が確認されている(矛盾があれば設計書を先に更新済み)
- [ ] `latest_handoff`が更新されている

---

## 7. 開始・中断・引き継ぎの手順

**作業開始時**

1. 担当Issueを1件選ぶ(`status: todo`かつ`ready: true`かつ`blocked-by`が全てcloseされているもの。`status: hold`が付いているIssueは、`ready: true`であっても選ばない)
2. Issue本文と関連設計書を読む
3. `owner`を自分(claude-code または codex)に更新、`status`を`in_progress`に更新

**作業中断時(利用制限・エラー等)**

1. `current_state` / `completed` / `remaining`を最新化
2. `latest_handoff`に、日時・中断理由・次にやるべき具体的な一歩を記載
3. `status`を`blocked`または`in_progress`のまま保持(人間の指示なしに`done`にしない)

**別エージェントが引き継ぐ時**

1. まず`latest_handoff`を「再開サマリー」として読み、直前の状況・次の一歩の見当をつける
2. 続けて、Issue本文全体(objective / scope / acceptance_criteria / test_cases / current_state / completed / remaining / next_action)と関連設計書を読み、`latest_handoff`の要約だけでは分からない背景・条件・除外事項を確認する
3. `latest_handoff`はあくまで導入であり、それ単体を根拠に作業内容を確定させない。Issue本文・設計書と矛盾する場合は本文・設計書を優先する
4. 追加説明を人間に求めずに再開する。不明点がありIssue記述からも判断できない場合のみ、Issueにコメントで疑問点を残し、人間へエスカレーションする

---

## 8. PRマージのTier分類

### Tier 1: 自動マージ対象

以下の条件を**すべて**満たすPRのみ、Actions workflowによる自動マージ対象とする。1つでも判定できない場合はマージせず、人間へエスカレーションする(fail-closed)。

- `Does this PR change a Decision?`が本文パースで`No`と確認できる
- 変更ファイルが`docs`のgitlink1件のみ(API・`git diff`の二重確認)
- 現在の`main`側gitlinkとPR側の対象gitlinkがどちらもdocs repo `main`から到達可能であり、現在のgitlinkが対象gitlinkの祖先である（forward updateのみ。巻き戻し・別系統commitは対象外）
- Draftでない
- マージコンフリクトがない
- CI(存在する場合)が全て成功している
- Bubble Production関連ファイルを含まない
- `.github/workflows/`配下の変更を含まない
- base branchが`main`、head repositoryが同一リポジトリである

判定処理でエラー、取得不能、想定外の値が発生した場合もTier 1として扱わない。詳細な判定ロジックとbootstrap手順は親リポジトリ`hiroshino728/water-denki-emergency-mvp`のIssue #34を参照する。

### Tier 2: 篠さんの承認必須(デフォルト)

上記Tier 1条件を満たさないすべてのPR。特に以下は常にTier 2とする。

- `Does this PR change a Decision?`が`Yes`
- ADRの新規追加・既存ADRの変更
- Gate判定(GATE-A〜E等)に関わる変更
- Bubble Production環境の変更
- 法務・契約・金銭・責任分界に関わる変更
- `.github/workflows/`配下の変更(自動マージロジック自体の変更)

### エスカレーション条件

自動マージ判定処理でエラー・条件未達が発生した場合、PRに`needs-human-review`ラベルを付与し、PRへエスカレーションコメントを投稿する。この場合、篠さんの確認まで放置せず、次にGitHubを開いたエージェント(Claude Code/Codex)が状況を要約して報告する。

### 初回導入と有人監視

自動マージの導入はdry-run-onlyのStage 1と、篠さんの承認を受けてlive化するStage 2に分ける。dry-runでは判定条件をすべて評価するがmerge APIを呼ばない。Stage 2 live化後、最初の実際の自動マージ対象PRを検知した場合は、マージが実行される前に篠さんへ通知する。最初の2〜3回はマージcommitと関連Issueへの反映を直後に目視確認し、異常が1件でもあればworkflowを直ちに無効化して人間レビューへ戻す。

ロールバック手順は[`auto_merge_rollback.md`](auto_merge_rollback.md)を参照する。

---

## 更新履歴

- v2.1: owner/execution分離、Label/本文分離、Bubble実行3段階優先順位、Close条件、引き継ぎ手順を反映。親リポジトリ`AGENTS.md`(ブランチ保護・PR運用・no-fixed-role・記憶非共有の原則)およびこのリポジトリの`AI_COLLABORATION.md`(サブモジュール運用・ADR優先順位)と役割分担のうえ運用する。
- 2026-08-16: Issue #33/#34に基づき、第8節「PRマージのTier分類」とbootstrap・有人監視方針を追加。
- 2026-08-18: Issueと設計書の不一致に新しいCEO Decisionが関係する場合のDecision Reconciliation手順を第1節へ追加。
- 2026-08-16: Stage 2初回有人監視でstaleな巻き戻しPRを検出したため、Tier 1にdocs gitlinkの到達可能性・forward ancestry条件を追加。
- 2026-08-15(Gate Check Round 1-3反映): `status:hold`ラベルを追加(2節)。着手条件に`status:todo`を明記し、`status:hold`をAI自律着手の対象外とすることを明確化(7節)。T1/T2の「本番仕様固定」という記述を「Phase 1の暫定値、加盟店ヒアリング・Pilot Matching結果を踏まえて再評価」に修正(5節)。
