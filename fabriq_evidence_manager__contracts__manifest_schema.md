# manifest スキーマ消費契約（fabriq_evidence_manager 側）

> **対象**: fabriq_evidence_manager / contracts
> **対象バージョン**: 3.8.1（取得元: `E:\fabriq_evidence_manager\FabriqEvidenceManager\FabriqEvidenceManager.csproj` `<Version>`、最新コミット `45eae22` (2026-05-07)）
> **対応 producer 契約**: fabriq kernel `EVIDENCE_MANIFEST.md schemaVersion=1`（kernel 2.2.2+ / evidence_config 1.3.0+）
> **ドキュメント更新日**: 2026-05-07

## 位置づけ

本ドキュメントは fabriq kernel の公開契約 [fabriq__contracts__evidence_manifest_contract.md](fabriq__contracts__evidence_manifest_contract.md) を **fabriq_evidence_manager（外部 evidence consumer）側がどう消費しているか** を明文化する。producer 側は `evidence_config` モジュールが manifest を出力する規約、consumer 側は本アプリが受け入れて型付きモデルへ写像する規約であり、両者は SemVer 互換として独立に進化する。

implementaion entry point は [E:\fabriq_evidence_manager\FabriqEvidenceManager\Services\ManifestReaderService.cs](file:///E:/fabriq_evidence_manager/FabriqEvidenceManager/Services/ManifestReaderService.cs)、関連型は `Models/EvidenceManifest.cs` と `Models/ManifestSection.cs`、例外は `Services/ManifestExceptions.cs`。

## サポート対象

本アプリは **schemaVersion=1 のみ** をサポートする。producer 側で schemaVersion=2 が将来出力された場合は **silent fallback せず** `UnsupportedManifestSchemaException` で停止する設計（producer 側契約 §「外部ツール（manager 等）の責任」§1 「未知 major 版を検知したら警告 + legacy mode へフォールバック」のうち、本アプリは「警告 + 停止」を採用）。

`manifestType` は固定値 `"fabriq-evidence-manifest"` のみ受理。type discrimination 用の field のため、空文字列・null・他値はすべて拒否する。

## 5 段検証パイプライン

`ManifestReaderService.Read(pcInformationDirectoryPath)` は次の順で検証する。**前段が通らない限り後段は実行しない**。

| 段 | チェック | 実装 | 失敗時の例外 |
|---|---|---|---|
| 1 | `manifest.json` がディレクトリに物理存在するか | `File.Exists(manifestPath)` | `ManifestNotFoundException` |
| 2 | UTF-8 として読めるか | `File.ReadAllText(... Encoding.UTF8)` | `ManifestParseException`（inner = IO/Unauthorized） |
| 3 | JSON として妥当か | `JsonDocument.Parse(json)` で root 要素取得 | `ManifestParseException`（inner = `JsonException`） |
| 4 | `schemaVersion == 1`（Number 型）かつ `manifestType == "fabriq-evidence-manifest"`（String 型） | プリチェック: `JsonElement.TryGetProperty` | schemaVersion 不一致は `UnsupportedManifestSchemaException`、manifestType 不一致は `ManifestParseException` |
| 5 | `EvidenceManifest` 型へのデシリアライズ + `required` フィールド検証 | `JsonSerializer.Deserialize<EvidenceManifest>(json, JsonOptions)` | `ManifestParseException`（inner = `JsonException` / `InvalidOperationException`） |

段 4 は **段 5 のデシリアライズ前にプリチェック** することで、不正 manifest を typed model に流し込んでから「必須フィールド欠損」エラーで落ちる動作を避け、明示的な「未対応 schema」「manifestType 不一致」のメッセージで早期に止める設計。

## 専用例外型

`Services/ManifestExceptions.cs` で 3 種を定義。`Discovery` 側は `try/catch` で 4 種すべて（ManifestNotFound / UnsupportedManifestSchema / ManifestParse / 想定外例外）を捕捉して PC 個別の `PcEvidence.ManifestError` に詰めるため、**1 PC の manifest 不良で fleet 全体の load を中断しない**。

| 例外 | フィールド | 発生条件 |
|---|---|---|
| `ManifestNotFoundException` | `Path : string` | manifest.json 物理不在（kernel 2.2.1 以前 / evidence_config 1.2.0 以前で収集された旧 evidence） |
| `UnsupportedManifestSchemaException` | `DetectedSchemaVersion : int`、定数 `ExpectedSchemaVersion = 1` | `schemaVersion != 1`。検知値が `Number` 型でなければ `-1` |
| `ManifestParseException` | `InnerException` | 段 2 / 段 3 / 段 4（manifestType） / 段 5 のいずれか |

```csharp
// Discovery 側の使用パターン
try
{
    pc.Manifest = _manifestReader.Read(pcInformationDirectoryPath);
    pc.ManifestError = null;
    pc.ActualComputerName = pc.Manifest.ComputerName;
}
catch (ManifestNotFoundException ex)        { pc.ManifestError = $"manifest.json 不在: {ex.Path}"; }
catch (UnsupportedManifestSchemaException ex){ pc.ManifestError = ex.Message; }
catch (ManifestParseException ex)           { pc.ManifestError = $"manifest 解析失敗: {ex.Message}"; }
```

エラーメッセージは UI（`PcEvidence.ManifestError`）に直接表示する想定で、Path / 検知 schemaVersion 等の **トラブルシュート可能な情報を必ず含める**。

## 旧 evidence の扱い（producer 契約との差分）

producer 側契約は §「manifest 不在の旧 evidence」で「**外部ツールはサポートし続けることが期待される**（manifest が無ければファイル列挙ベースで動作する従来挙動を維持）」と宣言している。

しかし fabriq_evidence_manager 3.8.0 は **本機能を採用していない**：

- `Discovery` は manifest 不在 PC をリストには載せる（`PcName` / `SerialNumber` / `CollectionDate` はフォルダ名から取得）が、`PcInformationDirectoryPath` への parse は `pc.Manifest is null` チェックで丸ごとスキップする
- 結果として `LocalUsers / FirewallProfiles / WindowsLicense / SecurityBaseline ...` 全フィールドが空のままの「不完全エントリ」になる
- UI 上は `ManifestError` メッセージで原因が明示されるため、**監査人は「旧 evidence は再収集が必要」と判断できる**

これは producer 契約からの **意図的な逸脱**：旧形式 evidence への暗黙対応をやめて schemaVersion=1 を必須化することで、consumer 側の挙動が「ファイル列挙モード」と「manifest モード」の 2 系統に分岐して fabriq 側の進化が止まるのを避けている。kernel 2.2.2+ の evidence_config を使えば取り扱える、というシンプルな前提条件で運用する。

## SectionStatus の正規化

`Models/ManifestSection.cs` の `SectionStatus` 列挙は producer 契約の 4 値を機械的にマップする：

```csharp
public enum SectionStatus { Success, Skipped, Failed, Partial }
```

`Services/ManifestReaderService.SectionStatusConverter`（`JsonConverter<SectionStatus>`）は **未知文字列値を `SectionStatus.Failed` に正規化** する。producer 契約 §「外部ツール（manager 等）の責任」§3「未知 status enum 値は Failed 扱い」「将来 `"InProgress"` 等が追加されても安全側に倒す」を実装した形。

```csharp
public override SectionStatus Read(...)
{
    var s = reader.GetString();
    return Enum.TryParse<SectionStatus>(s, ignoreCase: false, out var result)
        ? result
        : SectionStatus.Failed;
}
```

`ignoreCase: false` で大文字小文字も厳密にマッチさせる（producer 側が `"success"` 等の non-canonical 表記を出した場合も Failed に倒れる）。

## Reason フィールドの正規化

`Models/ManifestSection.Reason` は JSON 上 `null` と `""`（空文字列）の両方を許容しうるが、本アプリでは **`init` 時に空文字列を `null` に正規化** する：

```csharp
[JsonPropertyName("reason")]
public string? Reason
{
    get => _reason;
    init => _reason = string.IsNullOrEmpty(value) ? null : value;
}
```

これにより consumer コードでは `section.Reason is not null` の単一チェックで「理由文言あり」を判定できる（producer 契約 §「Status セマンティクス」「Success 時 reason は null」を踏まえた防御的正規化）。

## Files 配列の役割分担

`ManifestSection.Files` の各要素は producer 契約 §「ディレクトリ表現」に従い：

- **末尾 `/` 無**: 通常ファイル（manifest 親ディレクトリからの相対パス）
- **末尾 `/`**: ディレクトリ（forensic dump 等の opaque な集合）

consumer 側は便宜上 2 つの算出プロパティで分けて参照する：

```csharp
[JsonIgnore] public IEnumerable<string> RegularFiles => Files.Where(f => !f.EndsWith('/'));
[JsonIgnore] public IEnumerable<string> Directories => Files.Where(f => f.EndsWith('/'));
```

- **`RegularFiles`**: 各セクションの dispatch ロジックがファイル名サフィックス `EndsWith` で対象ファイルを引き当てる際に参照
- **`Directories`**: §20 System TEMP Backup 等の opaque dir。本アプリは **個別パースしない**（producer 契約 §「ディレクトリ表現」「opaque な forensic dump dir として扱い、内部のファイルは個別パースしない」準拠）

## 必須フィールド戦略

`Models/EvidenceManifest` のトップレベルフィールドは producer 契約の `Required: yes` 列に対応して **`required` キーワード付き init-only** で宣言する：

```csharp
public sealed class EvidenceManifest
{
    public required int SchemaVersion { get; init; }
    public required string ManifestType { get; init; }
    public required string EvidenceConfigVersion { get; init; }
    public required string FabriqKernelVersion { get; init; }
    public required DateTimeOffset CollectedAt { get; init; }
    public required string ComputerName { get; init; }
    public required string HardwareUniqueId { get; init; }
    public required string SelectedNewPcName { get; init; }
    public string? WorkerName { get; init; }   // producer Required=no
    public required IReadOnlyList<ManifestSection> Sections { get; init; }
    public required EvidenceManifestSummary Summary { get; init; }
}
```

`required` の不在は段 5 で `JsonException` / `InvalidOperationException` として検出され、`ManifestParseException` でラップされる。producer 側が必須フィールドを欠いた場合は **段 5 で必ず止まる**（typed model に "Empty string default" が紛れ込まない）。

`WorkerName` のみ producer 契約上 `nullable`：profile 実行外（モジュール単発実行）では `$env:FABRIQ_WORKER_NAME` が unset で `null` になりうるため、本アプリは `string?` で受ける。

## Summary 不変条件の活用

`EvidenceManifestSummary` は producer 契約 §「Summary オブジェクト」の不変条件 `successCount + skippedCount + failedCount + partialCount === sectionCount` を **明示的には verify しない**。理由：

- producer 側で実装上の不変条件として保証されており、consumer 側が改めて検証してもエラー時の対処が「警告ログのみ」となる
- 仮に producer の bug で破れていた場合、警告を出すよりは sections 配列を信頼して dispatch を続けたほうが UI が機能する
- 検証が必要な監査機能は将来別 layer（`troubleshooting` カテゴリで提案）として追加する余地を残す

ただし Excel 出力の M6 案件メタデータブロック（`ExcelExportService.WriteDeliveryMetadataBlock`）は manifest summary から「成功 N / スキップ M / 失敗 K」のフリート集計を直接拾って表示するため、summary 値の整合性は表示精度として効いてくる。

## 前方互換戦略

producer 契約 §「前方互換ルール」と consumer 側実装の対応関係：

| producer ルール | consumer 実装 |
|---|---|
| schemaVersion=1 内での **新 section 追加は OK** | `EvidenceConstants.KnownSections` に未登録の ID は `EvidenceParserService` の switch default に流れ、`UnknownSection` に raw 保持 → Excel `WriteUnknownSectionSheets` が動的シート生成 |
| **任意フィールドの追加は OK**（required 不変） | `JsonSerializerOptions.PropertyNameCaseInsensitive = true` + 未知プロパティは無視（`System.Text.Json` のデフォルト挙動）|
| schemaVersion=1 内での **status enum 拡張は禁止**（schemaVersion=2 を伴う） | `SectionStatusConverter` で未知 enum 値を `Failed` に倒す（防御的） |
| schemaVersion=2 への昇格は **破壊的変更を伴う** | `UnsupportedManifestSchemaException` で停止、UI に「未対応 manifest schema」と表示 |

新 section ID 追加への耐性は `EvidenceParserService.DispatchSection` の switch + `CaptureUnknownSection` で確保されており、本アプリのバージョンアップなしで新 evidence を「raw 表示」状態で取り込める。詳細は [fabriq_evidence_manager__contracts__section_dispatch.md](fabriq_evidence_manager__contracts__section_dispatch.md) §「未知セクション ID の捕捉」を参照。

## 関連ドキュメント

- producer 側 manifest 契約（fabriq kernel）: [fabriq__contracts__evidence_manifest_contract.md](fabriq__contracts__evidence_manifest_contract.md)
- §01〜§31 セクション ID 別 dispatch 表: [fabriq_evidence_manager__contracts__section_dispatch.md](fabriq_evidence_manager__contracts__section_dispatch.md)
- 入力 evidence 構造と Discovery → Parser フロー: [fabriq_evidence_manager__architecture__02_evidence_input.md](fabriq_evidence_manager__architecture__02_evidence_input.md)
- 階層構造と DI: [fabriq_evidence_manager__architecture__01_layers.md](fabriq_evidence_manager__architecture__01_layers.md)
