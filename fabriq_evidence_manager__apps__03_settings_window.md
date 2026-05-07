# 設定ダイアログ（SettingsWindow）

> **対象**: fabriq_evidence_manager / apps / 設定ダイアログ
> **対象バージョン**: 3.8.0（取得元: `E:\fabriq_evidence_manager\FabriqEvidenceManager\FabriqEvidenceManager.csproj` `<Version>`）
> **ドキュメント更新日**: 2026-05-07

メインウィンドウ右上の `⚙ 設定` ボタンから起動する modeless ダイアログ。`EvidenceCheckOptions`（取得チェック）と `IBaselineService.EnabledCategoryIds`（ベースライン突合カテゴリ）の 2 軸を変更する。OK/Cancel パターンではなく **変更は即時反映** の方針：チェックボックスを 1 つ切り替えるたびに全 PC が再評価される。

| 項目 | 内容 |
|---|---|
| ViewModel | `SettingsViewModel`（DI 登録なし、`MainWindowViewModel.CreateSettingsViewModel()` ヘルパで構築） |
| View | `Views/SettingsWindow.xaml`（137 行）+ `Views/SettingsWindow.xaml.cs` |
| 起動 | `MainWindow.xaml` の `⚙ 設定` ボタン → `MainWindow.xaml.cs.OnSettingsClick` ハンドラ |
| ウィンドウ仕様 | `Title="設定" / Height=600 Width=500 / WindowStartupLocation=CenterOwner / ShowInTaskbar=False / Background="#D2D6DB"` |
| 確定方式 | 即時反映（CheckBox の `IsChecked` 変更が即 `EvidenceCheckOptions` / `IBaselineService` に流れる）。`閉じる` ボタンは window 解体のみ |

## 起動コード

`MainWindow.xaml.cs.OnSettingsClick`：

```csharp
private void OnSettingsClick(object sender, RoutedEventArgs e)
{
    if (DataContext is not MainWindowViewModel vm) return;
    var settingsVm = vm.CreateSettingsViewModel();
    var settingsWindow = new SettingsWindow(settingsVm) { Owner = this };
    settingsWindow.Show();
}
```

`MainWindowViewModel.CreateSettingsViewModel()` は SettingsViewModel を構築する DI 補助ヘルパで、`EvidenceCheckOptions` の **同一インスタンス参照を共有** し、ベースラインカテゴリ変更後の再評価コールバックを束ねる：

```csharp
public SettingsViewModel CreateSettingsViewModel()
{
    return new SettingsViewModel(
        CheckOptions,                               // ← MainWindowVM と同一参照
        _baselineService,
        onBaselineCategoriesChanged: () =>
        {
            EvaluateCautionsForAllPcs();            // ← 変更直後にフリート再評価
        });
}
```

## レイアウト構成

`<DockPanel>` 内：

- 下端 (`DockPanel.Dock=Bottom`): `閉じる` ボタン
- 上部 (`ScrollViewer + StackPanel`): 注意書き + セクション 1 + セクション 2

```
┌──────────────────────────────────────────────┐
│ 変更は即時反映されます。設定値はアプリ終了で │
│ 破棄されます。                                │
│                                              │
│ ◆ エビデンス取得チェック                      │
│ OFF にした項目は欠損警告(赤行)の対象から外れる│
│ ┌──────────────────────────────────────────┐ │
│ │ □ MACアドレス (有線)                      │ │
│ │ □ MACアドレス (Wi-Fi)                     │ │
│ │ □ BitLocker 回復キー                       │ │
│ │ □ pc_information ディレクトリ              │ │
│ │ □ auto_capture ディレクトリ                │ │
│ │ □ チェックリストファイル                    │ │
│ │ ─────────────────────                  │ │
│ │   ライセンス認証 (§21 / §22)              │ │
│ │ □ Windows ライセンス認証                   │ │
│ │ □ Office インストール                      │ │
│ │ □ Office ライセンス認証                    │ │
│ │ ─────────────────────                  │ │
│ │   hostlist 突合の調整                     │ │
│ │ □ 余剰プリンタ検出                          │ │
│ └──────────────────────────────────────────┘ │
│                                              │
│ ◆ ベースライン突合カテゴリ                    │
│ OFF にしたカテゴリは突合判定でスキップされ、 │
│ 差分件数の集計からも外れます。                │
│ ┌──────────────────────────────────────────┐ │
│ │ □ 実行サマリ                                │ │
│ │ □ SystemInfo                               │ │
│ │ □ チェックリスト                             │ │
│ │ □ インストール済みアプリ                      │ │
│ │ □ ライセンス                                 │ │
│ │ □ ドメイン参加状態                            │ │
│ └──────────────────────────────────────────┘ │
│                                  [ 閉じる ]  │
└──────────────────────────────────────────────┘
```

## セクション 1: エビデンス取得チェック

`EvidenceCheckOptions`（`ObservableObject`）の各 `bool` プロパティに **CheckBox を 2 way バインド**。`MainWindowViewModel` のコンストラクタで `CheckOptions.PropertyChanged += (_,_) => { RevalidateAllPcs(); EvaluateCautionsForAllPcs(); };` というリスナが仕掛けられているため、チェック切替→ プロパティ変更通知→ 全 PC 再検証 + Caution 再評価、が即座に走る。

### 取得チェック項目（基本 6 項目）

| ラベル | プロパティ | 既定 | 警告対象（ON 時） |
|---|---|---|---|
| MACアドレス (有線) | `CheckEthernetMac` | true | `EthernetMac` 空 → "MACアドレス(有線)欠損" |
| MACアドレス (Wi-Fi) | `CheckWifiMac` | true | `WifiMac` 空 → "MACアドレス(Wi-Fi)欠損" |
| BitLocker 回復キー | `CheckBitLocker` | true | `BitLockerKeys.Count == 0` → "BitLockerキー未取得" |
| pc_information ディレクトリ | `CheckPcInformation` | true | `PcInformationDirectoryPath is null` → "pc_information未検出" |
| auto_capture ディレクトリ | `CheckAutoCapture` | true | `AutoCaptureDirectoryPath is null` → "auto_capture未検出" |
| チェックリストファイル | `CheckChecklist` | true | `ChecklistFilePath is null` → "チェックリスト未検出" |

`SerialNumber` 取得不能（"SN取得不能"）は **CheckOption に紐づかず常時警告対象**。これは「シリアル不在は監査の根本不能事象」であり、案件特性で OFF にできない方針。

### ライセンス認証 §21 / §22（区切り線下）

| ラベル | プロパティ | 既定 | 警告条件 |
|---|---|---|---|
| Windows ライセンス認証 | `CheckWindowsLicense` | true | `WindowsLicense is null` → "Windowsライセンス未取得" / `!IsWindowsLicensed` → "Windowsライセンス未認証" |
| Office インストール | `CheckOfficeInstalled` | true | `OfficeLicense is null` → "Office情報未取得" / `!IsOfficeInstalled` → "Office未インストール" |
| Office ライセンス認証 | `CheckOfficeLicense` | true | インストール済 + `OfficeLicenseEvaluation = AuthFailed` → "Officeライセンス認証失敗"（赤）/ `= SignInPending` → "Officeサインイン待ち"（黄、Caution 側） |

「Office を必須としない案件」（PC 提供のみで Office 配備外）では `CheckOfficeInstalled` を OFF にする。「インストールはあるが認証は不要（オフライン環境等）」では `CheckOfficeLicense` を OFF にする。両者は独立に切れる。

`Office ライセンス認証` の `Partial` (= `SignInPending`) は **Caution 側（黄）に流れる** ことに注意（`MainWindowViewModel.EvaluateCaution` 内の "Officeサインイン待ち"）。Warning 側（赤）には「`AuthFailed` のみ」が出る、という赤黄の責務分離。

### hostlist 突合の調整（区切り線下）

| ラベル | プロパティ | 既定 | 効果 |
|---|---|---|---|
| 余剰プリンタ検出 | `CheckExtraPrinters` | **false** | ON 時、hostlist に未登録のプリンタを `EvidenceVerificationService` が「余剰プリンタ」として `Mismatch` に計上（OS デフォルト = Microsoft Print to PDF / XPS / OneNote / Fax は除外） |

既定 OFF。fabriq 側の通常運用では PC が hostlist 外のプリンタを既に持っているケース（メーカ配布ドライバ等）が多く、ON にすると Caution 件数が増えすぎるため。フィールドで「余剰検出が必要な案件」のときだけ ON に切る。

## セクション 2: ベースライン突合カテゴリ

`IBaselineService.AvailableCategories`（6 件）に対する `ItemsControl` 表示。各カテゴリの `IBaselineComparator.IsEnabledByDefault=true` を初期値として `BaselineCategoryOption.IsEnabled` を持ち、チェック ON/OFF を `IBaselineService.SetEnabledCategoryIds(...)` に反映する。

### カテゴリ一覧（6 件）

| 表示名 | CategoryId | 比較対象 |
|---|---|---|
| 実行サマリ | `ExecutionSummary` | `export_history.csv` の `ModuleName × Status` |
| SystemInfo | `SystemInfo` | `01_SystemInfo.txt` の OS名 / OSバージョン / CPU / メモリ |
| チェックリスト | `Checklist` | チェックリスト HTML の `OverallStatus + VerifyItems` |
| インストール済みアプリ | `InstalledApps` | §11 Desktop + Store アプリの `Name × Version` |
| ライセンス | `License` | §21 Windows + §22 Office の license posture（typed model 直比較） |
| ドメイン参加状態 | `DomainStatus` | §05 `DomainStatus`（CurrentUser を除く 7 項目） |

`AvailableCategories` の表示順は `App.xaml.cs` での `IBaselineComparator` 登録順に依存（DI コンテナが `IEnumerable<IBaselineComparator>` を順序保持で解決する）：

```csharp
services.AddSingleton<IBaselineComparator, ExecutionSummaryComparator>();
services.AddSingleton<IBaselineComparator, SystemInfoComparator>();
services.AddSingleton<IBaselineComparator, ChecklistComparator>();
services.AddSingleton<IBaselineComparator, InstalledAppsComparator>();
services.AddSingleton<IBaselineComparator, LicenseComparator>();
services.AddSingleton<IBaselineComparator, DomainStatusComparator>();
```

### 即時反映ロジック

`BaselineCategoryOption` は `ObservableObject` を継承し `IsEnabled` 変更時に `PropertyChanged` を発火。`SettingsViewModel.OnCategoryOptionChanged` がそれを購読：

```csharp
private void OnCategoryOptionChanged(object? sender, PropertyChangedEventArgs e)
{
    if (e.PropertyName != nameof(BaselineCategoryOption.IsEnabled)) return;

    var enabledIds = BaselineCategories.Where(c => c.IsEnabled).Select(c => c.CategoryId);
    _baselineService.SetEnabledCategoryIds(enabledIds);
    _onBaselineCategoriesChanged.Invoke();   // ← MainWindowVM の EvaluateCautionsForAllPcs()
}
```

`SetEnabledCategoryIds` は `BaselineService` 内の `_enabledCategoryIds : HashSet<string>` を差し替える。**キャッシュ済みベースラインデータ自体は破棄しない**：OFF 切替は `CompareAll` で対応 Comparator をスキップするだけで、再 ON 時に `LoadFromPc` をやり直す必要はない（同じ baseline PC を保持し続ける）。

### ベースライン未設定時のヒント

`BaselinePcName is null`（ベースライン未設定）のとき、`<TextBlock>` でヒントを表示：

> ※ ベースラインが未設定でもチェックは保存されます。次回ベースライン設定時に有効になります。

`Visibility` は `IsBaselineLoaded=False` のときのみ Visible（`DataTrigger`）。「カテゴリ ON/OFF を先に設定 → 後で MainWindow で baseline PC を選ぶ」というワークフローを案内する補助テキスト。

## ライフサイクル

| イベント | 動作 |
|---|---|
| 起動 | `MainWindowViewModel.CreateSettingsViewModel()` で 新規 `SettingsViewModel` 作成 → `SettingsWindow` modeless 起動 |
| チェック変更（取得チェック） | `EvidenceCheckOptions.<Property>` 変更 → `MainWindowViewModel` のリスナで `RevalidateAllPcs() + EvaluateCautionsForAllPcs()` |
| チェック変更（ベースラインカテゴリ） | `BaselineCategoryOption.IsEnabled` 変更 → `IBaselineService.SetEnabledCategoryIds` + `onBaselineCategoriesChanged` コールバック → `MainWindowViewModel.EvaluateCautionsForAllPcs()` |
| `閉じる` | `OnCloseClick` で `Close()`。設定値は `EvidenceCheckOptions` / `BaselineService` にメモリ保持されている（**プロセス終了で破棄、永続化しない**） |
| 再起動 | 全設定が既定値（取得チェック ON 多数 / 余剰プリンタ OFF / ベースラインカテゴリ全 ON）に戻る |

設定値の **永続化は v3.8.0 では未実装**。CLAUDE.md `evidence_manager` 想定では `appsettings.json` 等に保存する設計余地があるが、現時点ではアプリ終了で破棄される。「セッション内で必要に応じて切り替える」というユースケースに最適化されている。

## 永続化されない理由（設計判断）

- 案件ごとに必要な取得チェック・ベースラインカテゴリは大きく異なる（Office 不要案件 / 余剰プリンタ検出必要案件 等）
- 永続化すると「前案件の設定が残ったまま新案件を始める」事故が起きやすい
- 1 セッションで完結する短時間運用が前提（フリート 50 台規模を 1 日で検収する用途）

将来案件 profile として保存する機能を追加する場合、CLAUDE.md R4 のカテゴリ語彙には `profiles` が用意されているため、`fabriq_evidence_manager__profiles__settings_profile.md` 等の文書化方針で対応できる。

## 関連ドキュメント

- フリート画面（メイン）: [fabriq_evidence_manager__apps__01_main_window.md](fabriq_evidence_manager__apps__01_main_window.md)
- PC 詳細ウィンドウ: [fabriq_evidence_manager__apps__02_pc_detail_window.md](fabriq_evidence_manager__apps__02_pc_detail_window.md)
- 階層構造と DI: [fabriq_evidence_manager__architecture__01_layers.md](fabriq_evidence_manager__architecture__01_layers.md)
- セクション ID dispatch 契約: [fabriq_evidence_manager__contracts__section_dispatch.md](fabriq_evidence_manager__contracts__section_dispatch.md)
