# その他ツール（Looper / Printer Driver Detector / Backup / Update）

> **対象**: fabriq_studio / apps / その他ツール群
> **対象バージョン**: commit `3897c6e`
> **ドキュメント更新日**: 2026-05-06

メイン編集系・Pianist Editor・レジストリ辞書以外の補助ツール 4 種について解説する。

---

## 1. Script Looper（LooperEditor）

| 項目 | 内容 |
|---|---|
| ViewModel | `LooperEditorViewModel` |
| View | `LooperEditorView.xaml` |
| Service | `ILooperService` / `LooperService` |
| Model | `LooperEntry`（[Models/LooperEntry.cs](file:///E:/fabriq_studio/FabriqStudio/Models/LooperEntry.cs)） |
| 出力先 | `<workspace>/modules/extended/<moduleName>/`（ユーザ指定） |
| テンプレート | `<exe>/template/template_fabriq/looper_template/` |

### 機能

リトライ条件付きタスクの繰り返し実行モジュールを生成する。fabriq の `script_looper.ps1` を中身とする独立モジュールとして export できる。

### LooperEntry のスキーマ（looper_list.csv）

| カラム | 型 | 説明 |
|---|---|---|
| `Enabled` | int | 0 = スキップ / 1 = 実行 |
| `ScriptPath` | string | 対象スクリプトのパス（fabriq ルート相対 or 絶対） |
| `MaxRetry` | int | **1 以上**（無限ループ防止）。1 未満は `INotifyDataErrorInfo` で UI 上にエラー表示 |
| `IntervalSec` | int | リトライ間隔（秒、0 以上） |
| `Condition` | string | `"OnError"` または `"Always"` |
| `Description` | string | 管理用説明（コンソール表示にも使用） |
| `Segment` | string | `[Optional]` |

UI 側は `LooperEntry.AllConditions` を ComboBox の選択肢ソースとして使用。

### モジュール export 機能

`ILooperService.ExportModuleAsync(moduleName, entries, overwrite)`:

`<workspace>/modules/extended/<moduleName>/` 直下に以下 4 ファイルを生成（テンプレートからのコピー + データ書き込み）:

```
<moduleName>/
├── script_looper.ps1   ── テンプレートから複製（実行ロジック）
├── module.csv          ── モジュールメタデータ
├── looper_list.csv     ── ユーザが UI で組み立てたループ設定
└── Guide.txt           ── 操作ガイド
```

`overwrite=false` で既存ディレクトリ衝突時は `InvalidOperationException`。

### テスト実行（TestRunAsync）

ループ設定を一時ディレクトリにコピーし、kernel を dot-source した PowerShell 子プロセスで実行する:

- カーネル解決: **Dual Resolution** — workspace の `kernel/` を優先、無ければテンプレート同梱の kernel にフォールバック
- 作業ディレクトリ: ワークスペースルート（`looper_list.csv` 内の相対パスを正しく解決させるため）
- 戻り値: 標準出力 + 標準エラーをマージしたログ文字列

---

## 2. プリンタドライバ検出（PrinterDriverDetector）

| 項目 | 内容 |
|---|---|
| ViewModel | `PrinterDriverDetectorViewModel` |
| View | `PrinterDriverDetectorView.xaml` |
| Service | `IPrinterDriverDetectorService` / `PrinterDriverDetectorService` |
| Model | `PrinterDriverInfo` / `ArchiveExtractResult` / `DriverExportResult` |
| ワークスペース依存 | スキャン時は不要、エクスポート時のみ必要 |

### 機能

任意フォルダ配下の INF ファイルを再帰スキャンし、プリンタドライバ情報を抽出。`.exe` / `.zip` アーカイブの解凍にも対応。検出したドライバを workspace の `printer_driver_list.csv` に転記する。

### サービス API

[Services/IPrinterDriverDetectorService.cs](file:///E:/fabriq_studio/FabriqStudio/Services/IPrinterDriverDetectorService.cs):

| メソッド | 役割 |
|---|---|
| `ScanAsync(scanDir, ct)` | `scanDir` 配下を再帰スキャンし `*.inf` を解析（`Helpers/InfParser`）。ドライバモデル名一覧を返す |
| `ExtractArchivesAsync(scanDir, sevenZipPath, ct)` | `scanDir` 直下の `.exe` / `.zip` を同名フォルダへ展開。冪等。7-Zip があれば `.exe` も解凍可、無ければ `.zip` のみ ZipFile fallback |
| `ExportToWorkspaceAsync(driver, workspaceRootPath, ct)` | 1 件を `modules/standard/printer_driver_config/printer_driver_list.csv` に追記。`DriverName` で重複検知（大文字小文字無視）。CSV が無ければ新規作成 |

### 7-Zip パスの扱い

ユーザーが `sevenZipPath` を指定した場合のみ 7z.exe で `.exe` SFX を展開できる。指定が無ければ:

- 対象が `.zip` → `System.IO.Compression.ZipFile` でフォールバック展開
- 対象が `.exe` → 展開不可として `ArchiveExtractResult.Failed` にカウント

7-Zip 自体は **同梱しない**（ライセンスの分散管理を避けるため）。fabriq 本体に同梱されている 7z.exe を流用するか、ユーザが別途インストールしたものを参照させる。

---

## 3. fabriq バックアップ（FabriqBackup）

| 項目 | 内容 |
|---|---|
| ダイアログ | `FabriqBackupDialog.xaml` |
| Service | `IFabriqBackupService` / `FabriqBackupService` |
| Model | `FabriqBackupRequest` / `FabriqBackupResult` |

### 機能

現在開いているワークスペース全体を、PS1 等を除外した **設定ファイルのみのミラーコピー** として別フォルダに複製する。

### 出力構造

```
<ParentFolder>/fabriq_backup_yyyyMMdd_HHmmss/
├── kernel/
├── modules/
├── profiles/
├── ...（fabriq の通常構成をミラー）
├── USER_MEMO.txt          ── ダイアログでユーザが入力したメモ
└── BACKUP_INFO.txt        ── 自動生成（タイムスタンプ・件数・元ワークスペースパス等）
```

### 除外対象

- PS1（`.ps1`） — ロジックは fabriq 本体に存在するため、設定値だけバックアップする方針
- バージョン管理メタデータ（`.git` / `.svn` 等）
- ビルド出力 / 一時ディレクトリ

---

## 4. fabriq オーバーレイ更新（FabriqUpdate）

| 項目 | 内容 |
|---|---|
| ダイアログ | `FabriqUpdateDialog.xaml` |
| ViewModel | `FabriqUpdateDialogViewModel` |
| Service | `IFabriqUpdateService` / `FabriqUpdateService` |
| Model | `OverlayRules` / `FabriqUpdatePlan` / `BundleUpdateItem` / `FabriqUpdateResult` / `SemVer` |
| 契約 | fabriq KERNEL_API.md §9（更新オーバーレイ契約） |

### 機能

Studio に同梱したテンプレート（`<exe>/template/template_fabriq/fabriq/`）の中身を、現在開いているワークスペースに **SemVer 比較で安全に上書き** する。Studio 自身を更新するのではなく、Studio が抱える「fabriq 本体の最新版テンプレート」をユーザのワークスペースに適用する仕組み。

fabriq 本体の `dev/framework_overlay_rules.json` に書かれた **オーバーレイ契約** に厳密準拠する（schemaVersion / 除外パス / bundle 構造）。

### 更新フロー（4 段階）

#### 段階 1. ルール読み込み — LoadRulesAsync(templateRoot)

`<templateRoot>/dev/framework_overlay_rules.json` を読み、[Models/OverlayRules.cs](file:///E:/fabriq_studio/FabriqStudio/Models/OverlayRules.cs) にデシリアライズ。`schemaVersion` が未対応（現行は 1）の場合は処理を拒否（§9.8）。

ルールに含まれるフィールド:

- `excludeDirsTopLevel` — トップレベルのみ除外するディレクトリ
- `excludeDirsRecursive` — 再帰的に除外するディレクトリ（`.git` 等）
- `excludeFilesKernelLevel` — kernel bundle で除外するファイル
- `moduleCsvWhitelist` — モジュールバンドル内で更新対象とする CSV のホワイトリスト
- `bundles` — kernel bundle / module bundle の定義（VERSION ファイルパス、includePaths 等）

#### 段階 2. Plan 計算 — ComputePlanAsync(templateRoot, targetRoot)

各 bundle ごとに template と target の `VERSION` を読んで `SemVer` 比較し、`UpdateAction` を決定する。

`UpdateAction` 種別（[Models/FabriqUpdatePlan.cs](file:///E:/fabriq_studio/FabriqStudio/Models/FabriqUpdatePlan.cs)）:

| Action | 条件 | 既定選択 |
|---|---|---|
| `Update` | template > target | ✓ 選択 |
| `New` | template にあり target に無い | ✓ 選択 |
| `SkipSame` | 両 version 同一 | (選択可) |
| `SkipTargetNewer` | template < target（target が新しい） | (選択可、警告表示) |
| `SkipNoTemplate` | template に VERSION 無し | 不可 |
| `Preserve` | target にあり template に無いモジュール（site-custom 扱い） | 不可 |

返り値の `BundleUpdateItem` は `IsSelected` のみ可変、その他は init-only。`CanSelect` プロパティで UI 上のチェックボックスを有効化するかを判定する（`Preserve` / `SkipNoTemplate` は手動操作不可）。

#### 段階 3. Preflight — RunPreflight(plan, selected)

更新前の安全チェック（§9.7）:

- **Fabriq.exe 実行中か検出** — 実行中ならブロック（ファイル占有エラーを未然に防ぐ）
- **`resume_state.json` 残存** — 中断中の kitting セッションがあれば警告
- **書込権限** — target ディレクトリへの実書込テスト
- **`REQUIRES_KERNEL` 整合性** — 各モジュールが要求する最小 kernel 版が、更新後の kernel 版以下かチェック

#### 段階 4. 適用 — ApplyAsync(request, progress)

1. workspace 全体を **zip でバックアップ** （ファイル名にタイムスタンプ）
2. `request.DryRun = true` の場合は zip 作成のみで終了
3. 選択された bundle の includePaths を template から target にコピー
4. 進捗は `IProgress<string>` でメッセージ通知（UI のログ表示用）
5. 結果は `FabriqUpdateResult`（処理件数・スキップ件数・エラー一覧）

### UI 構成（FabriqUpdateDialog）

| 領域 | 内容 |
|---|---|
| 上部 | 対象ワークスペース表示、テンプレート版 / 現行版の比較サマリ |
| 中央 | DataGrid（kernel + modules/standard + modules/extended の bundle リスト、Action / 旧版 / 新版 / 選択チェック） |
| 下部 | DryRun スイッチ / 適用ボタン / 進捗ログ |
| グルーピング | `BundleUpdateItem.GroupName`（`"kernel"` / `"modules/standard"` / `"modules/extended"`） |

### 補助モデル

- `Models/SemVer.cs` — `^(\d+)\.(\d+)\.(\d+)$` 専用パーサ。`System.Version`（4 コンポーネント）を意図的に使わない。`TryParseFile(path)` で VERSION ファイル直読み
- `BundleUpdateItem.WarningMessage` — Preview 時に REQUIRES_KERNEL 不整合などを警告表示
