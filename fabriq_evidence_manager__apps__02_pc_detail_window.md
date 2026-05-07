# PC 詳細ウィンドウ（PcDetailWindow）

> **対象**: fabriq_evidence_manager / apps / PC 個別詳細
> **対象バージョン**: 3.8.0（取得元: `E:\fabriq_evidence_manager\FabriqEvidenceManager\FabriqEvidenceManager.csproj` `<Version>`）
> **ドキュメント更新日**: 2026-05-07

フリート画面（`MainWindow`）の DataGrid 行ダブルクリックで起動する **modeless ウィンドウ**。1 PC = 1 ウィンドウで完結する設計のため、複数の異なる PC を別ウィンドウで並べて並列に比較する用途を許す。サブセクションは取得状況に応じて自動表示 / 非表示が切り替わる（Visibility binding）。

| 項目 | 内容 |
|---|---|
| ViewModel | `PcDetailViewModel`（DI 登録なし、`MainWindow.xaml.cs.OnPcRowDoubleClick` 内で手動 `new`） |
| View | `Views/PcDetailWindow.xaml`（1,897 行）+ `Views/PcDetailWindow.xaml.cs` |
| 起動 | `MainWindow.xaml` の `DataGridRow.Style` `EventSetter Event=MouseDoubleClick Handler=OnPcRowDoubleClick` |
| ウィンドウ仕様 | `Title="{Binding Pc.PcName, StringFormat='PC 詳細: {0}'}" / Height=900 Width=800 / WindowStartupLocation=CenterOwner` |
| レイアウト | 全体が 1 つの `ScrollViewer + StackPanel` で縦に長い構成（Tab / Expander 等は使わない） |

## スナップショット保持と並列比較

`PcDetailViewModel` は **起動時に渡された各種レポートを `init` プロパティとして固定** する。`MainWindowViewModel` の `SelectedPc` 変更や `LoadEvidenceCommand` 再実行に追従しない：

```csharp
public sealed partial class PcDetailViewModel : ObservableObject
{
    public PcEvidence Pc { get; }                                // 参照
    public EvidenceCheckOptions CheckOptions { get; }            // 参照（共有）
    public bool IsHostlistLoaded { get; }                        // 起動時値
    public bool IsBaselineLoaded { get; }                        // 起動時値
    public VerificationReport? VerificationReport { get; }       // 起動時スナップショット
    public ChecklistResult? ChecklistResult { get; }
    public IReadOnlyList<ExportHistoryEntry>? ExportHistory { get; }
    public BaselineComparisonReport? BaselineReport { get; }
}
```

これにより **PC A の詳細ウィンドウを開いたまま MainWindow で PC B を選んで詳細を開く** と、2 つのウィンドウが**それぞれ別の PC のスナップショットを持って並ぶ**。後勝ちで上書きされない。`PcDetailWindow.xaml.cs` の `OnSourceInitializedClampSize` / `OnLoadedClampPosition` でウィンドウ位置・サイズを `WorkArea` 内にクランプするのも、複数ウィンドウが画面端にあふれて操作不能にならないための配慮。

ただし `Pc : PcEvidence` はインスタンス参照のため、自動リロードで `MainWindowViewModel.RefreshEvidenceAsync` が同 PC のフィールドを更新すると **詳細ウィンドウ側にも即座に反映** される（`ObservableObject` 経由）。「**スナップショットなのは付帯レポート（VerificationReport 等）であり、本体 PcEvidence は live 参照**」という非対称性に注意。

## ウィンドウサイズ / 位置のクランプ

XAML 既定 `Height=900` が画面 `WorkArea` より大きい低解像度環境（高 DPI ノート等）で `WindowStartupLocation=CenterOwner` の中心計算が画面外に出るのを防ぐため、`PcDetailWindow.xaml.cs` で 2 段クランプ：

| イベント | 処理 |
|---|---|
| `SourceInitialized` | `Width / Height` を `WorkArea.Width / Height` 以内にクランプ（HWND 作成直後で CenterOwner 配置の前） |
| `Loaded` | `Top / Left` を `WorkArea` 内に強制クランプ（CenterOwner 配置後の最終位置がそれでも画面外なら補正、タイトルバー欠落でドラッグ不能になる症状の防止） |

## 全体構造

`<ScrollViewer><StackPanel>` 直下に **32 セクション** が縦に並ぶ。各セクションは独立した `TextBlock` 見出し + 内容ブロックで構成され、`Visibility={Binding HasXxx, Converter=BoolToVis}` で取得済みのものだけ表示される。

セクション見出しは色分けされた小さなラベル `FontSize=10 / FontWeight=Bold / Foreground=#757B82` で、1 つのウィンドウ内に 30+ のラベルが連続的に並ぶ「縦長の検査表」スタイル。

### セクション一覧（出現順）

| # | XAML コメント | 見出し | 表示条件 | 表示内容 |
|---|---|---|---|---|
| 1 | (固定) | PC 名ヘッダー / 実 PC 名 / シリアル | 常時 | `PcName` 16pt + `ActualComputerNameDisplay` + `SerialNumberDisplay`（フォールバック時オレンジ） |
| 2 | 管理者メモ | `◆ 管理者メモ` | 常時 | `MemoText` TextBox（multi-line）+ `💾 保存` ボタン + `MemoLastUpdatedDisplay`。詳細は §「管理者メモ」 |
| 3 | キャプチャプレビュー | `AUTO CAPTURE` | 常時（画像なしは "No Image"） | `Image Source="{Binding CurrentPreviewImagePath}"` + `< 前へ / 1 / 10 / 次へ >` ナビ |
| 4 | エビデンス詳細 | `EVIDENCE DETAILS` | 常時 | 有線 MAC / Wi-Fi MAC / BitLocker（ドライブ別 ID + 回復キー）/ Checklist ファイル名 |
| 4-w | (Border) | (赤背景) `WarningMessage` | `HasWarning` | 欠損警告メッセージ（カンマ区切り、ラップ） |
| 4-c | (Border) | (黄背景) `CautionMessage` | `HasCaution` | 要確認メッセージ（カンマ区切り、ラップ） |
| 5 | ドメイン参加状態 | `DOMAIN STATUS` | `HasDomainStatus` | ドメイン所属 / ドメイン / DomainRole / 現在のユーザー + AzureAD 参加 / ドメイン参加 / ドメイン名 / テナント名 |
| 6 | ローカルユーザー | `LOCAL USERS` | `HasLocalUsers` | DataGrid（ユーザー名 / 有効 / フルネーム / 説明 / ソース） |
| 7 | ローカルグループ | `LOCAL GROUPS` | `HasLocalGroups` | DataGrid（グループ名 / 説明） |
| 8 | グループメンバー | `GROUP MEMBERS` | `HasLocalGroupMembers` | DataGrid（グループ名 / メンバー名 / 種別 / ソース） |
| 9 | FW プロファイル | `FIREWALL PROFILES` | `HasFirewallProfiles` | DataGrid（プロファイル / 有効 / 受信 / 送信 / ログファイル） |
| 10 | FW ルール | `FIREWALL RULES` | `HasFirewallRules` | DataGrid（ルール名 / 有効 / 方向 / アクション / プロファイル） |
| 11 | オプション機能 | `OPTIONAL FEATURES` | `HasOptionalFeatures` | DataGrid（機能名 / 状態） |
| 12 | ユーザープロファイル | `USER PROFILES` | `HasUserProfiles` | DataGrid（パス / SID / 最終使用 / 読込済） |
| 13 | ディスク | `DISKS` | `HasDisks` | DataGrid（# / 名前 / SN / 容量 GB / 形式 / 状態） |
| 14 | パーティション | `PARTITIONS` | `HasPartitions` | DataGrid（Disk# / Part# / ドライブ / 容量 GB / 種別 / System / Boot） |
| 15 | 電源設定 | `POWER SETTINGS  アクティブプラン: ○○` | `HasPowerSettings` | アクティブプラン名見出し + raw text TextBox（読み取り専用、Consolas 10pt） |
| 16 | WiFi プロファイル | `WIFI PROFILES (N)` | `HasWiFiProfiles` | ItemsControl（SSID 名のリスト） |
| 17 | 復元ポイント | `RESTORE POINTS (N)` | `HasRestorePoints` | DataGrid（# / 説明 / 種別 / 作成日時） |
| 18 | Windows Defender | `WINDOWS DEFENDER` | `HasDefenderStatus` | Antivirus / RealTime / Antispyware / NIS の `True/False` 表示 + エンジン / 製品 / 定義 / 定義更新日 |
| 19 | Windows Update | `WINDOWS UPDATES (N)` | `HasWindowsUpdates` | DataGrid（KB / 種別 / インストール者 / 日時） |
| 20 | メモリインベントリ §29 | `MEMORY INVENTORY` | `HasMemoryInventory` | 装着総容量 / マザーボード上限 / スロット使用 + DIMM 個別 DataGrid + アレイサマリ DataGrid |
| 21 | ハードウェア識別子 §31 | `HARDWARE IDENTIFIERS` | `HasHardwareIdentifiers` | ComputerSystem / CSP / BaseBoard / SystemEnclosure の 4 ブロック（取得失敗ブロックは個別非表示） |
| 22 | 環境変数 §27 | `ENVIRONMENT VARIABLES (M / U)` | `HasEnvironmentVariables` | DataGrid（Scope / Name / Value）+ Machine/User 件数表示 |
| 23 | 自動起動項目 §28 | `STARTUP ITEMS (R / T)` | `HasStartupItems` | DataGrid（Source / Name / User / Location / Command / 有効）+ Registry/Task 件数表示 |
| 24 | PnP デバイス §30 | `PNP DEVICES (T / P / D)` | `HasPnpDevices` | DataGrid（Class / FriendlyName / Status / 接続 / Manufacturer / DriverVersion / DriverDate / InstanceId）+ Total/Present/Disconnected 件数 |
| 25 | グループポリシー §24 | `GROUP POLICY REPORT` | `HasGroupPolicyReport` | gpresult exit code / Domain / ExecutingUser + `OpenGroupPolicyHtml` ボタン（HTML を OS 既定ブラウザで開く） |
| 26 | HOSTLIST 突合結果 | `HOSTLIST VERIFICATION` | (起動時 hostlist 読込済 + レポート生成済) | 各項目: ItemName / Expected / Actual / Status（Match=緑 / Mismatch=赤 / NoActual=オレンジ / NoExpected=グレー） |
| 27 | チェックリスト結果 | `CHECKLIST` | `ChecklistResult is not null` | OverallStatus + VerifyItems（PASS/FAIL）+ ModuleItems |
| 28 | 実行履歴 | `EXECUTION HISTORY` | `ExportHistory is not null` | DataGrid（Timestamp / ModuleName / Category / Status / Message） |
| 29 | ベースライン §SystemInfo | `BASELINE — SYSTEM INFO` | `BaselineReport.SystemInfoComparison is not null` | 4 項目: OS名 / OSバージョン / CPU / メモリ |
| 30 | ベースライン §Checklist | `BASELINE — CHECKLIST` | `BaselineReport.ChecklistComparison is not null` | OverallStatus 期待値/実測値 + 各 VerifyItem の比較 |
| 31 | ベースライン §InstalledApps | `BASELINE — INSTALLED APPS` | `BaselineReport.InstalledAppsComparison is not null` | 一致件数 / 差分項目（Mismatch / NoActual / NoExpected） |
| 32 | ベースライン §DomainStatus | `BASELINE — DOMAIN STATUS` | `BaselineReport.DomainStatusComparison is not null` | 7 項目: ドメイン参加 / ドメイン名 / ドメインロール / Azure AD 参加 / AD ドメイン参加 / AD ドメイン名 / Azure AD テナント |
| 33 | ベースライン §License | `BASELINE — LICENSE` | `BaselineReport.LicenseComparison is not null` | Windows: ファミリ / チャネル / 状態 / KMS Machine + Office: インストール / C2R Release / Channel / 状態 |
| 34 | ベースライン §ExecutionSummary | `BASELINE — EXECUTION SUMMARY` | `BaselineReport.Items.Count > 0` | モジュール ×（baseline / actual / 一致 / 不一致 / Missing / Extra） |

> **§21 / §22 / §23 / §25 / §26 のサブセクションは PcDetailWindow に表示されない**：Windows License / Office License / Security Baseline / Certificates / Battery は本ウィンドウでは「ヘッダの SerialNumber 周りに認証状態が出る」「警告メッセージとしてラップされる」のみで、構造化フィールドの完全表示は **Excel 台帳**（`pc_details/` サブブック）が担う。これは「画面で見るならフリート視点 + サマリ」「監査表として読むなら Excel」という役割分担。

## 1. 管理者メモ（manager_memo.json）

PC ルートディレクトリ直下の `manager_memo.json` に保存される自由記述メモ。fabriq 側 evidence ツリーには触らず co-located で配置する設計（CLAUDE.md R1 ソース不変）。

```
{evidenceRoot}/{timestamp}_{PCName}_{Serial}_evidence/
├── evidence/                    ← fabriq が出力（read-only）
└── manager_memo.json            ← 本アプリが書く唯一のファイル
```

### UI 構成

| 要素 | 役割 |
|---|---|
| `◆ 管理者メモ` ラベル | 強調表示（FontSize=11 Bold） |
| `💾 保存` ボタン | `SaveMemoCommand`（`CanExecute = Pc.PcRootDirectoryPath is not null`） |
| `MemoText` TextBox | `AcceptsReturn=True / TextWrapping=Wrap / MinHeight=60 MaxHeight=120` |
| `MemoLastUpdatedDisplay` | `"最終更新: 2026/05/07 14:32 (suzuki)"` または `"(未保存)"` |

### 保存処理

```csharp
private void SaveMemo()
{
    if (Pc.PcRootDirectoryPath is null) return;
    var memo = new PcMemo
    {
        Text = MemoText,
        LastUpdatedAt = DateTimeOffset.Now,
        LastUpdatedBy = Environment.UserName,
    };
    _memoService.Save(Pc.PcRootDirectoryPath, memo);
    Pc.Memo = memo;
    OnPropertyChanged(nameof(MemoLastUpdatedDisplay));
}
```

`IPcMemoService.Save` は `JsonSerializer.Serialize` で indented + camelCase 書き出し。失敗時は例外を呼び出し側に投げる（UI でエラーメッセージ表示は未実装、将来課題）。

`MainWindow` 側の DataGrid 行に `📝` アイコンが表示される条件（`HasMemo = Memo is not null && !string.IsNullOrEmpty(Memo.Text)`）が `Pc.Memo` 更新で即座に再評価される。

## 2. AUTO CAPTURE プレビュー

`auto_capture/` 配下の画像ファイル（`.png/.jpg/.jpeg/.bmp/.gif`、名前順）を順次表示するナビゲータ。

| 要素 | バインド |
|---|---|
| `Image Source` | `CurrentPreviewImagePath`（`PreviewImagePaths[CurrentPreviewIndex]`） |
| プレースホルダ | 画像 null 時 `No Image` を表示 |
| `< 前へ` ボタン | `PreviousPreviewImageCommand`（`CanExecute = CurrentPreviewIndex > 0`） |
| `次へ >` ボタン | `NextPreviewImageCommand`（`CanExecute = CurrentPreviewIndex < PreviewImagePaths.Count - 1`） |
| `PreviewCounterText` | `"3 / 10"` 形式 |
| ナビ全体 | `Visibility={Binding HasMultipleImages}`（複数あるときのみ） |

`PcDetailViewModel.LoadPreviewImages()` がコンストラクタで `Pc.AutoCaptureDirectoryPath` を `Directory.EnumerateFiles` で走査し、対応拡張子の画像をソート済みでロード。本ウィンドウは modeless かつスナップショット保持なので、後から auto_capture/ にファイル追加してもこのウィンドウのリストには反映されない（再起動で取得し直す前提）。

## 3. 警告 / 要確認メッセージ

`HasWarning=True` のとき赤背景 (`#FFEBEE`) + 赤枠の `Border` で `WarningMessage` を表示。`HasCaution=True` のとき黄背景 (`#FFF8E1`) + オレンジ枠で `CautionMessage` を表示。

両者ともカンマ区切りメッセージで、ラップ表示。検収判定の根拠を一目で識別できるよう EVIDENCE DETAILS の直下に配置されている。詳細な判定軸は `EvidenceParserService.ValidateEvidence`（赤）と `MainWindowViewModel.EvaluateCaution`（黄）が決める（[fabriq_evidence_manager__architecture__01_layers.md](fabriq_evidence_manager__architecture__01_layers.md) §「Services 層」 §「突合・比較サービス」を参照）。

## 4. ドメイン参加状態セクション

§05 `DomainStatusData` の主要 9 フィールドを「ヘッダーセクション」（`PartOfDomain` / `Domain` / `DomainRole` / `CurrentUser`）と「dsregcmd セクション」（`AzureAdJoined` / `DomainJoined` / `DomainName` / `TenantName`）に分けて縦リスト表示。

`PartOfDomain` / `AzureAdJoined` / `DomainJoined` の `True/False` は **`DataTrigger` で文字色 + 表示文字を切り替え**：

| 値 | テキスト | 色 |
|---|---|---|
| `True` | `YES` | `#2E7D32`（緑、Bold） |
| `False`（既定） | `NO` | `#666666`（グレー） |

`TenantName` は `Pc.DomainStatus.AzureAdJoined=True` のときのみ Visibility 表示（Azure AD 非参加機ではテナント名行自体を隠す）。

## 5. メモリインベントリ §29 セクション

3 ブロック構成：

```
[上段サマリ]
  装着総容量(GB):  64       マザーボード上限(GB):  128
  スロット使用:    2/4

[DIMM 個別 DataGrid]
  Bank | DeviceLocator | GB | Type | Form | MHz | メーカ | 型番 | SN

[アレイサマリ DataGrid]
  Tag | Loc | Use | ECC | MaxGB | スロット数
```

上段は `MemoryInstalledTotalGB` / `MemoryMaxCapacityGB` / `MemorySlotUsageDisplay` の算出プロパティ（`PcDetailViewModel`）を表示。装着 DIMM の `CapacityGB` 合計と `MemoryArraySummaries[].MaxCapacityGB` 合計を比較することで、増設余地が一目で分かる。`MemorySlotUsageDisplay` は `"2/4"`（装着数 / アレイの総スロット数、アレイ情報無時 `"2/-"`）形式。

## 6. ハードウェア識別子 §31 セクション

4 ブロック（`Win32_ComputerSystem` / `Win32_ComputerSystemProduct` / `Win32_BaseBoard` / `Win32_SystemEnclosure`）を縦に並べ、**probe 個別失敗時はそのブロックのみ Visibility=Collapsed** で隠す（`HasHwComputerSystem` / `HasHwComputerSystemProduct` / `HasHwBaseBoard` / `HasHwSystemEnclosure` の 4 フラグで個別判定）。

これは fabriq 側 evidence_config §31 が **inner try/catch で 4 probe を独立退避** する仕様（取得失敗ブロックは対応プロパティが null になる）に整合した設計で、「§31 セクション自体は Success だが Win32_BaseBoard だけ取れていない」というケースで実害ある情報のみ表示する。

## 7. グループポリシー §24 セクション

gpresult `/h` の HTML（240KB 規模）は **メモリに読み込まず**、`HtmlFilePath` 絶対パスのみ `GroupPolicyReport` に保持する設計。本セクションでは：

- `ExecutionStatus`（exit code、0=成功）/ `Domain` / `ExecutingUser` を表示
- `GroupPolicyStatusDisplay` プロパティが `"OK"` / `"Failed (code N)"` / `"(取得不能)"` を返す
- `OpenGroupPolicyHtmlCommand`（`CanExecute = CanOpenGroupPolicyHtml`、HTML 実在チェック）で `Process.Start(new ProcessStartInfo { FileName = path, UseShellExecute = true })` を発火し OS 既定ブラウザで HTML を開く

`UseShellExecute=true` により `.html` 拡張子の関連付けで Edge / Chrome 等が起動する。Excel 出力には HTML hyperlink を入れない（delivery folder 相対パス計算が複雑なため、後続フェーズに保留）。

## 8. ベースライン突合（6 サブセクション）

`BaselineReport`（`BaselineComparisonReport`）の 6 つのサブフィールドを 6 つのセクションに展開：

| セクション | データソース | 表示内容 |
|---|---|---|
| `BASELINE — SYSTEM INFO` | `SystemInfoComparison` | 4 項目（OS名 / OSバージョン / CPU / メモリ）の Expected/Actual/Status |
| `BASELINE — CHECKLIST` | `ChecklistComparison` | OverallStatus の期待値/実測値 + 各 VerifyItem の比較 |
| `BASELINE — INSTALLED APPS` | `InstalledAppsComparison` | 一致件数 / 不一致 / Missing / Extra（差分のみ表示、一致は集計のみ） |
| `BASELINE — DOMAIN STATUS` | `DomainStatusComparison` | 7 項目（ドメイン参加 / ドメイン名 / ロール / Azure AD 参加 / AD 参加 / AD 名 / テナント名） |
| `BASELINE — LICENSE` | `LicenseComparison` | Windows 4 + Office 4 = 最大 8 項目（typed model 直比較） |
| `BASELINE — EXECUTION SUMMARY` | `Items` | モジュール × Status の対比リスト（MatchStatus = Match / Mismatch / MissingInActual / ExtraInActual） |

各サブセクションは「全項目が Match のとき緑色のサマリ行（"○○ すべて一致"）を冒頭表示」する仕様で、不一致時は項目別 DataTemplate で一覧化。

詳細な比較ロジックは `Services/Baseline/*Comparator.cs` の各実装に閉じている（[fabriq_evidence_manager__architecture__01_layers.md](fabriq_evidence_manager__architecture__01_layers.md) §「Baseline プラグインチェーン」）。

## 関連ドキュメント

- フリート画面（メイン）: [fabriq_evidence_manager__apps__01_main_window.md](fabriq_evidence_manager__apps__01_main_window.md)
- 設定ダイアログ: [fabriq_evidence_manager__apps__03_settings_window.md](fabriq_evidence_manager__apps__03_settings_window.md)
- 階層構造と DI: [fabriq_evidence_manager__architecture__01_layers.md](fabriq_evidence_manager__architecture__01_layers.md)
- 入力 evidence 構造: [fabriq_evidence_manager__architecture__02_evidence_input.md](fabriq_evidence_manager__architecture__02_evidence_input.md)
- セクション ID dispatch 契約: [fabriq_evidence_manager__contracts__section_dispatch.md](fabriq_evidence_manager__contracts__section_dispatch.md)
