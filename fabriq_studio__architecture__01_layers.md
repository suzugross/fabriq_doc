# アーキテクチャ — 層構成と依存関係

> **対象**: fabriq_studio / architecture
> **対象バージョン**: commit `3897c6e`
> **ドキュメント更新日**: 2026-05-06

## 全体像

```
┌────────────────────────────────────────────────────────────────────┐
│  View (XAML, 29 ファイル)                                           │
│   ・WPF DataBinding でのみ ViewModel と通信                         │
│   ・Code-Behind は XAML イベントの受け流し限定（ロジック禁止）       │
└──────────────┬─────────────────────────────────────────────────────┘
               │  TwoWay binding / RelayCommand
               ▼
┌────────────────────────────────────────────────────────────────────┐
│  ViewModel (16 クラス)                                              │
│   ・ObservableObject + [ObservableProperty] / [RelayCommand]        │
│   ・WeakReferenceMessenger でページ間遷移メッセージを送受信          │
│   ・編集系画面は IDirtyAwareViewModel を実装                        │
└──────────────┬─────────────────────────────────────────────────────┘
               │  Service interface （DI 注入）
               ▼
┌────────────────────────────────────────────────────────────────────┐
│  Service (16 件、DI に 15 件登録)                                    │
│   ・全て Singleton。データロードは「初回問い合わせ時に I/O」          │
│   ・I/O は async/await で UI スレッドをブロックしない                 │
└──────────────┬─────────────────────────────────────────────────────┘
               │  CsvHelper / System.IO / System.Text.Json
               ▼
┌────────────────────────────────────────────────────────────────────┐
│  fabriq ワークスペース (kernel/ + modules/ + profiles/ + ...)       │
└────────────────────────────────────────────────────────────────────┘
```

中央の制御点は `IWorkspaceService`。ワークスペースが Open / Close / Reload されると `WorkspaceChanged` イベントが発火し、各 ViewModel が自身のデータを再ロードする。

## DI コンテナ初期化（[App.xaml.cs](file:///E:/fabriq_studio/FabriqStudio/App.xaml.cs)）

`App.OnStartup` で以下を順序通りに行う:

1. `ServiceCollection` を組み立て、`BuildServiceProvider()` する
2. **VM 構築前** に `IWorkspaceService.TryRestorePersisted()` を呼ぶ（イベント発火なしで前回パスを silent 復元）
3. `IRegistryCollectionService.EnsureInitializedAsync()` でレジストリ辞書 catalog.json をロード（ワークスペース非依存）
4. `MainWindow` を取得して `Show()`

VM のコンストラクタは `IWorkspaceService.IsOpen` を見て、true なら直接ワークスペースのデータをロードする。これにより、起動直後 `WorkspaceChanged` を発火させなくても VM は適切な初期状態になる。

## サービス登録（DI スコープ）

すべて `Singleton`:

```
IWorkspaceService           → WorkspaceService
ICsvService                 → CsvService
IProfileService             → ProfileService
IModuleService              → ModuleService
IFileService                → FileService
ILooperService              → LooperService
IRegistryCollectionService  → RegistryCollectionService
IPrinterDriverDetectorService → PrinterDriverDetectorService
ICryptoService              → CryptoService
IModulePresetService        → ModulePresetService
IHostListExportService      → HostListExportService
IFabriqBackupService        → FabriqBackupService
IFabriqUpdateService        → FabriqUpdateService
IPianistProfileService      → PianistProfileService
IPianistTestRunService      → PianistTestRunService
```

ViewModel もすべて Singleton（理由: ナビゲーション再訪時に同じインスタンスへ戻り、編集中状態を維持できる）。`MainWindow` のみ `Transient`。

非 DI 登録（手動インスタンス化）: `IAppSettingsService` / `AppSettingsService` — `appsettings.json` のリーダで現状ほぼ空（将来用）。

## ViewModel 一覧と責務

| ViewModel | 責務 |
|---|---|
| `MainViewModel` | 左ペインのナビゲーション + ワークスペース変更ハンドラ + Messenger 受信ハブ + パスフレーズ設定ダイアログ起動 |
| `WelcomeViewModel` | ワークスペース未設定時の選択画面（フォルダ選択 / 検証） |
| `BasicParamsViewModel` | workers / categories / log_destinations / passphrase_verify などの基本 CSV 編集 |
| `HostListViewModel` | hostlist.csv の表編集（行追加・削除・複製・暗号化バッチ・エクスポート） |
| `HostDetailViewModel` | 1 端末の詳細編集（IP / Wi-Fi / DNS / BitLocker PIN / プリンタ 1〜10） |
| `ModuleEditViewModel` | モジュールマスタ一覧（`modules/{standard,extended}/<name>/module.csv` 統合表示・カテゴリ別グルーピング） |
| `ModuleDetailViewModel` | 個別モジュールの設定 CSV / preset.csv 連携編集 |
| `AppConfigViewModel` | `app_config` モジュール専用の編集画面（一般モジュールと UI が異なるため特殊化） |
| `ProfileDetailViewModel` | `profiles/*.csv` のスクリプト実行リスト編集（Order 10 刻み振り直し） |
| `LooperEditorViewModel` | `looper_list.csv` 編集 + モジュール export + テスト実行 |
| `RegistryCollectionViewModel` | レジストリ辞書カタログ管理 + workspace への export |
| `RegistryPickerViewModel` | エクスポート対象選択ダイアログ用 |
| `PrinterDriverDetectorViewModel` | INF スキャン + アーカイブ展開 + hostlist 転記 |
| `PianistProfileEditorViewModel` | Pianist Profile (5 タブ + 4 sub-tab) の編集と Test Run |
| `FabriqUpdateDialogViewModel` | オーバーレイ更新の Plan 表示 / Preflight / Apply 進捗 |
| `IDirtyAwareViewModel` | 未保存編集検知用インターフェース（実装ではなく契約） |

## ナビゲーションモデル

`MainViewModel.CurrentPage` が右ペイン（単一 `ContentControl`）の表示を決める。3 種類の遷移パスがある:

### 1. 左ペインの静的ボタン → `NavigateCommand`

`MainWindow.xaml` 内の各 `Button` が `CommandParameter` で識別子を渡し、`MainViewModel.Navigate(string?)` が `switch` 式で対応する VM に切り替える:

```
"BasicParams"            → _basicParamsVm
"ModuleEdit"             → _moduleEditVm
"HostList"               → _hostListVm
"LooperEditor"           → _looperEditorVm
"RegistryCollection"     → _registryCollectionVm
"PrinterDriverDetector"  → _printerDriverDetectorVm
"PianistProfile"         → _pianistEditorVm
```

ナビゲーション前に必ず `ConfirmDiscardIfDirty()` を通る。

### 2. 一覧 → 詳細遷移 — `WeakReferenceMessenger`

`Messages/NavigationMessage.cs` に集約:

| メッセージ | 送信元 | 受信先の動作 |
|---|---|---|
| `ShowHostDetailMessage(HostEntry)` | `HostListViewModel` | `_hostDetailVm.Load(host)` → `CurrentPage = _hostDetailVm` |
| `ShowModuleDetailMessage(ModuleMasterEntry)` | `ModuleEditViewModel` | `app_config` のみ `_appConfigVm`、それ以外は `_moduleDetailVm` に分岐 |
| `ShowProfileDetailMessage(ProfileEntry)` | `BasicParamsViewModel` | `_profileDetailVm.Load(profile)` → `CurrentPage = _profileDetailVm` |

### 3. 一覧画面への戻り — `NavigateBackMessage(string)`

詳細画面が「保存して戻る」「キャンセル」時に発信:

```
"HostList"     → _hostListVm
"ModuleEdit"   → _moduleEditVm
"BasicParams"  → _basicParamsVm
```

ただしモーダルダイアログ表示中（`Application.Current.Windows.Count > 1`）は MainWindow 側のナビゲーションを抑止し、ダイアログ側に責務を委ねる（Pianist 系ダイアログ等）。

### 4. データ更新通知 — `WorkspaceDataUpdatedMessage(string)`

詳細画面で保存が完了したことを通知。`BasicParamsViewModel` 等が受信して自身のデータを自動リフレッシュする。

## ワークスペース変更時の連鎖

`IWorkspaceService` の `Open / Close / Reload` は `WorkspaceChanged` イベントを発火し、`MainViewModel` のハンドラが分岐する:

| 種別 | 判定条件 | MainViewModel の動作 |
|---|---|---|
| Close | `e.NewPath is null` | `WelcomeView` に戻し、`IsWorkspaceOpen = false` |
| Reload | `e.NewPath == e.OldPath` | `CurrentPage` 維持（各 VM が `WorkspaceChanged` 受信で自動再ロード） |
| Open | `e.NewPath != e.OldPath` | `BasicParamsView` に遷移し、`IsWorkspaceOpen = true` |

各 ViewModel は自身のコンストラクタで `WorkspaceChanged` を購読しておき、変更時にデータを再ロードする責務を持つ。

## 非同期 I/O 方針

すべての CSV / JSON / ファイル操作は `async/await`。UI スレッドからの呼び出しは `await` を介し、CPU バウンドな整形処理は `Task.Run` で別スレッドへ逃がす。

例: `WorkspaceService.CreateFromTemplateAsync` はテンプレートのフォルダ階層全コピーを `Task.Run(() => CopyDirectoryRecursive(...))` でラップする。

## 共通ヘルパー

| ヘルパー | 役割 |
|---|---|
| `DirtyConfirmHelper` | 未保存変更ダイアログの文言と DiscardChanges 呼び出しを集約（[fabriq_studio__architecture__02_workspace.md](fabriq_studio__architecture__02_workspace.md) 参照） |
| `CryptoHelper` | バッチ暗号化対象列の allowlist / `ValidatePassphrase` / `BatchCryptoResult` |
| `DataGridRowDragDropBehavior` + `DropIndicatorAdorner` | DataGrid 行ドラッグ＆ドロップ並び替え |
| `InfParser` | INF ファイルからプリンタモデル名抽出 |
| `PresetColumnFactory` / `CheckBoxColumnFactory` | DataGrid 列の動的生成（preset.csv 連携用） |
| `WindowEnumerator` | デスクトップウィンドウ一覧取得（Pianist Window Picker 用） |
| `PianistInstructionParser` | section marker DSL のパース / シリアライズ（pianist.ps1 とバイト一致） |
| `PianistValueTemplateSelector` | DataTemplate 選択（`*` 行と通常行で表示変える） |

## NRT と保守姿勢

C# 12 + Nullable Reference Types 有効。CSV モデルは初期値で空文字を割り当て、null 戻りは `IWorkspaceService.RootPath` のように意味的に「未設定」を表す箇所に限定する。
