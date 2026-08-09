# ADR-004: LINE連携の責任分界

## ステータス

Accepted

## コンテキスト

Phase 1のMVPにおいて、LINE公式アカウントを顧客受付チャネルとして組み込むにあたり、LINE・LIFF・Bubble・Make.com・Messaging APIの間で「どのシステムが何を担うか」を明確にしておく必要がある。曖昧なままだと、業務ロジック（緊急度判定、業者選定、ステータス遷移等）が複数システムに分散し、将来の保守性・移行性を損なうリスクがある。

既存のADR-001（Bubble採用）およびAPI First原則（Bubbleはビューレイヤー、データモデルはBubble非依存で設計）を踏まえ、LINE導入によってこの原則が崩れないようにする。

## 決定

1. **LINEは顧客との受付・連絡チャネルに限定する。** 業務データベースとしては利用しない。
2. **業務データの正本はBubbleに置く。** 顧客・案件・業者・ステータス等はすべてBubble側で管理する。
3. **修理依頼の入力はLIFF経由でBubbleがホストするページに直接送信する。** LINE Bot形式の会話フロー（状態管理が必要な方式）は採用しない。
4. **Make.comは「Bubble → LINE通知」の片方向連携、および軽量な自動化に限定する。** 緊急度判定・業者選定・ステータス遷移といった業務ロジックをMake.com上に実装しない。
5. **顧客の識別はChannelIdentity（channel / channel_user_id）をCustomerと分離して持つ。** LINEのuserIdをそのままCustomerの主キーにはしない。将来的にWeb・電話等の他チャネルが追加されても同一顧客として名寄せできる構造とする。

## 責任分界表

| システム | 主な責任 |
| --- | --- |
| LINE | 顧客との入口、通知の受信UI、リッチメニュー |
| LIFF（Bubbleホスト） | 修理依頼の入力フォーム |
| Bubble | 業務データの正本、業務ロジック全般 |
| Make.com | Bubble→LINEの通知配信、軽量な連携（業務ロジックは持たない） |
| Messaging API | LINEとBubble/Makeを繋ぐ接続層 |

## 結果

- LINE導入後も、Bubbleを業務基盤の中心に据えるという既存方針（ADR-001、API First原則）が維持される。
- Make.comに業務ロジックが漏れ出すことを防ぎ、将来n8nやSupabaseへの移行時の影響範囲を限定できる。
- LINE以外の受付チャネル（Webフォーム、電話）を追加する際も、ChannelIdentityの仕組みを流用できる。
- 詳細な受付フロー・データ項目・実装WBSは `docs/line_channel_design_phase1.md` を参照。

## 関連ドキュメント

- ADR-001: Bubble採用
- ADR-002: 決済モデル
- `docs/data_model_phase1.md`
- `docs/business_workflow.md`
- `docs/line_channel_design_phase1.md`
