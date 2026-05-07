# フリート画面（MainWindow）

> **対象**: fabriq_evidence_manager / apps / フリート画面
> **対象バージョン**: 3.8.0（取得元: `E:\fabriq_evidence_manager\FabriqEvidenceManager\FabriqEvidenceManager.csproj` `<Version>`）
> **ドキュメント更新日**: 2026-05-07

アプリ起動直後に表示されるトップウィンドウ。1 つの evidence ルートディレクトリを開き、その配下の全 PC を 1 行 = 1 PC の `DataGrid` で一覧表示し、hostlist との突合・ベースライン PC との比較・納品データ出力を一括で行うフリート操作画面。

| 項目 | 内容 |
|---|---|
| ViewModel | `MainWindowViewModel`（`Transient`、DI 経由） |
| View | `Views/MainWindow.xaml`（524 行）+ `Views/MainWindow.xaml.cs` |
| 起動 | `App.OnStartup` で `IServiceProvider.GetRequiredService<MainWindow>()` 解決 → `Show()` |
| ウィンドウ仕様 | `Title="Fabriq Evidence Manager" / Height=700 Width=1100 / WindowState=Maximized / Background="#BABEc2"` |
| シャットダウン | `App.xaml` `ShutdownMode="OnMainWindowClose"`（メインを閉じればプロセス終了） |

## 画面構成（5 段グリッド）

```
┌────────────────────────────────────────────────────────────┐
│ Row 0: ヘッダーバー（FABRIQ EVIDENCE MANAGER + 青黄赤線）  │ Auto
├────────────────────────────────────────────────────────────┤
│ Row 1: GroupBox「Evidence フォルダ」                        │ Auto
│   - 参照... / 読み込み / 自動更新(30秒)                    │
│   - Hostlist: CSV 読込 + 件数表示                          │
│   - Baseline PC: 選択PCをベースラインに設定 / クリア         │
├────────────────────────────────────────────────────────────┤
│ Row 2: 検索バー + DataGrid（メインコンテンツ）              │ *  (Star)
│   - ⚙ 設定 ボタン（右上）                                  │
│   - FILTER テキストボックス + 表示件数カウンタ              │
│   - DataGrid（10 列、行ハイライトと行ダブルクリック）       │
├────────────────────────────────────────────────────────────┤
│ Row 3: 納品データ出力 ボタン                                │ Auto
├────────────────────────────────────────────────────────────┤
│ Row 4: ステータスバー（LINK/ACT LED + ProgressBar + Msg）   │ Auto
└────────────────────────────────────────────────────────────┘
```

## 1. ヘッダーバー（Row 0）

`Background="#4A4A4A"` のダークバー。`FABRIQ` の太字 + `EVIDENCE MANAGER` の細字。直下に CentreCOM 風の **青 (#4A90D9) / 黄 (#F2C94C) / 赤 (#EB5757) の 3 色 2px 横線** をストライプ表示する。テーマ全体の意匠（Allied Telesis 風）は `App.xaml` の Resources で統一されており、その視覚的アクセントとして本バーが機能する。

機能はなく、ブランディング目的のみ。

## 2. Evidence フォルダ + Hostlist + Baseline 設定（Row 1）

`<GroupBox Header="Evidence フォルダ">` 内に 3 段の操作行を縦積み。

### 2-1. Evidence フォルダ選択

| 操作 | 関連プロパティ / コマンド | 挙動 |
|---|---|---|
| `参照...` | `SelectEvidenceFolderCommand` | `OpenFolderDialog` を表示。前回値があればそこを `InitialDirectory` に。**選択直後に `LoadEvidenceCommand` を自動実行** する設計（`SelectEvidenceFolderAsync` メソッド内で `await LoadEvidenceCommand.ExecuteAsync(null)`） |
| `読み込み` | `LoadEvidenceCommand`（`CanExecute = nameof(CanLoadEvidence)`） | `EvidenceRootPath` が非空かつディレクトリ存在で実行可。`NestedEvidenceDiscoveryService.DiscoverAsync` → 各 PC に `IEvidenceParserService.PopulateDetails` → `RevalidateEvidence(CheckOptions)` → `EvaluateCautionsForAllPcs()`。読み込み中は `IsLoading=true` でボタン無効化 + ステータスバー progress bar |
| `自動更新 (30秒)` | `IsAutoReloadEnabled` | チェックで `DispatcherTimer (30秒間隔)` 起動。`tick` で `RefreshEvidenceAsync` を駆動。詳細は §「自動リロード」参照 |

`EvidenceRootPath` テキストボックスは `IsReadOnly="True"` で直接編集不可（必ず `参照...` 経由で選択）。

### 2-2. Hostlist 読み込み

`hostlist.csv` を読み込んで期待値として保持する。

| 操作 | 関連プロパティ / コマンド |
|---|---|
| `CSV 読込...` | `LoadHostlistCommand` |
| 状態表示 | `HostlistPath`（読み込み済みファイルパス）/ `HostlistStatusText`（"3件読込済" 等） |

`OpenFileDialog` で `*.csv` を選択 → `IHostlistService.Load(path)` → 件数を返す。読み込み成功で `EvaluateCautionsForAllPcs()` を再実行（既に PC 一覧が読み込み済みなら hostlist 不一致 caution が即時反映される）。失敗時は `MessageBox.Show` で例外メッセージを表示。

hostlist 未読み込み状態でも本アプリは動作する（突合判定は `HostlistFound=false` のレポートを返すだけ）。

### 2-3. Baseline PC 設定

フリート全体の整合性検証の基準点となる PC を「ベースライン」として設定する。CSV 読み込みではなく **既に DataGrid に表示中の PC を 1 つ選んで** 基準化する形式：

| 操作 | コマンド |
|---|---|
| `選択PCをベースラインに設定` | `SetBaselinePcCommand` |
| `クリア` | `ClearBaselineCommand` |

`SetBaselinePc` は `IBaselineService.LoadFromPc(SelectedPc)` を呼び、登録された 6 件の `IBaselineComparator` 全てに対して `CacheBaseline(baselinePc)` を発火する。これにより SystemInfo / Checklist / InstalledApps / License / DomainStatus / ExecutionSummary の各カテゴリで「baseline PC のスナップショット」がメモリに保持され、以降の `CompareAll(targetPc)` で全 PC を baseline と比較できる。

選択 PC 未指定で押すと `MessageBox.Show("ベースラインに設定するPCを一覧から選択してください。")`。設定後は `BaselineStatusText="ベースラインPC: {PcName}"` で表示。

ベースライン設定 / クリア後は `EvaluateCautionsForAllPcs()` を再実行し、`SelectedPc` のサイドパネル情報も `UpdateVerificationInfo` で更新。

## 3. 検索バー + DataGrid（Row 2）

中核。1 行 = 1 PC の縦長フリート一覧。

### 3-1. 検索フィルタ

`FILTER:` ラベル + テキストボックス。`SearchText` プロパティを `UpdateSourceTrigger=PropertyChanged` で即時バインド、`OnSearchTextChanged` で `TargetPCsView.Refresh()` を呼ぶリアルタイムフィルタ。

`FilterPcEvidence(obj)` は `pc.PcName / pc.SerialNumber / pc.WarningMessage` を **OR 検索**（部分一致 / `OrdinalIgnoreCase`）。`DisplayCountText` プロパティが `"3 / 10 件"` 形式で「フィルタ後 / 全体」の件数を表示する。

### 3-2. DataGrid（10 列）

`ItemsSource="{Binding TargetPCsView}"`（`CollectionViewSource` 経由）、`SelectionMode=Single / SelectionUnit=FullRow / IsReadOnly=True / ClipboardCopyMode=IncludeHeader`。

| 列 | 内容 | バインド |
|---|---|---|
| 計画 PC 名 | hostlist で選択された名前（= manifest.selectedNewPcName）。`HasMemo=true` のときに末尾 `📝` アイコン | `PcName` + `HasMemo` |
| 実 PC 名 | manifest.computerName（rename 後の OS 上名前）。不一致時赤背景 | `ActualComputerNameDisplay` + `HasPcNameMismatch` |
| シリアル | 採用ソース併記。フォールバック取得時オレンジ強調 | `SerialNumberDisplay` + `HasSerialFallbackWarning` |
| 収集日 | `yyyy_MM_dd_HHmmss` | `CollectionDate` |
| MAC (イーサネット) | 算出プロパティ | `EthernetMac` |
| MAC (Wi-Fi) | 算出プロパティ | `WifiMac` |
| BitLocker (回復キー) | 各ドライブ 1 行 = `{drive}: {recoveryPassword}` の `ItemsControl` | `BitLockerKeys` |
| Win 認証 | 認証済 / 未認証 / (未取得)。`LicenseStatusToBg` で背景色 | `WindowsLicenseStatusDisplay` |
| Office 認証 | 認証済 / サインイン待ち / 認証失敗 / 未インストール / (未取得)。`LicenseStatusToBg` で背景色 | `OfficeLicenseStatusDisplay` |
| 検出エビデンス | "PC情報 / キャプチャ / BitLocker / チェックリスト / 履歴" の存在組合せ | `DetectedItems` |
| ステータス | `WarningMessage`（赤）+ `CautionMessage`（オレンジ）の縦並び。固定 320px 幅 | `WarningMessage` + `CautionMessage` |

### 3-3. 行ハイライト

`DataGridRow.RowStyle` の `DataTrigger` で 2 軸ハイライト：

| 条件 | 背景色 | 意味 |
|---|---|---|
| `HasCaution=True` | `#FFF9C4`（薄黄） | 要確認: PC名不一致 / SNフォールバック / hostlist 不一致 / ベースライン差異 / Office サインイン待ち |
| `HasWarning=True` | `#FFCDD2`（薄赤、Caution より優先） | 欠損: MAC/BitLocker/SN/ライセンス未取得・未認証 |
| `IsMouseOver` | `#D2D4D9` | hover フィードバック |

赤と黄が同時に立つ場合は赤が優先（`Triggers` 評価順）。

### 3-4. 行ダブルクリック → PC 詳細ウィンドウ

`EventSetter Event="MouseDoubleClick" Handler="OnPcRowDoubleClick"` で `MainWindow.xaml.cs` の `OnPcRowDoubleClick` ハンドラを発火。

```csharp
var detailVm = new PcDetailViewModel(
    pc:               vm.SelectedPc,
    checkOptions:     vm.CheckOptions,
    isHostlistLoaded: vm.IsHostlistLoaded,
    isBaselineLoaded: vm.IsBaselineLoaded,
    verification:     vm.CurrentVerificationReport,
    checklist:        vm.CurrentChecklistResult,
    history:          vm.CurrentExportHistory,
    baseline:         vm.CurrentBaselineReport,
    memoService:      vm.MemoService);

var detailWindow = new PcDetailWindow(detailVm) { Owner = this };
detailWindow.Show();
```

`Show()`（`ShowDialog` ではない）で **modeless 起動**。`PcDetailViewModel` は起動時のレポートを **スナップショット保持** するため、別 PC を選んでもう 1 度ダブルクリックすれば **2 ウィンドウが並列に並ぶ**（後勝ちで上書きされない）。詳細は [fabriq_evidence_manager__apps__02_pc_detail_window.md](fabriq_evidence_manager__apps__02_pc_detail_window.md) §「スナップショット保持と並列比較」を参照。

### 3-5. 設定ボタン

DataGrid 右上の `⚙ 設定` ボタンは `OnSettingsClick` ハンドラで `SettingsWindow` を modeless 起動：

```csharp
var settingsVm = vm.CreateSettingsViewModel();
var settingsWindow = new SettingsWindow(settingsVm) { Owner = this };
settingsWindow.Show();
```

`MainWindowViewModel.CreateSettingsViewModel()` ヘルパで `SettingsViewModel` を構築する。設定ダイアログ内の変更は `SettingsViewModel` 経由で `EvidenceCheckOptions` および `IBaselineService.SetEnabledCategoryIds(...)` に即時反映され、`MainWindowViewModel` の `EvidenceCheckOptions.PropertyChanged` リスナまたは `onBaselineCategoriesChanged` コールバックが全 PC を再評価する。詳細は [fabriq_evidence_manager__apps__03_settings_window.md](fabriq_evidence_manager__apps__03_settings_window.md) を参照。

## 4. 納品データ出力ボタン（Row 3）

右下に大きく `納品データ出力`（`Padding=20,8 / FontSize=14 / FontWeight=SemiBold`）。`ExportDeliveryCommand`（`CanExecute = !IsLoading && TargetPCs.Count > 0`）に紐づく。

### 出力フロー

```
1. OpenFolderDialog で出力先親ディレクトリを選択
   → キャンセルなら何もせず終了

2. EvidenceCollectorService.CreateDeliveryDirectory(parentPath)
   → "{YYYY_MM_DD_HHmmss}_fabriq_evi" を作成、絶対パスを返す

3. EvidenceCollectorService.CollectAsync(pcList, deliveryPath, progress, ct)
   → 各 PC ディレクトリへの再帰コピー + manifest_sha256.txt 生成
   → IProgress<int> で 0〜100 を報告 → "フォルダ整理中... N/M 台完了 (X%)"

4. BuildExcelExportOptions(pcList) で:
   - VerificationReports / ChecklistResults / ExportHistories / BaselineReports
     を Dictionary<PcEvidence, ...> として参照等価で組み立て

5. ExcelExportService.ExportAsync(pcList, "{ts}_PC情報一覧表.xlsx", options, ct)
   → main book + pc_details/ サブブックを出力

6. コピー失敗ファイルがあれば MessageBox.Warning でサマリ表示
   → 10 件超は先頭 10 + "他 N 件"

7. 完了 MessageBox（成功時 Information / 警告時 Warning）
```

`IsLoading=true` でボタン無効化、`StatusMessage` で進捗表示、`OperationCanceledException` はキャンセル扱いで silent 終了。コピー失敗（`IOException` / `UnauthorizedAccessException`）は **個別ファイル単位でスキップ + リスト返却** する設計のため、1 ファイル不良で出力全体は止めない。

## 5. ステータスバー（Row 4）

`#4A4A4A` のダークバー。`Padding="12,6"`。

### 5-1. ProgressBar

`IsIndeterminate=True` の薄い 3px バー。`Visibility={Binding IsLoading, Converter=BoolToVis}` で読み込み / 出力中のみ表示。

### 5-2. LINK/ACT LED

8x8 px の `Ellipse`、`Fill="#00FF00"`（緑色）、`DropShadowEffect` でグロー。`IsLoading=True` の `DataTrigger` で **`DoubleAnimation`（Opacity 1.0 ↔ 0.15、0.5 秒、AutoReverse、Forever）** が起動して点滅する。停止時は `StopStoryboard`。

ネットワーク機器の LINK/ACT LED を模した演出で、操作中 / アイドルが視覚的に区別できる。`TextBlock Text="LINK/ACT" FontSize=9 Foreground=#AAAAAA` をラベルとして並置。

### 5-3. ステータスメッセージ

`StatusMessage` プロパティをそのまま表示。Discovery / Parse / Export の各段で逐次更新される（"evidence を読み込み中..." → "読み込み完了: 10 台のPCを検出しました。" → "フォルダ整理中... 5/10 台完了 (50%)" → "出力完了: E:\..." 等）。

## 6. 自動リロード（差分マージ）

`MainWindowViewModel` は **30 秒間隔の `DispatcherTimer`** で `RefreshEvidenceAsync` を駆動する（自動更新トグル ON 時）。通常の `LoadEvidenceCommand` のように `TargetPCs.Clear()` してから再構築する経路ではなく、**既存リストを温存して差分マージ** する：

```
key = $"{PcName}\t{SerialNumber}\t{CollectionDate}"
```

| 状態 | 動作 |
|---|---|
| 既存 key | そのインスタンスのプロパティ（`SerialNumber / 各 DirectoryPath / HasWarning / WarningMessage / MacAddresses / BitLockerKeys`）を上書き。`ObservableObject` の通知で UI が自動更新 |
| 新規 key | `TargetPCs.Add(newPc)` |

これにより `SelectedPc / SearchText / DataGrid のスクロール位置` が維持され、現場で「監査作業中に追加 PC を検出」しても操作が中断されない。`addedCount > 0` なら `StatusMessage = "自動更新: N 台の新規PCを検出しました。"`。

例外は `try/catch` で完全に握りつぶす（自動リロードのエラーは `MessageBox` を出さない、サイレント）。再入防止フラグ `_isRefreshing` と `IsLoading` チェックで手動読み込み中はスキップ。

## 7. SelectedPc 同期と UpdatePreviewInfo

`OnSelectedPcChanged(value)` で `UpdatePreviewInfo(value)` が発火し、サイドパネル系算出プロパティを通知更新する：

- `HasPreview` / `HasDomainStatus` / `HasLocalUsers` / `HasFirewallProfiles` / `HasOptionalFeatures` / ... 等の各セクション存在フラグ
- `PreviewImagePaths` を `auto_capture/` から再収集（`.png/.jpg/.jpeg/.bmp/.gif`、名前順）
- `ChecklistFileName` を `pc.ChecklistFilePath` から再導出
- `UpdateVerificationInfo(pc)` で `CurrentVerificationReport / CurrentChecklistResult / CurrentExportHistory / CurrentBaselineReport` を再計算

これらは MainWindow XAML 側ではほぼ使われない（DataGrid 列での表示のみ）が、PC 詳細ウィンドウ起動時のスナップショットを生成するために裏で計算される。

> 注: 現行 v3.8.0 の MainWindow には**サイドパネルの個別表示はなく**、DataGrid 1 ペインの構成。プレビュー画像と各セクションの可視化は **PC 詳細ウィンドウ側で行う** 設計に統一されている。

## 関連ドキュメント

- PC 詳細ウィンドウ: [fabriq_evidence_manager__apps__02_pc_detail_window.md](fabriq_evidence_manager__apps__02_pc_detail_window.md)
- 設定ダイアログ: [fabriq_evidence_manager__apps__03_settings_window.md](fabriq_evidence_manager__apps__03_settings_window.md)
- 階層構造と DI: [fabriq_evidence_manager__architecture__01_layers.md](fabriq_evidence_manager__architecture__01_layers.md)
- 入力 evidence 構造と Discovery → Parser フロー: [fabriq_evidence_manager__architecture__02_evidence_input.md](fabriq_evidence_manager__architecture__02_evidence_input.md)
- セクション ID dispatch 契約: [fabriq_evidence_manager__contracts__section_dispatch.md](fabriq_evidence_manager__contracts__section_dispatch.md)
