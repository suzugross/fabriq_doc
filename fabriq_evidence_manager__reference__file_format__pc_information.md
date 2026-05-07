# pc_information ファイル形式リファレンス

> **対象**: fabriq_evidence_manager / reference
> **対象バージョン**: 3.8.0（取得元: `E:\fabriq_evidence_manager\FabriqEvidenceManager\FabriqEvidenceManager.csproj` `<Version>`）
> **対応 producer 契約**: fabriq kernel 3.0.0+ / evidence_config 1.6.0+（schemaVersion=1）
> **ドキュメント更新日**: 2026-05-07

`pc_information/` 配下の §01〜§31 各ファイルが consumer 側でどう解釈されるかの完全リファレンス。**fabriq 側 evidence_config モジュールの出力仕様** と **本アプリのパース実装** の両者を突合させた形で、各ファイルの行構造・列構造・特殊規則を列挙する。

producer 側の契約は [fabriq__contracts__evidence_manifest_contract.md](fabriq__contracts__evidence_manifest_contract.md) と [fabriq__modules__evidence_config.md](fabriq__modules__evidence_config.md)。consumer 側の dispatch は [fabriq_evidence_manager__contracts__section_dispatch.md](fabriq_evidence_manager__contracts__section_dispatch.md)。

---

## 共通規約

| 項目 | 仕様 |
|---|---|
| エンコーディング | **BOM 付き UTF-8**（`File.ReadAllText / ReadAllLines (Encoding.UTF8)`） |
| 改行 | CRLF（PowerShell 既定）。`File.ReadAllLines` で吸収 |
| ファイル名 | ID 接頭辞付き（例: `01_SystemInfo.txt`、`8b` のみ `8b` 接頭辞）。consumer は `EndsWith` で末尾サフィックス一致 |
| CSV 区切り | `,` カンマ。ダブルクォート囲み + `""` エスケープ対応の自前パーサ |
| CSV ヘッダ | **1 行目固定でヘッダ列名**。consumer は header-index lookup で **列順非依存** に読む |
| TXT 区切り | `---- {Block Name} ----` 4 ハイフンによるブロック区切り（多くのテキスト形式で共通） |
| Key:Value | `Key: Value`（コロン + 空白）。`SplitKeyValue` ヘルパで `idx = line.IndexOf(':')` で分離 |
| fallback marker | `(Get-Tpm not available: ...)` のような **括弧で始まる行** = probe 失敗（block null 化） |

---

## §01 System Basic Info — `01_SystemInfo.txt`

**形式**: TXT、Key:Value 縦並び。ヘッダ行・空行・`====` 区切り行はスキップ。

| Key | バインド先 |
|---|---|
| `Hostname` | `SystemInfoData.Hostname` |
| `OS Name` | `OsName` |
| `Version` | `Version` |
| `CPU` | `Cpu` |
| `Memory` | `Memory` |

`hostlist 突合 / SystemInfo ベースライン` の両方で使われる on-demand parse 対象。`PopulateDetails` で常時格納はしない。

---

## §02 Local Users — `02_LocalUsers.csv`

**形式**: CSV、11 列。1 行 = 1 ローカルユーザー。列順固定で実装。

```csv
"Name","Enabled","FullName","Description","SID","LastLogon","PasswordLastSet","PasswordRequired","PasswordExpires","AccountExpires","PrincipalSource"
```

| 列 | 型 | バインド先（`LocalUserEntry`） |
|---|---|---|
| Name | string | `Name` |
| Enabled | bool | `Enabled`（`bool.TryParse`） |
| FullName | string | `FullName` |
| Description | string | `Description` |
| SID | string | `SID` |
| LastLogon | string（日時） | `LastLogon` |
| PasswordLastSet | string（日時） | `PasswordLastSet` |
| PasswordRequired | bool | `PasswordRequired` |
| PasswordExpires | string（未使用） | — |
| AccountExpires | string（未使用） | — |
| PrincipalSource | string | `PrincipalSource` |

---

## §03 Local Groups — `03_LocalGroups.csv`

```csv
"Name","Description","SID"
```

| 列 | バインド先（`LocalGroupEntry`） |
|---|---|
| Name | `Name` |
| Description | `Description` |
| SID | `SID` |

---

## §04 Local Group Members — `04_LocalGroupMembers.csv`

```csv
"GroupName","MemberName","ObjectClass","PrincipalSource"
```

| 列 | バインド先（`LocalGroupMemberEntry`） |
|---|---|
| GroupName | `GroupName` |
| MemberName | `MemberName` |
| ObjectClass | `ObjectClass`（`User` / `Group` 等） |
| PrincipalSource | `PrincipalSource`（`Local` / `ActiveDirectory` 等） |

---

## §05 Domain & User Profiles — 2 ファイル

### `05_DomainStatus.txt`

**形式**: TXT、複数セクション混在の Key:Value。先頭は General セクション、その後 dsregcmd 出力（`+------+ | Section Name | +------+` 形式）。

ヘッダ / 区切り行（`====` / `----`）はスキップ。`+------` 行を見つけたら次行（`| Section Name |`）からセクション名を抽出してから 2 行スキップ。

各セクションは `Dictionary<string, string>` で保持され、`DomainStatusData.Sections` に全部入る。**主要フィールド**は General と dsregcmd の特定セクションから抽出：

| 抽出元セクション / Key | バインド先（`DomainStatusData`） |
|---|---|
| `General.PartOfDomain` | `PartOfDomain` |
| `General.Domain` | `Domain` |
| `General.DomainRole` | `DomainRole` |
| `General.Current User` | `CurrentUser` |
| `Device State.AzureAdJoined` | `AzureAdJoined`（YES/NO） |
| `Device State.DomainJoined` | `DomainJoined` |
| `Device State.DomainName` | `DomainName` |
| `Tenant Details.TenantName` | `TenantName` |

dsregcmd セクション内では **`Key : Value`（コロン両側スペース）** 区切り、ヘッダー側は **`Key:   Value`（コロン直後スペース）** 区切り、両方に対応。

### `05_UserProfiles.csv`

```csv
"LocalPath","SID","LastUseTime","Loaded"
```

| 列 | 型 | バインド先（`UserProfileEntry`） |
|---|---|---|
| LocalPath | string | `LocalPath` |
| SID | string | `SID` |
| LastUseTime | string | `LastUseTime` |
| Loaded | bool | `Loaded` |

---

## §06 Network Settings — `06_NetworkConfig.csv`

```csv
"Interface","IPv4Address","SubnetMask","DefaultGateway","DNSServers"
```

`DNSServers` は **カンマ区切りの DNS リスト**（CSV 内のセル値、ダブルクォートで全体を囲む）。consumer 側は：

- カンマで分割 → `TrimEntries / RemoveEmptyEntries`
- **IPv6（`:` を含む）を除外**（fec0:0:0:ffff::1 等のリンクローカル DNS）
- ソートして `NetworkInterfaceInfo.DnsServers : List<string>` に格納

`InterfaceName` の判定で `IsEthernet` / `IsWifi` を `NetworkInterfaceInfo` の算出プロパティで分岐させる。

on-demand parse 対象（`hostlist 突合` 専用）。

---

## §07 Printers — `07_Printers.csv`

```csv
"Name","DriverName","PortName","Shared","PrinterStatus"
```

| 列 | バインド先（`PrinterEntry`） |
|---|---|
| Name | `Name` |
| DriverName | `Driver` |
| PortName | `Port` |

`Shared / PrinterStatus` 列は consumer では未使用（解析対象外）。on-demand parse。

---

## §08 BitLocker Status — `08_BitLocker.txt`

**形式**: TXT、ボリューム単位ブロック構造。

```
====================
BitLocker Status
====================

Volume C: [OSDrive]
Conversion Status: FullyEncrypted
Encryption Percentage: 100%
Encryption Method: XtsAes256
Protection Status: On

Volume D: [Data]
...
```

ボリューム開始行は正規表現 `^Volume\s+([A-Z]):\s*\[(\w+)\]` でマッチ → `DriveLetter` / `VolumeLabel` 抽出。続く Key:Value 4 行を集約して 1 ブロック完成、次のボリュームヘッダで前ブロックをフラッシュ → `BitLockerVolumeStatus` 生成。

| Key | バインド先 |
|---|---|
| Conversion Status | `ConversionStatus` |
| Encryption Percentage | `EncryptionPercentage` |
| Encryption Method | `EncryptionMethod` |
| Protection Status | `ProtectionStatus` |

`IsProtected = ProtectionStatus.Equals("On", IgnoreCase)` の算出プロパティ。on-demand parse 対象（`hostlist 突合` 専用、C ドライブ判定）。

---

## §8b Disk & Partition — 2 ファイル

### `08b_Disks.csv`

```csv
"Number","FriendlyName","SerialNumber","SizeGB","PartitionStyle","HealthStatus","OperationalStatus"
```

| 列 | 型 | バインド先（`DiskEntry`） |
|---|---|---|
| Number | int | `Number` |
| FriendlyName | string | `FriendlyName` |
| SerialNumber | string | `SerialNumber` |
| SizeGB | string | `SizeGB`（数値だが string で保持、評価は表示用途のみ） |
| PartitionStyle | string | `PartitionStyle`（`MBR` / `GPT`） |
| HealthStatus | string | `HealthStatus` |
| OperationalStatus | string | `OperationalStatus` |

### `08b_Partitions.csv`

```csv
"DiskNumber","PartitionNumber","DriveLetter","SizeGB","Type","IsSystem","IsBoot","IsActive"
```

`DiskNumber / PartitionNumber` は int、`IsSystem / IsBoot / IsActive` は bool。`DriveLetter` は `Trim()` で正規化（"C:" のような末尾コロンの揺れを吸収）。

---

## §09 MAC Addresses — `09_MacAddress.csv`

```csv
"Name","InterfaceDescription","MacAddress","Status"
```

| 列 | バインド先（`MacAddressEntry`） |
|---|---|
| Name | `ConnectionName` |
| InterfaceDescription | `AdapterName` |
| MacAddress | `PhysicalAddress`（**`XX-XX-XX-XX-XX-XX` 正規表現で妥当性チェック**、不一致行は破棄） |
| Status | `Status` |

`EthernetMac` / `WifiMac` の算出は `ConnectionName` の部分一致：

- `イーサネット` / `Ethernet` → Ethernet 扱い
- `Wi-Fi` / `Wireless` / `ワイヤレス` → Wi-Fi 扱い

---

## §10 PC Serial Number — `10_SerialNumber.txt`

**形式**: TXT、4 ブロック構成。state machine でブロックを切り替えながら読む。

```
==== PC Serial Number ====

---- Canonical Serial Number ----
ABC123XYZ
(Source: Win32_BIOS.SerialNumber)

---- All Sources ----
Win32_BIOS.SerialNumber                          : ABC123XYZ                [VALID, MATCH]
Win32_ComputerSystemProduct.IdentifyingNumber    : ABC123XYZ                [VALID, MATCH]
Win32_SystemEnclosure.SerialNumber               : (empty)                  [INVALID: empty]
Win32_BaseBoard.SerialNumber                     : MB-987654                [VALID]
Registry SystemSerialNumber                      : ABC123XYZ                [VALID, MATCH]

---- Reference ID ----
Win32_ComputerSystemProduct.UUID                 : AAAA-BBBB-CCCC-DDDD-EEEE [VALID]

---- Selection Policy ----
(human-readable description of which source was selected)
```

| ブロック | 読み方 | 出力先 |
|---|---|---|
| `Canonical Serial Number` | 値 1 行 + `(Source: ...)` 1 行。`(Unretrievable)` のときは空文字列 | `SerialNumberDetail.CanonicalValue` / `CanonicalSourceLabel` |
| `All Sources` | `Label : Value [Tag]` 形式の正規表現 `^(?<label>.+?)\s+:\s+(?<value>.*?)\s+\[(?<tag>.+)\]\s*$` | `Sources : List<SerialSourceEntry>` |
| `Reference ID` | 同パターンの 1 行（典型的に `Win32_ComputerSystemProduct.UUID`） | `ReferenceUuid` |
| `Selection Policy` | 人間向け、consumer はスキップ | — |

### Validity タグ解釈

| タグ | Validity | Reason | MatchesCanonical |
|---|---|---|---|
| `[VALID, MATCH]` | `Valid` | null | true |
| `[VALID, DIFFERENT]` | `Valid` | null | false |
| `[VALID]`（修飾子なし） | `Valid` | null | false |
| `[INVALID: ...]` | `Invalid` | 詳細文字列 | false |
| `[INVALID]` | `Invalid` | null | false |
| `[QUERY FAILED: ...]` | `QueryFailed` | 詳細文字列 | false |
| `[QUERY FAILED]` | `QueryFailed` | null | false |
| その他（未知） | `Invalid` | tag 全体 | false |

### Canonical Candidate 判定（ハードコード）

| Label | IsCanonicalCandidate |
|---|---|
| `Win32_BIOS.SerialNumber` | true（**Primary canonical**） |
| `Win32_ComputerSystemProduct.IdentifyingNumber` | true（フォールバック候補） |
| `Win32_SystemEnclosure.SerialNumber` | true（フォールバック候補） |
| `Win32_BaseBoard.SerialNumber` | **false**（マザーボード SN、record-only） |
| `Registry SystemSerialNumber` | true |

`IsSelectedCanonical = (Label == CanonicalSourceLabel)`。`HasSerialFallbackWarning` は **採用ソースが Primary 以外のとき true**（OEM SMBIOS 不整合の可能性）。

---

## §11 Installed Software — 2 ファイル

### `11_DesktopApps.csv`

```csv
"Name","Version","Publisher","InstallDate"
```

| 列 | バインド先（`InstalledAppEntry`） |
|---|---|
| Name | `Name` |
| Version | `Version` |
| Publisher | `Publisher` |
| InstallDate | `InstallDate` |
| (固定) | `Source = InstalledAppSource.Desktop` |

### `11_StoreApps.csv`

```csv
"Name","Version","Publisher"
```

`InstallDate` 列は無し（Store アプリは取得不能）。`Source = InstalledAppSource.Store`。

両ファイルは on-demand parse（`InstalledAppsComparator` 専用）。

---

## §12 Firewall — 2 ファイル

### `12_FirewallProfiles.csv`

```csv
"Name","Enabled","DefaultInboundAction","DefaultOutboundAction","LogFileName"
```

3 プロファイル固定（`Domain` / `Private` / `Public`）。`Enabled` は bool、Action は `Block` / `Allow` / `NotConfigured`。

### `12_FirewallRules.csv`

```csv
"DisplayName","Enabled","Direction","Action","Profile"
```

`Direction = Inbound / Outbound`、`Action = Allow / Block`。

---

## §13 Optional Features — `13_OptionalFeatures.csv`

```csv
"FeatureName","State"
```

`State = Enabled / Disabled / DisabledWithPayloadRemoved`。

---

## §14 Server Roles & Features

Client OS では `Skipped` で出力されるため consumer はパース対象外（`DispatchSection` の switch にも case なし、`KnownSections` には登録）。

---

## §15 Power Settings — `15_PowerSettings.txt`

**形式**: TXT、複数セクション。`---- Power Plans ----` セクション内で `[ACTIVE]` を含む行が現れたら、その行から **`[ACTIVE]` の手前を `ActivePlanName` として抽出**：

```
  バランス [ACTIVE]
```

→ `ActivePlanName = "バランス"`

ファイル全体を `RawText` にも保持（PcDetailWindow で `TextBox` raw 表示用）。

---

## §16 WiFi Profiles — `16_WiFiProfiles.txt`

**形式**: TXT、`netsh wlan show profiles` の出力。`プロファイル` または `Profile` を含む key 行から SSID を抽出：

```
すべてのユーザー プロファイル     : MyHomeWiFi
```

→ `["MyHomeWiFi"]`

`<` で始まる値（`<None>` 等）は除外。

---

## §17 Restore Points — `17_RestorePoints.csv`

```csv
"SequenceNumber","Description","RestorePointType","CreationTime"
```

`SequenceNumber` は int、その他 string。

---

## §18 Windows Defender — `18_DefenderStatus.txt`

**形式**: TXT、Key:Value 単純構造（ヘッダ行と `====` のみスキップ）。`Get-MpComputerStatus` の出力。

| Key | 型 | バインド先（`DefenderStatusData`） |
|---|---|---|
| `AMServiceEnabled` | bool | `AMServiceEnabled` |
| `AntispywareEnabled` | bool | `AntispywareEnabled` |
| `AntivirusEnabled` | bool | `AntivirusEnabled` |
| `RealTimeProtectionEnabled` | bool | `RealTimeProtectionEnabled` |
| `BehaviorMonitorEnabled` | bool | `BehaviorMonitorEnabled` |
| `IoavProtectionEnabled` | bool | `IoavProtectionEnabled` |
| `NISEnabled` | bool | `NISEnabled` |
| `OnAccessProtectionEnabled` | bool | `OnAccessProtectionEnabled` |
| `AMEngineVersion` | string | `AMEngineVersion` |
| `AMProductVersion` | string | `AMProductVersion` |
| `AntivirusSignatureVersion` | string | `AntivirusSignatureVersion` |
| `AntivirusSignatureLastUpdated` | string | `AntivirusSignatureLastUpdated` |
| `QuickScanEndTime` | string | `QuickScanEndTime` |
| `FullScanEndTime` | string | `FullScanEndTime` |

---

## §19 Windows Update — `19_WindowsUpdates.csv`

```csv
"HotFixID","Description","InstalledBy","InstalledOn"
```

`Get-HotFix` の出力。

---

## §20 System TEMP Backup — `20_TempBackup/` + `20_TempBackup.txt`

forensic dump。consumer は **opaque dir として個別パースしない**（`KnownSections` に登録のみ、Excel 出力なし）。`files[]` 配列に `"20_TempBackup/"`（末尾 `/`）が入っているのが目印。

---

## §21 Windows License — `21_WindowsLicense.txt`

**形式**: TXT、3 ブロック構成。

```
---- SoftwareLicensingProduct (per-product) ----
Name: Windows(R), Professional edition
Description: Windows(R) Operating System, VOLUME_KMSCLIENT channel
LicenseStatus: Licensed (1)
PartialProductKey: T83GX
LicenseFamily: Professional
ProductKeyChannel: Volume:GVLK
GracePeriodRemaining: 0 minutes
KMS (configured): kms.example.com
KMS (discovered): kms.example.com:1688

(空行で次の Product に切り替え、複数製品を順次列挙)

---- SoftwareLicensingService (machine-wide) ----
ClientMachineID: {AAAAAAAA-BBBB-CCCC-DDDD-EEEEEEEEEEEE}
KeyManagementServiceMachine: kms.example.com
KeyManagementServicePort: 1688
OA3xOriginalProductKey:
OA3xOriginalProductKeyDescription:
PolicyCacheRefreshRequired: 0

---- slmgr /dlv (raw) ----
（slmgr.vbs /dlv の人間向け raw 出力、複数 Product 分が並ぶ）
```

### Block 1: `SoftwareLicensingProduct`

`Name:` 行で **新製品ブロック開始**（前ブロックを flush）。空行も flush トリガ。各 Key:

| Key | バインド先（`WindowsLicensingProduct`） |
|---|---|
| Name | `Name` |
| Description | `Description` |
| LicenseStatus | **`"Notification Mode (5)"` のような `Text (Code)` 形式** → `LicenseStatusText` / `LicenseStatusCode`（正規表現 `^(.+?)\s*\((\d+)\)\s*$`）。Code を見て `LicenseStatusMap` で再マップ |
| PartialProductKey | `PartialProductKey`（5 文字） |
| LicenseFamily | `LicenseFamily` |
| ProductKeyChannel | `ProductKeyChannel` |
| GracePeriodRemaining | **数値 + " minutes"** から先頭数値部分のみ抽出 → `GracePeriodMinutes : int?` |
| KMS (configured) | `KmsConfigured` |
| KMS (discovered) | `KmsDiscovered` |

### LicenseStatusCode マップ（slmgr 由来）

| Code | LicenseStatusText |
|---|---|
| 0 | Unlicensed |
| 1 | **Licensed**（`LicensedStatusCode` 定数）|
| 2 | OOB Grace Period |
| 3 | Out of Tolerance Grace |
| 4 | Non-Genuine Grace |
| 5 | Notification Mode |
| 6 | Extended Grace Period |
| その他 | `Unknown` にマップ |

### Block 2: `SoftwareLicensingService`

Key:Value 6 行。`[WARN]` で始まる Key を含む場合は **`serviceFailed = true`** として `Service = null` で返す。

### Block 3: `slmgr /dlv (raw)`

残り全行を string として `SlmgrDlvRaw` に保持（人間向け raw text）。

---

## §22 Office License — `22_OfficeLicense.txt` + `22_OfficeVnextLicenses.csv`

### `22_OfficeLicense.txt`（v1.5.0+ では 4 ブロック）

```
---- Office Click-to-Run Configuration (registry) ----
ProductReleaseIds: O365ProPlusRetail
VersionToReport: 16.0.17932.20742
Platform: x64
CDNBaseUrl: ...
UpdateChannel: ...
AudienceData: Production::LTSC2024
ClientCulture: ja-jp
DetectedAsSubscription: True            (v1.5.0+ のみ)

---- OSPP.vbs path ----
C:\Program Files\Microsoft Office\Office16\OSPP.VBS

---- cscript OSPP.vbs /dstatus (raw) ----
（OSPP.vbs /dstatus の human-readable 出力、複数 Product ブロックが "---" で区切られる）

---- vNext Per-User License Files ----
（v1.5.0+、人間向けサマリ。consumer は raw を捨てて CSV を真値とする）

---- INTERPRETATION ----                (v1.5.0+ のみ)
OSPP shows Grace/Notify:    True
vNext Provisioned:          2
  CONCLUSION: LICENSED (M365 subscription, 2 Provisioned vNext license(s)).
```

### Block 1: Click-to-Run (registry)

8 Key:Value（v1.4 以前は 7、`DetectedAsSubscription` が新規）→ `OfficeClickToRunConfig`。`ProductReleaseIds` キーが含まれていれば C2R あり、無ければ `ClickToRun = null`。

`DetectedAsSubscription` の bool 値を `IsSubscriptionDetected : bool?` に格納（v1.4 以前は null）。

### Block 2: OSPP.vbs

3 状態 detection：

- ヘッダ `---- OSPP.vbs ----` のみ → not found（`OsppPath = null`）
- ヘッダ `---- OSPP.vbs path ----` + 次行に絶対パス → `OsppPath` 設定
- ヘッダ `---- cscript OSPP.vbs /dstatus (raw) ----` → 残り全行が `dstatusRaw`

`dstatusRaw` 内の Product ブロックは `---` で区切られた `Key: Value` 集合。`ParseOsppProducts` がそれを `OfficeProductLicense` リストに展開（実装は本ドキュメント範囲外、`Models/OfficeLicenseData.cs` 参照）。

### Block 3: vNext per-user

サマリ raw は捨て、**真値は `22_OfficeVnextLicenses.csv` から取る**。

### Block 4: INTERPRETATION

| 行パターン | バインド先 |
|---|---|
| `OSPP shows Grace/Notify: <bool>` | `OsppShowsGrace : bool?` |
| `vNext Provisioned: <int>` | `ProvisionedCount : int?` |
| `CONCLUSION: <text>` | `InterpretationConclusion`（`CONCLUSION:` 接頭辞除去後の本文） |

その他（NOTE / 空行 / 二重ソース項目）はスキップ。

### `22_OfficeVnextLicenses.csv`（v1.5.0+ のみ、任意）

```csv
"UserProfile","Category","LicenseFile","LicenseType","ProductReleaseId","Status","IsTrial","Beneficiary","LicenseId","Acid","TenantId","UserId","HardwareIdBound","NotBefore","NotAfter","ParseStatus"
```

16 列、header-index lookup でロード。CSV 不在時は `VnextEntries = []`（旧 evidence 互換）。`OfficeVnextLicenseEntry.IsProvisioned` は `Status == "Provisioned"`（OrdinalIgnoreCase）の算出プロパティ。

---

## §23 Security Baseline — `23_SecurityBaseline.txt`

**形式**: TXT、5 ブロック（`---- TPM ----` / `---- Secure Boot ----` / `---- Virtualization-Based Security (Win32_DeviceGuard) ----` / `---- LSA Protection (RunAsPPL) ----` / `---- BIOS / Firmware ----`）。

各ブロックは Key:Value 集合。**`(` で始まる行は probe 失敗マーカー**（`(Get-Tpm not available: ...)` 等）→ そのブロックの集約は `null` で返す。

### TPM ブロック（`Get-Tpm`）

| Key | 型 | バインド（`TpmInfo`） |
|---|---|---|
| `TpmPresent` | bool? | `Present` |
| `TpmReady` | bool? | `Ready` |
| `TpmEnabled` | bool? | `Enabled` |
| `TpmActivated` | bool? | `Activated` |
| `TpmOwned` | bool? | `Owned` |
| `ManufacturerId` | string | `ManufacturerId` |
| `ManufacturerVersion` | string（trim） | `ManufacturerVersion` |
| `ManagedAuthLevel` | string | `ManagedAuthLevel` |
| `OwnerAuth` | string | `OwnerAuth` |
| `LockedOut` | bool? | `LockedOut` |
| `LockoutCount` | int? | `LockoutCount` |
| `LockoutMax` | int? | `LockoutMax` |

### Secure Boot ブロック

| Key | 型 | バインド |
|---|---|---|
| `SecureBootEnabled` | bool? | `SecurityBaselineData.SecureBootEnabled`（直接） |

### VBS ブロック（`Win32_DeviceGuard`）

| Key | バインド（`VbsInfo`） |
|---|---|
| `VirtualizationBasedSecurityStatus` | `(text, code)` に分解 → `VbsStatusText` / `VbsStatusCode`（正規表現 `^(.+?)\s*\((-?\d+)\)\s*$`） |
| `SecurityServicesRunning` | カンマ分割 → `IReadOnlyList<string>` |
| `SecurityServicesConfigured` | 同上 |
| `CodeIntegrityPolicyEnforcementStatus` | `(text, code)` 分解 → `CodeIntegrityEnforcementText` / `Code` |
| `UsermodeCodeIntegrityPolicyEnforcementStatus` | 同上 |
| `AvailableSecurityProperties` | カンマ分割 → `IReadOnlyList<int>`（parse 失敗要素はスキップ） |
| `RequiredSecurityProperties` | 同上 |

### LSA ブロック（HKLM レジストリ）

| Key | バインド（`LsaProtectionInfo`） |
|---|---|
| `RunAsPPL` | raw → `RunAsPplRawText`、enum マップ → `RunAsPpl : LsaRunAsPplState` |
| `RunAsPPLBoot` | int? → `RunAsPplBoot` |

`LsaRunAsPplState` マップ：

| raw text | enum |
|---|---|
| 含 `UEFI Lock` | `OnWithUefiLock` |
| `Off ...` 始まり | `Off` |
| `On ...` 始まり | `On` |
| その他 / 空 | `Unknown` |

### BIOS ブロック（`Win32_BIOS`）

| Key | 型 | バインド（`BiosInfo`） |
|---|---|---|
| `Manufacturer` | string | `Manufacturer` |
| `SMBIOSBIOSVersion` | string | `SmbiosBiosVersion` |
| `ReleaseDate` | string | `ReleaseDate` |
| `BIOSVersion` | string | `BiosVersion` |
| `SystemBIOSMajor` / `SystemBIOSMinor` | int? | `SystemBiosMajor` / `SystemBiosMinor` |
| `SMBIOSMajor` / `SMBIOSMinor` | int? | `SmbiosMajor` / `SmbiosMinor` |

---

## §24 Group Policy — `24_GroupPolicySummary.txt` + `24_GroupPolicy.html`

### `24_GroupPolicySummary.txt`

**形式**: Key:Value 単純構造、5〜6 行。ヘッダ行はスキップ：

- `====`
- `Group Policy Report`
- `Running ...`
- `Temp HTML ...`
- `Target HTML ...`
- `(NOTE: ...)` などの括弧始まり行

| Key | 型 | バインド（`GroupPolicyReport`） |
|---|---|---|
| `gpresult exit code` | int? | `ExecutionStatus`（0 = 成功） |
| `HTML file size` | long?（**末尾 `bytes` を除去して数値抽出**）| `HtmlFileSizeBytes` |
| `PartOfDomain` | bool? | `PartOfDomain` |
| `Domain` | string | `Domain` |
| `ExecutingUser` | string | `ExecutingUser` |

### `24_GroupPolicy.html`

**読み込まない**。絶対パスのみ `HtmlFilePath` に保持し、PcDetailWindow の `OpenGroupPolicyHtmlCommand` で OS 既定ブラウザに `Process.Start` で渡す（240KB 規模のメモリ常駐回避）。

`gpresult` 失敗時は HTML 不在で `htmlFilePath = null` 渡し（GroupPolicyParserService 側で受ける）。

---

## §25 Certificates — `25_Certificates.csv`

```csv
"Store","Subject","Issuer","Thumbprint","NotBefore","NotAfter","HasPrivateKey","EnhancedKeyUsageList","FriendlyName","SerialNumber"
```

10 列、4 ストア統合（`LocalMachine\My` / `LocalMachine\Root` / `CurrentUser\My` / `CurrentUser\Root` 等）。

| 列 | 型 | バインド（`CertificateEntry`） |
|---|---|---|
| Store | string | `Store` |
| Subject | string | `Subject` |
| Issuer | string | `Issuer` |
| Thumbprint | string | `Thumbprint` |
| NotBefore | DateTime? | `NotBefore`（書式: `yyyy/MM/dd H:mm:ss` 系、InvariantCulture） |
| NotAfter | DateTime? | `NotAfter` |
| HasPrivateKey | bool | `HasPrivateKey` |
| EnhancedKeyUsageList | string | `EnhancedKeyUsageList` |
| FriendlyName | string | `FriendlyName` |
| SerialNumber | string | `SerialNumber` |

### 日時フォーマット

```
yyyy/MM/dd H:mm:ss
yyyy/M/d H:mm:ss
yyyy/MM/dd HH:mm:ss
```

の順で `TryParseExact`、失敗したら `TryParse(InvariantCulture)` で fallback。空文字列は null。

`DaysToExpiry(today)` 算出プロパティで残日数を算出。Excel 出力で着色判定に使う。

---

## §26 Battery Report — `26_BatteryReport.html`

**形式**: HTML（`powercfg /batteryreport` 出力）。HtmlAgilityPack 等を使わず、正規表現 1 本でパース：

```regex
<span class="label">([^<]+)</span></td><td>([^<]*)
```

label と value のペアを `Dictionary` に格納（同 label 複数出現時は **先頭のみ採用**、Installed batteries が先頭で出る前提）。

| label（大文字キー） | バインド（`BatteryReportData`） |
|---|---|
| `NAME` | `Name` |
| `MANUFACTURER` | `Manufacturer` |
| `SERIAL NUMBER` | `SerialNumber` |
| `CHEMISTRY` | `Chemistry` |
| `DESIGN CAPACITY` | `DesignCapacityMwh : int?`（`"42,068 mWh"` から `42068` 抽出、正規表現 `([\d,]+)`） |
| `FULL CHARGE CAPACITY` | `FullChargeCapacityMwh : int?` |
| `CYCLE COUNT` | `CycleCount : int?`（カンマ除去） |

`HealthPercent` は `FullChargeCapacityMwh / DesignCapacityMwh * 100` の算出プロパティ（Model 側）。

デスクトップ等のバッテリ非搭載機では HTML 自体が存在せず、§26 が `Skipped` になる。dispatch 段階でファイル存在チェックを通過した PC のみここに来る。

---

## §27 Environment Variables — `27_EnvironmentVariables.csv`

```csv
"Scope","Name","Value"
```

3 列、列順非依存（header-index lookup）。`Scope = Machine / User`、Path 系の長文 Value は数百〜数千文字になりうる。Excel 出力では `TruncateForCell` で 32,767 文字制限に対応。

---

## §28 Startup Items — `28_StartupItems.csv`

```csv
"Source","Name","User","Command","Location","Enabled"
```

6 列、列順非依存。

| 列 | バインド（`StartupItemEntry`） |
|---|---|
| Source | `Source`（`Win32_StartupCommand` / `ScheduledTask`） |
| Name | `Name` |
| User | `User` |
| Command | `Command`（引数含む長文 → Excel `TruncateForCell` 適用） |
| Location | `Location` |
| Enabled | `Enabled : bool` |

---

## §29 Memory — 2 ファイル

### `29_MemorySlots.csv`

```csv
"BankLabel","DeviceLocator","Capacity_GB","Speed_MHz","ConfiguredClockSpeed_MHz","ConfiguredVoltage_mV","Manufacturer","PartNumber","SerialNumber","FormFactor","SMBIOSMemoryType","DataWidth_bit","TotalWidth_bit"
```

13 列、`Win32_PhysicalMemory` 1 件 = 1 行。**空スロットは fabriq 側で行を出力しない仕様**（装着 DIMM のみ）。

| 列 | 型 | バインド（`MemorySlotEntry`） |
|---|---|---|
| BankLabel | string | `BankLabel` |
| DeviceLocator | string | `DeviceLocator` |
| Capacity_GB | int? | `CapacityGB` |
| Speed_MHz | int? | `SpeedMhz` |
| ConfiguredClockSpeed_MHz | int? | `ConfiguredClockSpeedMhz` |
| ConfiguredVoltage_mV | int? | `ConfiguredVoltageMv` |
| Manufacturer | string | `Manufacturer` |
| PartNumber | string | `PartNumber` |
| SerialNumber | string | `SerialNumber` |
| FormFactor | string | `FormFactor` |
| SMBIOSMemoryType | string | `SmbiosMemoryType` |
| DataWidth_bit | int? | `DataWidthBit` |
| TotalWidth_bit | int? | `TotalWidthBit` |

### `29b_MemoryArraySummary.csv`

```csv
"Tag","Location","Use","MemoryErrorCorrection","MaxCapacity_GB","MemoryDevices"
```

`Win32_PhysicalMemoryArray` 1 件 = 1 行。シングル CPU 機では 1 件、デュアル CPU サーバでは複数件。

| 列 | 型 | バインド（`MemoryArraySummaryEntry`） |
|---|---|---|
| Tag | string | `Tag` |
| Location | string | `Location` |
| Use | string | `Use` |
| MemoryErrorCorrection | string | `MemoryErrorCorrection` |
| MaxCapacity_GB | int? | `MaxCapacityGB` |
| MemoryDevices | int? | `MemoryDevices`（総スロット数） |

---

## §30 PnP Devices — `30_PnpDevices.csv`

```csv
"Class","FriendlyName","Status","Present","Manufacturer","Service","DriverVersion","DriverDate","InstanceId"
```

9 列、列順非依存。`Win32_PnPEntity` 由来、1 PC あたり 200〜400 件規模で行数が多いため List 容量を初期確保（`new List<PnpDeviceEntry>(lines.Length - 1)`）。

| 列 | 型 | バインド（`PnpDeviceEntry`） |
|---|---|---|
| Class | string | `Class` |
| FriendlyName | string | `FriendlyName` |
| Status | string | `Status`（`OK` / `Error` / `Unknown` 等） |
| Present | bool | `Present`（true = 現在接続中、false = 過去キャッシュ） |
| Manufacturer | string | `Manufacturer` |
| Service | string | `Service` |
| DriverVersion | string | `DriverVersion` |
| DriverDate | string | `DriverDate` |
| InstanceId | string | `InstanceId`（200 文字超もありうる → Excel `TruncateForCell` 適用） |

---

## §31 Hardware Identifiers — `31_HardwareIdentifiers.txt`

**形式**: TXT、4 ブロック（`---- Win32_ComputerSystem ----` / `---- Win32_ComputerSystemProduct ----` / `---- Win32_BaseBoard ----` / `---- Win32_SystemEnclosure ----`）。

§23 と同じ probe 個別退避ポリシー。`(` で始まる行は失敗マーカー、対応ブロックを `null` 化。

ヘッダの**前方一致衝突回避**: `Win32_ComputerSystemProduct` を `Win32_ComputerSystem` より先に判定する。

### Win32_ComputerSystem ブロック

| Key | 型 | バインド（`ComputerSystemInfo`） |
|---|---|---|
| Manufacturer | string | `Manufacturer` |
| Model | string | `Model` |
| SystemFamily | string | `SystemFamily` |
| SystemSKUNumber | string | `SystemSKUNumber` |
| TotalPhysicalMem | string | `TotalPhysicalMem` |
| NumberOfProcessors | int? | `NumberOfProcessors` |
| NumberOfLogicalProcessors | int? | `NumberOfLogicalProcessors` |

### Win32_ComputerSystemProduct ブロック

| Key | バインド（`ComputerSystemProductInfo`） |
|---|---|
| Vendor | `Vendor` |
| Name | `Name` |
| Version | `Version` |
| IdentifyingNumber | `IdentifyingNumber`（SN candidate） |
| UUID | `Uuid`（SMBIOS UUID）|
| SKUNumber | `SkuNumber` |

### Win32_BaseBoard ブロック

マザーボード情報（**SerialNumber は record-only、PC SN ではない**）。

| Key | バインド（`BaseBoardInfo`） |
|---|---|
| Manufacturer | `Manufacturer` |
| Product | `Product` |
| Version | `Version` |
| SerialNumber | `SerialNumber` |
| Tag | `Tag` |

### Win32_SystemEnclosure ブロック

| Key | 型 | バインド（`SystemEnclosureInfo`） |
|---|---|---|
| Manufacturer | string | `Manufacturer` |
| Model | string | `Model` |
| ChassisTypes | string | `ChassisTypes`（数値配列の文字列、例: `"3"` = Desktop） |
| SerialNumber | string | `SerialNumber`（SN candidate） |
| SMBIOSAssetTag | string | `SmbiosAssetTag` |
| AssetTag | string | `AssetTag` |
| SecurityStatus | int? | `SecurityStatus` |
| LockPresent | bool? | `LockPresent` |

---

## bitlocker/ ディレクトリ（manifest 外）

`pc_information/manifest.json` には載らないが、`evidence/bitlocker/{pcName}_{drive}.txt` という形で別出しされる。

**形式**: `manage-bde -protectors -get` の出力 raw text。

| 抽出パターン | 抽出元 | バインド（`BitLockerEntry`） |
|---|---|---|
| ファイル名末尾 `_{drive}` | ファイル名 | `DriveLetter`（取れなければ本文） |
| `Mount Point: X:` 行 | 本文（フォールバック） | `DriveLetter` |
| `Identifier:\s*\{?([0-9A-Fa-f]{8}-...-[0-9A-Fa-f]{12})\}?` | 本文 | `KeyId` |
| `Recovery Password:\s*(\d{6}-\d{6}-...-\d{6})`（6 桁 × 8） | 本文 | `RecoveryPassword` |

**Recovery Password が取れない行は破棄**（実用価値なし、`null` を返す）。

---

## checklist/ ディレクトリ（manifest 外）

`evidence/checklist/checklist_*.html`。**最新 1 ファイル**を採用（lexical 後勝ち）。

`ChecklistParserService` が `<table class="verify-table">` と `<div class="table-wrap">` を Regex で抽出：

| 構造 | 抽出 |
|---|---|
| `class="overall-badge overall-{cls}">{text}</div>` | `OverallStatus` |
| verify-table 内の `<tr><td>4 列</td>` + 末尾 `<span>` | `VerifyItems[]`（ItemName / Expected / Actual / Status） |
| table-wrap 内の `<tr><td>7 列</td>` + 4 番目に Result `<span>`、5 番目に **Verified `<span>`**（kernel 2.x で追加） | `ModuleItems[]`（Order / ModuleName / Category / Status / VerifiedStatus / Time / Message） |

HTML タグ除去は `HtmlTagPattern = <[^>]+>` の汎用置換 + `Trim`。

---

## export_history/ ディレクトリ（manifest 外）

`evidence/export_history/history_export_*.csv`。最新 1 ファイル採用。

```csv
"Timestamp","KanriNo","PCName","ModuleName","Category","Status","Message","WindowsUser","Worker","MediaSerial","SessionID"
```

11 列、列順非依存（header-index lookup）。

| 列 | バインド（`ExportHistoryEntry`） |
|---|---|
| Timestamp | `Timestamp` |
| KanriNo | `KanriNo` |
| PCName | `PCName` |
| ModuleName | `ModuleName` |
| Category | `Category`（`Registry` / `Security` / `System` / `Evidence` 等） |
| Status | `Status`（`Success` / `Error` / `Skipped`） |
| Message | `Message` |
| WindowsUser | `WindowsUser` |
| Worker | `Worker` |
| MediaSerial | `MediaSerial` |
| SessionID | `SessionID` |

`IsSuccess` / `IsError` は算出プロパティ。

---

## 関連ドキュメント

- producer 側 manifest 契約: [fabriq__contracts__evidence_manifest_contract.md](fabriq__contracts__evidence_manifest_contract.md)
- producer 側 evidence_config モジュール: [fabriq__modules__evidence_config.md](fabriq__modules__evidence_config.md)
- consumer 側 manifest 消費契約: [fabriq_evidence_manager__contracts__manifest_schema.md](fabriq_evidence_manager__contracts__manifest_schema.md)
- consumer 側 dispatch 表: [fabriq_evidence_manager__contracts__section_dispatch.md](fabriq_evidence_manager__contracts__section_dispatch.md)
- 入力 evidence 構造（ディレクトリ階層）: [fabriq_evidence_manager__architecture__02_evidence_input.md](fabriq_evidence_manager__architecture__02_evidence_input.md)
- Excel 出力（同じデータが Excel 列にどう並ぶか）: [fabriq_evidence_manager__reference__excel_layout.md](fabriq_evidence_manager__reference__excel_layout.md)
