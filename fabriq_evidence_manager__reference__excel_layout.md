# Excel 台帳の列構成・色・ハイライト規則（監査用）

> **対象**: fabriq_evidence_manager / reference
> **対象バージョン**: 3.8.0（取得元: `E:\fabriq_evidence_manager\FabriqEvidenceManager\FabriqEvidenceManager.csproj` `<Version>`）
> **ドキュメント更新日**: 2026-05-07

`ExcelExportService.cs`（2,809 行、ClosedXML 0.105.0）が生成する **30+ シート構成の納品 Excel 台帳** の完全カタログ。監査用に「どのシートのどの列に何が出るか / 何色のセルが何を意味するか」を一覧化する。

---

## ファイル構成（main + pc_details/）

```
{deliveryRoot}/
├── {timestamp}_PC情報一覧表.xlsx           ← main 台帳 (1 PC = 1 行のフリート総覧 + 横断シート群)
└── pc_details/                              ← PC 個別詳細 (1 PC = N 行になる詳細データ)
    ├── {PcName}_{CollectionDate}.xlsx
    ├── {PcName}_{CollectionDate}_{Serial先頭6}.xlsx       ← 衝突回避
    ├── {PcName}_{CollectionDate}_{Serial先頭6}_2.xlsx     ← さらに衝突時連番
    └── ...
```

「**1 PC = 1 行で表現できるシート**」は main 側、「**1 PC = N 行になるシート**」は pc_details/ 側に出る。`HasAnyDetailData(pc)` が真の PC のみ pc_details/ サブブックを生成し、main 台帳の PC 名セルからハイパーリンクで飛べる。

ハイパーリンク実装上の注意：`XLHyperlink(string)` は内部セル参照と解釈される（Excel が現在シート名を prefix してリンクが壊れる）ため、外部ファイル相対リンクは `XLHyperlink(new Uri(detailRelativePath, UriKind.Relative))` を渡す。

---

## ヘッダ色 13 種（用途別）

`ExcelExportService` 冒頭で `XLColor.FromHtml(...)` で定義された色定数：

| 色定数 | HTML 色 | 視覚 | 用途 |
|---|---|---|---|
| `BaseHeaderBg` | `#4472C4` | 青 | 基本情報（PC名 / Serial / 収集日 / 突合判定 / 管理者メモ） |
| `NicHeaderBg` | `#548235` | 緑 | NIC（MAC / アダプタ / 接続名） |
| `KeyHeaderBg` | `#BF8F00` | 黄 | BitLocker 回復キー |
| `VerifyHeaderBg` | `#7030A0` | 紫 | hostlist 突合 |
| `ChecklistHeaderBg` | `#C55A11` | オレンジ | チェックリスト |
| `HistoryHeaderBg` | `#538DD5` | 水色 | 実行履歴 |
| `LicenseHeaderBg` | `#7B1FA2` | 紫 | ライセンス監査（Win/Office/SecBase/証明書） |
| `HwIdHeaderBg` | `#455A64` | 濃グレー青 | HW 識別子（§31） |
| `MemoryHeaderBg` | `#00838F` | 青緑 | メモリ（§29） |
| `EnvVarHeaderBg` | `#5D4037` | 茶 | 環境変数（§27） |
| `StartupHeaderBg` | `#388E3C` | 深緑 | 自動起動（§28） |
| `PnpHeaderBg` | `#D81B60` | 濃ピンク | PnP デバイス（§30） |
| `GpHeaderBg` | `#E65100` | 濃オレンジ | グループポリシー（§24） |
| `BaselineHeaderBg` | `#5B5EA6` | 青紫 | ベースライン突合 6 シート |
| `UserGroupHeaderBg` | `#00695C` | ティール | ユーザー・グループ |
| `DomainHeaderBg` | `#37474F` | ダークグレー | ドメイン状態 |
| `FwHeaderBg` | `#D84315` | ディープオレンジ | FW プロファイル / ルール |
| `FeatureHeaderBg` | `#6A1B9A` | パープル | オプション機能 |
| `ProfileHeaderBg` | `#00838F` | シアン | ユーザープロファイル |
| `DiskHeaderBg` | `#4E342E` | ブラウン | ディスク・パーティション |
| `PowerHeaderBg` | `#827717` | オリーブ | 電源設定 |
| `WiFiHeaderBg` | `#1565C0` | ブルー | WiFi プロファイル |
| `RestoreHeaderBg` | `#AD1457` | ピンク | 復元ポイント |
| `DefenderHeaderBg` | `#1B5E20` | ダークグリーン | Windows Defender |
| `UpdateHeaderBg` | `#283593` | インディゴ | Windows Update |

ヘッダセル共通: 文字色 = `White`、太字、Arial 10pt、中央寄せ、`WrapText=true`（`SetHeaderCell` ヘルパ）。

---

## ステータス色 7 種（セル背景・文字色）

| 色定数 | HTML 色 | 用途 |
|---|---|---|
| `MatchBg` | `#E2EFDA`（薄緑） | OK / 一致 / 認証済 |
| `MismatchBg` | `#FCE4EC`（薄赤） | NG / 不一致 / 認証失敗 / 期限切れ |
| `WarningBg` | `#FFF8E1`（薄黄） | Partial / 要確認（M11 監査用） |
| `InfoBg` | `#F5F5F5`（薄グレー） | 情報行（Office 未インストール等、判定対象外） |
| `CautionBg` | `#FFF3E0`（薄オレンジ） | フォールバック取得（要確認） |
| `MismatchFont` | `#C62828`（赤） | 文字色（NG 強調） |
| `MatchFont` | `#2E7D32`（緑） | 文字色（OK 強調） |
| `CautionFont` | `#E65100`（オレンジ） | 文字色（フォールバック警告） |

汎用ステータス着色ヘルパ `ApplyStatusColor(cell, status)`：

| status 値（大文字化後） | 効果 |
|---|---|
| `OK / SUCCESS / MATCH / PASS` | `MatchFont` |
| `NG / ERROR / MISMATCH / FAIL` | `MismatchFont` + 太字 |
| `SKIP / SKIPPED / - / NOTRUN` | `Gray` |

監査用 4 段階着色 `AuditLevel`（M12 SecurityBaseline 等で使用）：

| AuditLevel | 効果 |
|---|---|
| `Good` | `MatchBg` 背景 + `MatchFont` 文字色 |
| `Warning` | `WarningBg` 背景のみ |
| `Bad` | `MismatchBg` 背景 + `MismatchFont` 文字色 + 太字 |
| `Na` | `Gray` 文字色（`"N/A"` 値） |

---

## 共通スタイル規約（`ApplyStyles`）

各シートの末尾で `ApplyStyles(ws, dataRowCount, totalColumns, headerRow)` を呼ぶ：

- ヘッダ + データ行に **薄細罫線**（`XLBorderStyleValues.Thin` の Outside + Inside）
- データ行のフォントを **Arial 10pt** に統一
- 全列を **`AdjustToContents`** で内容に合わせる、幅上限は `EvidenceConstants.ExcelMaxColumnWidth = 55`
- ヘッダ行を **`FreezeRows`** で固定スクロール
- ヘッダ + データ範囲に **`AutoFilter`** 設定（フリート横断で絞込み可能）

セル値書き込みの安全装置 `TruncateForCell`：

- Excel セル 1 個の上限 32,767 文字を超えるテキストは **末尾 `\n... [TRUNCATED]` マーカー付きで切り詰め**
- 適用箇所: 管理者メモ / Description / Message / 環境変数 Value / Command / InstanceId / 未知 section RawContent

---

## main 台帳のシート一覧（最大 19 シート）

main 台帳 `{timestamp}_PC情報一覧表.xlsx` に出るシート。**条件付き生成** のため、該当データを 1 PC でも持っていればシート出現。

| # | シート名 | 出現条件 | 行構造 | 詳細 |
|---|---|---|---|---|
| 1 | `PC情報一覧` | 必ず出る | 1 PC = 1 行 | フリート総覧。後述 §「Sheet 1」 |
| 2 | `ドメイン状態` | 1 PC でも `DomainStatus is not null` | 1 PC = 1 行 | §「ドメイン状態シート」 |
| 3 | `電源設定` | 1 PC でも `PowerSettings is not null` | 1 PC = 1 行 | PC 名 + アクティブプラン |
| 4 | `Windows Defender` | 1 PC でも `DefenderStatus is not null` | 1 PC = 1 行 | §「Windows Defender シート」 |
| 5 | `Windowsライセンス` | 1 PC でも `WindowsLicense is not null` | 1 PC = N 行（製品ごと） | §「Windows ライセンスシート」 |
| 6 | `Officeライセンス` | 1 PC でも `OfficeLicense is not null` | 1 PC = N 行（製品ごと） | §「Office ライセンスシート」 |
| 7 | `セキュリティベースライン` | 1 PC でも `SecurityBaseline is not null` | 1 PC = 1 行 | §「セキュリティベースラインシート」 |
| 8 | `HW識別子` | 1 PC でも `HardwareIdentifiers is not null` | 1 PC = 1 行 | §「HW 識別子シート」 |
| 9 | `グループポリシー` | 1 PC でも `GroupPolicyReport is not null` | 1 PC = 1 行 | §「グループポリシーシート」 |

§25 証明書 / §29 メモリ / §27 環境変数 / §28 自動起動 / §30 PnP デバイスは **N 行/PC で行数が膨大になるため main には出さず pc_details/ 側に出力**（フリートサマリは Sheet 1 主要列で代替）。

---

## pc_details/ 個別ブックのシート一覧（最大 19 種類）

各 PC の `pc_details/{PcName}_{Date}.xlsx` に出るシート。1 PC = N 行データを持つもののみ。`ExportPcDetailWorkbook` 内で `single = new[] { pc }` の単一要素配列で各 `WriteXxxSheet` を呼び出し再利用（DRY）。

| シート名 | 出現条件 | 行構造 | ヘッダ色 |
|---|---|---|---|
| `突合詳細` | `VerificationReports.ContainsKey(pc)` | 突合項目 1 件 = 1 行 | `VerifyHeaderBg` 紫 |
| `チェックリスト` | `ChecklistResults.ContainsKey(pc)` | モジュール 1 件 = 1 行 | `ChecklistHeaderBg` オレンジ |
| `実行履歴` | `ExportHistories.ContainsKey(pc)` | エントリ 1 件 = 1 行 | `HistoryHeaderBg` 水色 |
| `ベースライン_実行サマリ` | `BaselineReports[pc].Items.Count > 0` | モジュール 1 件 = 1 行 | `BaselineHeaderBg` 青紫 |
| `ベースライン_SystemInfo` | `BaselineReports[pc].SystemInfoComparison is not null` | 4 項目 = 4 行 | 同上 |
| `ベースライン_チェックリスト` | `BaselineReports[pc].ChecklistComparison is not null` | OverallStatus 1 行 + VerifyItems N 行 | 同上 |
| `ベースライン_アプリ` | `BaselineReports[pc].InstalledAppsComparison is not null` | 集計 1 行 + 差分 N 行 | 同上 |
| `ベースライン_ライセンス` | `BaselineReports[pc].LicenseComparison is not null` | 最大 8 項目 = 8 行 | 同上 |
| `ベースライン_ドメイン` | `BaselineReports[pc].DomainStatusComparison is not null` | 7 項目 = 7 行 | 同上 |
| `ユーザー・グループ` | `LocalUsers.Count > 0 \|\| LocalGroups.Count > 0` | ユーザー / グループ / メンバー 種別 1 件 = 1 行 | `UserGroupHeaderBg` ティール |
| `FWプロファイル` | `FirewallProfiles.Count > 0` | プロファイル 1 件 = 1 行 | `FwHeaderBg` ディープオレンジ |
| `FWルール` | `FirewallRules.Count > 0` | ルール 1 件 = 1 行 | 同上 |
| `オプション機能` | `OptionalFeatures.Count > 0` | 機能 1 件 = 1 行 | `FeatureHeaderBg` パープル |
| `ユーザープロファイル` | `UserProfiles.Count > 0` | プロファイル 1 件 = 1 行 | `ProfileHeaderBg` シアン |
| `ディスク・パーティション` | `Disks.Count > 0 \|\| Partitions.Count > 0` | ディスク / パーティション 種別 1 件 = 1 行 | `DiskHeaderBg` ブラウン |
| `WiFiプロファイル` | `WiFiProfiles.Count > 0` | プロファイル 1 件 = 1 行 | `WiFiHeaderBg` ブルー |
| `復元ポイント` | `RestorePoints.Count > 0` | 復元ポイント 1 件 = 1 行 | `RestoreHeaderBg` ピンク |
| `Windows Update` | `WindowsUpdates.Count > 0` | KB 1 件 = 1 行 | `UpdateHeaderBg` インディゴ |
| `証明書` | `Certificates.Count > 0` | 証明書 1 件 = 1 行 | `LicenseHeaderBg` 紫 |
| `メモリ` | `MemorySlots.Count > 0 \|\| MemoryArraySummaries.Count > 0` | スロット N 行 + アレイ N 行（2 セクション） | `MemoryHeaderBg` 青緑 |
| `環境変数` | `EnvironmentVariables.Count > 0` | 環境変数 1 件 = 1 行 | `EnvVarHeaderBg` 茶 |
| `スタートアップ` | `StartupItems.Count > 0` | 自動起動項目 1 件 = 1 行 | `StartupHeaderBg` 深緑 |
| `PnPデバイス` | `PnpDevices.Count > 0` | デバイス 1 件 = 1 行 | `PnpHeaderBg` 濃ピンク |
| `§{Id} {Title}` | `UnknownSections.Count > 0`（section ID ごとに動的生成） | 未知 section 1 ファイル = 1 行 | 黄タブ + `#7B1FA2` ヘッダ |

---

## Sheet 1 「PC情報一覧」（main 台帳の中核）

### 案件メタデータブロック（オプション・上 6 行）

`ExcelExportOptions.DeliveryMetadata is not null` のとき、Sheet 1 冒頭に 6 行のメタデータブロックを書き出す。データ部は 7 行目以降に開始する。

| 行 | A 列（ラベル、太字 + 薄グレー背景） | B 列 | C 列 | D 列 | E 列 | F 列 |
|---|---|---|---|---|---|---|
| 1 | 案件名 | `metadata.ProjectName` | | | | |
| 2 | 顧客 | `metadata.CustomerName` | | | | |
| 3 | 作業者 | `metadata.WorkerName` | | | | |
| 4 | 作業日 | `WorkDate?.ToString("yyyy/MM/dd")` | | | | |
| 5 | fabriqKernelVersion | `firstManifest?.FabriqKernelVersion` | evidenceConfigVersion | `firstManifest?.EvidenceConfigVersion` | | |
| 6 | 対象 PC 台数 | `pcEvidences.Count` | 突合 OK | `ok` | 突合 NG | `ng`（NG > 0 で赤太字） |

`fabriqKernelVersion` / `evidenceConfigVersion` は **最初に manifest が読めた PC の値を採用**（フリート全機の値が同じ前提、混在時は先頭値が代表）。

`ok / ng` の集計式：

```
ok = pcEvidences.Count(pc => report.AllMatch && rep.MismatchCount == 0)
ng = pcEvidences.Count(pc => rep.MismatchCount > 0)
```

### 列構成（動的）

固定列 6 + 動的 NIC + 動的 BitLocker + バッテリ 3 + メモリ 2 + GP 1 + HW 4 + メモ 1 = 17 + 動的列数。

| 列 | ヘッダ | ヘッダ色 | バインド | 着色 |
|---|---|---|---|---|
| 1 | 計画 PC 名 | `BaseHeaderBg` 青 | `pc.PcName` | 詳細 .xlsx へハイパーリンク（`#1565C0` 下線） |
| 2 | 実 PC 名 | 同 | `pc.ActualComputerNameDisplay` | `HasPcNameMismatch=true` で `MismatchBg` + `MismatchFont` + 太字 |
| 3 | シリアル番号 | 同 | `pc.SerialNumber` | |
| 4 | SN ソース | 同 | `SerialSourceEntry.ShortLabel(pc.SerialNumberSource)` | 取得不能=Gray、フォールバック=`CautionBg` + `CautionFont` + 太字 |
| 5 | 収集日 | 同 | `pc.CollectionDate` | |
| 6 | 突合判定 | 同 | OK / NG (N件) / 対象外 / 未突合 | OK=`MatchBg`+`MatchFont`+太字、NG=`MismatchBg`+`MismatchFont`+太字、対象外/未突合=Gray |
| 7〜 | NIC{N}_アダプタ / NIC{N}_接続名 / NIC{N}_MAC | `NicHeaderBg` 緑 | `pc.MacAddresses` を ConnectionName でソートして展開 | |
| | {drive}ドライブ_キーID / {drive}ドライブ_回復キー | `KeyHeaderBg` 黄 | フリート全機の DriveLetter 集合をソート、各 PC の対応キーを展開 | |
| | バッテリ健全性 (%) / 設計容量 (mWh) / 満充電容量 (mWh) | `LicenseHeaderBg` 紫 | `pc.BatteryReport` 各値 | 健全性: ≥95% Good / 90-95% Warning / <90% Bad / null `WriteNa` |
| | 総容量(GB) / スロット使用 | `MemoryHeaderBg` 青緑 | `MemorySlots.Sum(CapacityGB)` / `"装着/全スロット"` 形式 | MemorySlots 空時は `WriteNa` 2 件 |
| | GP Status | `GpHeaderBg` 濃オレンジ | `OK` / `Failed (code N)` / `(取得不能)` | `WriteGroupPolicyStatusCell` で着色 |
| | HWメーカ / HWモデル / マザーボードSN / SMBIOS UUID | `HwIdHeaderBg` 濃グレー青 | `cs.Manufacturer` (なければ `csp.Vendor`) / `cs.Model` / `bb.SerialNumber` / `csp.Uuid` | |
| 末尾 | 管理者メモ | `BaseHeaderBg` 青 | `TruncateForCell(pc.Memo?.Text)` | `WrapText=true` |

NIC 列数 = フリート全機の `MacAddresses.Count` の最大値（`maxNicCount`）。BitLocker 列数 = フリート全 `DriveLetter` の `Distinct + OrderBy` 集合のサイズ × 2。

PC 名列のハイパーリンクは pc_details/ サブブックがあるときのみ張る（`detailRelativePath is not null`）。

---

## ベースライン突合シート 6 種

`BaselineReports : Dictionary<PcEvidence, BaselineComparisonReport>` を入力として、6 サブカテゴリそれぞれが独立シートに展開される。**pc_details/ 側のみ**（main には出ない）。

### 共通の判定セル `ApplyVerificationStatusToCell`

`VerificationStatus` を Excel セルに反映する共通ヘルパ（SystemInfo / Checklist VerifyItems / InstalledApps / License / DomainStatus 5 シートで共用）：

| Status | 表示テキスト | 着色 | 行全体着色 |
|---|---|---|---|
| `Match` | `OK` | `MatchFont` + `MatchBg` | なし |
| `Mismatch` | `NG` | `MismatchFont` + `MismatchBg` + 太字 | 行全体 `MismatchBg` |
| `NoExpected` | `NEW`（ベースラインなし、対象 PC のみに値あり = 「追加」） | `#1565C0`（青） | なし |
| `NoActual` | `---`（ベースラインあり、対象 PC に値なし = 「欠損」） | `#E65100`（オレンジ）+ 太字 | 行全体 `MismatchBg` |

### `ベースライン_実行サマリ`（6 列）

| # | 列 | 内容 |
|---|---|---|
| 1 | PC名 (計画\|実) | `pc.PcNameDisplay` |
| 2 | モジュール名 | `item.ModuleName` |
| 3 | カテゴリ | `item.Category` |
| 4 | 期待ステータス | `item.ExpectedStatus` |
| 5 | 実測ステータス | `item.ActualStatus` |
| 6 | 判定 | `OK` / `NG` / `未実行` / `追加` |

`MatchStatus` 別の着色：

- `Match` → OK + `MatchFont` + `MatchBg`
- `Mismatch` → NG + `MismatchFont` + 太字 + 行全体 `MismatchBg`
- `MissingInActual` → 未実行（baseline にあるが対象 PC 未実行）+ オレンジ文字 + 行全体 `MismatchBg`
- `ExtraInActual` → 追加（対象 PC のみ）+ Gray

### `ベースライン_SystemInfo`（5 列）/ `_ドメイン`（5 列）/ `_ライセンス`（5 列）

すべて同じ列構成: PC名 / 項目 / 期待値 / 実測値 / 判定。`ApplyVerificationStatusToCell` で着色。

### `ベースライン_チェックリスト`（6 列）

| # | 列 |
|---|---|
| 1 | PC名 (計画\|実) |
| 2 | 種別（"全体判定" or "検証項目"） |
| 3 | 項目（"OverallStatus" or `item.ItemName`） |
| 4 | 期待値 |
| 5 | 実測値 |
| 6 | 判定 |

OverallStatus 行 + 各 VerifyItem 行で計 N+1 行/PC。OverallStatusMatch は `OK / NG` 表示で `ApplyVerificationStatusToCell` ではなく独自着色（行全体 `MismatchBg`）。

### `ベースライン_アプリ`（5 列）

PC ごとに「集計行 1 行 + 差分行 N 行」の 2 段構成：

**集計行**: アプリ名=`(集計)` / 期待バージョン=`ベースライン M件` / 実測バージョン=`対象 N件` / 判定=`一致 K件 / 差分 L件`（一致のみ緑太字、差分ありで赤太字）

**差分行**: 通常の Status 着色（`Mismatch` / `NoActual` / `NoExpected`）

---

## 1 PC 1 行系 シート詳細（main 台帳）

### `ドメイン状態`（9 列）

PC名 / PartOfDomain / ドメイン(WORKGROUP) / ドメインロール / 現在のユーザー / AzureAD参加 / ドメイン参加 / ドメイン名 / テナント名

`bool` 値はすべて `YES / NO`（着色なし）。ヘッダ色 `DomainHeaderBg` ダークグレー。

### `Windows Defender`（15 列）

| # | 列 | 着色 |
|---|---|---|
| 1 | PC名 (計画\|実) | |
| 2-9 | AMService / Antispyware / Antivirus / リアルタイム保護 / 動作監視 / IOAV保護 / NIS / オンアクセス保護 | `ON`=`MatchFont`、`OFF`=`MismatchFont`+太字 |
| 10 | エンジンVer | |
| 11 | 製品Ver | |
| 12 | 定義Ver | |
| 13 | 定義更新日 | |
| 14 | クイックスキャン（最終終了時刻） | |
| 15 | フルスキャン（最終終了時刻） | |

### `Windowsライセンス`（17 列）

`Products[]` ごとに 1 行（複数製品なら 1 PC = 複数行、製品 0 件は `(プロダクトキー未検出) / 未取得` で 1 行）。

PC名 / シリアル番号 / 収集日 / プロダクト名 / ライセンス状態 / プロダクトキー末尾 / ライセンスファミリ / プロダクトキーチャネル / 残り猶予期間(分) / KMS configured / KMS discovered / ClientMachineID / KMS Machine / KMS Port / OA3 Original Key / OA3 Key Description / PolicyCacheRefreshRequired

**監査着色**: `LicenseStatusCode != 1`（Licensed 以外）または `product is null` のとき **行全体に `MismatchBg` + `MismatchFont`**。

### `Officeライセンス`（23 列）

`Products[]` ごとに 1 行。製品 0 件 / 未インストールは 1 PC = 1 行（5-23 列が空）。

| 列範囲 | グループ |
|---|---|
| 1-3 | PC 名 / シリアル / 収集日 |
| 4 | Office インストール状態（`インストール済み / 未インストール`） |
| **5-9** | **vNext サマリブロック（M11 で追加）**: SubscriptionType / Verdict / Conclusion / LicensedUsers / TenantId |
| 10-15 | C2R 詳細: ProductReleaseIds / バージョン / Platform / UpdateChannel / AudienceData / ClientCulture |
| 16 | OSPP パス |
| 17-23 | OSPP 製品詳細: LICENSE NAME / LICENSE DESCRIPTION / ライセンス状態 / プロダクトキー末尾 / ERROR CODE / REMAINING GRACE / KMS Machine |

**SubscriptionType 列**（5 列目）の解決ロジック `ResolveSubscriptionType`：

- `!IsOfficeInstalled` → `"Not installed"`
- `IsSubscriptionDetected is null` → `"Unknown"`（旧 evidence v1.4 以前）
- `IsSubscriptionDetected = true` → `"M365 Subscription"`
- `IsSubscriptionDetected = false` + `ProductReleaseIds.Contains("Volume")` → `"Volume"`
- `IsSubscriptionDetected = false` + その他 → `"Buy-once"`

**監査着色** `ApplyOfficeLicenseRowColor`：

- `verdict is null`（旧 evidence、manifest §22 不在）: 未インストール=`InfoBg`、`!product.IsLicensed`=`MismatchBg`+`MismatchFont`
- `verdict = Failed`: 行全体 `MismatchBg` + `MismatchFont`
- `verdict = Partial`: 行全体 `WarningBg`（薄黄）
- `verdict = Success` + 未インストール: 行全体 `InfoBg`（情報行）
- `verdict = Success` + インストール済: 行全体 `MatchBg`
- `verdict = Skipped`: Gray 文字色

### `セキュリティベースライン`（10 列）

| # | 列 | バインド | 着色（`AuditLevel`） |
|---|---|---|---|
| 1 | PC名 | `pc.PcNameDisplay` | |
| 2 | TPM Present | `sb.Tpm?.Present` | `false=Bad`（`badIfFalse`）/ null=`Na` |
| 3 | TPM Ready | `sb.Tpm?.Ready` | 同上（M12 仕様: Ready=false は赤）|
| 4 | SecureBoot | `sb.SecureBootEnabled` | `false=Warning` / null=`Na` |
| 5 | VBS Status | `sb.Vbs?.VbsStatusText` | `"Not running"` で `Warning` |
| 6 | HVCI Running | `sb.Vbs?.HvciRunning` | `false` で `Warning` |
| 7 | CodeIntegrity Enforcement | `sb.Vbs?.CodeIntegrityEnforcementText` | `"Off"` で `Warning` |
| 8 | LSA RunAsPPL | `sb.Lsa?.RunAsPplRawText` | `Off=Bad` / `OnWithUefiLock=Good` / `Unknown=Na` / `On`=通常 |
| 9 | BIOS Version | `sb.Bios?.SmbiosBiosVersion` | |
| 10 | BIOS Date | `sb.Bios?.ReleaseDate` | |

### `HW識別子`（27 列）

4 ブロック全列を prefix 付きで展開。各ブロックが `null` のとき該当列範囲は `WriteNa` で `N/A` 埋め。

| 列範囲 | プレフィックス | 元 WMI クラス | 列 |
|---|---|---|---|
| 1 | (なし) | | PC名 (計画\|実) |
| 2-8 | `CS_` | Win32_ComputerSystem | Manufacturer / Model / SystemFamily / SystemSKUNumber / TotalPhysicalMem / NumberOfProcessors / NumberOfLogicalProcessors |
| 9-14 | `CSP_` | Win32_ComputerSystemProduct | Vendor / Name / Version / IdentifyingNumber / UUID / SKUNumber |
| 15-19 | `BB_` | Win32_BaseBoard | Manufacturer / Product / Version / SerialNumber / Tag |
| 20-27 | `SE_` | Win32_SystemEnclosure | Manufacturer / Model / ChassisTypes / SerialNumber / SMBIOSAssetTag / AssetTag / SecurityStatus / LockPresent |

### `グループポリシー`（7 列）

PC名 / GP Status（`OK` / `Failed (code N)` / `(取得不能)` 着色）/ Exit Code / HTML Size (bytes) / PartOfDomain / Domain / ExecutingUser

**HTML への hyperlink は付与しない**（delivery 相対パス計算が複雑、後続フェーズで保留）。HTML 確認は PcDetailWindow のブラウザボタン経由で行う。

---

## pc_details/ 側 シート詳細

### `突合詳細`（5 列）

PC名 / チェック項目 / 期待値 (hostlist) / 実測値 (evidence) / 判定

判定列：`OK / NG / 期待値なし / 実測値なし` で着色（`OK=MatchBg` / `NG=MismatchBg`+行全体 / `期待値なし=Gray` / `実測値なし=オレンジ太字`+行全体 `MismatchBg`）。

### `チェックリスト`（9 列）

PC名 / 全体判定 / # / モジュール名 / カテゴリ / 結果 / ベリファイ / 実行時刻 / メッセージ

**Verified 列（kernel 2.x で新設）が Result の右に挿入**。`ChecklistResult.ModuleItems` が空のときは PC 単位で全体判定 1 行のみ。`ApplyStatusColor` で各セルを着色。Message は `TruncateForCell` で切り詰め。

### `実行履歴`（8 列）

PC名 / タイムスタンプ / モジュール名 / カテゴリ / ステータス / メッセージ / 作業者 / セッションID

ステータスセルは `ApplyStatusColor`、メッセージは `TruncateForCell`。

### `ユーザー・グループ`（8 列）

`LocalUsers` / `LocalGroups` / `LocalGroupMembers` を 1 シートに 3 種別混在で展開。

| # | 列 | バインド |
|---|---|---|
| 1 | PC名 (計画\|実) | |
| 2 | 種別 | `"ユーザー"` / `"グループ"` / `"メンバー"` |
| 3 | 名前 | ユーザー名 / グループ名 / `"{GroupName} → {MemberName}"` |
| 4 | 有効 | ユーザーのみ `有効/無効`、無効=Gray、グループ・メンバーは空 |
| 5 | フルネーム | ユーザーのみ |
| 6 | 説明 | ユーザー / グループ |
| 7 | ソース | ユーザー / メンバーの PrincipalSource |
| 8 | 最終ログオン | ユーザーのみ |

### `FWプロファイル`（6 列）

PC名 / プロファイル / 有効（`有効/無効`） / 受信デフォルト / 送信デフォルト / ログファイル

### `FWルール`（6 列）

PC名 / ルール名 / 有効 / 方向 / アクション / プロファイル

### `オプション機能`（3 列）

PC名 / 機能名 / 状態（`Enabled` 緑、その他は Gray）

### `ユーザープロファイル`（5 列）

PC名 / ローカルパス / SID / 最終使用日時 / ロード済み（YES/NO）

### `ディスク・パーティション`（9 列）

`Disks` と `Partitions` を 2 種別混在で展開：

| # | 列 | ディスク行 | パーティション行 |
|---|---|---|---|
| 1 | PC名 | | |
| 2 | 種別 | `"ディスク"` | `"パーティション"` |
| 3 | 番号 | `Number` | `"{DiskNumber}-{PartitionNumber}"` |
| 4 | 名前 | `FriendlyName` | `DriveLetter` |
| 5 | シリアル番号 | `SerialNumber` | (空) |
| 6 | サイズ(GB) | `SizeGB` | `SizeGB` |
| 7 | パーティションスタイル | `PartitionStyle` | `Type` |
| 8 | 正常性 | `HealthStatus` | `IsSystem ? "System" : ""` |
| 9 | 動作状態 | `OperationalStatus` | `IsBoot ? "Boot" : ""` |

### `WiFiプロファイル`（2 列）

PC名 / プロファイル名

### `復元ポイント`（5 列）

PC名 / シーケンス番号 / 説明（Truncate） / 種類 / 作成日時

### `Windows Update`（5 列）

PC名 / HotFixID / 説明（Truncate） / インストール者 / インストール日

### `証明書`（12 列）

PC名 / Store / Subject / Issuer / Thumbprint / NotBefore / NotAfter / **DaysToExpiry**（着色）/ HasPrivateKey / EnhancedKeyUsageList / FriendlyName / SerialNumber

DaysToExpiry の着色：

| 残日数 | AuditLevel |
|---|---|
| < 0（期限切れ） | `Bad`（赤太字） |
| 0〜90 | `Warning`（薄黄背景） |
| > 90 | 通常（着色なし） |
| null（NotAfter なし） | `Na`（Gray、`"N/A"`） |

### `メモリ`（最大 14 列、2 セクション構成）

`AutoFilter` は付けない（`ApplyStyles` を使わずカスタム）。

**セクション 1: メモリスロット (Win32_PhysicalMemory)** — 14 列：

PC名 / BankLabel / DeviceLocator / Capacity (GB) / Speed (MHz) / ConfiguredClockSpeed (MHz) / ConfiguredVoltage (mV) / Manufacturer / PartNumber / SerialNumber / FormFactor / SMBIOSMemoryType / DataWidth (bit) / TotalWidth (bit)

セクションタイトル行: 太字 + 12pt + 薄シアン背景 (`#E0F2F1`)、列方向にセル結合。

**区切り**: 空行 1 行

**セクション 2: メモリアレイサマリ (Win32_PhysicalMemoryArray)** — 7 列：

PC名 / Tag / Location / Use / MemoryErrorCorrection / MaxCapacity (GB) / MemoryDevices

ヘッダ行（セクション 1 の row 2）を `FreezeRows(2)` で固定。

### `環境変数`（4 列）

PC名 / Scope / Name / Value（Truncate）

### `スタートアップ`（7 列）

PC名 / Source / Name / User / Location / Command（Truncate） / Enabled

### `PnPデバイス`（10 列）

PC名 / Class / FriendlyName / Status / Present / Manufacturer / Service / DriverVersion / DriverDate / InstanceId（Truncate）

200〜400 行/PC 規模のため、AutoFilter で絞込みが効くことが重要。

### 未知 section シート `§{Id} {Title}`

タブ色 `Yellow` で警告表示。シート名は `SanitizeSheetName` で禁則文字 `\ / * ? : [ ]` を除去 + 31 文字上限。

| 行 | 内容 |
|---|---|
| 1 | `fabriq_evidence_manager v{ManagerVersion} (manifest 経由・未対応セクション)`（紫太字） |
| 2 | `§{Id} {Title}`（太字） |
| 3 | (空) |
| 4 | データテーブルヘッダ（紫 `#7B1FA2`）: PC名 / ファイル名 / 種別 / raw 内容 |
| 5- | section 1 ファイル = 1 行、`raw 内容` セルは `WrapText` + `VerticalAlignment.Top`、`TruncateForCell` |

列幅: 1-3 列 `AdjustToContents`、4 列目（raw 内容）は固定 80 幅。`FreezeRows(headerRow=4)` で 4 行目のヘッダを固定。

`ManagerVersion` は `typeof(ExcelExportService).Assembly.GetName().Version?.ToString(3)` で本アプリのアセンブリバージョンを取得（前方互換情報の記録）。

---

## ファイル名衝突回避（pc_details/ 側）

`GetPcDetailFileName(pc, usedNames)` の段階的回避：

1. `{PcName}_{CollectionDate}.xlsx` を試す → 衝突なら次へ
2. `{PcName}_{CollectionDate}_{Serial 先頭 6 文字 or "noserial"}.xlsx` を試す → 衝突なら次へ
3. `{PcName}_{CollectionDate}_{Serial 先頭 6}_{counter=2,3,4,...}.xlsx` で連番

`SanitizeFileNameSegment` で Windows ファイル名禁則文字 `\ / * ? : [ ] " < > |` を除去。

---

## 関連ドキュメント

- 納品出力フロー: [fabriq_evidence_manager__usage__04_export_delivery.md](fabriq_evidence_manager__usage__04_export_delivery.md)
- 入力 evidence 構造: [fabriq_evidence_manager__architecture__02_evidence_input.md](fabriq_evidence_manager__architecture__02_evidence_input.md)
- セクション ID dispatch 契約: [fabriq_evidence_manager__contracts__section_dispatch.md](fabriq_evidence_manager__contracts__section_dispatch.md)
- 階層構造（ExcelExportService 概要）: [fabriq_evidence_manager__architecture__01_layers.md](fabriq_evidence_manager__architecture__01_layers.md)
- ベースライン突合の使い方: [fabriq_evidence_manager__usage__03_baseline.md](fabriq_evidence_manager__usage__03_baseline.md)
- hostlist 突合の使い方: [fabriq_evidence_manager__usage__02_hostlist_verification.md](fabriq_evidence_manager__usage__02_hostlist_verification.md)
