# 行動調査レポート

## ヘッダー
| 項目             | 内容                                 |
|------------------|--------------------------------------|
| 調査対象         | u542@int.zava-corp.com               |
| 調査期間         | 2026-04-06T09:00:00Z ～ 2026-04-20T09:00:00Z |
| アラート発生日時 | 2026-04-13T09:00:00Z                |
| レポート生成日時 | 2026-04-13T09:00:00Z                |

---

## セクション 1: アラート情報サマリー
- アラート詳細情報は提供されていません。
- 本調査は、アラート発生前後1週間の社外通信パターンやクラウドストレージ利用の有無を中心に、情報漏洩リスクの有無を確認することを目的とします。

---

## セクション 2: Office/Teams アクティビティ概要（OfficeActivity）
**データソース: Sentinel OfficeActivity**

| OfficeWorkload      | 件数 | ExternalAccess=true の件数 |
|---------------------|------|---------------------------|
| Exchange            | 24   | 0                         |
| MicrosoftTeams      | 1    | 0                         |
| SharePoint/OneDrive | 0    | 0                         |

- ExternalAccess = true の操作: なし
- Exchange: Send 操作が多数（24件）確認され、主に社内宛て。外部アクセス操作はなし。
- Teams: 1件のTeamsSessionStarted（2026-04-13T03:48:08Z）のみ。外部チャネル参加やメッセージ送信は確認されず。
- SharePoint/OneDrive: ファイル共有・アップロード・エクスポート操作は確認されず。

---

## セクション 3: メール送信パターン分析（EmailEvents）
**データソース: Defender EmailEvents**

- 送信メール総数: 17件
- 外部ドメインへの送信 Top 10

| ドメイン                     | 件数 | 添付ファイル有り件数 |
|------------------------------|------|---------------------|
| ctf.alpineskihouse.co        | 2    | 0                   |
| その他（int.zava-corp.com等）| 15   | 0                   |

- 添付ファイル付きメール: 0件
- 脅威検知（ThreatTypesあり）メール: 0件
- アラート発生前後3日間（2026-04-10～2026-04-16）の送信件数: 7件（特段の増加傾向なし）

---

## セクション 4: クラウドアプリ利用状況（CloudAppEvents）
**データソース: Defender CloudAppEvents**

| アプリ名                      | イベント数 | 主な操作種別                                      | アクセス国 |
|-------------------------------|------------|---------------------------------------------------|------------|
| Microsoft 365 Copilot Chat    | 174        | CopilotInteraction, Create/Enable/Disable/DeletePlugin | JP         |
| Microsoft 365                 | 82         | Search, Validate, DEX Reporting API read           | （記載なし）|
| Microsoft Exchange Online     | 34         | AutoSensitivityLabelRuleMatch                      | （記載なし）|

- ⚠️ 外部クラウドストレージサービス（Dropbox, Google Drive, Box等）の利用は確認されませんでした。
- アクセス国はJP（日本）のみ。異常な国や複数デバイスからのアクセスは確認されませんでした。

---

## セクション 5: タイムライン（重要イベント）
- 2026-04-10～2026-04-13: Exchangeで複数のSend操作（社内・社外宛て）【Sentinel】
- 2026-04-11T02:58:06Z: 外部ドメイン（ctf.alpineskihouse.co）宛てメール送信【Defender EmailEvents】
- 2026-04-13T03:48:08Z: TeamsSessionStarted（Teams利用開始）【Sentinel】
- 2026-04-13T04:51:12Z: Exchangeでメール送信【Sentinel/Defender EmailEvents】
- 2026-04-13T03:49:12Z～05:37:12Z: Microsoft 365 Copilot Chat利用【Defender CloudAppEvents】

---

## セクション 6: 調査所見と推奨アクション

### 調査所見（事実ベース）
- Exchangeでのメール送信は主に社内宛てで、外部ドメイン宛ては2件（ctf.alpineskihouse.co）のみ。
- 添付ファイル付きメールや脅威検知メールは確認されませんでした。
- TeamsやSharePoint/OneDriveでの外部アクセスやファイル共有操作は確認されませんでした。
- クラウドアプリ利用はMicrosoft 365系のみで、外部ストレージや異常な国からのアクセスはなし。
- 調査期間中に特段のリスク指標は検出されませんでした。

### 推奨アクション
- 🔴 **即時対応**: 特に必要なし
- 🔍 **深掘り調査**: 外部ドメイン宛てメールの内容確認（必要に応じて）
- 🛡️ **再発防止**: DLP/IRMポリシーの継続運用と定期的な監査の実施

---

本レポートは Security Copilot エージェントにより自動生成されました。
すべての記載は収集されたデータに基づく事実です。推測は ※ 推定: として区別しています。