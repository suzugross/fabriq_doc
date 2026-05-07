# Script Looper Editor

> **対象**: fabriq_studio / apps / Script Looper Editor
> **対象バージョン**: commit `3897c6e`（取得元: `git -C E:\fabriq_studio rev-parse --short HEAD`）
> **ドキュメント更新日**: 2026-05-07

ループ条件付きでスクリプトを繰り返し実行する fabriq モジュール（`script_looper.ps1` 系）を、Studio の GUI で組み立て・テスト実行・workspace 内モジュールとしてエクスポートするエディタ。

エディタが扱う CSV スキーマ（`looper_list.csv`）の列定義は [fabriq_studio__reference__csv_schemas.md](fabriq_studio__reference__csv_schemas.md#3-looper_listcsv--script-looper-用ループ設定) を参照。本書は **エディタ側の責務とフロー** に集中する。

---

## 概要

| 項目 | 内容 |
|---|---|
| ViewModel | `LooperEditorViewModel`（`IDirtyAwareViewModel` 実装） |
| View | [Views/LooperEditorView.xaml](file:///E:/fabriq_studio/FabriqStudio/Views/LooperEditorView.xaml) |
| Service | `ILooperService` / `LooperService` |
| Model | `LooperEntry`（[Models/LooperEntry.cs](file:///E:/fabriq_studio/FabriqStudio/Models/LooperEntry.cs)） |
| 出力先 | `<workspace>/modules/extended/<moduleName>/`（4 ファイル一括生成） |
| テンプレート | `<exe>/template/template_fabriq/looper_template/` |

Script Looper は fabriq の **拡張モジュール枠**（`modules/extended/`）でユーザーが任意のスクリプトを安全にリトライ実行するためのモジュール。Studio はその設定 CSV（`looper_list.csv`）と必須付随ファイル一式を **新規モジュールとして export** する I/O ツールに徹する。

---

## ViewModel の責務

[ViewModels/LooperEditorViewModel.cs](file:///E:/fabriq_studio/FabriqStudio/ViewModels/LooperEditorViewModel.cs) は以下の状態を保持する:

| 状態 | 型 | 役割 |
|---|---|---|
| `Rows` | `ObservableCollection<LooperEntry>` | 編集中のループエントリ |
| `SelectedRow` | `LooperEntry?` | 行操作の対象（追加・削除・並び替えの基準） |
| `ModuleName` | `string` | エクスポート時のフォルダ名・メニュー表示名（必須） |
| `IsDirty` | `bool` | 未保存変更フラグ（`IDirtyAwareViewModel.HasUnsavedChanges` に転送） |
| `IsExporting` / `IsRunning` | `bool` | コマンド競合制御（`CanExecute` の入力） |
| `StatusMessage` / `ErrorMessage` | `string?` | UI バー表示用 |

### ConditionTypes（ComboBox バインド用）

```csharp
public IReadOnlyList<string> ConditionTypes { get; } = ["OnError", "Always"];
```

XAML 側の `Condition` 列 `DataGridComboBoxColumn` は `LooperEntry.AllConditions` を直接静的バインドする（`{x:Static models:LooperEntry.AllConditions}`）。

### Dirty 検知の仕掛け

- `Rows` の `CollectionChanged` 購読 → 行の追加・削除で `IsDirty = true`
- 各 `LooperEntry` の `PropertyChanged` 購読 → セル編集で `IsDirty = true`
- 新規行が `Rows` に追加されたタイミング（`OnRowsCollectionChanged`）でも購読を継続して張る

`IDirtyAwareViewModel` 実装により、ワークスペースを閉じる／別画面に遷移する際に保存確認ダイアログをトリガできる。

### DirtyDescription

```csharp
public string DirtyDescription => string.IsNullOrEmpty(ModuleName)
    ? "Script Looper"
    : $"Script Looper ({ModuleName})";
```

保存確認ダイアログに「Script Looper (my_looper) に未保存の変更があります」のように具体名で表示するため、ModuleName を取り込んで動的生成する。

### DiscardChanges

`Rows.Clear()` + `ModuleName = ""` + `IsDirty = false` のフルリセット。**このエディタは固定の保存先を持たない**（ファイルを開いた後でも、別フォルダにエクスポートできる）ため、破棄＝クリアの意味になる点に注意。

---

## 主要コマンドのフロー

### NewCommand

`Rows.Clear()` + `ModuleName = ""` + `IsDirty = false`。`DiscardChanges` と等価。

### LoadCommand

1. `OpenFileDialog` で `looper_list.csv` を選択させる。
2. `_looperService.LoadLooperListAsync(filePath)` で CSV をパース。
3. **親ディレクトリ名をモジュール名として推定**: `modules/extended/my_looper/looper_list.csv` → `ModuleName = "my_looper"`。
4. 新しいコレクションに対して `SubscribeDirty` を再張り付けし、`IsDirty = false` で読み込み完了状態に。

### ExportCommand（`CanExecute`: `!IsExporting && Rows.Count > 0 && !string.IsNullOrWhiteSpace(ModuleName)`）

```csharp
try
{
    await _looperService.ExportModuleAsync(ModuleName.Trim(), Rows, overwrite: false);
}
catch (InvalidOperationException ex)
{
    // 既存ディレクトリ → 上書き確認
    var result = MessageBox.Show($"{ex.Message}\n上書きしますか？", ...);
    if (result == MessageBoxResult.Yes)
        await _looperService.ExportModuleAsync(ModuleName.Trim(), Rows, overwrite: true);
}
```

**まず `overwrite: false` で試行 → `InvalidOperationException` を捕捉して上書き確認** という 2 段階パターン。`InvalidOperationException` を判定文字列に頼らず型で識別する設計。

成功後 `IsDirty = false` に戻し、ステータスバーに `エクスポート完了: modules/extended/<name>/` を表示。

### TestRunCommand（`CanExecute`: `!IsRunning && Rows.Count > 0`）

`_looperService.TestRunAsync(Rows)` を呼び出し、戻ってきたログ文字列を `LogViewerDialog.ShowLog("テスト実行結果", log)` でモーダル表示。**実モジュールを export せず、一時ディレクトリで dry-run** する仕組み（後述）。

### 行操作コマンド

| Command | CanExecute 条件 | 動作 |
|---|---|---|
| `AddRowCommand` | 常に有効 | 末尾に新規 `LooperEntry` を追加し、選択 |
| `InsertRowCommand` | `SelectedRow is not null` | 選択行の前に挿入 |
| `DeleteRowCommand` | `SelectedRow is not null` | 選択行を削除し、隣接行を再選択 |
| `MoveUpCommand` / `MoveDownCommand` | 選択行があり境界外でない | `Rows.Move(idx, idx±1)` で並び替え |

`SelectedRow` の `[NotifyCanExecuteChangedFor]` で挿入・削除・移動コマンドの `CanExecute` 状態が自動再評価される（CommunityToolkit.Mvvm Source Generator）。

---

## XAML 構成（[LooperEditorView.xaml](file:///E:/fabriq_studio/FabriqStudio/Views/LooperEditorView.xaml)）

```
┌───────────────────────────────────────────────────────────────────┐
│ ツールバー (Row 0)                                                │
│   📄 新規  📂 読み込み  📤 エクスポート(Ctrl+E)  ▶ テスト実行    │
│                            [● 未保存] [実行中...] [ステータス]   │
├───────────────────────────────────────────────────────────────────┤
│ モジュール名バー (Row 1)                                          │
│   モジュール名: [______________]                                  │
├──────────────────────────────────────┬────────────────────────────┤
│ DataGrid (Row 2 / Col 0)             │ リファレンスペイン         │
│   有効 | ScriptPath | MaxRetry |     │ (Col 2, 230px 固定)        │
│   Interval(s) | Condition |          │                            │
│   Description | Segment              │ Enabled / ScriptPath /     │
│                                      │ MaxRetry / IntervalSec /   │
│   [＋行追加] [挿入] [🗑削除] [↑] [↓]  │ Condition の説明           │
└──────────────────────────────────────┴────────────────────────────┘
```

### キーボードショートカット

```xml
<UserControl.InputBindings>
    <KeyBinding Key="E" Modifiers="Control" Command="{Binding ExportCommand}" />
    <KeyBinding Key="Delete" Command="{Binding DeleteRowCommand}" />
</UserControl.InputBindings>
```

- `Ctrl+E`: エクスポート
- `Delete`: 選択行削除（DataGrid フォーカス時）

### DataGrid のセル編集タイミング

各列の `UpdateSourceTrigger=LostFocus` で **フォーカス離脱時**に ViewModel に伝搬。`PropertyChanged` ではなく `LostFocus` を採用しているのは、`MaxRetry`/`IntervalSec` の `INotifyDataErrorInfo` バリデーションが「タイプ中に毎回赤くなる」のを避けるため。

### Condition 列の ComboBox 静的ソース

```xml
<DataGridComboBoxColumn Header="Condition"
    SelectedItemBinding="{Binding Condition, UpdateSourceTrigger=LostFocus}"
    ItemsSource="{x:Static models:LooperEntry.AllConditions}"
    Width="100" />
```

`LooperEntry.AllConditions` は `static readonly IReadOnlyList<string>` で `[ConditionOnError, ConditionAlways]`。Condition の選択肢を追加する場合は **モデル側の定数を追加** する。ViewModel の `ConditionTypes` も同等のリストだが、現在 XAML からの参照は静的版の方を使っている（モデル側を Single Source of Truth とする方針）。

### ダブルクリックでファイルダイアログ

`PreviewMouseDoubleClick="DataGrid_PreviewMouseDoubleClick"` イベントが ScriptPath セルでファイルダイアログを開く（XAML.cs 側で実装）。リファレンスペインに「💡 セルをダブルクリックでファイルダイアログを開く」と注記。

---

## エクスポートサービス（`ExportModuleAsync`）

[Services/LooperService.cs:61-116](file:///E:/fabriq_studio/FabriqStudio/Services/LooperService.cs):

```csharp
public async Task<string> ExportModuleAsync(
    string moduleName, IEnumerable<LooperEntry> entries, bool overwrite = false)
{
    // 1. バリデーション
    if (string.IsNullOrWhiteSpace(moduleName))
        throw new ArgumentException("モジュール名を入力してください。");
    var invalidChars = Path.GetInvalidFileNameChars();
    if (moduleName.Any(c => invalidChars.Contains(c)))
        throw new ArgumentException("モジュール名に使用できない文字が含まれています。");

    var sanitized = moduleName.Trim();
    var fabriqRoot = GetRoot();   // workspace 必須
    var destDir    = Path.Combine(fabriqRoot, "modules", "extended", sanitized);

    // 2. 既存チェック
    if (Directory.Exists(destDir))
    {
        if (!overwrite)
            throw new InvalidOperationException($"モジュール「{sanitized}」は既に存在します。");
        await Task.Run(() => Directory.Delete(destDir, recursive: true));
    }

    // 3. 4 ファイル生成
    Directory.CreateDirectory(destDir);
    File.Copy(template/script_looper.ps1, destDir/script_looper.ps1, overwrite: true);
    File.Copy(template/Guide.txt,         destDir/Guide.txt,         overwrite: true);
    // module.csv 生成
    writer.WriteLine("MenuName,Category,Script,Order,Enabled");
    writer.WriteLine($"{sanitized},Scripts,script_looper.ps1,90,0");
    // looper_list.csv 保存
    await SaveLooperListAsync(destDir/looper_list.csv, entries);

    return destDir;
}
```

### 生成される 4 ファイル

```
<workspace>/modules/extended/<moduleName>/
├── script_looper.ps1   ── テンプレート（<exe>/template/template_fabriq/looper_template/）からコピー
├── Guide.txt           ── 操作ガイド（同上）
├── module.csv          ── 1 行の固定内容（<moduleName>,Scripts,script_looper.ps1,90,0）
└── looper_list.csv     ── ユーザが UI で組み立てたエントリ（UTF-8 BOM + CsvHelper で書き込み）
```

### 上書きの安全性

`overwrite: true` の場合 `Directory.Delete(destDir, recursive: true)` で **既存ディレクトリを完全削除してから再作成**。素材ファイル（`screenshots/` 等のサブフォルダ）が同名モジュール内にあった場合は失われる点に注意。ViewModel 側で確認ダイアログを必ず挟む。

### モジュール名のサニタイズ

- `null` / 空白だけ → `ArgumentException`
- Windows 不正文字（`< > : " / \ | ? *` 等、`Path.GetInvalidFileNameChars()`）含む → `ArgumentException`
- 前後空白は `Trim()` で除去

サニタイズ後の文字列が `module.csv` の `MenuName` 列にも転記されるため、メニュー表示にもそのまま使われる。

---

## テスト実行サービス（`TestRunAsync`）

ループ設定を **本番 fabriq に触れずに dry-run** する独自フロー。

### フロー全体

[Services/LooperService.cs:120-254](file:///E:/fabriq_studio/FabriqStudio/Services/LooperService.cs):

1. **一時ディレクトリ作成**: `Path.Combine(Path.GetTempPath(), $"fabriq_looper_test_{Guid.NewGuid():N}")`
2. **looper_list.csv を一時ディレクトリに保存**: `SaveLooperListAsync` を再利用
3. **script_looper.ps1 テンプレートを一時ディレクトリにコピー**: `$PSScriptRoot` が tempDir に解決される設計
4. **kernel/common.ps1 のパスを Dual Resolution で解決**（後述）
5. **PowerShell コマンド構築**（モック付き、後述）
6. **子プロセス起動**: `powershell.exe -NoProfile -ExecutionPolicy Bypass -EncodedCommand <Base64>`
   - Working directory: **ワークスペースルート**（`looper_list.csv` 内の fabriq 相対パスが正しく解決されるように）
   - stdin 即時 close（`Read-Host` 入力待ちを防止）
   - stdout/stderr を非同期読み取り
7. **タイムアウト 5 分**: `WaitForExitAsync(cts.Token)`、超過時 `Kill(entireProcessTree: true)`
8. **CLIXML フィルタ**: stderr に `#< CLIXML` で始まる Write-Progress/Write-Information のシリアライズが混入する場合があり、それを除外
9. **一時ディレクトリ削除**: `finally` で best effort（失敗は無視）

### Dual Kernel Resolution

```csharp
private string ResolveKernelCommonPath()
{
    if (_workspace.RootPath is not null)
    {
        var workspacePath = Path.Combine(_workspace.RootPath, "kernel", "common.ps1");
        if (File.Exists(workspacePath))
            return workspacePath;
    }
    var templatePath = Path.Combine(TemplateFabriqPath, "kernel", "common.ps1");
    if (File.Exists(templatePath))
        return templatePath;
    throw new FileNotFoundException("カーネル (kernel/common.ps1) が見つかりません。...");
}
```

- **優先 1**: ワークスペース内 `kernel/common.ps1`（ユーザが触っている版で再現性を取る）
- **フォールバック**: アプリ同梱の `<exe>/template/template_fabriq/fabriq/kernel/common.ps1`（ワークスペース未開放でもテストできるように）

### モック PowerShell コマンド

実際に組み立てられるコマンド文字列（[LooperService.cs:158-170](file:///E:/fabriq_studio/FabriqStudio/Services/LooperService.cs)）:

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8;
$ProgressPreference = 'SilentlyContinue';
$InformationPreference = 'SilentlyContinue';
$global:AutoPilotMode = $true;
function Read-Host { return 'Y' };
. '<kernelPath>';
function Confirm-Execution { param([string]$Message) return $true };
function Confirm-ModuleExecution {
    param([string]$Message)
    return (New-ModuleResult -Status 'Cancelled' -Message 'Test run: validation only')
};
function Wait-KeyPress { param([string]$Message) return };
& '<tempScript>';
```

順序:

1. **(a) UTF-8 エンコーディング強制**
2. **(b) AutoPilotMode + Read-Host モック**（**dot-source 前** に配置 → 読み込み中の対話を防止）
3. **(c) `common.ps1` をドットソース**でカーネル関数群をロード
4. **(d) `Confirm-ModuleExecution` / `Wait-KeyPress` をモック上書き**（**dot-source 後** に再定義）
5. **(e) `script_looper.ps1` 実行**

### Looper 固有の安全設計

`Confirm-ModuleExecution` を **`Cancelled` を返すモック** で上書きしているため、Looper はユーザー指定の外部スクリプトを `&` で実行するが、テスト環境では Steps 1-3（CSV 解析・パス解決・バリデーション表示）のみ実行され、**Step 5（実際のループ実行）はスキップ** される。

> Looper が外部スクリプトを実行するのは fabriq 本体の関数だが、それを test run でそのまま走らせると、ユーザーの本番設定（プリンタ削除・レジストリ書込み等）が **意図せず適用される危険性**がある。Confirm-ModuleExecution Cancelled モックでこのリスクを断ち切る。

### 戻り値（ログ）

stdout + (CLIXML 除外後の) stderr を文字列結合して返す。`LogViewerDialog.ShowLog` でモーダル表示。

### タイムアウト 5 分の根拠

Looper は本番でリトライ間隔を秒〜分単位で設定する想定。Test run は **Steps 1-3 だけなので 5 分で十分** だが、極端に大きい `looper_list.csv`（100 エントリ超）でも失敗しないよう余裕を取った値。

---

## 関連ドキュメント

| ドキュメント | 関係 |
|---|---|
| [fabriq_studio__reference__csv_schemas.md](fabriq_studio__reference__csv_schemas.md#3-looper_listcsv--script-looper-用ループ設定) | `looper_list.csv` 7 列スキーマ + INotifyDataErrorInfo バリデーション仕様 |
| [fabriq_studio__reference__services_catalog.md](fabriq_studio__reference__services_catalog.md) | `ILooperService` のシグネチャ全体 |
| [fabriq_studio__reference__models_catalog.md](fabriq_studio__reference__models_catalog.md) | `LooperEntry` のフィールド一覧 |
| [fabriq__modules__script_looper.md](fabriq__modules__script_looper.md) | fabriq 本体側の `script_looper.ps1` の動作（読み込み先 / 結果ハンドリング） |
| [fabriq_studio__architecture__01_layers.md](fabriq_studio__architecture__01_layers.md) | `IDirtyAwareViewModel` パターン全体像 |

---

## 変更履歴

- 2026-05-07 初版作成（`apps__04_other_tools.md` の Script Looper セクションから個別化、ViewModel/Service/View を網羅的に取り込み）
