# Services 索引（16 件）

> **対象**: fabriq_studio / reference
> **対象バージョン**: commit `3897c6e`（取得元: `git -C E:\fabriq_studio rev-parse --short HEAD`、2026-05-06）
> **ドキュメント更新日**: 2026-05-07

`FabriqStudio/Services/` 配下の **16 のインターフェース** + **16 の実装**（合計 32 ファイル）の責務索引。アーキテクチャ概要は [fabriq_studio__architecture__01_layers.md](fabriq_studio__architecture__01_layers.md)、本ドキュメントは **各 service の責務 + 主要メソッド + 永続化先** を表形式で網羅する。

---

## DI 登録方針

`App.OnStartup` で `Microsoft.Extensions.DependencyInjection` の `ServiceCollection` に **すべて Singleton で登録**：

```csharp
services.AddSingleton<IWorkspaceService, WorkspaceService>();
services.AddSingleton<ICsvService, CsvService>();
services.AddSingleton<IFileService, FileService>();
// ... 16 種すべて
```

15 サービスが DI 登録されており、`IAppSettingsService` のみ将来拡張用の placeholder（現状実装は最低限のスタブ）。詳細は [fabriq_studio__architecture__01_layers.md](fabriq_studio__architecture__01_layers.md) §「サービス層」。

ViewModel は constructor 注入で必要な service を受ける。テスト容易性のため **interface に依存し実装には依存しない** 規約。

---

## サービス全体俯瞰

| # | Interface | 実装 | 責務カテゴリ | ワークスペース依存 |
|---|---|---|---|---|
| 1 | `IWorkspaceService` | `WorkspaceService` | 中核（ルートディレクトリ管理 + 永続化） | (自身が決める) |
| 2 | `ICsvService` | `CsvService` | 汎用 CSV I/O（ワークスペース相対）| ○ |
| 3 | `IFileService` | `FileService` | 汎用ファイル I/O（絶対パス）| × |
| 4 | `IProfileService` | `ProfileService` | profiles/*.csv の編集 | ○ |
| 5 | `IModuleService` | `ModuleService` | modules/*/module.csv の検出 | ○ |
| 6 | `IModulePresetService` | `ModulePresetService` | preset.csv の読み込み | ○ |
| 7 | `IHostListExportService` | `HostListExportService` | hostlist.csv のタイムスタンプ付き export | ○ |
| 8 | `IPrinterDriverDetectorService` | `PrinterDriverDetectorService` | INF ファイル走査 + ドライバ抽出 | × （export のみ ○） |
| 9 | `IRegistryCollectionService` | `RegistryCollectionService` | catalog.json の永続化 + ws へのエクスポート | × （export のみ ○） |
| 10 | `ILooperService` | `LooperService` | looper_list.csv 編集 + module export + テスト実行 | ○ |
| 11 | `IPianistProfileService` | `PianistProfileService` | Pianist Profile (5+ ファイル) の I/O | ○ |
| 12 | `IPianistTestRunService` | `PianistTestRunService` | Pianist テスト実行（子プロセス起動） | ○ |
| 13 | `IFabriqBackupService` | `FabriqBackupService` | fabriq バックアップ複製 | ○ |
| 14 | `IFabriqUpdateService` | `FabriqUpdateService` | template からのオーバーレイ更新 | ○ |
| 15 | `ICryptoService` | `CryptoService` | AES-256-CBC + PBKDF2 暗号化 | × |
| 16 | `IAppSettingsService` | `AppSettingsService` | appsettings.json（将来拡張） | × |

---

## 1. `IWorkspaceService` — ルートディレクトリ管理

[Services/IWorkspaceService.cs](file:///E:/fabriq_studio/FabriqStudio/Services/IWorkspaceService.cs)

| メンバ | シグネチャ | 用途 |
|---|---|---|
| `RootPath` | `string?` | 現在開いているワークスペース絶対パス、未設定時 null |
| `IsOpen` | `bool` | ワークスペース有効か |
| `WorkspaceChanged` | `event EventHandler<WorkspaceChangedEventArgs>` | 変更通知（ViewModel が購読してデータ再ロード） |
| `Validate(path)` | `string?` | 検証、null = OK / 文字列 = エラーメッセージ |
| `Open(path)` | `void` | 検証して open、`WorkspaceChanged` 発火 |
| `Close()` | `void` | RootPath を null、永続化クリア、`WorkspaceChanged` 発火 |
| `Reload()` | `void` | 同 path で再ロード（NewPath==OldPath==RootPath で発火）|
| `TryRestorePersisted()` | `void` | `<exe>/config/workspace.json` から前回 path 復元、**WorkspaceChanged は発火しない**（VM 構築前に呼ばれるため） |
| `CreateFromTemplateAsync(path)` | `Task<string?>` | 組み込み template から新規 ws 作成 |

### 永続化

```
<exe directory>/config/workspace.json
```

ポータブル運用対応（`AppDomain.CurrentDomain.BaseDirectory`）。中身は `{ "RootPath": "C:\\fabriq" }` 程度。

### イベント駆動

`WorkspaceChanged` を全 ViewModel が購読。発火時に各 ViewModel が `OnWorkspaceChanged` ハンドラで自身のデータをリロード。

---

## 2. `ICsvService` — ワークスペース相対 CSV I/O

[Services/ICsvService.cs](file:///E:/fabriq_studio/FabriqStudio/Services/ICsvService.cs)

| メソッド | シグネチャ | 用途 |
|---|---|---|
| `ReadAsync<T>(relativePath)` | `Task<IReadOnlyList<T>>` | ws root + relativePath の CSV をモデル T のリストに |
| `WriteAsync<T>(relativePath, records)` | `Task` | モデルリストを CSV 上書き |

`relativePath` 例: `"kernel/csv/hostlist.csv"`、`"profiles/Master_Pre01.csv"`。`IWorkspaceService.RootPath` を起点に絶対パス解決。

CsvHelper の **RFC 4180 厳格モード**（共通 `ICsvService` 経由は厳格性を維持）。寛容パースが必要な Pianist 専用 CSV は `PianistProfileService.TolerantReadConfig` で個別対応する（[fabriq_studio__reference__pianist_profile_schema.md](fabriq_studio__reference__pianist_profile_schema.md) §「2. procedure.csv 寛容パース」）。

エンコーディング規約: **BOM 付き UTF-8** で書き込み（PowerShell `Import-Csv` 互換）、読み込みは BOM 有無自動判定。

---

## 3. `IFileService` — 絶対パス汎用 I/O

[Services/IFileService.cs](file:///E:/fabriq_studio/FabriqStudio/Services/IFileService.cs)

| メソッド | シグネチャ | 用途 |
|---|---|---|
| `ReadTextAsync(absolutePath)` | `Task<string?>` | テキスト読込、不在で null |
| `ReadCsvAsDataTableAsync(absolutePath)` | `Task<DataTable>` | カラム不定の CSV を DataTable で（汎用 grid 用） |
| `WriteTextAsync(absolutePath, content)` | `Task` | BOM 付き UTF-8 でテキスト書込 |
| `WriteCsvFromDataTableAsync(absolutePath, table)` | `Task` | DataTable → BOM 付き UTF-8 CSV |
| `LoadLinesFromFileAsync(absolutePath)` | `Task<List<string>>` | 1 行 1 項目、空行・重複除外（インポート用） |
| `LoadCsvAsModelAsync<T>(absolutePath)` | `Task<List<T>>` | 外部 CSV → モデル T |

`ICsvService` がワークスペース相対なのに対し、こちらは **絶対パス前提**。USB 上の外部ファイル / scratch / インポート系で使う。

---

## 4. `IProfileService` — profiles/*.csv 編集

[Services/IProfileService.cs](file:///E:/fabriq_studio/FabriqStudio/Services/IProfileService.cs)

| メソッド | シグネチャ | 用途 |
|---|---|---|
| `GetProfilesAsync()` | `Task<IReadOnlyList<ProfileEntry>>` | profiles/ 直下 .csv を `ProfileEntry` 列挙、ファイル名 = profile 名 |
| `GetProfileModulesAsync(profile)` | `Task<IReadOnlyList<ProfileScriptEntry>>` | 1 profile の Order 順 module list |
| `SaveProfileModulesAsync(profile, modules)` | `Task` | 渡し順で **`Order` を 10 刻みで振り直す** + CSV 上書き |
| `CreateProfileAsync(profileName)` | `Task<ProfileEntry>` | 新規空 profile、OS 禁則文字 / 重複チェック |

`Order` 自動振り直しが特徴。UI の D&D で並び替えた後は、保存時に `10, 20, 30, ...` で正規化される（手動編集の Order ノイズが除去される）。

---

## 5. `IModuleService` — modules/*/module.csv 検出

[Services/IModuleService.cs](file:///E:/fabriq_studio/FabriqStudio/Services/IModuleService.cs)

| メソッド | シグネチャ | 用途 |
|---|---|---|
| `GetAllModulesAsync()` | `Task<IReadOnlyList<ModuleMasterEntry>>` | `modules/standard/*/module.csv` + `modules/extended/*/module.csv` 全件統合 |

シンプル 1 メソッド。各モジュールフォルダ名 + `module.csv` のメタデータを `ModuleMasterEntry`（Kind=`standard`/`extended` 含む）に詰めて返す。

詳細な ModuleMasterEntry は [fabriq_studio__reference__models_catalog.md](fabriq_studio__reference__models_catalog.md)、CSV スキーマは [fabriq_studio__reference__csv_schemas.md](fabriq_studio__reference__csv_schemas.md) §「module.csv」。

---

## 6. `IModulePresetService` — preset.csv 読み込み

[Services/IModulePresetService.cs](file:///E:/fabriq_studio/FabriqStudio/Services/IModulePresetService.cs)

| メソッド | シグネチャ | 用途 |
|---|---|---|
| `LoadAsync(moduleDirAbsolutePath)` | `Task<IReadOnlyDictionary<string, IReadOnlyList<string>>>` | preset.csv → 列名 → 選択肢 Value 一覧 |

戻り値の辞書キーは `StringComparer.OrdinalIgnoreCase`。preset.csv 不在は **空辞書 fallback**（graceful degradation：既存モジュールの動作に影響させない）。

`ModuleDetailViewModel` がこの辞書を使って DataGrid の特定列を ComboBox 化する。CSV スキーマは [fabriq_studio__reference__csv_schemas.md](fabriq_studio__reference__csv_schemas.md) §「preset.csv」。

---

## 7. `IHostListExportService` — hostlist タイムスタンプ export

[Services/IHostListExportService.cs](file:///E:/fabriq_studio/FabriqStudio/Services/IHostListExportService.cs)

| メソッド | シグネチャ | 用途 |
|---|---|---|
| `ExportAsync(request)` | `Task<HostListExportResult>` | `<ParentFolder>/hostlist_export_yyyyMMdd_HHmmss/` に hostlist.csv + README.txt 出力 |

`HostListExportRequest` の `Decrypt=true` で `ENC:` 復号（[fabriq_studio__reference__hostlist_csv_schema.md](fabriq_studio__reference__hostlist_csv_schema.md) §「Decrypt オプションの挙動」）。

---

## 8. `IPrinterDriverDetectorService` — INF 走査 + ドライバ抽出

[Services/IPrinterDriverDetectorService.cs](file:///E:/fabriq_studio/FabriqStudio/Services/IPrinterDriverDetectorService.cs)

| メソッド | シグネチャ | 用途 |
|---|---|---|
| `ScanAsync(scanDir, ct)` | `Task<IReadOnlyList<PrinterDriverInfo>>` | 再帰 *.inf 走査 + `Helpers.InfParser` で抽出 |
| `ExtractArchivesAsync(scanDir, sevenZipPath, ct)` | `Task<ArchiveExtractResult>` | 直下の .exe / .zip を同名フォルダへ展開 |
| `ExportToWorkspaceAsync(driver, ws, ct)` | `Task<DriverExportResult>` | `modules/standard/printer_driver_config/printer_driver_list.csv` に追記、DriverName 重複は skip |

### Extract の優先順位

1. `sevenZipPath` 指定 + 7z.exe 存在 → **7z.exe 使用**（exe / zip 両対応）
2. 7z.exe 無し + 対象 .zip → `System.IO.Compression.ZipFile` フォールバック
3. 7z.exe 無し + 対象 .exe → 展開不可、Failed カウント

7-Zip 25.01 が fabriq 本体に同梱されている（`modules/standard/printer_driver_config/tools/7z.exe`）。Studio 側はそのパスを `sevenZipPath` で受け取る運用。

### スキャン非依存性

`ScanAsync` / `ExtractArchivesAsync` は **ワークスペース非依存**（任意フォルダをスキャン可）。`ExportToWorkspaceAsync` のみ ws root が必要。

---

## 9. `IRegistryCollectionService` — catalog.json 管理

[Services/IRegistryCollectionService.cs](file:///E:/fabriq_studio/FabriqStudio/Services/IRegistryCollectionService.cs)

| メソッド | シグネチャ | 用途 |
|---|---|---|
| `Entries` | `IReadOnlyList<RegistryTemplateEntry>` | 現在ロード済み全エントリ（順序保持） |
| `EnsureInitializedAsync()` | `Task` | App.OnStartup で 1 回呼ぶ、catalog.json があればロード |
| `ReloadAsync()` | `Task` | catalog.json 再読込 |
| `AddAsync(entry)` | `Task` | 末尾追加 + 保存 |
| `UpdateAsync(entry)` | `Task` | Id 一致で差替 + 保存（不一致は no-op）|
| `RemoveAsync(id)` | `Task` | Id 削除 + 保存（ヒット時のみ）|
| `ExportToWorkspaceAsync(entry, wsRoot)` | `Task<ExportResult>` | reg_hklm_list.csv / reg_hkcu_list.csv に追記、KeyPath+KeyName 重複 skip |

詳細は [fabriq_studio__reference__registry_catalog.md](fabriq_studio__reference__registry_catalog.md)。

---

## 10. `ILooperService` — Script Looper

[Services/ILooperService.cs](file:///E:/fabriq_studio/FabriqStudio/Services/ILooperService.cs)

| メソッド | シグネチャ | 用途 |
|---|---|---|
| `LoadLooperListAsync(filePath)` | `Task<IReadOnlyList<LooperEntry>>` | looper_list.csv 読込（絶対パス） |
| `SaveLooperListAsync(filePath, entries)` | `Task` | looper_list.csv 上書き |
| `ExportModuleAsync(moduleName, entries, overwrite)` | `Task<string>` | `modules/extended/<name>/` に 4 ファイル生成（script_looper.ps1 / module.csv / looper_list.csv / Guide.txt）、戻り値は出力先絶対パス |
| `TestRunAsync(entries)` | `Task<string>` | 一時ディレクトリにコピー → kernel ドットソース PowerShell でテスト実行、ログ返却 |

### Export 時の 4 ファイル

| ファイル | 出処 |
|---|---|
| `script_looper.ps1` | template の `script_looper_template.ps1` をコピー |
| `module.csv` | モジュールメタデータ生成 |
| `looper_list.csv` | ユーザー設定（`entries` 引数）|
| `Guide.txt` | 操作ガイド |

`overwrite=false` で同名ディレクトリ既存時 `InvalidOperationException`。

### Test Run の Dual Resolution

カーネル解決は **「ワークスペースを優先、無ければテンプレートにフォールバック」**：

```
1. workspace/kernel/common.ps1 が存在 → それを dot-source
2. 無ければ template/template_fabriq/fabriq/kernel/common.ps1 を dot-source
```

作業ディレクトリをワークスペースルートに設定するため、looper_list.csv 内の **相対パスも解決される**。

---

## 11. `IPianistProfileService` — Pianist Profile I/O

[Services/IPianistProfileService.cs](file:///E:/fabriq_studio/FabriqStudio/Services/IPianistProfileService.cs)

| メソッド | シグネチャ | 用途 |
|---|---|---|
| `GetProfilesAsync()` | `Task<IReadOnlyList<PianistProfileEntry>>` | `modules/extended/pianist/profiles/` 直下のサブディレクトリ列挙、名前順 |
| `LoadProfileAsync(entry)` | `Task<PianistProfileData>` | 1 profile の全データ読込（json / 3 csv / instructions/*.txt） |
| `SaveProfileAsync(data, crypto)` | `Task<string?>` | §10 規約で書き出し、null=成功 / string=エラー |
| `LoadLegacyValuesAsync(entry)` | `Task<IReadOnlyList<PianistLegacyValueEntry>>` | 旧 long format values.csv（移行用、Phase 7） |
| `ValidateNewProfileName(name)` | `string?` | 半角英数 + アンダースコア / 未使用名チェック、null=OK |
| `CreateNewProfileAsync(name)` | `Task<PianistProfileEntry>` | テンプレートから 5 ファイル生成（最小 placeholder） |
| `DeleteProfileAsync(entry)` | `Task<string?>` | フォルダ再帰削除（取消不可、catalog 連動は呼出側責務） |
| `LoadPianistListAsync()` | `Task<IReadOnlyList<PianistListEntry>>` | pianist_list.csv（catalog）読込、不在で空 |
| `SavePianistListAsync(entries)` | `Task<string?>` | pianist_list.csv 上書き、null=成功 |

詳細は [fabriq_studio__reference__pianist_profile_schema.md](fabriq_studio__reference__pianist_profile_schema.md) §「9. ファイル I/O の規約」。

### CreateNewProfileAsync の最小 placeholder

```
P01 phase に Wait 1000ms の Step 1 件
values.csv は `*` 行のみ（変数列なし）
instructions/P01.txt は空 [Manual] section
shortcuts.csv は空（ヘッダのみ）
```

ユーザーが Editor で本格編集する前の **動作可能な雛形**。

---

## 12. `IPianistTestRunService` — Pianist テスト実行

[Services/IPianistTestRunService.cs](file:///E:/fabriq_studio/FabriqStudio/Services/IPianistTestRunService.cs)

| メソッド | シグネチャ | 用途 |
|---|---|---|
| `RunAsync(profileName, newPCName, ct)` | `Task<PianistTestRunResult>` | 子プロセス（PowerShell + pianist.ps1）でテスト実行、ログ + ExitCode + ModuleResult |

### 実装上のキーポイント

1. **`powershell.exe -STA -EncodedCommand` で子プロセス起動**（STA = pianist の WinForms 起動に必須）
2. **`kernel/common.ps1` を dot-source** してから `pianist.ps1` を呼ぶ（pianist.ps1 は kernel 関数群に強く依存し単体実行不可）
3. **profile picker のモック上書き**: `Import-ModuleCsv` を上書きして合成 1 行を返す → pianist.ps1:891 の `Items.Count -eq 1` shortcut で auto-skip
   - これにより workspace 側 `pianist_list.csv` を改変せずに任意 profile をテスト実行できる
4. **`$global:FabriqMasterPassphrase` 注入**: ラッパスクリプト内で `ICryptoService.MasterPassphrase` を流して `ENC:` セルの復号を有効化
5. **既定タイムアウトなし**: GUI 操作待ちのため。`CancellationToken` 経由でユーザーがキャンセルしたときだけ `Process.Kill(entireProcessTree: true)`

### `PianistTestRunResult`

```csharp
public record PianistTestRunResult(
    string  Log,
    int     ExitCode,
    string? ModuleResultStatus,
    string? ModuleResultMessage,
    bool?   ModuleResultVerified,
    bool    WasCancelled);
```

`ModuleResult*` は pianist.ps1 が末尾に出力する **Sentinel** `===PIANIST_TEST_RESULT===` 直後の JSON を parse して取得。プロセスが crash / Kill された場合は null。

---

## 13. `IFabriqBackupService` — バックアップ複製

[Services/IFabriqBackupService.cs](file:///E:/fabriq_studio/FabriqStudio/Services/IFabriqBackupService.cs)

| メソッド | シグネチャ | 用途 |
|---|---|---|
| `BackupAsync(request)` | `Task<FabriqBackupResult>` | `<ParentFolder>/fabriq_backup_yyyyMMdd_HHmmss/` に fabriq ミラー + USER_MEMO.txt + BACKUP_INFO.txt |

PS1 / バージョン管理ファイル等を除外しながらディレクトリ構造を再現する。詳細な除外パターンは実装側の `FabriqBackupService` 参照。

---

## 14. `IFabriqUpdateService` — オーバーレイ更新

[Services/IFabriqUpdateService.cs](file:///E:/fabriq_studio/FabriqStudio/Services/IFabriqUpdateService.cs)

| メソッド | シグネチャ | 用途 |
|---|---|---|
| `LoadRulesAsync(templateRoot)` | `Task<OverlayRules>` | template/dev/framework_overlay_rules.json 読込、schemaVersion 検証 |
| `ComputePlanAsync(templateRoot, targetRoot)` | `Task<FabriqUpdatePlan>` | kernel + 各モジュール bundle の VERSION 比較 → アクション決定 |
| `RunPreflight(plan, selected)` | `PreflightResult` | Fabriq.exe 実行状態 / resume_state / 書込権限 / REQUIRES_KERNEL チェック |
| `ApplyAsync(request, progress)` | `Task<FabriqUpdateResult>` | バックアップ zip 作成 → overlay 実行、`DryRun=true` で zip のみ |

fabriq 公開契約 [fabriq__contracts__overlay_contract.md](fabriq__contracts__overlay_contract.md) （`KERNEL_API.md §9`）に準拠。bundle 単位で SemVer 比較してアクション（Install / Upgrade / Skip / Downgrade-Block）を決定する。

### Preflight の §9.7 安全チェック

- Fabriq.exe が実行中でないこと
- `kernel/json/resume_state.json` が無いこと（中断中セッション保護）
- 書き込み権限あること
- 各 bundle の `REQUIRES_KERNEL` がアップグレード後 KERNEL_VERSION で満たされること

---

## 15. `ICryptoService` — AES-256-CBC 暗号化

[Services/ICryptoService.cs](file:///E:/fabriq_studio/FabriqStudio/Services/ICryptoService.cs)

| メンバ | シグネチャ | 用途 |
|---|---|---|
| `MasterPassphrase` | `string?` get/set | 現セッションのパスフレーズ |
| `HasPassphrase` | `bool` | 設定済みか |
| `Encrypt(plainText, passphrase)` | `string` | 平文 → `"ENC:" + Base64` |
| `Decrypt(cipherText, passphrase)` | `string` | `"ENC:" + Base64` → 平文 |

PowerShell `Unprotect-FabriqValue` と完全互換（AES-256-CBC + PBKDF2-HMAC-SHA256、100k iterations、固定ソルト）。詳細は [fabriq_studio__contracts__crypto_interop.md](fabriq_studio__contracts__crypto_interop.md)。

`MasterPassphrase` はセッション内でメモリに平文保持（DPAPI に永続化はしない）。Studio 終了で消える。

---

## 16. `IAppSettingsService` — appsettings.json（将来拡張）

[Services/IAppSettingsService.cs](file:///E:/fabriq_studio/FabriqStudio/Services/IAppSettingsService.cs)

```csharp
public interface IAppSettingsService
{
    // 将来の追加設定はここに定義する
}
```

**現状実装は最低限のスタブ**。`FabriqRootPath` は当初ここに置かれていたが `IWorkspaceService` に移管済み。将来追加設定（テーマ / 言語 / etc.）が生じた際に拡張する placeholder。

`appsettings.json` 自体は csproj `CopyToOutputDirectory=Always` で配布される枠（中身は空）。

---

## 横断的トピック

### 非同期 I/O 規約

すべてのファイル I/O 系は `async/await` で UI スレッドをブロックしない。`Task.Run` で同期 API（`File.ReadAllText` 等）をラップしているケースもある（`PianistProfileService.LoadStepsAsync` 等）。

### エラーハンドリング規約

| パターン | 例 |
|---|---|
| 戻り値 `string?`（null=成功） | `IPianistProfileService.SaveProfileAsync` / `DeleteProfileAsync` |
| 例外（致命的）| `IWorkspaceService.Open`（Validate NG で `ArgumentException`）|
| 結果オブジェクト | `ExportResult` / `BatchCryptoResult` / `FabriqUpdateResult` 等で `Error` プロパティ |
| silent 失敗 | `RegistryCollectionService.SaveAsync` 等、ランタイム継続を優先 |

「保存系で UI を止めたくない」場合は silent + 結果オブジェクト、「致命的エラー」は例外、「ユーザーに通知すべきエラー」は戻り値の `string?`。

### IDirtyAwareViewModel との連携

`HostDetailViewModel` / `ModuleDetailViewModel` / `ProfileDetailViewModel` / `LooperEditorViewModel` / `PianistProfileEditorViewModel` 等が `IDirtyAwareViewModel` 実装。`DirtyConfirmHelper` が画面遷移時に `IsDirty` チェック → 未保存変更ある時は破棄確認ダイアログ（kernel 3.x で全 Editor に展開）。

各 service は **Dirty を意識しない**（Dirty は ViewModel 側の責務、service は I/O のみ）。

---

## 関連ドキュメント

- アーキテクチャ概要（DI 構成 + MVVM パターン）: [fabriq_studio__architecture__01_layers.md](fabriq_studio__architecture__01_layers.md)
- ワークスペース概念: [fabriq_studio__architecture__02_workspace.md](fabriq_studio__architecture__02_workspace.md)
- 暗号化仕様: [fabriq_studio__contracts__crypto_interop.md](fabriq_studio__contracts__crypto_interop.md)
- メイン編集画面の UI: [fabriq_studio__apps__01_main_pages.md](fabriq_studio__apps__01_main_pages.md)
- Pianist Editor の UI: [fabriq_studio__apps__02_pianist_profile_editor.md](fabriq_studio__apps__02_pianist_profile_editor.md)
- HostEntry スキーマ: [fabriq_studio__reference__hostlist_csv_schema.md](fabriq_studio__reference__hostlist_csv_schema.md)
- Pianist Profile スキーマ: [fabriq_studio__reference__pianist_profile_schema.md](fabriq_studio__reference__pianist_profile_schema.md)
- Registry catalog: [fabriq_studio__reference__registry_catalog.md](fabriq_studio__reference__registry_catalog.md)
- Models 索引: [fabriq_studio__reference__models_catalog.md](fabriq_studio__reference__models_catalog.md)
- 小さな CSV スキーマ集合: [fabriq_studio__reference__csv_schemas.md](fabriq_studio__reference__csv_schemas.md)
