# セクション ID dispatch 契約

> **対象**: fabriq_evidence_manager / contracts
> **対象バージョン**: 3.8.1（取得元: `E:\fabriq_evidence_manager\FabriqEvidenceManager\FabriqEvidenceManager.csproj` `<Version>`、最新コミット `45eae22` (2026-05-07)）
> **対応 producer 契約**: fabriq kernel `EVIDENCE_MANIFEST.md schemaVersion=1` / evidence_config 1.6.0 (kernel 3.0.0)
> **ドキュメント更新日**: 2026-05-07

## 位置づけ

`EvidenceParserService.DispatchSection(pc, dir, section)` 内の switch が **manifest セクション ID を入力として、対応するパース処理と PcEvidence 出力先を決定** する。本ドキュメントは ID → ファイル名サフィックス → パーサ実装 → 出力 Model の写像を全 31 セクション（+ 副 ID `8b`）について明文化する。

producer 側の出力ファイル名規約・status セマンティクスは [fabriq__contracts__evidence_manifest_contract.md](fabriq__contracts__evidence_manifest_contract.md)、consumer 側の manifest 検証は [fabriq_evidence_manager__contracts__manifest_schema.md](fabriq_evidence_manager__contracts__manifest_schema.md)、データフロー全体像は [fabriq_evidence_manager__architecture__02_evidence_input.md](fabriq_evidence_manager__architecture__02_evidence_input.md) を参照。

implementation entry point は [E:\fabriq_evidence_manager\FabriqEvidenceManager\Services\EvidenceParserService.cs](file:///E:/fabriq_evidence_manager/FabriqEvidenceManager/Services/EvidenceParserService.cs) の `DispatchSection` メソッド、既知 ID マスタは [Helpers/EvidenceConstants.cs](file:///E:/fabriq_evidence_manager/FabriqEvidenceManager/Helpers/EvidenceConstants.cs) `KnownSections` ディクショナリ。

## dispatch ポリシー

`EvidenceParserService.PopulateDetails(pc)` は次の流れで manifest 駆動パースを行う：

```
1. pc.Manifest is null なら全スキップ（ManifestError は UI に出ているはず）
2. pc.Manifest.Sections を順次走査
3.   section.Status ∈ { Failed, Skipped } → スキップ（壊れた / 意図的省略）
4.   section.Status ∈ { Success, Partial } → DispatchSection(pc, dir, section)
5. 最後に bitlocker/*.txt を manifest 外で個別走査
6. ValidateEvidence で初回 Warning を立てる
```

### Status 別の取り扱い

| Status | 取り扱い | 理由 |
|---|---|---|
| `Success` | dispatch する | 正常完了、`files[]` 全部書き込まれている |
| `Partial` | **dispatch する** | 単一 section 内で複数処理の一部のみ成功（例: §11 DesktopApps + StoreApps の片方失敗）。書き込まれた分は読みたい |
| `Skipped` | スキップ | 意図的省略（Server-only セクションを Client OS で実行 等）。`files[]` は通常空 |
| `Failed` | スキップ | 例外発生で完了不可。producer 契約により `files[]` は **常に空配列**（途中まで書かれた壊れたファイルは載せない） |

### 3 種類の dispatch ルート

セクションは出力先・パース手段により次の 3 種に分類される：

- **Eager parse（自前）**: `EvidenceParserService` 内蔵の `Parse...` メソッドで直接パース、`PcEvidence` のフィールドに格納
- **Eager parse（サブパーサ委譲）**: `IXxxParserService` インターフェース経由でサブパーサに委譲、`PcEvidence` のフィールドに格納
- **On-demand parse**: `PopulateDetails` では格納せず、`EvidenceVerificationService` / `BaselineService` 等が必要時に該当ファイルパスを `IEvidenceParserService` に渡して parse する。メモリ消費抑制のため

## §01〜§31 完全 dispatch 表

| ID | KnownSections.DisplayName | files[] サフィックス | dispatch | パーサ | PcEvidence 出力 |
|---|---|---|---|---|---|
| 01 | System Basic Info | `01_SystemInfo.txt` | On-demand | `IEvidenceParserService.ParseSystemInfo / ParseHostname` | (Verification + Baseline 経由でのみ参照) |
| 02 | Local Users | `_LocalUsers.csv` | Eager（自前） | `ParseLocalUsers` | `LocalUsers : List<LocalUserEntry>` |
| 03 | Local Groups | `_LocalGroups.csv` | Eager（自前） | `ParseLocalGroups` | `LocalGroups` |
| 04 | Local Group Members | `_LocalGroupMembers.csv` | Eager（自前） | `ParseLocalGroupMembers` | `LocalGroupMembers` |
| 05 | Domain & User Profiles | `_DomainStatus.txt` + `_UserProfiles.csv` | Eager（自前 / 2 ファイル） | `ParseDomainStatus` + `ParseUserProfiles` | `DomainStatus : DomainStatusData?` + `UserProfiles` |
| 06 | Network Settings | `06_NetworkConfig.csv` | On-demand | `ParseNetworkConfig` | (Verification 経由) |
| 07 | Printers | `07_Printers.csv` | On-demand | `ParsePrinters` | (Verification 経由) |
| 08 | BitLocker Status | `08_BitLocker.txt` | On-demand | `ParseBitLockerStatus` | (Verification 経由) |
| 8b | Disk & Partition | `_Disks.csv` + `_Partitions.csv` | Eager（自前 / 2 ファイル） | `ParseDisks` + `ParsePartitions` | `Disks` + `Partitions` |
| 09 | MAC Addresses | `_MacAddress.csv` | Eager（自前） | `ParseMacAddresses` | `MacAddresses` |
| 10 | PC Serial Number | `_SerialNumber.txt` | Eager（自前 / 専用ロジック） | `ApplySerialNumberDetail` → `ParseSerialNumberDetailed` | `SerialNumber / SerialNumberSource / SerialReferenceUuid / SerialSourceTrail` |
| 11 | Installed Software | `11_DesktopApps.csv` + `11_StoreApps.csv` | On-demand | `ParseDesktopApps` + `ParseStoreApps` | (`DesktopAppsComparator` + `StoreAppsComparator` 経由、v3.8.1 で分割) |
| 12 | Firewall | `_FirewallProfiles.csv` + `_FirewallRules.csv` | Eager（自前 / 2 ファイル） | `ParseFirewallProfiles` + `ParseFirewallRules` | `FirewallProfiles` + `FirewallRules` |
| 13 | Optional Features | `_OptionalFeatures.csv` | Eager（自前） | `ParseOptionalFeatures` | `OptionalFeatures` |
| 14 | Server Roles & Features | (Client OS では Skipped) | dispatch 対象外 | — | — |
| 15 | Power Settings | `_PowerSettings.txt` | Eager（自前） | `ParsePowerSettings` | `PowerSettings : PowerSettingsData?` |
| 16 | WiFi Profiles | `_WiFiProfiles.txt` | Eager（自前） | `ParseWiFiProfiles` | `WiFiProfiles : List<string>` |
| 17 | Restore Points | `_RestorePoints.csv` | Eager（自前） | `ParseRestorePoints` | `RestorePoints` |
| 18 | Windows Defender | `_DefenderStatus.txt` | Eager（自前） | `ParseDefenderStatus` | `DefenderStatus : DefenderStatusData?` |
| 19 | Windows Update | `_WindowsUpdates.csv` | Eager（自前） | `ParseWindowsUpdates` | `WindowsUpdates` |
| 20 | System TEMP Backup | (forensic dump dir、本アプリはパース対象外) | dispatch 対象外 | — | — |
| 21 | Windows License | `_WindowsLicense.txt` | Eager（サブパーサ） | `IWindowsLicenseParserService.Parse` | `WindowsLicense : WindowsLicenseData?` |
| 22 | Office License | `_OfficeLicense.txt` + `_OfficeVnextLicenses.csv`（任意） | Eager（サブパーサ / 専用ロジック） | `ApplyOfficeLicenseDetail` → `IOfficeLicenseParserService.Parse(txt, csv?)` | `OfficeLicense : OfficeLicenseData?` |
| 23 | Security Baseline | `_SecurityBaseline.txt` | Eager（サブパーサ） | `ISecurityBaselineParserService.Parse` | `SecurityBaseline : SecurityBaselineData?`（5 ブロック個別退避） |
| 24 | Group Policy Report | `_GroupPolicySummary.txt` + `_GroupPolicy.html` | Eager（サブパーサ / 専用ロジック） | `ApplyGroupPolicyDetail` → `IGroupPolicyParserService.Parse(summary, html?)` | `GroupPolicyReport : GroupPolicyReport?`（HTML はパス保持のみ） |
| 25 | Certificates | `_Certificates.csv` | Eager（サブパーサ） | `ICertificatesParserService.Parse` | `Certificates : List<CertificateEntry>` |
| 26 | Battery Report | `_BatteryReport.html` | Eager（サブパーサ） | `IBatteryReportParserService.Parse` | `BatteryReport : BatteryReportData?`（デスクトップ機では null） |
| 27 | Environment Variables | `_EnvironmentVariables.csv` | Eager（サブパーサ） | `IEnvironmentVariablesParserService.Parse` | `EnvironmentVariables : List<EnvironmentVariableEntry>`（Machine + User 混在） |
| 28 | Startup Items | `_StartupItems.csv` | Eager（サブパーサ） | `IStartupItemsParserService.Parse` | `StartupItems : List<StartupItemEntry>`（Registry + Task 混在） |
| 29 | Memory Slots & Array | `_MemorySlots.csv` + `_MemoryArraySummary.csv` | Eager（サブパーサ / 2 メソッド） | `IMemoryInventoryParserService.ParseSlots / ParseArraySummary` | `MemorySlots` + `MemoryArraySummaries` |
| 30 | PnP Devices | `_PnpDevices.csv` | Eager（サブパーサ） | `IPnpDevicesParserService.Parse` | `PnpDevices : List<PnpDeviceEntry>`（200〜400 件規模） |
| 31 | Hardware Identifiers | `_HardwareIdentifiers.txt` | Eager（サブパーサ） | `IHardwareIdentifiersParserService.Parse` | `HardwareIdentifiers : HardwareIdentifiersData?`（4 ブロック個別退避） |

## ファイルサフィックス一致ルール

`section.RegularFiles` の各要素は producer 側で「`{番号 or ID}_{名前}.{拡張子}`」形式で出力されるが、consumer 側は **`EndsWith(suffix, OrdinalIgnoreCase)` で末尾一致** で引き当てる：

```csharp
var fileName = section.RegularFiles.FirstOrDefault(
    f => f.EndsWith("_LocalUsers.csv", StringComparison.OrdinalIgnoreCase));
```

これにより producer 側のファイル名プレフィックス（`02_` 等の番号部分）が将来変わっても consumer 側を改修せずに追従できる。一致しなければスキップ（throw しない）。

## On-demand parse の理由

On-demand に分類される §01 / §06 / §07 / §08 / §11 は **`PopulateDetails` で `PcEvidence` フィールドに常時格納しない**。理由：

- **§01 SystemInfo** は OS 名・バージョン・CPU・メモリ等の文字列フィールドで構造が薄く、`Verification` と `BaselineService` が必要時にパースする方が多態的に対応できる
- **§06 NetworkConfig / §07 Printers** は hostlist 突合（`EvidenceVerificationService`）の入力としてのみ使用。フリート 100 台×NIC 5 件の格納は無駄
- **§08 BitLocker Status** は `Verification` の C ドライブ判定でのみ使用。ボリューム単位の状態は Excel 主シートには出さず、`bitlocker/` の回復キー情報（manifest 外）を主軸にしているため
- **§11 InstalledApps** は `BaselineService.InstalledAppsComparator` でのみ使用。1 PC あたり 100〜500 件で、フリート 50 台なら最大 25,000 件のメモリ常駐は避けたい

これらは `IEvidenceParserService` のインターフェースとして parse メソッドが公開されており、利用側が `pc.PcInformationDirectoryPath` と `EvidenceConstants.FileXxx` 定数を組み合わせて呼び直す。

## 多ファイルセクションの取り扱い

1 セクションが複数ファイルを出力するパターン：

| ID | ファイル群 | dispatch 実装 |
|---|---|---|
| §05 | `_DomainStatus.txt` + `_UserProfiles.csv` | switch case 内で `TryParseSingle` + `TryParseList` を 2 回 |
| §8b | `_Disks.csv` + `_Partitions.csv` | 同上 |
| §11 | `11_DesktopApps.csv` + `11_StoreApps.csv` | On-demand（`InstalledAppsComparator` 内で 2 回パース） |
| §12 | `_FirewallProfiles.csv` + `_FirewallRules.csv` | switch case 内で `TryParseList` を 2 回 |
| §22 | `_OfficeLicense.txt` + `_OfficeVnextLicenses.csv`（任意） | 専用ヘルパ `ApplyOfficeLicenseDetail`（CSV 不在時 null 渡し） |
| §24 | `_GroupPolicySummary.txt` + `_GroupPolicy.html` | 専用ヘルパ `ApplyGroupPolicyDetail`（HTML はパス保持のみ） |
| §29 | `_MemorySlots.csv` + `_MemoryArraySummary.csv` | switch case 内で `TryParseList` を 2 回（サブパーサに 2 メソッド） |

`TryParseSingle<T>(dir, section, suffix, parser, setter)` と `TryParseList<T>(dir, section, suffix, parser, destination)` は `EvidenceParserService` 内の汎用ヘルパ。section.RegularFiles から suffix 一致ファイルを 1 つ選び、I/O 例外時は Debug.WriteLine + 続行する設計（詳細は [fabriq_evidence_manager__architecture__02_evidence_input.md](fabriq_evidence_manager__architecture__02_evidence_input.md) §「ファイル I/O の規約」参照）。

## 専用ロジックを持つセクション

汎用ヘルパで処理せず専用メソッドが切られている 3 セクション：

### §10 PC Serial Number

`ApplySerialNumberDetail(pc, dir, section)` が `_SerialNumber.txt` 1 ファイルを 4 ブロック構成（`Canonical Serial Number` / `All Sources` / `Reference ID` / `Selection Policy`）の state machine でパースする。`ParseSerialNumberDetailed` の戻り値 `SerialNumberDetail` を 4 つの PcEvidence プロパティ（`SerialNumber / SerialNumberSource / SerialReferenceUuid / SerialSourceTrail`）に分解して充填。Primary canonical = `Win32_BIOS.SerialNumber` の判定とフォールバック警告ロジックを含む。

### §22 Office License

`ApplyOfficeLicenseDetail(pc, dir, section)` が `.txt` を必須・`.csv` を任意として `IOfficeLicenseParserService.Parse(txtPath, csvPath?)` に渡す。CSV 不在時（v1.4 以前 evidence や C2R 未検出機）は `null` を渡し、サブパーサ側で `OfficeLicenseData.VnextEntries` を空配列にする。

### §24 Group Policy

`ApplyGroupPolicyDetail(pc, dir, section)` が `_GroupPolicySummary.txt` を必須・`_GroupPolicy.html` をパス参照として `IGroupPolicyParserService.Parse(summary, html?)` に渡す。**HTML 本体（240KB 規模）は読み込まず**、`GroupPolicyReport.HtmlFilePath` に絶対パスのみ保持し、`PcDetailViewModel.OpenGroupPolicyHtmlCommand` で OS 既定ブラウザに `Process.Start` で渡す設計（メモリ消費抑制）。HTML 不在時（gpresult 失敗）は `htmlPath = null` で渡す。

## bitlocker/ ディレクトリ（manifest 外）

回復キーは fabriq 側の運用上、**`pc_information/manifest.json` ではなく `bitlocker/{pcName}_{drive}.txt` という別ディレクトリに別出し** されている（manifest §08 BitLocker は status 情報のみを保持）。

`PopulateDetails` は manifest 走査が終わった後に `pc.BitLockerDirectoryPath` 配下の全 `.txt` を `ParseBitLockerKey` で順次パースし、`pc.BitLockerKeys : List<BitLockerEntry>` に詰め込む：

| 抽出フィールド | 抽出元 |
|---|---|
| ドライブレター | ファイル名末尾 `_{drive}` から、取れなければ本文 `Mount Point: X:` |
| Identifier (GUID) | 本文中の `Identifier: {xxxxxxxx-...}` 正規表現マッチ |
| Recovery Password | 本文中の `Recovery Password: 123456-123456-...`（6 桁 × 8 ハイフン区切り） |

回復パスワードが取れない場合はエントリ生成しない（実用価値なし、`null` を返す）。

## 未知セクション ID の捕捉

producer 契約 §「前方互換ルール」に基づき、本アプリは **`EvidenceConstants.KnownSections` に登録のない section.Id を `UnknownSection` として raw 保持** する。これにより producer 側が将来 `§32` 等を追加しても、consumer のバージョンアップなしに raw 表示状態で取り込める。

実装は `EvidenceParserService.DispatchSection` の switch default 節：

```csharp
default:
    if (!EvidenceConstants.KnownSections.ContainsKey(section.Id))
        CaptureUnknownSection(pc, dir, section);
    break;
```

`KnownSections` に登録されている **既知だが dispatch 対象外** の §14 / §20 はここでも捕捉対象外（switch default に入っても `ContainsKey` が true で skip）。これは「producer 側が意図的に出力していない / forensic dump として opaque」なセクションなので、空のまま放置するのが正しい。

`CaptureUnknownSection` の動作：

```csharp
foreach (var file in section.RegularFiles)
{
    var path = Path.Combine(dir, file);
    if (!File.Exists(path)) continue;

    var fileType = file.EndsWith(".csv", OrdinalIgnoreCase)
        ? UnknownSectionFileType.Csv : UnknownSectionFileType.Text;

    pc.UnknownSections.Add(new UnknownSection {
        Id = section.Id,
        Title = section.Title,
        FilePath = path,
        FileType = fileType,
        RawContent = File.ReadAllText(path, Encoding.UTF8),
    });
}
```

その後 `ExcelExportService.WriteUnknownSectionSheets` が **未知 ID ごとに専用シート（命名規則: `§<id> {Title}`）を動的生成** し、CSV 列頭は実データから推定、Text は raw 出力する。これにより監査人が新セクションの中身を確認できる状態を維持する（producer 契約 §「外部ツール（manager 等）の責任」§2「未知 section ID は raw 表示。クラッシュさせない」準拠）。

## §14 / §20 の dispatch 対象外扱い

| ID | KnownSections 登録 | dispatch 対象外 | 理由 |
|---|---|---|---|
| §14 Server Roles & Features | あり | 対象外 | Client OS では producer 側が `Skipped` で出すため、そもそも `DispatchSection` まで来ない。Server OS で実行された場合の dispatch は将来課題（現状は raw 化もしない） |
| §20 System TEMP Backup | あり | 対象外 | producer が forensic dump dir として `files: ["20_TempBackup.txt", "20_TempBackup/"]` を出すが、consumer は ディレクトリ要素を opaque に扱う方針（producer 契約 §「ディレクトリ表現」準拠）。`20_TempBackup.txt` 単独もパース対象外 |

両者とも `KnownSections` に登録されているのは Excel 出力やダイアログ表示で「セクション名」が必要なため。

## 既知 ID 一覧（KnownSections マスタ）

`EvidenceConstants.KnownSections` ディクショナリ（`IReadOnlyDictionary<string, KnownSection>`、`StringComparer.Ordinal`）の現行値：

```csharp
["01"] = "System Basic Info",        ["02"] = "Local Users",
["03"] = "Local Groups",             ["04"] = "Local Group Members",
["05"] = "Domain & User Profiles",   ["06"] = "Network Settings",
["07"] = "Printers",                 ["08"] = "BitLocker Status",
["8b"] = "Disk & Partition",         ["09"] = "MAC Addresses",
["10"] = "PC Serial Number",         ["11"] = "Installed Software",
["12"] = "Firewall",                 ["13"] = "Optional Features",
["14"] = "Server Roles & Features",  ["15"] = "Power Settings",
["16"] = "WiFi Profiles",            ["17"] = "Restore Points",
["18"] = "Windows Defender",         ["19"] = "Windows Update",
["20"] = "System TEMP Backup",       ["21"] = "Windows License",
["22"] = "Office License",           ["23"] = "Security Baseline",
["24"] = "Group Policy Report",      ["25"] = "Certificates",
["26"] = "Battery Report",           ["27"] = "Environment Variables",
["28"] = "Startup Items",            ["29"] = "Memory Slots & Array",
["30"] = "PnP Devices",              ["31"] = "Hardware Identifiers",
```

新セクション ID を本アプリで「raw でなく構造化」サポートする場合は次の 3 点を同時更新：

1. `EvidenceConstants.KnownSections` に `[id] = new("Display Name")` を追加
2. `EvidenceParserService.DispatchSection` switch に case を追加（必要なら新サブパーサを `IXxxParserService` として追加 + `App.xaml.cs` で DI 登録）
3. `Models/PcEvidence.cs` に格納フィールドを追加（必要なら新 DTO 型を `Models/` に追加）

## 関連ドキュメント

- consumer 側 manifest スキーマ消費契約: [fabriq_evidence_manager__contracts__manifest_schema.md](fabriq_evidence_manager__contracts__manifest_schema.md)
- producer 側 manifest 契約（fabriq kernel）: [fabriq__contracts__evidence_manifest_contract.md](fabriq__contracts__evidence_manifest_contract.md)
- 入力 evidence 構造と Discovery → Parser フロー: [fabriq_evidence_manager__architecture__02_evidence_input.md](fabriq_evidence_manager__architecture__02_evidence_input.md)
- 階層構造と DI: [fabriq_evidence_manager__architecture__01_layers.md](fabriq_evidence_manager__architecture__01_layers.md)
- fabriq kernel 側 evidence_config モジュール: [fabriq__modules__evidence_config.md](fabriq__modules__evidence_config.md)
