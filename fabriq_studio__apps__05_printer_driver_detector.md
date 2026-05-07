# プリンタドライバ検出（Printer Driver Detector）

> **対象**: fabriq_studio / apps / Printer Driver Detector
> **対象バージョン**: commit `3897c6e`（取得元: `git -C E:\fabriq_studio rev-parse --short HEAD`）
> **ドキュメント更新日**: 2026-05-07

任意フォルダ配下の **INF ファイルを再帰スキャンしてプリンタドライバ名（モデル名）を抽出**する補助ツール。検出した DriverName は workspace の `printer_driver_list.csv` に転記でき、また `hostlist.csv` のプリンタ列にも転用できる。

`.exe` / `.zip` SFX ドライバアーカイブの **自動展開機能**（Phase 2）を備えており、ベンダ配布の固まりをそのままスキャン先に置けば内部の INF 群まで自動で取り出される。

---

## 概要

| 項目 | 内容 |
|---|---|
| ViewModel | `PrinterDriverDetectorViewModel` |
| View | [Views/PrinterDriverDetectorView.xaml](file:///E:/fabriq_studio/FabriqStudio/Views/PrinterDriverDetectorView.xaml) |
| Service | `IPrinterDriverDetectorService` / `PrinterDriverDetectorService` |
| Model | `PrinterDriverInfo` / `ArchiveExtractResult` / `DriverExportResult` |
| INF パーサ | [Helpers/InfParser.cs](file:///E:/fabriq_studio/FabriqStudio/Helpers/InfParser.cs)（fabriq の `Get-ValidInfFiles` の C# 移植） |
| ワークスペース依存 | スキャン: 不要 / エクスポート: 必須 |

スキャン対象フォルダは workspace 内外を問わず任意（**ワークスペース非依存**）。エクスポート（CSV 追記）のみ workspace パスを呼び出し側から受け取る設計。

---

## ViewModel の責務

[ViewModels/PrinterDriverDetectorViewModel.cs](file:///E:/fabriq_studio/FabriqStudio/ViewModels/PrinterDriverDetectorViewModel.cs):

| 状態 | 型 | 役割 |
|---|---|---|
| `ScanPath` | `string` | スキャン起点ディレクトリ（絶対パス） |
| `AutoExtractArchives` | `bool` | スキャン前に `.exe`/`.zip` を自動展開（既定 `true`） |
| `Drivers` | `ObservableCollection<PrinterDriverInfo>` | 抽出結果一覧 |
| `SelectedDriver` | `PrinterDriverInfo?` | エクスポート / クリップボードコピー対象 |
| `IsScanning` | `bool` | コマンド競合制御 |
| `IsWorkspaceOpen` | `bool` | エクスポートの `CanExecute` 入力 |
| `StatusMessage` / `ErrorMessage` | `string?` | 結果表示用 |

### 既定スキャン先の自動投入

ワークスペースが開いていて、`<workspace>/modules/standard/printer_driver_config/INF/` フォルダが既に存在する場合、`ScanPath` をそのパスで初期化する（`TryFillDefaultScanPath`）。`ScanPath` が既に空でない（手動入力済）場合は **上書きしない**。

```csharp
private const string WorkspaceInfRelPath =
    @"modules\standard\printer_driver_config\INF";
```

### ワークスペース切替への追従

```csharp
workspace.WorkspaceChanged += (_, _) =>
{
    IsWorkspaceOpen = _workspace.IsOpen;
    TryFillDefaultScanPath();
};
```

`IWorkspaceService.WorkspaceChanged` イベントを購読して、ワークスペース変更のたびに既定パスを再投入し、`Export` ボタンの活性状態を切り替える。

---

## INF パーサ（[Helpers/InfParser.cs](file:///E:/fabriq_studio/FabriqStudio/Helpers/InfParser.cs)）

fabriq 本体の `printer_driver_config/printer_driver_install.ps1` 内 `Get-ValidInfFiles` のロジックを **C# に忠実移植** したパーサ。判定ロジックを Studio 側で再実装するのは、fabriq 起動前にドライバの妥当性を検証できるため。

### 有効な INF の判定条件（両方満たすもののみ）

1. **`[Manufacturer]` セクション**内に `NTamd64` の文字列参照がある
2. **`[*.NTamd64]` セクション**（任意の prefix を許容）内に `"ModelName" = ...` 形式の行が **1 つ以上**ある

両方を満たさなければ「INF はあるが対象アーキテクチャで使えない」または「モデル定義が未記載」として無効扱い → 抽出 0 件。

### 採用している正規表現

| 用途 | パターン |
|---|---|
| セクションヘッダ全般 | `^\[.*\]` |
| `[Manufacturer]` | `^\[Manufacturer\]\s*$`（IgnoreCase） |
| `[*.NTamd64]` | `^\[.+\.NTamd64\]\s*$`（IgnoreCase） |
| モデル行 | `^"(.+?)"\s*=` |

`Architecture = "NTamd64"` で固定（64bit Windows 前提）。将来 `NTamd64.10.0`/`NTarm64` 等への拡張が必要な場合はこの定数を切り替える。

### エンコーディング

INF は **UTF-16 LE (BOM 付き)** が一般的だが、UTF-8(BOM)・UTF-16 BE・ASCII も混在する。`File.ReadAllText(infPath)` の **BOM 自動検出**に委ねる設計。

### 重複排除

同一 INF 内で `"ModelName" = ...` が複数回登場しても、`models.Contains(name, StringComparer.Ordinal)` で **大小区別の重複排除**を行う。

### エラーフォールバック

`File.ReadAllText` 失敗時（権限・ロック・破損 INF 等）は `Array.Empty<string>()` を返し、スキャン全体を止めない（**graceful degradation**）。

---

## サービス API（`IPrinterDriverDetectorService`）

### `ScanAsync(scanDir, ct)`

```csharp
Task<IReadOnlyList<PrinterDriverInfo>> ScanAsync(string scanDir, CancellationToken ct = default);
```

- `scanDir` 配下を **再帰スキャン**（`SearchOption.AllDirectories`）。
- 各 `*.inf` を `InfParser.ExtractModelNames` で解析し、抽出された各モデル名を 1 件の `PrinterDriverInfo` に展開。
- **スキャン起点直下のトップフォルダ名**（`ResolveTopFolderName`）を `FolderName` として保存。
  - 例: 起点が `INF/`、対象 INF が `INF/EPSON PX-S505 Series/sub/x.inf` → `FolderName = "EPSON PX-S505 Series"`
  - INF が起点直下にある場合は `FolderName = ""` （スキャン根からの直接ヒット）
- 戻り値は **`DriverName` の昇順ソート**（`StringComparer.OrdinalIgnoreCase`）。
- `scanDir` が存在しない場合 `DirectoryNotFoundException`。

### `ExtractArchivesAsync(scanDir, sevenZipPath, ct)`

```csharp
Task<ArchiveExtractResult> ExtractArchivesAsync(
    string scanDir, string? sevenZipPath, CancellationToken ct = default);
```

- `scanDir` の **直下のみ** （`SearchOption.TopDirectoryOnly`）の `.exe` / `.zip` を対象に同名フォルダへ展開する。
- **冪等**: 同名フォルダが既に存在する場合は `Skipped` にカウントして再展開しない。

#### 展開戦略の分岐

| 条件 | 戦略 |
|---|---|
| `sevenZipPath` 指定あり + 該当 exe 存在 | **7z.exe 経由**: `x "<archive>" "-o<targetDir>" -y -bso0 -bsp0` (引数は fabriq の PS 実装と同じ) |
| 7z 無し + 対象が `.zip` | **`System.IO.Compression.ZipFile.ExtractToDirectory` フォールバック**（`.zip` 限定） |
| 7z 無し + 対象が `.exe` | **展開不可**として `Failed` カウント。メッセージ: `"不可: <name> (7z.exe 未配置のため .exe は展開できません)"` |

### `ExportToWorkspaceAsync(driver, workspaceRootPath, ct)`

```csharp
Task<DriverExportResult> ExportToWorkspaceAsync(
    PrinterDriverInfo driver, string workspaceRootPath, CancellationToken ct = default);
```

- 1 件のドライバを workspace の `modules/standard/printer_driver_config/printer_driver_list.csv` に追記。
- **DriverName 重複検知**（大文字小文字無視）: 既存行と一致する場合は `Skipped = 1` で何もしない。
- CSV が無ければ `Directory.CreateDirectory` してから新規作成。
- BOM 付き UTF-8 で **全件書き戻し**（PowerShell 5.1 `Import-Csv` 互換）。
- 例外捕捉:
  - `UnauthorizedAccessException` → `Error = "アクセスが拒否されました: ..."`
  - `IOException` → `Error = "ファイル操作エラー: ..."`
  - その他 → `Error = "予期しないエラー: ..."`

#### printer_driver_list.csv の内部スキーマ（[DriverListRow](file:///E:/fabriq_studio/FabriqStudio/Services/PrinterDriverDetectorService.cs)）

| 列名 | 役割 | デフォルト値 |
|---|---|---|
| `Enabled` | 有効フラグ | `"1"` |
| `TargetHost` | 対象ホスト名（空 = 全機） | `""` |
| `DriverName` | INF から抽出したモデル名 | 必須 |
| `Description` | 自由記述 | `"Detected from <InfFileName>"` |

`PrinterDriverDetectorService` 内 `private sealed class DriverListRow` で `[Name(...)]` 属性付き定義。

---

## 7-Zip パスの扱い

`PrinterDriverDetectorViewModel.ResolveWorkspaceSevenZipPath()`:

```csharp
private const string WorkspaceSevenZipRelPath =
    @"modules\standard\printer_driver_config\tools\7z.exe";
```

ワークスペース内の `modules/standard/printer_driver_config/tools/7z.exe` が存在する場合のみ自動的に `ExtractArchivesAsync` の `sevenZipPath` 引数として渡す。**Studio 側に 7-Zip を同梱しない**（ライセンスの分散管理を避けるため）。

代替: ユーザが workspace に `tools/7z.exe` を配置するか、`.zip` 形式で配布されたドライバのみ扱う。

---

## XAML 構成（[PrinterDriverDetectorView.xaml](file:///E:/fabriq_studio/FabriqStudio/Views/PrinterDriverDetectorView.xaml)）

```
┌───────────────────────────────────────────────────────────────────┐
│ 🖨 プリンタドライバ検出 (Row 0)                                  │
│ 指定フォルダ配下の INF をスキャンしてモデル名を抽出します        │
├───────────────────────────────────────────────────────────────────┤
│ スキャン先入力 (Row 1, セクション)                                │
│   スキャン先: [_____________________________] [📂選択] [🔄スキャン]│
│   ☑ EXE/ZIP を自動展開 (workspace の tools/7z.exe を使用)         │
├───────────────────────────────────────────────────────────────────┤
│ 結果テーブル (Row 2, セクション)                                  │
│   [ステータス: ...] [スキャン中...] [➕ list.csv に追加] [📋 コピー]│
│   DataGrid: DriverName / FolderName / InfFileName / InfFilePath   │
└───────────────────────────────────────────────────────────────────┘
```

### 主要コマンド

| Command | CanExecute | 動作 |
|---|---|---|
| `BrowseCommand` | `!IsScanning` | `OpenFolderDialog` で `ScanPath` を設定 |
| `ScanCommand` | `!IsScanning && !string.IsNullOrWhiteSpace(ScanPath)` | (オプション) ExtractArchives → Scan |
| `CopyDriverNameCommand` | `SelectedDriver is not null` | `Clipboard.SetText(SelectedDriver.DriverName)` |
| `ExportCommand` | `SelectedDriver is not null && IsWorkspaceOpen && !IsScanning` | `printer_driver_list.csv` 追記 |

### スキャンフロー（`ScanAsync` ViewModel メソッド）

1. `IsScanning = true`、`Drivers.Clear()`、ステータス「スキャン中...」表示
2. `AutoExtractArchives = true` の場合:
   - ステータス「アーカイブを展開中...」
   - `ResolveWorkspaceSevenZipPath()` で 7z.exe パスを解決（無ければ null）
   - `ExtractArchivesAsync(ScanPath, sevenZip)` 呼び出し
   - 結果（Extracted/Skipped/Failed）をステータスに合成: `アーカイブ: 展開 3 件 / スキップ 1 件 / 失敗 0 件。スキャン中...`
3. `ScanAsync(ScanPath)` 呼び出し
4. 結果を `Drivers` にバルク追加
5. ステータス: 検出件数または「有効なプリンタドライバ INF は見つかりませんでした。」
6. `finally` で `IsScanning = false`

### エクスポートフロー（`ExportAsync` ViewModel メソッド）

```csharp
StatusMessage = result switch
{
    { Error: not null } => null,
    { Skipped: > 0    } => $"「{SelectedDriver.DriverName}」は既に printer_driver_list.csv に登録済みです。",
    _                   => $"printer_driver_list.csv に追加しました: {SelectedDriver.DriverName}",
};
if (result.Error is not null)
    ErrorMessage = $"エクスポート失敗: {result.Error}";
```

`DriverExportResult` の `Error` / `Skipped` / `Added` の 3 状態を **switch 式 + パターンマッチ** で UI メッセージにマッピング。

---

## ドライバ転記の典型ワークフロー

ベンダから受領したプリンタドライバ配布物を fabriq に取り込む手順:

1. **配布物を受領**（`epson_printer.zip`、`canon_printer.exe` 等）
2. **配布物を `<workspace>/modules/standard/printer_driver_config/INF/` に直接コピー**
3. Studio で **プリンタドライバ検出** ツールを開く（ScanPath は自動投入される）
4. `☑ EXE/ZIP を自動展開` を有効のまま **🔄 スキャン** を押下
5. 抽出されたドライバ一覧から必要なものを選択
6. **➕ printer_driver_list.csv に追加** で workspace に登録
7. （オプション）DriverName を 📋 コピーして `hostlist.csv` の `Printer1Driver` 等に貼付

この流れで `printer_driver_install.ps1`（fabriq 本体）が `printer_driver_list.csv` を読んで実機にドライバをインストールするときに使うモデル名を、**実 INF と整合した形でメンテナンス**できる。

---

## 関連ドキュメント

| ドキュメント | 関係 |
|---|---|
| [fabriq_studio__reference__models_catalog.md](fabriq_studio__reference__models_catalog.md) | `PrinterDriverInfo` / `ArchiveExtractResult` / `DriverExportResult` |
| [fabriq_studio__reference__services_catalog.md](fabriq_studio__reference__services_catalog.md) | `IPrinterDriverDetectorService` のシグネチャ |
| [fabriq__modules__printer_driver_config.md](fabriq__modules__printer_driver_config.md) | fabriq 本体側で `printer_driver_list.csv` を消費するモジュール |
| [fabriq_studio__reference__hostlist_csv_schema.md](fabriq_studio__reference__hostlist_csv_schema.md) | hostlist.csv のプリンタ 1〜10 列に `DriverName` を転記する |

---

## 変更履歴

- 2026-05-07 初版作成（`apps__04_other_tools.md` の Printer Driver Detector セクションから個別化、INF パーサ・7z 戦略・既定パス自動投入を網羅）
