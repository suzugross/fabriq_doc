# fabriq オーバーレイ更新（FabriqUpdate）

> **対象**: fabriq_studio / apps / Fabriq Update Dialog
> **対象バージョン**: commit `3897c6e`（取得元: `git -C E:\fabriq_studio rev-parse --short HEAD`）
> **ドキュメント更新日**: 2026-05-07

Studio に **同梱された fabriq テンプレート** （`<exe>/template/template_fabriq/fabriq/`）の最新版を、現在開いているワークスペース（実運用の fabriq）に **SemVer 比較で安全に上書き** するダイアログツール。

Studio 自身を更新するのではなく、Studio が抱えている「**fabriq 本体の最新版テンプレート**」を、ユーザのワークスペース（カスタマイズされた fabriq）に **設計の根幹を壊さずに反映** する仕組み。

fabriq 本体の `dev/framework_overlay_rules.json`（**オーバーレイ契約**）に厳密準拠する。schemaVersion / 除外パス / bundle 構造は契約側が制御するため、Studio は契約に従う消費者の立場。

---

## 概要

| 項目 | 内容 |
|---|---|
| ダイアログ | `FabriqUpdateDialog.xaml` |
| ViewModel | `FabriqUpdateDialogViewModel` |
| Service | `IFabriqUpdateService` / `FabriqUpdateService` |
| Model | `OverlayRules` / `FabriqUpdatePlan` / `BundleUpdateItem` / `FabriqUpdateRequest` / `BundleUpdateResult` / `FabriqUpdateResult` / `PreflightResult` / `SemVer` |
| 契約 | fabriq KERNEL_API.md §9（更新オーバーレイ契約） |
| ルールファイル | `<templateRoot>/dev/framework_overlay_rules.json` |
| schemaVersion | **現行 1**。未対応版は `NotSupportedException` で拒否（§9.8） |

---

## 更新フロー（4 段階）

```
LoadRulesAsync(templateRoot)
        │
        ↓
ComputePlanAsync(templateRoot, targetRoot)
        │ (SemVer 比較で UpdateAction 決定)
        ↓
RunPreflight(plan, selected)
        │ (Fabriq.exe 実行 / resume_state / 書込権限 / REQUIRES_KERNEL)
        ↓
ApplyAsync(request, progress)
        │ (zip backup → bundle ごとに overlay → schema 警告抽出 → ログ書出し)
        ↓
FabriqUpdateResult
```

各段階の責務は `IFabriqUpdateService` の 4 メソッドに 1:1 対応する（[Services/IFabriqUpdateService.cs](file:///E:/fabriq_studio/FabriqStudio/Services/IFabriqUpdateService.cs)）。

---

## 段階 1. ルール読み込み — `LoadRulesAsync(templateRoot)`

[Services/FabriqUpdateService.cs:31-53](file:///E:/fabriq_studio/FabriqStudio/Services/FabriqUpdateService.cs):

```csharp
var rulesPath = Path.Combine(templateRoot, "dev", "framework_overlay_rules.json");
if (!File.Exists(rulesPath))
    throw new FileNotFoundException(
        "template に dev/framework_overlay_rules.json が見つかりません。"
      + "このバージョンの fabriq template は本機能に対応していません（kernel 2.2.0 以上が必要）。",
        rulesPath);

var rules = JsonSerializer.Deserialize<OverlayRules>(json, ...);

if (rules.SchemaVersion != SupportedSchemaVersion)   // 現行 1
    throw new NotSupportedException(
        $"Unsupported framework_overlay_rules.json schemaVersion: {rules.SchemaVersion}. ...");
```

### `OverlayRules` の構造（[Models/OverlayRules.cs](file:///E:/fabriq_studio/FabriqStudio/Models/OverlayRules.cs)）

| フィールド | 型 | 役割 |
|---|---|---|
| `schemaVersion` | int | 契約バージョン（現行 1） |
| `description` | string? | ルール説明（任意） |
| `excludeDirsTopLevel` | List\<string\> | トップレベルでのみ除外するディレクトリ |
| `excludeDirsRecursive` | List\<string\> | 再帰的に除外するディレクトリ（`profiles` 等） |
| `excludeFilesKernelLevel` | List\<string\> | kernel bundle でのみ除外するファイル |
| `moduleCsvWhitelist` | List\<string\> | モジュール bundle で更新対象とする CSV のホワイトリスト（**それ以外の CSV は site-specific として保持**） |
| `bundles.kernel` | KernelBundleDef | kernel bundle 定義（versionFile + includePaths） |
| `bundles.module` | ModuleBundleDef | module bundle 定義（pathPattern, versionFilePattern, requiresKernelFilePattern, typeValues） |

### `bundles.kernel`（KernelBundleDef）

```json
{
    "description": "Fabriq kernel and root files",
    "versionFile": "kernel/KERNEL_VERSION",
    "includePaths": [
        "kernel/", "Fabriq.exe", "Deploy.bat", "README.md"
    ]
}
```

`includePaths` は **ファイル単体**または**ディレクトリ**を混在で指定できる。Service 側で `File.Exists` / `Directory.Exists` で分岐処理。

### `bundles.module`（ModuleBundleDef）

```json
{
    "pathPattern": "modules/<type>/<name>",
    "versionFilePattern": "modules/<type>/<name>/VERSION",
    "requiresKernelFilePattern": "modules/<type>/<name>/REQUIRES_KERNEL",
    "typeValues": ["standard", "extended"]
}
```

`typeValues` で対象とする `<type>` を限定。現状は `standard` と `extended` の 2 種類。

### schemaVersion 拒否の意義

> 現行 schemaVersion=1 でしか対応していない。kernel 側で 2 を採用したルール変更（例: 新フィールド追加で `bundles.assets` を導入）が来た場合、Studio が **誤認識して破壊的更新する** リスクがある。それを防ぐため schemaVersion 不一致は `NotSupportedException` で **完全拒否**。

---

## 段階 2. Plan 計算 — `ComputePlanAsync(templateRoot, targetRoot)`

target 側 `Fabriq.exe` の存在を必須チェック → `LoadRulesAsync` → bundle 列挙 → SemVer 比較で `UpdateAction` 決定。

### `UpdateAction` 種別（[Models/FabriqUpdatePlan.cs](file:///E:/fabriq_studio/FabriqStudio/Models/FabriqUpdatePlan.cs)）

| Action | 条件 | 既定選択 | UI 表示 |
|---|---|---|---|
| `Update` | template > target | ✓ 選択 | `UPDATE` |
| `New` | template にあり target に無い | ✓ 選択 | `NEW` |
| `SkipSame` | 両 version 同一 | (選択可) | `SKIP (same)` |
| `SkipTargetNewer` | template < target | (選択可、警告) | `SKIP (target newer)` |
| `SkipNoTemplate` | template に VERSION 無し | 不可 | `SKIP (no template)` |
| `Preserve` | target にあり template に無い | 不可 | `PRESERVE (site-custom)` |

### `BundleUpdateItem` の不変性

```csharp
public partial class BundleUpdateItem : ObservableObject
{
    public string  BundleKey   { get; init; } = "";   // "kernel" or "modules/{type}/{name}"
    public string  DisplayName { get; init; } = "";
    public string  GroupName   { get; init; } = "";   // "kernel" / "modules/standard" / "modules/extended"
    public SemVer?    TargetVersion   { get; init; }
    public SemVer?    TemplateVersion { get; init; }
    public SemVer?    RequiresKernel  { get; init; }
    public UpdateAction Action        { get; init; }
    public string? WarningMessage     { get; init; }

    public bool CanSelect => Action is UpdateAction.Update or UpdateAction.New
                                or UpdateAction.SkipSame or UpdateAction.SkipTargetNewer;

    [ObservableProperty] private bool _isSelected;   // 唯一可変
    [ObservableProperty] private bool _isBlocked;    // 唯一可変（Preflight 結果）
}
```

UI から手動で操作できるのは `IsSelected` のみ。`Action`/`TargetVersion` 等は **計算結果として固定**。`Preserve` / `SkipNoTemplate` は `CanSelect = false` で **チェックボックス自体を無効化**。

### kernel 先頭、modules を順序付き

```csharp
foreach (var (type, name) in moduleNames.OrderBy(t => t.type).ThenBy(t => t.name))
```

DataGrid 上の表示は `kernel` → `modules/standard/<name 昇順>` → `modules/extended/<name 昇順>` の順。`GroupName` でグループヘッダ表示も可能。

### REQUIRES_KERNEL 事前チェック（§9.5）

各モジュールの `REQUIRES_KERNEL` ファイルが存在すれば SemVer として読み、**更新後の effective kernel** と比較する:

```csharp
var effectiveKernelAfterUpdate = IsUpdatingAction(kernelAction)
    ? templateKernel
    : targetKernel;

if (IsUpdatingAction(act) && requiresKer.HasValue && effectiveKernelAfterUpdate.HasValue
    && requiresKer.Value > effectiveKernelAfterUpdate.Value)
{
    blocked = true;
    warning = $"requires kernel {requiresKer}+, effective kernel would be {effectiveKernelAfterUpdate}. "
            + "check the kernel bundle first.";
}
```

「kernel を更新すれば満たせる」ケースでも、ユーザーが **kernel チェックを外している** 場合は `IsBlocked = true` で更新を阻止する。`WarningMessage` で UI に「kernel bundle first」と促す。

### モジュール名 union 列挙

```csharp
private static IEnumerable<(string type, string name)> EnumerateAllModuleNames(
    string templateRoot, string targetRoot, OverlayRules rules)
{
    var set = new HashSet<(string, string)>();
    foreach (var type in rules.Bundles.Module.TypeValues)
        foreach (var root in new[] { templateRoot, targetRoot })
            foreach (var sub in Directory.GetDirectories(Path.Combine(root, "modules", type)))
                set.Add((type, Path.GetFileName(sub)));
    return set;
}
```

`template` と `target` の両方からモジュール名を集めて union とする。これにより:

- target にしか無い → `Preserve`（site-custom）
- template にしか無い → `New`（lazy seed）
- 両方にある → SemVer 比較

### SemVer の独自実装

`Models/SemVer.cs` は `^(\d+)\.(\d+)\.(\d+)$` 専用パーサ。`System.Version`（4 コンポーネント）を意図的に使わない:

- `1.0.0` のような 3 コンポーネント形式に確定
- `TryParseFile(path)`: `VERSION` ファイルを 1 行読んで SemVer に
- `IComparable<SemVer>` 実装で `>` `<` `==` 比較

詳細は [fabriq_studio__reference__models_catalog.md](fabriq_studio__reference__models_catalog.md) 参照。

---

## 段階 3. Preflight — `RunPreflight(plan, selected)`

更新前の安全チェック（[FabriqUpdateService.cs:181-220](file:///E:/fabriq_studio/FabriqStudio/Services/FabriqUpdateService.cs)）。**§9.7 準拠**。

### 4 つのチェック

#### (1) Fabriq.exe プロセス実行チェック

```csharp
var running = Process.GetProcessesByName("Fabriq");
if (running.Length > 0)
    errors.Add($"Fabriq.exe が実行中です（{running.Length} プロセス）。終了してから再実行してください。");
```

実行中だと `File.Copy(..., overwrite: true)` がロック競合で失敗するため、**事前ブロック**で安全側に倒す。

#### (2) `resume_state.json` 残存チェック

```csharp
var resumePath = Path.Combine(plan.TargetRoot, "kernel", "json", "resume_state.json");
if (File.Exists(resumePath))
    errors.Add($"キッティング中断状態 (resume_state.json) が検出されました: {resumePath}");
```

中断中の kitting セッションを上書きすると **再開時に状態破壊** する可能性があるため、警告。

#### (3) target フォルダ書込権限テスト

```csharp
var testPath = Path.Combine(plan.TargetRoot, ".fabriq_studio_write_test");
File.WriteAllBytes(testPath, Array.Empty<byte>());
File.Delete(testPath);
```

UNC パス・ReadOnly 属性・ACL 等で書き込めないケースを **実書き込みテスト**で検出。失敗例外を `errors` に追加。

#### (4) REQUIRES_KERNEL ブロック

`Plan` 計算時に既に `IsBlocked = true` 付きの bundle を `RequiresKernelBlocks` リストに転記する（再判定はしない）。

### `PreflightResult.CanProceed`

```csharp
public bool CanProceed => Errors.Count == 0 && RequiresKernelBlocks.Count == 0;
```

`Apply` ボタンの `IsEnabled` 条件として使う。1 件でもブロック要因があれば実行不可。

---

## 段階 4. 適用 — `ApplyAsync(request, progress)`

[FabriqUpdateService.cs:226-327](file:///E:/fabriq_studio/FabriqStudio/Services/FabriqUpdateService.cs)。

### `FabriqUpdateRequest` の構造

```csharp
public record FabriqUpdateRequest(
    FabriqUpdatePlan                Plan,
    IReadOnlyList<BundleUpdateItem> SelectedBundles,   // ユーザがチェックしたもののみ
    string                          BackupZipPath,     // 自動生成タイムスタンプパス
    bool                            DryRun,            // true でファイル書き込みスキップ
    string                          LogFilePath);      // 実行ログ出力先
```

### フロー

1. **バックアップ zip 作成**（`!DryRun` のとき）:
   - `ZipFile.CreateFromDirectory(plan.TargetRoot, request.BackupZipPath, CompressionLevel.Optimal, includeBaseDirectory: false)`
   - 失敗時 → ログ書き出して **throw**（更新前ロールバック相当の保護）
2. **DryRun = true の場合**: バックアップは作らず、`Log("Dry-run: skipping backup zip creation")`
3. **bundle ごとに overlay**:
   - `IsBlocked == true` → `SKIP (blocked)` でログのみ
   - `Action != Update/New` → `SKIP` でログのみ
   - `BundleKey == "kernel"` → `OverlayKernel`
   - それ以外 → `OverlayModule`
4. **CHANGELOG schema 警告抽出**: `<templateRoot>/CHANGELOG.md` から `[Unreleased]` + 最新 `[X.Y.Z]` セクション内の `schema` を含む行を抽出
5. **ログファイル書き出し**: `LogFilePath` に時刻付きで全行
6. `FabriqUpdateResult` を返却（成功・失敗・スキップ件数 + 警告リスト）

### `OverlayKernel` の挙動

`includePaths` の各エントリについて:

- **ファイル単体**（`Fabriq.exe`/`Deploy.bat`/`README.md` 等）: そのまま 1 ファイルコピー（`excludeFilesKernelLevel` で除外チェック）
- **ディレクトリ**（`kernel/` 等）: `Directory.EnumerateFiles(srcDir, "*", SearchOption.AllDirectories)` で全ファイル走査、3 段階の除外フィルタ:
  1. `excludeDirsTopLevel` のディレクトリは丸ごとスキップ
  2. `excludeDirsRecursive` のディレクトリ（`profiles` 等）はパス中に出現すればスキップ
  3. `excludeFilesKernelLevel` の相対パス完全一致はスキップ

### `OverlayModule` の挙動

bundle ディレクトリ（`modules/{type}/{name}`）配下の全ファイルを走査:

- `*.csv` のうち `moduleCsvWhitelist` に含まれない CSV は **site-specific として保持**（コピーしない）
  - これにより `printer_driver_list.csv`/`hostlist.csv` 等のユーザーデータが上書きされない
- それ以外（`.ps1`/`.json`/`VERSION`/whitelisted CSV）は無条件で上書きコピー

### `TryCopy` のログ戦略

```csharp
void Log(string msg)     { logLines.Add(...); progress?.Report(msg); }   // UI 通知
void FileLog(string msg) { logLines.Add(...); }                          // ログファイルのみ
```

bundle 単位のサマリは `Log`（UI 進捗にも通知）、ファイル単位の詳細は `FileLog`（ログファイルのみ）。

> **理由**: 数千ファイルを毎回 `progress.Report` で通知すると、UI スレッド上の文字列連結が **O(n²) で爆発する**。bundle 単位の進捗なら 100 件未満で収まる。

### バックアップ失敗時の保護

```csharp
catch (Exception ex)
{
    Log($"Backup FAILED: {ex.Message}");
    WriteLog(request.LogFilePath, logLines);
    throw;
}
```

**バックアップが失敗した時点で更新を中止して例外を投げる**。bundle のコピーが始まる前に止めることで、「バックアップ無し + 部分更新」という最悪状態を防ぐ。

### CHANGELOG schema 警告抽出

```csharp
var lines = File.ReadAllLines(changelog);
bool inSection = false;
int sectionsRead = 0;
foreach (var line in lines)
{
    if (line.StartsWith("## ", StringComparison.Ordinal))
    {
        if (sectionsRead >= 2) break;            // [Unreleased] + 最新版の 2 セクション
        inSection = true;
        sectionsRead++;
        continue;
    }
    if (!inSection) continue;
    if (line.Contains("schema", StringComparison.OrdinalIgnoreCase))
        warnings.Add(line.TrimStart('-', ' ', '*').Trim());
}
```

最新 2 セクション（通常 `[Unreleased]` + 直近の `[X.Y.Z]`）の中から `schema` を含む行のみピックアップ。**「スキーマ変更を伴う更新だ」という注意喚起**を Apply 後に UI に反映するため。

---

## UI 構成（FabriqUpdateDialog）

| 領域 | 内容 |
|---|---|
| 上部 | 対象ワークスペース表示、テンプレート版 / 現行版の比較サマリ |
| 中央 | DataGrid（`Bundles` を `GroupName` でグループ化、Action / 旧版 / 新版 / 選択チェック / 警告） |
| 下部 | DryRun スイッチ / 適用ボタン / 進捗ログ |

### グルーピング

`BundleUpdateItem.GroupName` の値別:

- `"kernel"`: 1 件（ヘッダ「kernel」）
- `"modules/standard"`: N 件（標準モジュール群）
- `"modules/extended"`: M 件（拡張モジュール群）

### 警告表示

`WarningMessage` が非 null の bundle 行は背景色を変える等で **視覚的に注意喚起**:

- `IsBlocked = true` → 赤系（操作不可）
- `Action == SkipTargetNewer` → 橙系（target が新しい）

---

## 結果モデル（[FabriqUpdateResult.cs](file:///E:/fabriq_studio/FabriqStudio/Models/FabriqUpdateResult.cs)）

```csharp
public record FabriqUpdateResult(
    IReadOnlyList<BundleUpdateResult> BundleResults,
    string                            BackupZipPath,
    string                            LogFilePath,
    bool                              DryRun,
    IReadOnlyList<string>             SchemaWarnings)
{
    public int SuccessCount => BundleResults.Count(r => r.Success
        && r.Action is UpdateAction.Update or UpdateAction.New);
    public int FailureCount => BundleResults.Count(r => !r.Success);
    public int SkippedCount => BundleResults.Count(r => r.Action is UpdateAction.SkipSame
        or UpdateAction.SkipTargetNewer or UpdateAction.SkipNoTemplate or UpdateAction.Preserve);
}
```

### `BundleUpdateResult` の Success 判定

```csharp
public record BundleUpdateResult(
    string BundleKey, bool Success, int TouchedFileCount, int SkippedCount,
    IReadOnlyList<string> Errors,
    SemVer? FromVersion, SemVer? ToVersion, UpdateAction Action);
```

`Success` は **「全ファイルのコピーが成功した場合 true」**。`Errors.Count == 0 && (Action is Update or New)` の意味で、Skip 系は `Success = false` でも UI 上はエラーとして扱わない（Action で区別）。

---

## オーバーレイ契約（fabriq KERNEL_API.md §9）の要点

Studio 側の実装が依拠している fabriq 本体の契約事項。

| §節 | 内容 |
|---|---|
| §9.4 | `Action` 判定マトリクス（template ↔ target の SemVer 比較ルール） |
| §9.5 | `REQUIRES_KERNEL` の事前チェック義務 |
| §9.6 | `Action` enum 値の意味（Update/New/SkipSame/SkipTargetNewer/SkipNoTemplate/Preserve） |
| §9.7 | Preflight の必須項目（プロセス実行・resume_state・書込権限・REQUIRES_KERNEL） |
| §9.8 | schemaVersion 不一致時は **処理拒否** が義務（誤動作で fabriq を破壊するのを防ぐ） |

これらは fabriq 側で定義され、Studio 側が **消費者として準拠する** 契約。Studio が独自に逸脱することは許されない（kernel が増えたら Studio も追随）。

---

## 関連ドキュメント

| ドキュメント | 関係 |
|---|---|
| [fabriq__contracts__overlay_contract.md](fabriq__contracts__overlay_contract.md) | fabriq 本体側のオーバーレイ契約（KERNEL_API.md §9 のミラー） |
| [fabriq_studio__contracts__crypto_interop.md](fabriq_studio__contracts__crypto_interop.md) | 同じ「契約消費者」ポジションの隣接機能 |
| [fabriq_studio__reference__services_catalog.md](fabriq_studio__reference__services_catalog.md) | `IFabriqUpdateService` の 4 メソッドシグネチャ |
| [fabriq_studio__reference__models_catalog.md](fabriq_studio__reference__models_catalog.md) | `OverlayRules` / `FabriqUpdatePlan` / `BundleUpdateItem` 等の構造 |
| [fabriq_studio__apps__06_fabriq_backup.md](fabriq_studio__apps__06_fabriq_backup.md) | Update が **ファイル単位の zip backup** で安全網を作るのに対し、Backup は **設定値スナップショット** を作る |

---

## 変更履歴

- 2026-05-07 初版作成（`apps__04_other_tools.md` の FabriqUpdate セクションから個別化、4 段階フロー全体・契約準拠・kernel/module overlay 戦略を網羅）
