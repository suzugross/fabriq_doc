# メイン編集画面（基本パラメータ・端末一覧・モジュール・プロファイル）

> **対象**: fabriq_studio / apps / 編集系メイン画面
> **対象バージョン**: commit `3897c6e`
> **ドキュメント更新日**: 2026-05-06

ワークスペースを開いた直後に表示される左ペインメイン領域の 4 機能（基本パラメータ・モジュール編集・端末一覧・プロファイル詳細）について、ViewModel の責務と CSV 入出力契約を整理する。

---

## 1. 基本パラメータ（BasicParams）

| 項目 | 内容 |
|---|---|
| ViewModel | `BasicParamsViewModel`（Singleton） |
| View | `BasicParamsView.xaml` |
| 主担当 CSV | `kernel/csv/workers.csv`, `kernel/csv/categories.csv`, `kernel/csv/log_destinations.csv` |
| 関連 | プロファイル一覧 → `ShowProfileDetailMessage` 送信 |

複数の小規模 CSV の集約画面。fabriq 起動時に必要となる作業者・カテゴリ・ログ送信先のメンテナンスを 1 画面でまとめて行う。プロファイル一覧もここから選択して `ProfileDetailViewModel` に遷移する。

`WorkspaceDataUpdatedMessage` を購読し、詳細画面（HostDetail / ModuleDetail / ProfileDetail）で保存が完了したときに自身の表示を自動リフレッシュする。

---

## 2. 端末一覧（HostList → HostDetail）

| 項目 | 内容 |
|---|---|
| ViewModel | `HostListViewModel` → `HostDetailViewModel` |
| View | `HostListView.xaml` → `HostDetailView.xaml` |
| 主担当 CSV | `kernel/csv/hostlist.csv` |
| Model | `HostEntry`（[Models/HostEntry.cs](file:///E:/fabriq_studio/FabriqStudio/Models/HostEntry.cs)、43 列を `[ObservableProperty]` で展開） |
| 関連ダイアログ | `HostListExportDialog`（タイムスタンプ付き export） |

### HostEntry のフィールド構成（全 43 列）

- 基本: `AdminID`, `OldPCName`, `NewPCName`
- 有線 LAN: `EthernetIP`, `EthernetSubnet`, `EthernetGateway`
- 無線 LAN: `WifiIP`, `WifiSubnet`, `WifiGateway`
- DNS: `DNS1` ～ `DNS4`
- BitLocker: `Pin`
- プリンタ 1〜10（各 Name / Driver / Port の 3 列 × 10）

### HostListView の機能

- 行追加・削除・複製
- バッチ暗号化／復号: 選択行の対象列（`CryptoHelper.ExcludedColumns` 以外）に対して `ENC:` 暗号化を一括適用
- パスフレーズ照合: バッチ操作前に `IWorkspaceService.RootPath/kernel/txt/passphrase_verify.txt` の `surkitinisme` トークンを復号して照合（ミスマッチでブロック）
- export: タイムスタンプフォルダ `<parent>/hostlist_export_yyyyMMdd_HHmmss/` に `hostlist.csv` + `README.txt` を出力（Decrypt オプション付き）

### Dirty 検知

`HostEntry` は `JsonSerializer.Serialize` の文字列比較で `ContentEquals(other)` を実装。`Clone()` も同じ手法で全フィールドのディープコピーを作る。`HostDetailViewModel` は Load 時にスナップショットを保持し、保存／キャンセル時に `ContentEquals` で Dirty を判定する。

---

## 3. モジュール編集（ModuleEdit → ModuleDetail / AppConfig）

| 項目 | 内容 |
|---|---|
| ViewModel | `ModuleEditViewModel` → `ModuleDetailViewModel` または `AppConfigViewModel` |
| View | `ModuleEditView.xaml` → `ModuleDetailView.xaml` / `AppConfigView.xaml` |
| 主担当 CSV | `modules/{standard,extended}/<name>/module.csv` 全件 |
| Model | `ModuleMasterEntry`（[Models/ModuleMasterEntry.cs](file:///E:/fabriq_studio/FabriqStudio/Models/ModuleMasterEntry.cs)） |

### ModuleMasterEntry のスキーマ

CSV 列:

- `MenuName` — UI 表示名
- `Category` — カテゴリ（`kernel/csv/categories.csv` の Category 列と整合）
- `Script` — 起動スクリプト相対パス
- `Order` — 同カテゴリ内の表示順
- `Enabled` — `"0"` / `"1"`

CSV 非マッピング（`[Ignore]`）:

- `ModuleDir` — フォルダ名（例: `pianist`, `bitlocker_config`）
- `Kind` — `"standard"` / `"extended"`

### ModuleEditView の機能

- `IModuleService.GetAllModulesAsync()` で `modules/standard/*/module.csv` と `modules/extended/*/module.csv` を全件読み込み・統合表示
- カテゴリ別グルーピング、Order 順ソート
- 行クリックで `ShowModuleDetailMessage(entry)` を送信
- `MainViewModel` 側のハンドラが `ModuleDir == "app_config"` のみ `AppConfigViewModel` に分岐させ、それ以外は `ModuleDetailViewModel` に渡す

### app_config だけ専用画面な理由

`app_config` は他モジュールと CSV スキーマが大きく異なる（アプリインストールリストとして特殊）。共通の `ModuleDetailView` で表現すると UI が破綻するため、専用 ViewModel + View を持たせている。

### ModuleDetail の preset 連携

`IModulePresetService` が `<module_dir>/preset.csv` を読んで「列名 → 選択肢配列」の辞書を返す。`ModuleDetailViewModel` はこれを使って DataGrid の特定列を ComboBox 化する（`Helpers/PresetColumnFactory`）。preset.csv が無いモジュールは graceful に空辞書扱い（=普通のテキスト入力欄）。

---

## 4. プロファイル詳細（ProfileDetail）

| 項目 | 内容 |
|---|---|
| ViewModel | `ProfileDetailViewModel`（Singleton） |
| View | `ProfileDetailView.xaml` |
| 主担当 CSV | `profiles/<name>.csv` |
| Model | `ProfileEntry`（[Models/ProfileEntry.cs](file:///E:/fabriq_studio/FabriqStudio/Models/ProfileEntry.cs)）+ `ProfileScriptEntry`（[Models/ProfileScriptEntry.cs](file:///E:/fabriq_studio/FabriqStudio/Models/ProfileScriptEntry.cs)） |
| 入口 | `BasicParamsView` のプロファイルリスト（`ShowProfileDetailMessage`） |

### ProfileScriptEntry のスキーマ

CSV 列（fabriq KERNEL_API.md §4.1 準拠）:

- `Order` — 実行順
- `ScriptPath` — 実行スクリプトパス。`__RESTART__` `__AUTOPILOT__` 等のシステム特殊マーカーは前後 `__` で識別
- `Enabled` — `"0"` / `"1"`
- `Description` — 行の注釈
- `Segment` — セグメント分離用（`[Optional]`）
- `Note` — 任意メモ（`[Optional]`）
- `ErrorMode` — `skip` / `retry` / `ask`（`[Optional]`）
- `Group` — kernel 3.2.0 で追加された任意列（`[Optional]`）。同一値の行群が FlexProfile dashboard の `[Run: <Group>]` ボタンに集約される

UI 補助プロパティ:

- `IsEnabled`（bool）— `Enabled` の `"0"/"1"` を bool ラップ。CheckBox バインド用
- `IsSystemCommand` — ScriptPath の前後 `__` で判定。View で行スタイルを切り替える
- `DisplayName` — System Command はそのまま、それ以外は拡張子なしファイル名

### Order の振り直し

`IProfileService.SaveProfileModulesAsync` は「渡した順番で Order を 10 刻みに振り直してから書き込む」。これにより手動 Order 編集での衝突を排除し、ユーザーは行順序の Drag & Drop だけに集中できる。10 刻みは将来の差し込み余地を確保する慣習。

### CSV 編集系画面共通の保存契約

- `CsvService` 経由で **CsvHelper の標準書式** で書く（BOM 付き UTF-8 / CRLF）
- `[Optional]` プロパティ列は CSV ヘッダー欠損を許容（古いファイルとの互換性のため）
- 書き込みは常に上書き（マージは行わない）。同時編集衝突は OS のファイルロックに任せる

---

## CSV I/O サービスの位置付け

すべての CSV R/W は `ICsvService`（`CsvService.cs`）を通る:

```csharp
Task<IReadOnlyList<T>> ReadAsync<T>(string relativePath);
Task                   WriteAsync<T>(string relativePath, IEnumerable<T> records);
```

`relativePath` は **ワークスペースルート相対**。`IWorkspaceService.RootPath` を内部で結合する。これにより各 ViewModel は絶対パスを意識せず、ワークスペース切替時に自動で正しいファイルを参照する。

CsvHelper の設定は標準（ヘッダーマッピング・自動型変換）。モデル側で `[Name]` `[Optional]` `[Ignore]` といった属性を付けて細かい挙動を制御する。

---

## 共通の Dirty / Save パターン

詳細画面（HostDetail / ModuleDetail / AppConfig / ProfileDetail）の共通フロー:

1. **Load**: 一覧画面から渡されたエンティティ参照を `_original` に保持し、`Clone()` で `_snapshot` を作る
2. **HasUnsavedChanges**: `!_original.ContentEquals(_snapshot)` を返す
3. **DiscardChanges()**: `_snapshot` の値を `_original` のフィールドにコピーし戻す（共有エンティティ参照のため、親リストへの leak を防ぐ）
4. **Save**: `ICsvService.WriteAsync` で書き込み → `_snapshot = _original.Clone()` で再スナップショット → `WorkspaceDataUpdatedMessage` を送信 → `NavigateBackMessage` で戻る

`_original` は **親リストと同じインスタンス参照** を編集する設計のため、保存ボタンを押す前にユーザーがキャンセルしても親リストに反映されてしまう懸念があり、これを `DiscardChanges()` で吸収する（`IDirtyAwareViewModel` の存在意義の中核）。
