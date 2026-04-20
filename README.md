# Purview User Behavior Investigator

Microsoft Security Copilot 用カスタム Interactive Agent プラグインです。  
Microsoft Purview の DLP/IRM アラート発砲後に、対象ユーザーの行動履歴を複数データソースから自動収集・分析し、事実ベースの Markdown 行動調査レポートを生成します。

---

## 概要

| 項目 | 内容 |
|---|---|
| プラグイン種別 | Interactive Agent（チャット形式） |
| モデル | gpt-4o |
| トリガー | 手動（Manual） |
| データソース | Sentinel OfficeActivity / Defender EmailEvents / Defender CloudAppEvents / Defender DeviceEvents |
| 出力形式 | Markdown レポート |

---

## 調査ワークフロー

```
ユーザーが UPN を入力
        ↓
Phase 1: GetOfficeActivityForUser
        Sentinel OfficeActivity
        Exchange / Teams / SharePoint / OneDrive アクティビティ
        外部アクセス（ExternalAccess=true）の操作
        ↓
Phase 2: GetEmailEventsForUser
        Defender EmailEvents
        社外ドメインへの送信メール・添付ファイル・脅威検知
        ↓
Phase 3: GetCloudAppUsageForUser
        Defender CloudAppEvents（IdentityInfo で UPN→ObjectId をマッピング）
        外部クラウドストレージ利用（Dropbox, Google Drive, Box 等）の検出
        ↓
Phase 4: GetUSBDeviceActivityForUser
        Defender DeviceEvents
        USB 接続イベント（PnPDeviceAllowed / PnPDeviceBlocked）
        リムーバブルストレージへの読み書きアクセス（RemovableStoragePolicyTriggered）
        AdditionalFields からアクセス種別・ポリシー判定・シリアル番号を展開
        ↓
Markdown 行動調査レポートを生成
```

---

## レポート構成

生成されるレポートは以下のセクションで構成されます。

| セクション | 内容 |
|---|---|
| ヘッダー | 調査対象・調査期間・アラート発生日時・生成日時 |
| Section 1 | アラート情報サマリー |
| Section 2 | Office/Teams アクティビティ概要（Sentinel OfficeActivity） |
| Section 3 | メール送信パターン分析（Defender EmailEvents） |
| Section 4 | クラウドアプリ利用状況（Defender CloudAppEvents） |
| Section 5 | USB デバイスアクティビティ（Defender DeviceEvents） |
| Section 6 | タイムライン（重要イベント時系列） |
| Section 7 | 調査所見（事実ベース）と推奨アクション |

---

## 前提条件

### Microsoft Security Copilot
- カスタムプラグインのアップロード権限

### Microsoft Sentinel
- **Microsoft 365 監査ログコネクタ**が有効化されていること
- `OfficeActivity` テーブルにデータが取り込まれていること

### Microsoft Defender XDR
- **Advanced Hunting** の実行権限
- `EmailEvents`・`CloudAppEvents`・`IdentityInfo`・`DeviceEvents` テーブルへのアクセス権限
- **Microsoft Defender for Endpoint** がデバイスにオンボード済みであること（DeviceEvents を使用するため）

---

## セットアップ

### 1. プラグインのアップロード

1. [Microsoft Security Copilot](https://securitycopilot.microsoft.com) にサインインします。
2. **設定** → **カスタムプラグインの管理** → **プラグインの追加** を選択します。
3. `PurviewUserBehaviorInvestigator.yaml` をアップロードします。

### 2. プラグイン設定の入力

プラグインアップロード後、以下の設定値を入力します。

| 設定名 | 説明 | 例 |
|---|---|---|
| `SentinelTenantId` | Sentinel が属する Azure AD テナント ID | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| `SentinelSubscriptionId` | Sentinel が属する Azure サブスクリプション ID | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| `SentinelResourceGroupName` | Sentinel ワークスペースのリソースグループ名 | `rg-sentinel` |
| `SentinelWorkspaceName` | Sentinel ワークスペース名 | `sentinel-workspace` |

---

## 使い方

プラグインを追加後、Security Copilot の **「このエージェントとチャット」** から会話形式で調査を開始できます。

### 入力例

```
user@contoso.com の DLP アラートを調査してください
```

```
対象ユーザー: jane@contoso.com
アラート発生日時: 2026-04-06T09:00:00Z
ポリシー: 社外への機密ファイル送信
上記のアラートについて前後1週間の行動を調査してください
```

> **AlertDateTime を指定した場合**: アラート前後 7 日間（合計 14 日間）を調査します。  
> **AlertDateTime を省略した場合**: 現在日時から過去 14 日間を調査します。

### スターターメニュー

エージェントを起動すると以下のスターターが表示されます。

- `user@contoso.com の DLP アラートを調査してください`
- `指定のユーザーのアラート前後1週間の行動を調査してください`

### フォローアップ質問の例

調査レポート生成後、以下のような質問が可能です。

- `調査結果の概要を教えてください`
- `社外ドメインへのメール送信一覧を詳しく見せてください`
- `クラウドアプリ利用状況を詳しく分析してください`
- `推奨アクションの優先度を説明してください`

---

## スキル一覧

| スキル名 | 種別 | データソース | 説明 |
|---|---|---|---|
| `InvestigateUserBehavior` | Agent (Interactive) | — | 調査オーケストレーション・レポート生成 |
| `GetOfficeActivityForUser` | KQL | Sentinel `OfficeActivity` | Exchange/Teams/SharePoint/OneDrive のアクティビティ取得 |
| `GetEmailEventsForUser` | KQL | Defender `EmailEvents` | 送信メール履歴・脅威検知状況の取得 |
| `GetCloudAppUsageForUser` | KQL | Defender `CloudAppEvents` | クラウドアプリ利用状況のアプリ別集計 |
| `GetUSBDeviceActivityForUser` | KQL | Defender `DeviceEvents` | USB 接続・読み書きアクセスイベントの取得 |

---

## KQL クエリの注意点

### GetCloudAppUsageForUser
`CloudAppEvents` の `AccountObjectId` を使用してユーザーをフィルタしています。  
UPN から ObjectId へのマッピングは `IdentityInfo` テーブルを `toscalar` でルックアップして行います。

```kql
let targetObjectId = toscalar(
    IdentityInfo
    | where AccountUpn =~ targetUser
    | project AccountObjectId
    | take 1
);
CloudAppEvents
| where AccountObjectId == targetObjectId
```

### GetOfficeActivityForUser
Sentinel の `OfficeActivity` テーブルを使用します。  
Microsoft 365 監査ログコネクタが有効でない場合はデータが取得できません。

### GetUSBDeviceActivityForUser
`DeviceEvents` テーブルを `InitiatingProcessAccountUpn` でフィルタしています。  
`AdditionalFields` を `parse_json` で展開して USB 関連フィールドを取得します。

```kql
DeviceEvents
| where InitiatingProcessAccountUpn =~ targetUser
| where ActionType in (
    "RemovableStoragePolicyTriggered",
    "PnPDeviceAllowed",
    "PnPDeviceBlocked"
  )
| extend parsed = parse_json(AdditionalFields)
| extend RemovableStorageAccess        = tostring(parsed.RemovableStorageAccess)
| extend RemovableStoragePolicyVerdict = tostring(parsed.RemovableStoragePolicyVerdict)
| extend SerialNumber                  = tostring(parsed.SerialNumber)
```

> **注意**: `RemovableStoragePolicyTriggered` は Defender for Endpoint のデバイス制御ポリシーが構成されている場合にのみ発生します。ポリシーが未構成の場合、このイベントは記録されません。USB 接続自体は `PnPDeviceAllowed` で確認できます。

---

## ファイル構成

```
PurviewUserBehaviorInvestigator/
├── PurviewUserBehaviorInvestigator.yaml   # プラグイン本体
├── PurviewUserBehaviorInvestigator_card.html  # プラグインカード（視覚的な説明）
└── README.md                               # このファイル
```

---

## 免責事項

- 本プラグインは公知のテーブルスキーマをもとに作成しています。環境によりカラム名が異なる場合があります。
- KQL クエリは本番環境での動作確認を推奨します。
- レポートの内容は収集されたデータに基づきます。データが存在しない場合は「データなし」として報告されます。
- 推測が含まれる場合は「※ 推定:」として事実と区別されます。

---

## 参考リンク

- [Microsoft Security Copilot カスタムプラグイン ドキュメント](https://learn.microsoft.com/ja-jp/copilot/security/custom-plugins)
- [Security Copilot Interactive Agent の作成](https://learn.microsoft.com/ja-jp/copilot/security/developer/build-interactive-agents)
- [Microsoft Purview DLP の概要](https://learn.microsoft.com/ja-jp/purview/dlp-learn-about-dlp)
- [Microsoft Purview インサイダー リスク管理](https://learn.microsoft.com/ja-jp/purview/insider-risk-management)
- [Microsoft Defender for Endpoint デバイス制御の概要](https://learn.microsoft.com/ja-jp/defender-endpoint/device-control-overview)
- [DeviceEvents テーブル スキーマ リファレンス](https://learn.microsoft.com/en-us/defender-xdr/advanced-hunting-deviceevents-table)
