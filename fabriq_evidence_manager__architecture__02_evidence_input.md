# 入力 evidence 構造と manifest dispatch

> **対象**: fabriq_evidence_manager / architecture
> **対象バージョン**: 3.8.0（取得元: `E:\fabriq_evidence_manager\FabriqEvidenceManager\FabriqEvidenceManager.csproj` `<Version>`）
> **対応 fabriq バージョン**: kernel 2.2.2+（schemaVersion=1）/ evidence_config 1.3.0〜1.6.0
> **ドキュメント更新日**: 2026-05-07

## 入力ディレクトリの全体像

fabriq_evidence_manager は **fabriq 側 `log_uploader` モジュールが配備サーバへ集約したディレクトリツリー** を「evidence ルート」として受け付ける。1 PC = 1 サブフォルダ、サブフォルダ名は固定書式の命名規則を持つ。

```
evidence_root/                                            ← UI で選択するルート
├── 2026_03_12_143025_NEW-PC-01_ABC123XYZ_evidence/       ← log_uploader が作る親フォルダ
│   ├── evidence/                                          ← 中間階層（固定名）
│   │   ├── pc_information/
│   │   │   ├── manifest.json                             ← schemaVersion=1（必須）
│   │   │   ├── 01_SystemInfo.txt
│   │   │   ├── 02_LocalUsers.csv
│   │   │   ├── 03_LocalGroups.csv
│   │   │   ├── ... (manifest.sections[].files[] で公開)
│   │   │   └── 31_HardwareIdentifiers.txt
│   │   ├── auto_capture/
│   │   │   └── *.png / *.jpg                              ── デジタル魚拓
│   │   ├── bitlocker/
│   │   │   └── {pcName}_C.txt                             ── ドライブごとの回復キー
│   │   ├── checklist/
│   │   │   └── checklist_*.html                           ── チェックリスト HTML（最新採用）
│   │   └── export_history/
│   │       └── history_export_*.csv                       ── 実行履歴（最新採用）
│   └── manager_memo.json                                   ← 本アプリが保存するメモ
├── 2026_03_12_143112_NEW-PC-02_DEF456UVW_evidence/
│   └── ...
└── 2026_03_12_143258_OLD-NAMING-FORMAT/                    ← _evidence サフィックス無
                                                              は無視（古い形式は非対応）
```

## 親フォルダ命名規則

`{yyyy_MM_dd_HHmmss}_{PCName}_{SerialNumber}_evidence`

- **タイムスタンプ**: 17 文字固定（`2026_03_12_143025`）
- **PCName**: `_` を含みうる（例: `NEW-PC-01` / `KITTING_PC_03`）
- **SerialNumber**: `_` を含まない前提（BIOS SN / MAC / `UNKNOWN` のいずれか）
- **末尾サフィックス**: `_evidence` 固定（CaseInsensitive 比較）

`Helpers/EvidencePathParser.cs` の `TryParseNewFormat` が「先頭 17 文字のタイムスタンプ + 末尾 `_evidence` をアンカーに、中間部の**最後の `_`** で `{PCName} | {SerialNumber}` を分割」する右端優先パースで、PCName 内のアンダースコアを正しく扱う。

サフィックス `_evidence` を持たないフォルダ（旧形式 / 偶発フォルダ）は `IsNewFormatFolder` 判定で `Discovery` に拾われない。

## Discovery → Parser のデータフロー

```
ユーザーが evidence_root を選択
        │
        ▼
NestedEvidenceDiscoveryService.DiscoverAsync(evidenceRootPath)
        │  ・evidence_root/ 直下を ASCII 順にスキャン
        │  ・親フォルダ名を EvidencePathParser でパース
        │  ・evidence/ サブツリーから category フォルダを直接割当て
        │  ・PC ルート直下の manager_memo.json を PcMemoService で読込
        │  ・pc_information/manifest.json を ManifestReaderService で読込
        │      → 成功: pc.Manifest = ...
        │      → ManifestNotFound / UnsupportedSchema / ParseError:
        │           pc.ManifestError = "<理由>" を立てるだけで Discovery は中断しない
        ▼
List<PcEvidence>（pcName で sort）
        │
        ▼
foreach pc:
   EvidenceParserService.PopulateDetails(pc)
       │  ・pc.Manifest が null なら何もしない（ManifestError は UI 側で表示）
       │  ・manifest.sections[] を順次走査
       │     - status ∈ {Failed, Skipped} → skip
       │     - status ∈ {Success, Partial} → DispatchSection(pc, dir, section)
       │       └─ section.Id によって自前 Parse メソッド or 11 サブパーサのどれかを呼ぶ
       │  ・bitlocker/*.txt は manifest 外。BitLockerDirectoryPath があれば全 .txt を Parse
       │  ・最後に ValidateEvidence(pc, options=null) で初回 Warning を立てる
        ▼
   EvidenceParserService.RevalidateEvidence(pc, CheckOptions)
       │  ・ON にされたチェック項目のみを欠損判定対象とする
       ▼
TargetPCs.Add(pc)
        │
        ▼
MainWindowViewModel.EvaluateCautionsForAllPcs() で Caution（黄色）を別軸で立てる
```

## Manifest 読み込みの厳格 5 段検証

`ManifestReaderService.Read(pcInformationDirectoryPath)` は次の順で検証し、**いずれの段で不適合でも silent fallback せず** 専用例外を投げる。fabriq kernel 公開契約 [EVIDENCE_MANIFEST.md §4.1.1] の「未対応版は警告 + 停止」方針を踏襲。

| 段 | チェック内容 | 失敗時の例外 |
|---|---|---|
| 1 | `manifest.json` がディレクトリに存在するか | `ManifestNotFoundException` |
| 2 | 中身が JSON として妥当か（`JsonDocument.Parse`） | `ManifestParseException` |
| 3 | `schemaVersion == 1`（`Number` 型） | `UnsupportedManifestSchemaException` |
| 4 | `manifestType == "fabriq-evidence-manifest"`（`String` 型） | `ManifestParseException` |
| 5 | `EvidenceManifest` 型へのデシリアライズ + `required` フィールド | `ManifestParseException` |

`SectionStatus` の文字列値（`"Success" / "Skipped" / "Failed" / "Partial"`）以外は **`SectionStatusConverter` で `Failed` 扱いに正規化** する（[EVIDENCE_MANIFEST.md §4.1.3] 「未知 status は Failed として安全側に倒す」準拠）。

`Discovery` 側はこれら 4 種の例外を `try/catch` し、`PcEvidence.ManifestError` に人間可読文言を入れて UI に表示する設計（**1 PC の manifest 不良で全体 load は止めない**）。

## セクション ID → サブパーサ → Model 対応表

`EvidenceParserService.DispatchSection(pc, dir, section)` 内の `switch` がセクション ID から呼び出す処理を決定する。下表の「ファイル拡張子マッチ」は `section.RegularFiles` 内で `EndsWith` 一致するファイル名のサフィックス（fabriq 側がプレフィックス番号とアンダースコアを付けて出力する慣例）。

| ID | KnownSections 表示名 | ファイルサフィックス | 取扱い | 出力先 (PcEvidence) |
|---|---|---|---|---|
| 01 | System Basic Info | `01_SystemInfo.txt` | (オンデマンド: `ParseSystemInfo` / `ParseHostname`) | (Verification 経由でのみ参照) |
| 02 | Local Users | `_LocalUsers.csv` | 自前 `ParseLocalUsers` | `LocalUsers : List<LocalUserEntry>` |
| 03 | Local Groups | `_LocalGroups.csv` | 自前 `ParseLocalGroups` | `LocalGroups` |
| 04 | Local Group Members | `_LocalGroupMembers.csv` | 自前 `ParseLocalGroupMembers` | `LocalGroupMembers` |
| 05 | Domain & User Profiles | `_DomainStatus.txt` + `_UserProfiles.csv` | 自前 2 ファイル | `DomainStatus` + `UserProfiles` |
| 06 | Network Settings | `06_NetworkConfig.csv` | (オンデマンド: `ParseNetworkConfig`) | (Verification 経由でのみ参照) |
| 07 | Printers | `07_Printers.csv` | (オンデマンド: `ParsePrinters`) | (Verification 経由でのみ参照) |
| 08 | BitLocker Status | `08_BitLocker.txt` | (オンデマンド: `ParseBitLockerStatus`) | (Verification 経由でのみ参照) |
| 8b | Disk & Partition | `_Disks.csv` + `_Partitions.csv` | 自前 2 ファイル | `Disks` + `Partitions` |
| 09 | MAC Addresses | `_MacAddress.csv` | 自前 `ParseMacAddresses` | `MacAddresses` |
| 10 | PC Serial Number | `_SerialNumber.txt` | 自前 `ApplySerialNumberDetail` | `SerialNumber / SerialNumberSource / SerialReferenceUuid / SerialSourceTrail` |
| 11 | Installed Software | `11_DesktopApps.csv` + `11_StoreApps.csv` | (オンデマンド: `ParseDesktopApps` / `ParseStoreApps`) | (Baseline 経由でのみ参照) |
| 12 | Firewall | `_FirewallProfiles.csv` + `_FirewallRules.csv` | 自前 2 ファイル | `FirewallProfiles` + `FirewallRules` |
| 13 | Optional Features | `_OptionalFeatures.csv` | 自前 `ParseOptionalFeatures` | `OptionalFeatures` |
| 14 | Server Roles & Features | (Client OS では Skipped) | dispatch 対象外 | — |
| 15 | Power Settings | `_PowerSettings.txt` | 自前 `ParsePowerSettings` | `PowerSettings` |
| 16 | WiFi Profiles | `_WiFiProfiles.txt` | 自前 `ParseWiFiProfiles` | `WiFiProfiles : List<string>` |
| 17 | Restore Points | `_RestorePoints.csv` | 自前 `ParseRestorePoints` | `RestorePoints` |
| 18 | Windows Defender | `_DefenderStatus.txt` | 自前 `ParseDefenderStatus` | `DefenderStatus` |
| 19 | Windows Update | `_WindowsUpdates.csv` | 自前 `ParseWindowsUpdates` | `WindowsUpdates` |
| 20 | System TEMP Backup | (本ツールはパース対象外) | dispatch 対象外 | — |
| 21 | Windows License | `_WindowsLicense.txt` | サブパーサ `IWindowsLicenseParserService` | `WindowsLicense` |
| 22 | Office License | `_OfficeLicense.txt` + `_OfficeVnextLicenses.csv` | サブパーサ `IOfficeLicenseParserService` (txt + 任意 csv) | `OfficeLicense` |
| 23 | Security Baseline | `_SecurityBaseline.txt` | サブパーサ `ISecurityBaselineParserService` | `SecurityBaseline` (5 ブロック個別退避) |
| 24 | Group Policy Report | `_GroupPolicySummary.txt` + `_GroupPolicy.html` | サブパーサ `IGroupPolicyParserService` (HTML はパス保持のみ) | `GroupPolicyReport` |
| 25 | Certificates | `_Certificates.csv` | サブパーサ `ICertificatesParserService` | `Certificates` |
| 26 | Battery Report | `_BatteryReport.html` | サブパーサ `IBatteryReportParserService` | `BatteryReport`（デスクトップ機では null のまま） |
| 27 | Environment Variables | `_EnvironmentVariables.csv` | サブパーサ `IEnvironmentVariablesParserService` | `EnvironmentVariables` (Machine + User 混在) |
| 28 | Startup Items | `_StartupItems.csv` | サブパーサ `IStartupItemsParserService` | `StartupItems` (Registry + Task 混在) |
| 29 | Memory Slots & Array | `_MemorySlots.csv` + `_MemoryArraySummary.csv` | サブパーサ `IMemoryInventoryParserService` (2 メソッド) | `MemorySlots` + `MemoryArraySummaries` |
| 30 | PnP Devices | `_PnpDevices.csv` | サブパーサ `IPnpDevicesParserService` | `PnpDevices`（200〜400 件規模） |
| 31 | Hardware Identifiers | `_HardwareIdentifiers.txt` | サブパーサ `IHardwareIdentifiersParserService` | `HardwareIdentifiers` (4 ブロック個別退避) |

「オンデマンド」と書かれた §01 / §06 / §07 / §08 / §11 は **`PopulateDetails` 内で `PcEvidence` フィールドに常時格納はしない**。`HostlistService` / `BaselineService` / `EvidenceVerificationService` が必要時に `IEvidenceParserService` の対応 `Parse...` メソッドを `pc.PcInformationDirectoryPath` 経由で呼び直す設計で、メモリ消費を抑えている。

`§14 Server Roles & Features` と `§20 System TEMP Backup` は dispatch 対象外（`switch` の `default` でも `KnownSections.ContainsKey` が true のため `UnknownSection` にも入らない）。

## 未知セクションの前方互換捕捉

`switch` の `default` 節で **`EvidenceConstants.KnownSections.ContainsKey(section.Id)` が false** のケース（fabriq 側で将来 §32+ 等が追加されたケース）を `CaptureUnknownSection` が次のように処理する：

- `section.RegularFiles` の各ファイルを `pc_information/` から読み込み
- 拡張子で `UnknownSectionFileType.Csv | Text` を分類
- `pc.UnknownSections.Add(new UnknownSection { Id, Title, FilePath, FileType, RawContent })`

その後 `ExcelExportService.WriteUnknownSectionSheets` が **未知 ID ごとに専用シート（`§<id> {Title}` 命名）を動的生成** し、CSV 列頭は実データから推定、Text は raw 出力する。これにより本アプリのバージョンが古い状態で新 evidence を取り込んでもクラッシュせず、監査人がシート上で中身を確認できる（[EVIDENCE_MANIFEST.md §4.1.2] 「未知 section ID は raw 表示してクラッシュさせない」準拠）。

## ファイル I/O の規約

- **エンコーディング**: 全テキスト/CSV は **BOM 付き UTF-8** として `File.ReadAllLines / ReadAllText (Encoding.UTF8)` で読む（fabriq 側の `evidence_config` 出力規約と一致）
- **CSV パース**: ダブルクォート囲みの `"" → "` エスケープ対応の自前 `ParseCsvLine` を使用（外部依存なし、`EvidenceParserService` / `HostlistService` / `ChecklistParserService` で同一実装）
- **CSV 列順非依存**: §27 / §28 / §30（行数が多い・将来列追加の可能性が高いセクション）は **header-index lookup** で読み、列順が変わっても影響しない設計
- **HTML パース**: `ChecklistParserService` は外部 HTML パーサを使わず、`<table class="verify-table">` と `<div class="table-wrap">` を `GeneratedRegex` で抽出
- **HTML 本体非保持**: §24 GroupPolicy の `gpresult /h` HTML（240KB 規模）は **読まずに絶対パスのみ Model に保持** し、`PcDetailViewModel.OpenGroupPolicyHtmlCommand` で OS 既定ブラウザに `Process.Start` で渡す。Excel 出力もパスのみ
- **エラー耐性**: 個別ファイルの `IOException / UnauthorizedAccessException` は `Debug.WriteLine` でログするだけで処理継続（1 ファイル不良で PC 全体を落とさない）

## SerialNumber の特殊扱い（§10）

`10_SerialNumber.txt` は単純な Key:Value 形式ではなく、4 ブロック構成（`Canonical Serial Number / All Sources / Reference ID / Selection Policy`）を持つ。`EvidenceParserService.ParseSerialNumberDetailed` は state machine でブロックを切り替えて読む。

採用ロジック：

- **Primary canonical** = `Win32_BIOS.SerialNumber`（唯一の正式ソース）
- それ以外（`Win32_ComputerSystemProduct.IdentifyingNumber` / `Win32_SystemEnclosure.SerialNumber` / `Registry SystemSerialNumber`）は **canonical 候補** ではあるが、採用された場合は `HasSerialFallbackWarning=true` でフォールバック取得として黄色強調（OEM SMBIOS 不整合の可能性を示唆）
- `Win32_BaseBoard.SerialNumber` は record-only（マザーボード SN であって PC SN ではない）。採用候補ではなく audit 用記録のみ

`PcEvidence.SerialNumberDisplay` で UI/Excel 用に "値 ⚠ (CSProduct)" 形式の併記を生成、`SerialSourceTrail` に全試行履歴を `SerialSourceEntry` として残す。

## Office ライセンス評価の二重ソース

`§22 OfficeLicense` は fabriq 側 `evidence_config` v1.5.0 で INTERPRETATION ブロック（`OfficeLicenseData.InterpretationConclusion`）が追加されている。本ツールは **両世代の evidence を扱う** ため、`PcEvidence.OfficeLicenseEvaluation` で次のように切り分ける：

- **v1.5.0+（INTERPRETATION あり）**: `manifest.sections[id="22"].status` を権威ソースとし、`Success/Partial/Failed → Licensed/SignInPending/AuthFailed` にマップ。fabriq 側で C2R + OSPP + vNext を突合済みなので C# 側はそれを信頼
- **v1.4 以前（INTERPRETATION なし）**: manifest §22 status が単に「セクション収集完了」の意味しかなかった世代の evidence。ここでは OSPP `IsLicensed` と vNext `IsProvisioned` を C# 側で個別判定して `Licensed / AuthFailed` を決める

判定結果は `OfficeLicenseEvaluation` 列挙（`NotEvaluated / NotInstalled / Licensed / SignInPending / AuthFailed`）として DataGrid 列・Caution 判定・Excel 着色で共用される。

## bitlocker/ ディレクトリの扱い（manifest 外）

回復キーは fabriq 側の運用上、`pc_information/manifest.json` ではなく **`bitlocker/{pcName}_{drive}.txt` という separate なディレクトリに別出し** されている（manifest §08 BitLocker は status 情報のみ）。`EvidenceParserService.PopulateDetails` は manifest 走査が終わった後に `pc.BitLockerDirectoryPath` 配下の全 `.txt` を `ParseBitLockerKey` で順次パースし、`pc.BitLockerKeys` に詰め込む：

- **ドライブレター**: ファイル名末尾 `_{drive}` から取得（取れなければ本文中の `Mount Point: X:` から）
- **Identifier (GUID)**: 本文中の `Identifier: {xxxxxxxx-...}` パターン
- **Recovery Password**: 本文中の `Recovery Password: 123456-123456-...`（6 桁 × 8 のハイフン区切り）
- 回復パスワードが取れない場合はエントリを生成しない（実用価値なしと判定）

## 自動リロード（差分マージ）

`MainWindowViewModel` は 30 秒間隔の `DispatcherTimer` で `RefreshEvidenceAsync` を駆動する。**通常の `LoadEvidenceCommand` のように `TargetPCs.Clear()` してから再構築する経路ではなく**、次のキーで既存リストとマージする差分更新方式：

```
key = $"{PcName}\t{SerialNumber}\t{CollectionDate}"
```

- **既存 key**: そのインスタンスのプロパティを書き換える（`ObservableObject` が UI に通知）
- **新規 key**: `TargetPCs.Add()`

これにより `SelectedPc` / `SearchText` / DataGrid のスクロール位置が維持され、現場で「監査作業中に追加 PC を検出」しても操作が中断されない。

## 関連ドキュメント

- 階層構造と DI: [fabriq_evidence_manager__architecture__01_layers.md](fabriq_evidence_manager__architecture__01_layers.md)
- プロジェクト概要: [fabriq_evidence_manager__overview__readme.md](fabriq_evidence_manager__overview__readme.md)
- fabriq kernel 側 manifest 公開契約: [fabriq__contracts__evidence_manifest_contract.md](fabriq__contracts__evidence_manifest_contract.md)
- fabriq kernel 側 evidence_config モジュール: [fabriq__modules__evidence_config.md](fabriq__modules__evidence_config.md)
