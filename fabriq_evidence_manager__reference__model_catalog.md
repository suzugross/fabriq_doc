# Models DTO 索引（開発引継ぎ用）

> **対象**: fabriq_evidence_manager / reference
> **対象バージョン**: 3.8.1（取得元: `E:\fabriq_evidence_manager\FabriqEvidenceManager\FabriqEvidenceManager.csproj` `<Version>` / commit `45eae22`）
> **ドキュメント更新日**: 2026-05-07

`FabriqEvidenceManager/Models/` 配下の **60+ DTO/Record/ObservableObject 型** の用途索引。型ごとに「役割 / 主要フィールド / 生成元 / 利用先 / 関連型」を 1 ブロックで明文化する。

集約ルートは **`PcEvidence`**（1 PC = 1 インスタンス、`ObservableObject` 継承）で、Discovery で生成され Parser で各セクションのフィールドが充填される。すべての他モデルは直接 / 間接に `PcEvidence` から参照される。

---

## 索引（型分類）

| 分類 | 件数 | 代表型 |
|---|---|---|
| 集約ルート | 1 | `PcEvidence` |
| 入力契約: manifest | 5 | `EvidenceManifest`, `EvidenceManifestSummary`, `ManifestSection`, `SectionStatus`, `UnknownSection` |
| 入力期待値（hostlist） | 2 | `HostlistEntry`, `PrinterExpectation` |
| 基本情報 (§01〜§10) | 9 | `SystemInfoData`, `MacAddressEntry`, `NetworkInterfaceInfo`, `PrinterEntry`, `BitLockerEntry`, `BitLockerVolumeStatus`, `SerialSourceEntry`, `SerialNumberDetail`, `DomainStatusData` |
| Identity / Domain (§02〜§05) | 4 | `LocalUserEntry`, `LocalGroupEntry`, `LocalGroupMemberEntry`, `UserProfileEntry` |
| インストール状態 (§11〜§19) | 3 | `InstalledAppEntry`, `OptionalFeatureEntry`, `WindowsUpdateEntry` |
| Storage (§8b) | 2 | `DiskEntry`, `PartitionEntry` |
| Security (§12, §17, §18, §23, §25) | 11 | `FirewallProfileEntry`, `FirewallRuleEntry`, `RestorePointEntry`, `DefenderStatusData`, `SecurityBaselineData`, `TpmInfo`, `VbsInfo`, `LsaProtectionInfo`, `BiosInfo`, `LsaRunAsPplState`, `CertificateEntry` |
| Hardware (§29, §31, §26) | 7 | `HardwareIdentifiersData`, `ComputerSystemInfo`, `ComputerSystemProductInfo`, `BaseBoardInfo`, `SystemEnclosureInfo`, `MemorySlotEntry`, `MemoryArraySummaryEntry`, `BatteryReportData` |
| Inventory (§27, §28, §30) | 3 | `EnvironmentVariableEntry`, `StartupItemEntry`, `PnpDeviceEntry` |
| Power / WiFi / GP / License (§15, §16, §21, §22, §24) | 7 | `PowerSettingsData`, `WindowsLicenseData`, `WindowsLicensingProduct`, `WindowsLicensingService`, `OfficeLicenseData`, `OfficeClickToRunConfig`, `OfficeProductLicense`, `OfficeVnextLicenseEntry`, `OfficeLicenseEvaluation`, `GroupPolicyReport` |
| Verification / Baseline | 13 | `VerificationReport`, `VerificationItem`, `VerificationStatus`, `ChecklistResult`, `ChecklistVerifyItem`, `ChecklistModuleItem`, `ExportHistoryEntry`, `BaselineComparisonReport`, `BaselineComparisonItem`, `BaselineMatchStatus`, `SystemInfoComparisonResult`, `ChecklistComparisonResult`, `InstalledAppsComparisonResult`, `LicenseComparisonResult`, `DomainStatusComparisonResult`, `BaselineCategoryInfo`, `BaselineCategoryOption` |
| Options / Output | 5 | `EvidenceCheckOptions`, `ExcelExportOptions`, `ExcelDeliveryMetadata`, `PcMemo`, `SerialSourceValidity`, `UnknownSectionFileType`, `InstalledAppSource` |

---

## 集約ルート

### `PcEvidence` （`Models/PcEvidence.cs`、`partial ObservableObject`）

**役割**: 1 台の PC のエビデンス全体を保持するルート。Discovery で `PcName / SerialNumber / CollectionDate` 必須 init で生成 → Parser で 30+ フィールドが充填される。

**主要フィールド分類**:

- 識別: `PcName : required` / `ActualComputerName : ObservableProperty`（rename 後の OS 上名）/ `SerialNumber` / `SerialNumberSource` / `SerialReferenceUuid` / `SerialSourceTrail` / `CollectionDate : required`
- パス: `PcRootDirectoryPath` / `PcInformationDirectoryPath` / `AutoCaptureDirectoryPath` / `BitLockerDirectoryPath` / `ChecklistFilePath` / `ExportHistoryFilePath`
- manifest: `Manifest : EvidenceManifest?` / `ManifestError : string?`
- メモ: `Memo : PcMemo?`
- 各セクションのコレクション or 単一値: `MacAddresses` / `BitLockerKeys` / `LocalUsers` / `LocalGroups` / `LocalGroupMembers` / `DomainStatus?` / `FirewallProfiles` / `FirewallRules` / `OptionalFeatures` / `UserProfiles` / `Disks` / `Partitions` / `PowerSettings?` / `WiFiProfiles : List<string>` / `RestorePoints` / `DefenderStatus?` / `WindowsUpdates` / `WindowsLicense?` / `OfficeLicense?` / `SecurityBaseline?` / `Certificates` / `BatteryReport?` / `HardwareIdentifiers?` / `MemorySlots` / `MemoryArraySummaries` / `EnvironmentVariables` / `StartupItems` / `PnpDevices` / `GroupPolicyReport?`
- 未知 section: `UnknownSections : List<UnknownSection>`
- 警告: `HasWarning` / `WarningMessage` / `HasCaution` / `CautionMessage`

**算出プロパティ（DataGrid / Excel 用）**:

- `EthernetMac` / `WifiMac`（`ConnectionName` 部分一致から抽出）
- `HasBitLocker`（`BitLockerKeys.Count > 0`）
- `DetectedItems`（"PC情報 / キャプチャ / BitLocker / チェックリスト / 履歴" の存在組合せ）
- `HasMemo` / `HasPcNameMismatch` / `HasSerialFallbackWarning`
- `IsSerialFromPrimarySource` / `IsWindowsLicensed` / `IsOfficeInstalled`
- `WindowsLicenseStatusDisplay`（"認証済/未認証/(未取得)"）
- `OfficeLicenseEvaluation`（v1.5+ INTERPRETATION 経路と v1.4 旧 OSPP 経路を切り分け）
- `OfficeLicenseStatusDisplay`（"認証済/サインイン待ち/認証失敗/未インストール/(未取得)"）
- `PcNameDisplay`（"計画|実" 併記、不一致時 ⚠）
- `ActualComputerNameDisplay` / `SerialNumberDisplay`（フォールバック警告併記）
- `HasExpiringCertWithin30Days`

**通知**: `NotifyParsedDetailsChanged()` で `MacAddresses / BitLockerKeys` 更新後に算出プロパティの再評価を発火。

---

## 入力契約: manifest

### `EvidenceManifest` （`Models/EvidenceManifest.cs`）

**役割**: `pc_information/manifest.json`（`schemaVersion=1`）のデシリアライズ結果。fabriq kernel 公開契約 [EVIDENCE_MANIFEST.md] に準拠。
**主要フィールド**: `SchemaVersion : int (required)` / `ManifestType : string (required)`（`"fabriq-evidence-manifest"` 固定）/ `EvidenceConfigVersion / FabriqKernelVersion / CollectedAt : DateTimeOffset / ComputerName / HardwareUniqueId / SelectedNewPcName : required` / `WorkerName : string?` / `Sections : IReadOnlyList<ManifestSection>` / `Summary : EvidenceManifestSummary`
**生成**: `ManifestReaderService.Read`
**利用**: `EvidenceParserService.PopulateDetails`（`Sections` から dispatch）

### `EvidenceManifestSummary`

`SectionCount / SuccessCount / SkippedCount / FailedCount / PartialCount`。不変条件: 4 状態の合計 = `SectionCount`。Excel main book の案件メタデータブロックの集計に使用。

### `ManifestSection` （`Models/ManifestSection.cs`）

**役割**: manifest の `sections[]` 1 エントリ。
**フィールド**: `Id / Title / Files : IReadOnlyList<string> / Status : SectionStatus / Reason : string? / ElapsedMs : int (required)`
**算出**: `RegularFiles` （`/` で終わらないもの）/ `Directories` （末尾 `/`）
**正規化**: `Reason` は init 時に空文字列を null に正規化

### `SectionStatus` （enum）

`Success / Skipped / Failed / Partial`。`SectionStatusConverter`（`ManifestReaderService` 内）で未知文字列を `Failed` にフォールバック。

### `UnknownSection` / `UnknownSectionFileType`

**役割**: `EvidenceConstants.KnownSections` に無い section ID の raw データ保持（前方互換）。
**フィールド**: `Id / Title / FilePath / FileType : UnknownSectionFileType / RawContent : required`
**生成**: `EvidenceParserService.CaptureUnknownSection`（switch default + KnownSections 未登録判定）
**利用**: Excel `WriteUnknownSectionSheets`（`§{Id} {Title}` 動的シート）
**FileType**: `Csv / Text` の 2 値

---

## 入力期待値: hostlist

### `HostlistEntry` （`Models/HostlistEntry.cs`）

**役割**: `hostlist.csv` 1 行。`NewPCName` が突合キー。
**識別**: `AdminID / OldPCName / NewPCName`
**ネットワーク**: Ethernet / Wifi の `IP / Subnet / Gateway` 各 3 件 = 6 列、`DNS1〜DNS4`
**その他**: `Pin`（BitLocker PIN、暗号化フィールド可能性あり）/ `Printers : List<PrinterExpectation>`（最大 10 台）
**算出**: `GetDnsList()`（DNS1〜4 の非空 + ソート）

### `PrinterExpectation`

`Name / Driver / Port` の 3 フィールド required。hostlist の `Printer{i}{Name|Driver|Port}` 列を `i=1..10` で展開。

---

## 基本情報

### `SystemInfoData` （§01）

`Hostname / OsName / Version / Cpu / Memory`（5 string）。on-demand parse、SystemInfo Baseline で使用。

### `MacAddressEntry` （§09）

`ConnectionName / AdapterName / PhysicalAddress / Status`（4 string）。`ConnectionName` で Ethernet/Wifi 判定。

### `NetworkInterfaceInfo` （§06）

`InterfaceName / IPv4Address / SubnetMask / DefaultGateway`（4 string）/ `DnsServers : List<string>`（IPv6 除外 + ソート済み）。算出: `IsEthernet / IsWifi`。on-demand parse（hostlist 突合専用）。

### `PrinterEntry` （§07）

`Name / Driver / Port`（3 string）。on-demand parse。

### `BitLockerEntry` （bitlocker/ ディレクトリ、manifest 外）

`DriveLetter / KeyId / RecoveryPassword`（3 string、required）。Recovery Password が取れない行は破棄。

### `BitLockerVolumeStatus` （§08）

`DriveLetter / VolumeLabel / ConversionStatus / EncryptionPercentage / EncryptionMethod / ProtectionStatus`（6 string）。算出: `IsProtected = ProtectionStatus.Equals("On", IgnoreCase)`。on-demand parse。

### `SerialSourceEntry` （§10）

**役割**: `10_SerialNumber.txt` の "All Sources" ブロック 1 行 = 1 取得試行。
**定数**: `PrimaryCanonicalLabel = "Win32_BIOS.SerialNumber"`
**フィールド**: `Label / Value : required` / `Validity : SerialSourceValidity / ValidityReason : string? / MatchesCanonical / IsCanonicalCandidate / IsSelectedCanonical`
**ヘルパ**: `ShortLabel(label)` で UI 用短縮名（`BIOS / CSProduct / Enclosure / BaseBoard / Registry`）

### `SerialSourceValidity` （enum）

`Valid / Invalid / QueryFailed`。`[VALID, MATCH]` / `[INVALID: ...]` / `[QUERY FAILED: ...]` タグ由来。

### `SerialNumberDetail`

§10 全体パース結果。`CanonicalValue / CanonicalSourceLabel : required / Sources : IReadOnlyList<SerialSourceEntry> / ReferenceUuid` + 算出 `IsFromPrimarySource`。

### `DomainStatusData` （§05）

**主要フィールド**: `PartOfDomain : bool / Domain / DomainRole / CurrentUser / AzureAdJoined : bool / DomainJoined : bool / DomainName / TenantName`
**全体保持**: `Sections : Dictionary<string, Dictionary<string, string>>`（dsregcmd 全セクションの raw 保持、PcDetailWindow には主要 9 項目のみ表示）

---

## Identity / Domain

### `LocalUserEntry` （§02）

`Name / Enabled : bool / FullName / Description / SID / LastLogon / PasswordLastSet / PasswordRequired : bool / PrincipalSource`（9 フィールド）。

### `LocalGroupEntry` （§03）

`Name / Description / SID`（3 string）。

### `LocalGroupMemberEntry` （§04）

`GroupName / MemberName / ObjectClass / PrincipalSource`（4 string）。

### `UserProfileEntry` （§05）

`LocalPath / SID / LastUseTime / Loaded : bool`（4 フィールド）。

---

## インストール状態

### `InstalledAppEntry` （§11）

`Name / Version / Publisher / InstallDate`（4 string）+ `Source : InstalledAppSource`（`Desktop / Store`）。on-demand parse。

### `InstalledAppSource` （enum）

`Desktop / Store`。

### `OptionalFeatureEntry` （§13）

`FeatureName / State`（2 string、`State = Enabled / Disabled / DisabledWithPayloadRemoved`）。

### `WindowsUpdateEntry` （§19）

`HotFixID / Description / InstalledBy / InstalledOn`（4 string）。

---

## Storage

### `DiskEntry` （§8b）

`Number : int / FriendlyName / SerialNumber / SizeGB / PartitionStyle / HealthStatus / OperationalStatus`（7 フィールド）。

### `PartitionEntry` （§8b）

`DiskNumber : int / PartitionNumber : int / DriveLetter / SizeGB / Type / IsSystem : bool / IsBoot : bool / IsActive : bool`（8 フィールド）。

---

## Security

### `FirewallProfileEntry` （§12）

`Name / Enabled : bool / DefaultInboundAction / DefaultOutboundAction / LogFileName`（5 フィールド）。

### `FirewallRuleEntry` （§12）

`DisplayName / Enabled : bool / Direction / Action / Profile`（5 フィールド）。

### `RestorePointEntry` （§17）

`SequenceNumber : int / Description / RestorePointType / CreationTime`（4 フィールド）。

### `DefenderStatusData` （§18）

**bool フィールド 8 件**: `AMServiceEnabled / AntispywareEnabled / AntivirusEnabled / RealTimeProtectionEnabled / BehaviorMonitorEnabled / IoavProtectionEnabled / NISEnabled / OnAccessProtectionEnabled`
**string フィールド 6 件**: `AMEngineVersion / AMProductVersion / AntivirusSignatureVersion / AntivirusSignatureLastUpdated / QuickScanEndTime / FullScanEndTime`

### `SecurityBaselineData` （§23）

5 ブロックの集約：`Tpm : TpmInfo? / SecureBootEnabled : bool? / Vbs : VbsInfo? / Lsa : LsaProtectionInfo? / Bios : BiosInfo?`。各 probe 個別退避（取得失敗ブロックは null）。

### `TpmInfo` （§23a）

bool? 6 件（`Present / Ready / Enabled / Activated / Owned / LockedOut`）+ string 4 件（`ManufacturerId / ManufacturerVersion / ManagedAuthLevel / OwnerAuth`）+ int? 2 件（`LockoutCount / LockoutMax`）。

### `VbsInfo` （§23c）

**定数**: `HvciServiceName = "HVCI"` / `CredentialGuardServiceName = "Credential Guard"`
**string + int? ペア 3 セット**: VBS Status / CodeIntegrity / UsermodeCodeIntegrity の各 `Text` + `Code`
**リスト 4 件**: `SecurityServicesRunning : IReadOnlyList<string> / SecurityServicesConfigured / AvailableSecurityProperties : IReadOnlyList<int> / RequiredSecurityProperties`
**算出**: `HvciRunning / CredentialGuardRunning / IsVbsRunning`（VbsStatusCode == 2）

### `LsaProtectionInfo` （§23d）

`RunAsPpl : LsaRunAsPplState / RunAsPplRawText / RunAsPplBoot : int?`。

### `LsaRunAsPplState` （enum）

`Unknown / Off / On / OnWithUefiLock`。

### `BiosInfo` （§23e）

`Manufacturer / SmbiosBiosVersion / ReleaseDate / BiosVersion`（4 string）+ int? 4 件（`SystemBiosMajor/Minor / SmbiosMajor/Minor`）。`SerialNumber` は本クラスに含めない（§10 で別取得）。

### `CertificateEntry` （§25）

`Store / Subject / Issuer / Thumbprint`（4 string）+ `NotBefore / NotAfter : DateTime?` + `HasPrivateKey : bool` + `EnhancedKeyUsageList / FriendlyName / SerialNumber`（3 string）。
**算出**: `DaysToExpiry(referenceDate) : int?`（NotAfter null 時は null、負数は期限切れ）

---

## Hardware

### `HardwareIdentifiersData` （§31）

4 ブロックの集約：`ComputerSystem / ComputerSystemProduct / BaseBoard / SystemEnclosure`、いずれも `?`。

### `ComputerSystemInfo` （§31a）

`Manufacturer / Model / SystemFamily / SystemSKUNumber / TotalPhysicalMem`（5 string）+ `NumberOfProcessors / NumberOfLogicalProcessors : int?`。

### `ComputerSystemProductInfo` （§31b）

`Vendor / Name / Version / IdentifyingNumber / Uuid / SkuNumber`（6 string）。`IdentifyingNumber` は §10 の SN candidate。

### `BaseBoardInfo` （§31c）

`Manufacturer / Product / Version / SerialNumber / Tag`（5 string）。**`SerialNumber` は §10 canonical 候補から除外**（マザーボード SN）。

### `SystemEnclosureInfo` （§31d）

`Manufacturer / Model / ChassisTypes / SerialNumber / SmbiosAssetTag / AssetTag`（6 string）+ `SecurityStatus : int?` + `LockPresent : bool?`。

### `MemorySlotEntry` （§29）

`BankLabel / DeviceLocator / Manufacturer / PartNumber / SerialNumber / FormFactor / SmbiosMemoryType`（7 string）+ int? 6 件（`CapacityGB / SpeedMhz / ConfiguredClockSpeedMhz / ConfiguredVoltageMv / DataWidthBit / TotalWidthBit`）。空スロットは出力されない（装着 DIMM のみ）。

### `MemoryArraySummaryEntry` （§29）

`Tag / Location / Use / MemoryErrorCorrection`（4 string、raw 数値コード保持）+ `MaxCapacityGB : int? / MemoryDevices : int?`。

### `BatteryReportData` （§26）

`Name / Manufacturer / SerialNumber / Chemistry`（4 string）+ `DesignCapacityMwh / FullChargeCapacityMwh / CycleCount : int?`。
**算出**: `HealthPercent : double?`（FullCharge / Design × 100、四捨五入小数 1 位）

---

## Inventory

### `EnvironmentVariableEntry` （§27）

`Scope / Name / Value`（3 string、`Scope = Machine / User`、enum 化せず raw）。

### `StartupItemEntry` （§28）

`Source / Name / User / Command / Location`（5 string）+ `Enabled : bool`。`Source = Win32_StartupCommand / ScheduledTask`。

### `PnpDeviceEntry` （§30）

`Class / FriendlyName / Status / Manufacturer / Service / DriverVersion / DriverDate / InstanceId`（8 string）+ `Present : bool`（true=現在接続中、false=過去キャッシュ）。

---

## Power / WiFi / GP

### `PowerSettingsData` （§15）

`ActivePlanName / RawText`（2 string）。`RawText` は全文保持（PcDetailWindow で TextBox 表示）。

### `GroupPolicyReport` （§24）

`ExecutionStatus : int? / HtmlFileSizeBytes : long? / PartOfDomain : bool? / Domain / ExecutingUser / HtmlFilePath : string?`
**算出**: `IsSuccess = ExecutionStatus == 0`
**HTML 不読込**: `HtmlFilePath` 絶対パス保持のみ、本体は `Process.Start` で OS 既定ブラウザに渡す

---

## Windows / Office License

### `WindowsLicenseData` （§21）

`Products : IReadOnlyList<WindowsLicensingProduct> (required) / Service : WindowsLicensingService? / SlmgrDlvRaw : string`

### `WindowsLicensingProduct`

**定数**: `LicensedStatusCode = 1`
**フィールド**: `Name / Description / LicenseStatusText / PartialProductKey / LicenseFamily / ProductKeyChannel`（6 string）+ `LicenseStatusCode : int / GracePeriodMinutes : int? / KmsConfigured : string? / KmsDiscovered : string?`

### `WindowsLicensingService`

`ClientMachineID / KeyManagementServiceMachine / KeyManagementServicePort / OA3xOriginalProductKey / OA3xOriginalProductKeyDescription / PolicyCacheRefreshRequired`（6 string）。

### `OfficeLicenseData` （§22）

`IsOfficeInstalled : bool` + サブブロック: `ClickToRun : OfficeClickToRunConfig?` / `OsppPath : string?` / `Products : IReadOnlyList<OfficeProductLicense> (required)` / `OsppDstatusRaw : string` / `VnextEntries : IReadOnlyList<OfficeVnextLicenseEntry>` + INTERPRETATION 由来 `IsSubscriptionDetected / OsppShowsGrace : bool? / InterpretationConclusion : string? / ProvisionedCount : int?`。

### `OfficeClickToRunConfig`

`ProductReleaseIds / VersionToReport / Platform / CDNBaseUrl / UpdateChannel / AudienceData / ClientCulture`（7 string）。

### `OfficeProductLicense`

**定数**: `LicensedStatusText = "LICENSED"`
**フィールド**: `ProductId / SkuId / LicenseName / LicenseDescription / LicenseStatusText / ErrorCode / ErrorDescription / RemainingGrace / LastFiveCharsProductKey / KmsMachineName`（10 string）
**算出**: `IsLicensed = LicenseStatusText == "LICENSED"`

### `OfficeVnextLicenseEntry`

**定数**: `ProvisionedStatusText = "Provisioned"`
**フィールド**: `UserProfile / Category / LicenseFile / LicenseType / ProductReleaseId / Status / IsTrial / Beneficiary / LicenseId / Acid / TenantId / UserId / HardwareIdBound / NotBefore / NotAfter / ParseStatus`（16 string、CSV 16 列に対応）
**算出**: `IsProvisioned = Status == "Provisioned"`

### `OfficeLicenseEvaluation` （enum）

`NotEvaluated / NotInstalled / Licensed / SignInPending / AuthFailed`。`PcEvidence.OfficeLicenseEvaluation` プロパティの戻り値型。v1.5+ INTERPRETATION 経路と v1.4 旧 OSPP 経路の判定結果を統一する。

---

## Verification（hostlist 突合）

### `VerificationItem`

`ItemName : required / Expected / Actual / Status : VerificationStatus`（突合 1 項目）。

### `VerificationStatus` （enum）

`Match / Mismatch / NoExpected / NoActual`。

### `VerificationReport`

`PcName : required / HostlistFound : bool / Items : List<VerificationItem>`。
**算出**: `AllMatch / MismatchCount`（Mismatch + NoActual の合計、NoExpected は除外）

---

## Checklist (HTML / CSV)

### `ChecklistResult`

`OverallStatus / VerifyItems : List<ChecklistVerifyItem> / ModuleItems : List<ChecklistModuleItem>`。
**集計算出**: `OkCount / NgCount / SkipCount`（ModuleItems の Status 大文字小文字無視マッチ）

### `ChecklistVerifyItem`

`ItemName / Expected / Actual / Status`（4 string、verify-table 1 行）。

### `ChecklistModuleItem`

`Order / ModuleName / Category / Status / VerifiedStatus / Time / Message`（7 string、kernel 2.x で `VerifiedStatus` 列追加）。

### `ExportHistoryEntry`

`Timestamp / KanriNo / PCName / ModuleName / Category / Status / Message / WindowsUser / Worker / MediaSerial / SessionID`（11 string、CSV 11 列）。
**算出**: `IsSuccess / IsError`（Status 文字列マッチ）

---

## Baseline 突合

### `BaselineComparisonReport`

集約レポート（1 PC 分）。
**フィールド**: `PcName : required / Items : List<BaselineComparisonItem>`（実行サマリ）+ サブカテゴリレポート 6 件（`SystemInfoComparison / ChecklistComparison / DesktopAppsComparison / StoreAppsComparison / LicenseComparison / DomainStatusComparison`、すべて `?`）。v3.8.1 で従来の `InstalledAppsComparison` が `DesktopAppsComparison` / `StoreAppsComparison` の 2 プロパティに分離された（型はともに `InstalledAppsComparisonResult`）。
**算出**: `AllMatch / DifferenceCount`（全カテゴリの不一致集計）/ `AllCategoriesMatch`

### `BaselineComparisonItem`

`ModuleName : required / Category / ExpectedStatus / ActualStatus / MatchStatus : BaselineMatchStatus`。

### `BaselineMatchStatus` （enum）

`Match / Mismatch / MissingInActual / ExtraInActual`。

### サブカテゴリレポート 6 件

すべて `Items : List<VerificationItem>` を持ち、共通プロパティ `AllMatch / MismatchCount` を提供：

- `SystemInfoComparisonResult` — §01 4 項目固定
- `ChecklistComparisonResult` — `BaselineOverallStatus / ActualOverallStatus / VerifyItemComparisons : List<VerificationItem>` + 算出 `OverallStatusMatch`
- `InstalledAppsComparisonResult` — `Items`（差分のみ）+ `TotalBaselineCount / TotalActualCount / MatchCount : int`。v3.8.1 で `BaselineComparisonReport.DesktopAppsComparison` と `StoreAppsComparison` の 2 プロパティから別々に参照される共通型として使われる
- `LicenseComparisonResult` — Win 4 + Office 4 = 最大 8 項目
- `DomainStatusComparisonResult` — 7 項目固定（CurrentUser 除外）

### `BaselineCategoryInfo` / `BaselineCategoryOption`

設定ダイアログ用ペア：

- `BaselineCategoryInfo`（**`record`**、immutable メタデータ）: `CategoryId / DisplayName / IsEnabledByDefault`
- `BaselineCategoryOption`（**`ObservableObject`**、UI バインディング用）: `CategoryId : required / DisplayName : required / IsEnabled : ObservableProperty`

---

## Options / Output

### `EvidenceCheckOptions` （`ObservableObject`）

設定ダイアログ § 「エビデンス取得チェック」のチェックボックス状態を保持する 9 個の `bool ObservableProperty`：

- `CheckEthernetMac / CheckWifiMac / CheckBitLocker / CheckPcInformation / CheckAutoCapture / CheckChecklist`（既定 true）
- `CheckExtraPrinters`（既定 **false**、余剰プリンタ検出）
- `CheckWindowsLicense / CheckOfficeInstalled / CheckOfficeLicense`（既定 true）

`MainWindowViewModel` と `SettingsViewModel` で **同一インスタンスを共有**（`PropertyChanged` リスナで自動再評価）。

### `ExcelExportOptions`

Excel 出力時の追加データ。**キーは PcEvidence インスタンス参照（参照同一性、計画 PC 名衝突対策）**：

- `VerificationReports : IReadOnlyDictionary<PcEvidence, VerificationReport>?`
- `ChecklistResults : IReadOnlyDictionary<PcEvidence, ChecklistResult>?`
- `ExportHistories : IReadOnlyDictionary<PcEvidence, IReadOnlyList<ExportHistoryEntry>>?`
- `BaselineReports : IReadOnlyDictionary<PcEvidence, BaselineComparisonReport>?`
- `DeliveryMetadata : ExcelDeliveryMetadata?`

### `ExcelDeliveryMetadata`

main 台帳の Sheet 1 冒頭 6 行ブロック用：`ProjectName / CustomerName / WorkerName / WorkDate : DateTime?`。`fabriqKernelVersion / evidenceConfigVersion / 台数 / 突合 OK・NG` は manifest と pcEvidences から自動算出するため本型に含めない。

### `PcMemo`

`Text / LastUpdatedAt : DateTimeOffset / LastUpdatedBy`（3 フィールド）。`manager_memo.json` に JSON CamelCase で永続化。

---

## 関連ドキュメント

- 階層構造（Models 全体俯瞰）: [fabriq_evidence_manager__architecture__01_layers.md](fabriq_evidence_manager__architecture__01_layers.md)
- ファイル形式（Models と CSV/TXT 列の対応）: [fabriq_evidence_manager__reference__file_format__pc_information.md](fabriq_evidence_manager__reference__file_format__pc_information.md)
- Excel 出力（Models の値が Excel 列にどう並ぶか）: [fabriq_evidence_manager__reference__excel_layout.md](fabriq_evidence_manager__reference__excel_layout.md)
- セクション ID dispatch 契約（dispatch 表→Model 出力先）: [fabriq_evidence_manager__contracts__section_dispatch.md](fabriq_evidence_manager__contracts__section_dispatch.md)
- consumer 側 manifest 消費契約: [fabriq_evidence_manager__contracts__manifest_schema.md](fabriq_evidence_manager__contracts__manifest_schema.md)
