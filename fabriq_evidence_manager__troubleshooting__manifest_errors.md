# manifest 例外対処（Troubleshooting）

> **対象**: fabriq_evidence_manager / troubleshooting
> **対象バージョン**: 3.8.0（取得元: `E:\fabriq_evidence_manager\FabriqEvidenceManager\FabriqEvidenceManager.csproj` `<Version>`）
> **ドキュメント更新日**: 2026-05-07

evidence 取り込み時に PcDetailWindow ヘッダや DataGrid の警告メッセージで「manifest 関連エラー」が表示された場合の症状別対処。`Services/ManifestExceptions.cs` で定義された 3 種の例外がそれぞれどう発生し、何を直せばよいかを整理する。

---

## エラー検知の場所

`Services/NestedEvidenceDiscoveryService.ReadManifest(pc, pcInformationDirectoryPath)` 内で manifest 読み込み時に try/catch：

```csharp
try
{
    pc.Manifest = _manifestReader.Read(pcInformationDirectoryPath);
    pc.ManifestError = null;
    pc.ActualComputerName = pc.Manifest.ComputerName;
}
catch (ManifestNotFoundException ex)
{
    pc.ManifestError = $"manifest.json 不在: {ex.Path}";
}
catch (UnsupportedManifestSchemaException ex)
{
    pc.ManifestError = ex.Message;
}
catch (ManifestParseException ex)
{
    pc.ManifestError = $"manifest 解析失敗: {ex.Message}";
}
```

エラーは **`PcEvidence.ManifestError`** に格納され UI に表示される。**Discovery 自体は中断しない**（1 PC 分のエラーで全体 load を止めない）：

- 本 PC は DataGrid に載るが「manifest 由来のフィールドが空のまま」になる（`SerialNumber / ActualComputerName / 各セクションの構造化結果`）
- 他の PC は正常に処理される

---

## エラー 1: `manifest.json 不在`

### 症状

- DataGrid 行: 通常通り表示（PC 名・SN・収集日はフォルダ名から取得済み）
- ステータス列: 各セクション解析がスキップされ、Warning に `pc_information未検出` が立つ場合あり（実際はディレクトリはあるが manifest.json 自体が無いケース）

### 原因

- **fabriq kernel 2.2.1 以前** で収集された evidence（schemaVersion=1 公開化前）
- **evidence_config 1.2.0 以前** で収集された evidence（manifest.json 自体を出力しない世代）
- fabriq 側 `evidence_config` モジュールが `Failed` で manifest 書き込みまで到達できなかった
- 手動でフォルダを mv/cp する際に manifest.json を入れ忘れた

### 例外 `ManifestNotFoundException`（5 段検証の段 1）

```csharp
public sealed class ManifestNotFoundException : Exception
{
    public string Path { get; }    // 探したファイルパス
    public ManifestNotFoundException(string path)
        : base($"manifest.json が見つかりません: {path}")
}
```

メッセージ例：

```
manifest.json 不在: \\fileserver\evidence\2026_03_12_..._evidence\evidence\pc_information\manifest.json
```

### 対処

| 状況 | 対応 |
|---|---|
| 旧 evidence （kernel 2.2.1 / evidence_config 1.2.0 以前） | **fabriq 側を kernel 2.2.2+ / evidence_config 1.3.0+ に更新して再収集** |
| evidence_config が Failed で停止 | fabriq 側のログを確認し再収集 |
| ファイル mv/cp で取りこぼし | `pc_information/` ディレクトリ全体を再コピー |

producer 契約 §「manifest 不在の旧 evidence」では「外部ツールはサポートし続けることが期待される」と書かれているが、本アプリ 3.8.0 は **暗黙のファイル列挙フォールバックを採用していない**（[contracts__manifest_schema](fabriq_evidence_manager__contracts__manifest_schema.md) §「旧 evidence の扱い」参照）。再収集が現実的な唯一の対処。

---

## エラー 2: `未対応の manifest schemaVersion`

### 症状

- DataGrid 行: 表示される（PC 名・SN・収集日はフォルダ名から）
- 全セクション parse が **スキップ** される（pc.Manifest is null のため `PopulateDetails` の section 走査ループに入らない）
- 各種フィールドが空（`LocalUsers / WindowsLicense / SecurityBaseline ...` すべて）

### 原因

- 将来 fabriq 側が **schemaVersion=2 を公開化**（破壊的変更を伴うため SemVer MAJOR 昇格）→ 本アプリのバージョンアップが必要

### 例外 `UnsupportedManifestSchemaException`（5 段検証の段 4 前段）

```csharp
public sealed class UnsupportedManifestSchemaException : Exception
{
    public const int ExpectedSchemaVersion = 1;
    public int DetectedSchemaVersion { get; }

    public UnsupportedManifestSchemaException(int detectedSchemaVersion)
        : base($"未対応の manifest schemaVersion: {detectedSchemaVersion} " +
               $"(このアプリは schemaVersion={ExpectedSchemaVersion} のみサポート)")
}
```

メッセージ例：

```
未対応の manifest schemaVersion: 2 (このアプリは schemaVersion=1 のみサポート)
```

`DetectedSchemaVersion` プロパティは検知値（manifest 上の値）。検知できなかった（field 不在 / 型違い）場合は `-1`。

### 対処

| 状況 | 対応 |
|---|---|
| 本アプリのバージョンが古い | **fabriq_evidence_manager の最新版にアップデート**（schemaVersion=2 対応版が出ている前提） |
| 本アプリが最新だが未対応版 evidence が混入 | 該当 PC の evidence を schemaVersion=1 互換の収集環境で取り直す（暫定対処） |

本アプリは silent fallback せず明示エラーで止める方針（producer 契約 §「外部ツール（manager 等）の責任」§1 「警告 + 停止」を採用）。silent な部分動作は「型違いフィールドを誤って typed model に流し込んだ結果クラッシュする」より監査の根本破壊が大きいため。

---

## エラー 3: `manifest 解析失敗`

### 症状

- DataGrid 行: 表示される
- 全セクション parse がスキップ
- メッセージに具体的な内訳（JSON 構文エラー / manifestType 不一致 / 必須フィールド欠損のいずれか）

### 原因

3 系統に分かれる。

#### 3a: JSON 構文不正（5 段検証の段 3）

`JsonDocument.Parse(json)` が `JsonException` を投げる。

```
manifest 解析失敗: manifest.json は有効な JSON ではありません: {path}
```

考えられる原因：
- ファイル末尾切れ（書き込み中にプロセス強制終了 / ディスク満杯）
- 文字化け（Shift-JIS 等のエンコーディング不正、本来は BOM 付き UTF-8）
- 手動編集で `}` 等の不整合
- BOM なしファイルが UTF-8 として読まれて先頭バイトが化ける

#### 3b: manifestType 不一致（5 段検証の段 4 後段）

```
manifest 解析失敗: manifestType が不正: 期待値 "fabriq-evidence-manifest" / 実値 "..." ({path})
```

考えられる原因：
- fabriq 系列以外のツールが `manifest.json` を上書きしている
- producer 側で manifestType 定数が変えられている（仕様違反）

#### 3c: 必須フィールド欠損（5 段検証の段 5）

```
manifest 解析失敗: manifest.json の必須フィールドが欠損: {path} - {detail}
```

`required` 修飾子付きフィールド（`SchemaVersion / ManifestType / EvidenceConfigVersion / FabriqKernelVersion / CollectedAt / ComputerName / HardwareUniqueId / SelectedNewPcName / Sections / Summary`）のいずれかが JSON にないと、`System.Text.Json` のデシリアライズ時に `JsonException` または `InvalidOperationException` が出る。

producer 側 `evidence_config` の改修ミス / 手動編集が原因。

### 例外 `ManifestParseException`

```csharp
public sealed class ManifestParseException : Exception
{
    public ManifestParseException(string message) : base(message) { }
    public ManifestParseException(string message, Exception innerException)
        : base(message, innerException) { }
}
```

InnerException に元の例外（`JsonException` / `IOException` / `UnauthorizedAccessException` / `InvalidOperationException`）が入る。

### 対処

| サブ症状 | 対応 |
|---|---|
| 3a JSON 構文 | 該当 PC の `pc_information/manifest.json` を直接エディタで開く → 構文確認 → 再収集 |
| 3b manifestType | producer 側ログ確認、fabriq 系列のツールが書いた manifest かを確認 |
| 3c 必須フィールド | manifest.json をエディタで開いて欠損フィールドを確認、producer 側のバージョン確認（古い `evidence_config` で `WorkerName` 以外の必須が欠ける状況は通常起きないはず） |

#### 文字コード確認

PowerShell で BOM 確認：

```powershell
# BOM 付き UTF-8 なら 0xEF, 0xBB, 0xBF で始まる
Get-Content "{path}\manifest.json" -AsByteStream -TotalCount 4 | ForEach-Object { '{0:X2}' -f $_ }
```

期待値: `EF BB BF 7B`（先頭 3 byte BOM + `{` = 0x7B）。

#### 構文確認（PowerShell の ConvertFrom-Json）

```powershell
Get-Content "{path}\manifest.json" -Raw | ConvertFrom-Json
```

エラーメッセージで構文不正の位置（行・列）が出る。

---

## SectionStatus 未知値の silent 化

manifest の `sections[].status` フィールドが `Success / Skipped / Failed / Partial` 以外の文字列だった場合、**例外は出ず silent に `Failed` 扱い** になる。

`Services/ManifestReaderService.SectionStatusConverter`：

```csharp
public override SectionStatus Read(...)
{
    var s = reader.GetString();
    return Enum.TryParse<SectionStatus>(s, ignoreCase: false, out var result)
        ? result
        : SectionStatus.Failed;       // ← 未知値はすべて Failed
}
```

producer 契約 §「外部ツール（manager 等）の責任」§3「未知 status enum 値は Failed 扱い」「将来 `"InProgress"` 等が追加されても安全側に倒す」を実装。

### 影響

- 本来 `Success` だったセクションが silent に Failed 化されると **その section が dispatch されない**（PopulateDetails の switch 内でスキップ）
- 結果としてそのセクションのデータが PcEvidence に格納されない
- DataGrid / Excel / PcDetailWindow の対応箇所が空のまま

### 検知方法

明示エラーは出ない（silent design）。発見方法：

- `pc.Manifest.Sections.Where(s => s.Status == Failed && s.Reason != null)` の条件で生 manifest の `Reason` を見る
- producer 側 fabriq の改版で新 status 値が追加されたら、本アプリのバージョンアップ + `SectionStatus` enum 拡張が必要

---

## ManifestError の UI 表示

`PcEvidence.ManifestError` （`ObservableProperty`、`string?`）が UI に表示される箇所：

| 場所 | 表示形式 |
|---|---|
| MainWindow DataGrid ステータス列 | （現状版では直接表示しない、Warning メッセージは別系統） |
| PcDetailWindow | 現状版では直接表示しない |
| **将来計画** | エラー専用の表示を増やす余地あり |

v3.8.0 では `ManifestError` 自体が UI に明示的に出る箇所は限定的：実用上は **`SerialNumber` 空 + 各セクションフィールド空 = manifest 不良の症状** でユーザーが推測する形。明示表示の追加は将来要件。

ただし `Debug.WriteLine` レベルではログされる：Visual Studio デバッガで起動して `[Discovery] ... ManifestError = ...` を観察できる。

---

## エラーが出ない異常: manifest はあるがセクションが Failed

`section.Status == Failed` のセクションは parse 対象外（`DispatchSection` の最初で skip）。これは **manifest 例外ではない**：

- producer 側の `evidence_config` モジュールが該当セクションだけ例外で完了不可だった
- `Reason` フィールドに具体的な原因が入っている

### 検知方法

PcDetailWindow / Excel ではセクション欠損として表示される（`HasDomainStatus` 等の Visibility flag で隠れる）。生 manifest を確認して `Reason` を読むのが早い：

```powershell
$m = Get-Content "{path}\manifest.json" -Raw | ConvertFrom-Json
$m.sections | Where-Object { $_.status -eq "Failed" } | Select-Object id, title, reason
```

### 対処

セクションごとに違う：

- §06 NetworkConfig が Failed → ネットワーク adapter 取得に失敗 / 権限不足
- §22 OfficeLicense が Failed → OSPP.vbs 実行失敗 / cscript 不在
- §31 HardwareIdentifiers が Failed → WMI クエリ失敗

producer 側 fabriq の `evidence_config` モジュールに依存するため、本アプリ側では対処不可。fabriq 側ログ確認が必要。

---

## まとめ: 3 例外の鑑別フローチャート

```
manifest 関連エラーが出た
↓
1. manifest.json は物理ファイルとして存在するか?
   いいえ → ManifestNotFoundException
            → fabriq kernel 2.2.2+ で再収集
↓
2. ファイルは UTF-8 + JSON として妥当か?
   いいえ → ManifestParseException (JsonException inner)
            → エディタで開いて構文確認、再収集
↓
3. schemaVersion == 1 か?
   いいえ → UnsupportedManifestSchemaException
            → 本アプリのバージョンアップ
↓
4. manifestType == "fabriq-evidence-manifest" か?
   いいえ → ManifestParseException (mismatch)
            → producer 側を確認
↓
5. required フィールドはすべてある?
   いいえ → ManifestParseException (InvalidOperationException inner)
            → 該当フィールドを producer 側で出力するよう確認、再収集
↓
manifest 自体は OK。各 section の status を確認
```

---

## 関連ドキュメント

- consumer 側 manifest 消費契約（5 段検証の詳細）: [fabriq_evidence_manager__contracts__manifest_schema.md](fabriq_evidence_manager__contracts__manifest_schema.md)
- producer 側 manifest 公開契約: [fabriq__contracts__evidence_manifest_contract.md](fabriq__contracts__evidence_manifest_contract.md)
- 取り込み手順（manifest エラー解釈の表）: [fabriq_evidence_manager__usage__01_import.md](fabriq_evidence_manager__usage__01_import.md)
- 入力 evidence 構造（フォルダ命名規則）: [fabriq_evidence_manager__architecture__02_evidence_input.md](fabriq_evidence_manager__architecture__02_evidence_input.md)
- セクション ID dispatch（status 別の扱い）: [fabriq_evidence_manager__contracts__section_dispatch.md](fabriq_evidence_manager__contracts__section_dispatch.md)
