# docs submodule PR自動マージのロールバック手順

対象workflow: 親リポジトリの`.github/workflows/auto-merge-docs-submodule-sync.yml`

## 発動条件

誤ったPRの自動マージ、判定結果と実際のdiffの不一致、関連Issueへの反映異常などを1件でも確認した場合、以下を直ちに実行する。

## 手順

1. **即時停止**: GitHubのSettings > Actions > Workflowsから対象workflowをdisableする。UI操作ができない場合は、workflowを無効化する変更を別ブランチで作成し、緊急PRとして篠さんの承認を受ける。mainへ直接pushしない。
2. **対象の保留**: 該当PRまたは関連Issueを`ready:false`かつ`status:hold`へ戻し、自動処理を継続しない。`status:blocked`は外部依存の解消で自動再開しうるため代用しない。
3. **切り戻し**: 対象PRのマージイベント（例: `gh pr view <PR番号> --json mergeCommit,mergedAt`）とworkflow実行ログの実マージ記録を照合してmerge commitを特定し、`git revert -m 1 <merge_commit_sha>`で切り戻すブランチとPRを作成する。
4. **切り戻しPRをTier 2化**: PR本文の`Does this PR change a Decision?`を`Yes`にするかDraftで起票し、必ず篠さんのレビュー・手動マージを経る。切り戻しを自動マージしない。
5. **原因調査**: Actionsログ、対象PRのAPI応答、ローカルdiff検証結果を照合し、どのfail-closed条件が誤って通過または評価不能になったかをIssueへ記録する。
6. **修正と再検証**: 修正PRをTier 2として作成し、Stage 1相当のdry-runケースをすべて再実行する。
7. **再有効化**: 全ケースPASS後、篠さんの明示承認を得てからworkflowを再度有効化する。

## 注意事項

- Rulesetをbypassしたり、mainへ直接pushしたりしない。
- ロールバック中もBubble Development / Production環境には触れない。
- GITHUB_TOKENで発生したイベントは原則として別workflowを新たに起動しないため、連鎖workflowが将来追加された場合は個別に実行確認する。
