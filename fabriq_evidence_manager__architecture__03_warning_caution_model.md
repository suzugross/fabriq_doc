# Warning / Caution 2 軸判定モデル

> **対象**: fabriq_evidence_manager / architecture
> **対象バージョン**: 3.8.0（取得元: `E:\fabriq_evidence_manager\FabriqEvidenceManager\FabriqEvidenceManager.csproj` `<Version>`）
> **ドキュメント更新日**: 2026-05-07

本アプリは **PC ごとの異常を 2 つの独立した軸（Warning = 赤 / Caution = 黄）で評価** する。両者は判定ロジック・トリガ・責務分担すべて分離されており、同時に立つこと（赤 + 黄）も独立にあり得る。本ドキュメントはその設計上の根拠と境界線を明文化する。

---

## なぜ 2 軸か

「**取得失敗 / 物理欠損**」と「**取得はできたが期待値と違う / 要確認**」は **検収作業者が打つべき次の手が違う** ため、画面上で同じ赤色で並べると判断ノイズになる。

| 軸 | 性質 | 例 | 担当 Service | 反映色 |
|---|---|---|---|---|
| **Warning** | 取得欠損・取得失敗 | MAC 取れていない / BitLocker キー不在 / Office 認証失敗 / 主要ディレクトリ無 | `EvidenceParserService.ValidateEvidence` | 赤 (`#FFCDD2`) |
| **Caution** | 期待値ズレ・要確認 | hostlist 不一致 / ベースライン差異 / チェックリスト NG / Office サインイン待ち / SN フォールバック / PC 名不一致 | `MainWindowViewModel.EvaluateCaution` | 黄 (`#FFF9C4`) |

「**Warning は再収集 / 物理対処**」「**Caution は人間が判断・許容判断・案件管理**」という対応の違いを色で素早く識別できることが目的。

---

## ハイライトの優先順位

DataGrid の `RowStyle` `DataTrigger` の評価順により、**Warning が Caution を上書き** する：

```xml
<!-- MainWindow.xaml -->
<Style.Triggers>
    <DataTrigger Binding="{Binding HasCaution}" Value="True">
        <Setter Property="Background" Value="#FFF9C4"/>   <!-- 先に評価される -->
    </DataTrigger>
    <DataTrigger Binding="{Binding HasWarning}" Value="True">
        <Setter Property="Background" Value="#FFCDD2"/>   <!-- 後勝ち、赤が見える -->
    </DataTrigger>
    <MultiTrigger>
        <MultiTrigger.Conditions>
            <Condition Property="IsMouseOver" Value="True"/>
        </MultiTrigger.Conditions>
        <Setter Property="Background" Value="#D2D4D9"/>
    </MultiTrigger>
</Style.Triggers>
```

- 両方 True の PC → 赤
- Warning のみ → 赤
- Caution のみ → 黄
- 両方 False → 透明

ただし**ステータス列（PC 一覧右端）には両方のメッセージが縦並びで表示** される（赤行に黄メッセージも見える）：

```xml
<DataGridTemplateColumn Header="ステータス" Width="320">
    <StackPanel Orientation="Vertical">
        <TextBlock Foreground="#C62828"
                   Text="{Binding WarningMessage}"
                   Visibility="{Binding HasWarning, Converter={StaticResource BoolToVis}}"/>
        <TextBlock Foreground="#E65100"
                   Text="{Binding CautionMessage}"
                   Visibility="{Binding HasCaution, Converter={StaticResource BoolToVis}}"/>
    </StackPanel>
</DataGridTemplateColumn>
```

**色 = 重要度の最大値（赤勝ち）/ メッセージ = 完全並列** という設計。検収作業者は「色で目を引く → メッセージで何件・何故か把握」の 2 段視認をする。

---

## Warning（赤）の判定軸

### 担当: `EvidenceParserService.ValidateEvidence(pc, options)`

`PopulateDetails` の最後（初回）と `RevalidateEvidence`（CheckOptions 変更時）で呼ばれる。`EvidenceCheckOptions` の各 `bool` プロパティを見て、ON のチェック項目だけを判定対象にする。

```csharp
private static void ValidateEvidence(PcEvidence pc, EvidenceCheckOptions? options)
{
    var warnings = new List<string>();

    if (options?.CheckEthernetMac != false && string.IsNullOrEmpty(pc.EthernetMac))
        warnings.Add("MACアドレス(有線)欠損");
    if (options?.CheckWifiMac != false && string.IsNullOrEmpty(pc.WifiMac))
        warnings.Add("MACアドレス(Wi-Fi)欠損");
    if (options?.CheckBitLocker != false && pc.BitLockerKeys.Count == 0)
        warnings.Add("BitLockerキー未取得");
    if (string.IsNullOrEmpty(pc.SerialNumber))
        warnings.Add("SN取得不能");                     // ← CheckOption 無し、常時判定
    if (options?.CheckPcInformation != false && pc.PcInformationDirectoryPath is null)
        warnings.Add("pc_information未検出");
    if (options?.CheckAutoCapture != false && pc.AutoCaptureDirectoryPath is null)
        warnings.Add("auto_capture未検出");
    if (options?.CheckChecklist != false && pc.ChecklistFilePath is null)
        warnings.Add("チェックリスト未検出");
    if (options?.CheckWindowsLicense != false)
    {
        if (pc.WindowsLicense is null) warnings.Add("Windowsライセンス未取得");
        else if (!pc.IsWindowsLicensed) warnings.Add("Windowsライセンス未認証");
    }
    if (options?.CheckOfficeInstalled != false)
    {
        if (pc.OfficeLicense is null) warnings.Add("Office情報未取得");
        else if (!pc.OfficeLicense.IsOfficeInstalled) warnings.Add("Office未インストール");
    }
    if (options?.CheckOfficeLicense != false
        && pc.OfficeLicense is { IsOfficeInstalled: true }
        && pc.OfficeLicenseEvaluation == OfficeLicenseEvaluation.AuthFailed)
    {
        warnings.Add("Officeライセンス認証失敗");
    }

    pc.HasWarning = warnings.Count > 0;
    pc.WarningMessage = string.Join(", ", warnings);
}
```

### Warning に立つ条件（ON 時の場合）

| 条件 | メッセージ | CheckOption | 既定 |
|---|---|---|---|
| `EthernetMac` 空 | `MACアドレス(有線)欠損` | `CheckEthernetMac` | true |
| `WifiMac` 空 | `MACアドレス(Wi-Fi)欠損` | `CheckWifiMac` | true |
| `BitLockerKeys.Count == 0` | `BitLockerキー未取得` | `CheckBitLocker` | true |
| `SerialNumber` 空 | `SN取得不能` | **(常時判定)** | — |
| `PcInformationDirectoryPath` null | `pc_information未検出` | `CheckPcInformation` | true |
| `AutoCaptureDirectoryPath` null | `auto_capture未検出` | `CheckAutoCapture` | true |
| `ChecklistFilePath` null | `チェックリスト未検出` | `CheckChecklist` | true |
| `WindowsLicense` null | `Windowsライセンス未取得` | `CheckWindowsLicense` | true |
| `!IsWindowsLicensed` | `Windowsライセンス未認証` | `CheckWindowsLicense` | true |
| `OfficeLicense` null | `Office情報未取得` | `CheckOfficeInstalled` | true |
| `!IsOfficeInstalled` | `Office未インストール` | `CheckOfficeInstalled` | true |
| `OfficeLicenseEvaluation = AuthFailed` | `Officeライセンス認証失敗` | `CheckOfficeLicense` | true |

`SerialNumber` 取得不能は **CheckOption に紐づかず常時警告**。「シリアル不在は監査の根本不能事象」であり、案件特性で OFF にできない方針。

### `OfficeLicenseEvaluation = AuthFailed` の判定

`PcEvidence.OfficeLicenseEvaluation` 算出プロパティが返す値は v1.5+ INTERPRETATION 経路と v1.4 旧 OSPP 経路で分岐する。詳細は [fabriq_evidence_manager__reference__office_license_evaluation.md](fabriq_evidence_manager__reference__office_license_evaluation.md)。

ここでは **`AuthFailed` のみが Warning（赤）に流れる** ことが重要。`SignInPending`（v1.5 manifest §22 verdict=Partial）は Caution 側に分離している。

---

## Caution（黄）の判定軸

### 担当: `MainWindowViewModel.EvaluateCaution(pc)`

`MainWindowViewModel.EvaluateCautionsForAllPcs()` が全 PC に対して呼ぶ。トリガは：

| トリガ | 経路 |
|---|---|
| `LoadEvidenceCommand` 完了時 | 取り込み直後 |
| `LoadHostlistCommand` 成功時 | hostlist 読み込み後 |
| `SetBaselinePcCommand` / `ClearBaselineCommand` | baseline 切替後 |
| `EvidenceCheckOptions.PropertyChanged` | 設定変更時（`MainWindowViewModel` のリスナ経由） |
| `SettingsViewModel.onBaselineCategoriesChanged` | ベースラインカテゴリ ON/OFF 変更時 |
| 自動更新タイマ完了時 | （差分マージで増えた PC のみ追加判定） |

```csharp
private void EvaluateCaution(PcEvidence pc)
{
    var cautions = new List<string>();

    if (pc.HasPcNameMismatch)
        cautions.Add($"PC名不一致(計画={pc.PcName} / 実={pc.ActualComputerName})");

    if (pc.HasSerialFallbackWarning)
        cautions.Add($"SNフォールバック(source={SerialSourceEntry.ShortLabel(pc.SerialNumberSource)})");

    if (_hostlistService.IsLoaded)
    {
        try
        {
            var report = _verificationService.Verify(pc, CheckOptions);
            if (report.HostlistFound && report.MismatchCount > 0)
                cautions.Add($"hostlist不一致({report.MismatchCount}件)");
        }
        catch { /* 判定失敗時はスキップ */ }
    }

    if (pc.ChecklistFilePath is not null && File.Exists(pc.ChecklistFilePath))
    {
        try
        {
            var checklist = _checklistParserService.ParseChecklistHtml(pc.ChecklistFilePath);
            if (checklist.NgCount > 0) cautions.Add($"チェックリストNG({checklist.NgCount}件)");
        }
        catch { /* 判定失敗時はスキップ */ }
    }

    if (CheckOptions.CheckOfficeLicense
        && pc.OfficeLicenseEvaluation == OfficeLicenseEvaluation.SignInPending)
    {
        cautions.Add("Officeサインイン待ち");
    }

    if (_baselineService.IsLoaded)
    {
        try
        {
            var baseline = _baselineService.CompareAll(pc);
            if (baseline.DifferenceCount > 0)
                cautions.Add($"ベースライン差異({baseline.DifferenceCount}件)");
        }
        catch { /* 判定失敗時はスキップ */ }
    }

    pc.HasCaution = cautions.Count > 0;
    pc.CautionMessage = string.Join(", ", cautions);
}
```

### Caution に立つ条件

| 条件 | メッセージ | 必要状態 |
|---|---|---|
| `HasPcNameMismatch` | `PC名不一致(計画=X / 実=Y)` | manifest あり、`PcName != ComputerName` |
| `HasSerialFallbackWarning` | `SNフォールバック(source=CSProduct/Enclosure/...)` | SN は取れたが Primary canonical 以外 |
| `report.MismatchCount > 0` | `hostlist不一致(N件)` | hostlist 読込済み + 該当 PC が hostlist にあり、Mismatch + NoActual の合計 > 0 |
| `checklist.NgCount > 0` | `チェックリストNG(N件)` | チェックリスト HTML あり、Status が NG/Error/Fail のモジュール件数 |
| `OfficeLicenseEvaluation = SignInPending` | `Officeサインイン待ち` | `CheckOfficeLicense=ON` かつ verdict=Partial（v1.5+）|
| `baseline.DifferenceCount > 0` | `ベースライン差異(N件)` | ベースライン PC 設定済み、6 カテゴリ全合計 |

**全条件は AND ではなく OR**。複数該当する PC は `, ` 区切りで全部並ぶ：

```
PC名不一致(計画=NEW-PC-01 / 実=NEW-PC-001), hostlist不一致(2件), ベースライン差異(5件)
```

---

## 責務分離の境界線

### 担当層

| 軸 | 担当層 | 理由 |
|---|---|---|
| Warning | `EvidenceParserService`（Service 層） | パースの最後の段で「取れたか/取れていないか」が確定する → そこで赤判定する |
| Caution | `MainWindowViewModel`（VM 層） | hostlist / baseline / chexklist は **複数 Service の組合せ判定** が必要、単一 Service の責務を超える |

`Caution` が Service 層に下りていない理由は「**`HostlistService` も `BaselineService` も `ChecklistParserService` も**フリート他 PC や案件設定（hostlist 読込済か等）に依存する判定」のため。`PcEvidence` 単体の純粋関数では決まらず、ViewModel が状態を集約する位置が自然。

### Office License の振り分け

Office に関する判定は **3 つの状態に分岐し、それぞれ違う軸に流れる**：

| `OfficeLicenseEvaluation` | 流れる軸 | 設定スイッチ |
|---|---|---|
| `NotEvaluated` （manifest §22 不在） | Warning（`CheckOfficeInstalled=ON` で `Office情報未取得`） | `CheckOfficeInstalled` |
| `NotInstalled` （C2R/OSPP どちらも未検出） | Warning（`CheckOfficeInstalled=ON` で `Office未インストール`） | `CheckOfficeInstalled` |
| `Licensed` | どちらにも立たない（正常） | — |
| `SignInPending` （v1.5 manifest §22 verdict=Partial） | **Caution**（`CheckOfficeLicense=ON` で `Officeサインイン待ち`） | `CheckOfficeLicense` |
| `AuthFailed` （v1.5 verdict=Failed / v1.4 旧 OSPP IsLicensed=false） | Warning（`CheckOfficeLicense=ON` で `Officeライセンス認証失敗`） | `CheckOfficeLicense` |

**`Partial` を Caution（黄）に分離した設計判断**：

- M365 サブスクリプション機で「OS イメージ展開直後でユーザーがまだサインインしていない」状態は **真の認証失敗ではない**（24h 以内にサインインすれば自動 provision される）
- 検収作業者は黄色を見て「明日もう一度確認」「ユーザー側のサインイン手順案内」等の運用判断を打つ
- 一方 `Failed` は「VL ライセンスが切れている / KMS 不通 / プロダクトキー消失」等の **物理対処が必要な事象**

---

## EvidenceCheckOptions のリスナ機構

`EvidenceCheckOptions` は `ObservableObject` で、各 `bool` プロパティが `[ObservableProperty]` で notification 対応。`MainWindowViewModel` のコンストラクタで購読：

```csharp
CheckOptions.PropertyChanged += (_, _) =>
{
    RevalidateAllPcs();           // 全 PC で ValidateEvidence 再実行 → Warning 更新
    EvaluateCautionsForAllPcs();  // 全 PC で EvaluateCaution 再実行 → Caution 更新
};
```

設定ダイアログでチェックを 1 つ変えると **両方の軸が同時に再評価される**。Warning 系のチェック（`CheckEthernetMac` 等）は `RevalidateAllPcs` 経由でのみ効くが、`CheckOfficeLicense` は両 ValidateEvidence と EvaluateCaution の両方で参照されているため両方に効く。

---

## ベースラインカテゴリ ON/OFF の Caution 反映

`SettingsViewModel.OnCategoryOptionChanged` がベースラインカテゴリの `IsEnabled` 変更で発火するコールバック：

```csharp
private void OnCategoryOptionChanged(...)
{
    var enabledIds = BaselineCategories.Where(c => c.IsEnabled).Select(c => c.CategoryId);
    _baselineService.SetEnabledCategoryIds(enabledIds);
    _onBaselineCategoriesChanged.Invoke();   // ← MainWindowVM の EvaluateCautionsForAllPcs()
}
```

`BaselineService.CompareAll(pc)` が `_enabledCategoryIds` 集合を見て Comparator をスキップ／実行するので、カテゴリ OFF にすると **そのカテゴリ起因の差分が `DifferenceCount` から消え、Caution が下がりうる**。

例：「ベースライン差異(8件)」表示の PC が、`License` カテゴリだけ OFF にすると差分集計から Win 4 + Office 4 = 最大 8 項目が外れ、`(0件)` になれば Caution 解除。

---

## Excel 出力での反映

主に **Sheet 1「PC情報一覧」の「突合判定」列** と **行全体着色** で反映：

| 状態 | 突合判定列 | 行着色 |
|---|---|---|
| `report.MismatchCount > 0` | `NG (N件)`（赤背景・赤文字・太字） | （行全体は着色しない） |
| `report.AllMatch` | `OK`（緑背景・緑文字・太字） | （着色なし） |
| `!report.HostlistFound` | `対象外`（Gray） | （着色なし） |
| `report` 未生成 | `未突合`（Gray） | （着色なし） |

各 license / certificates シートでは個別に AuditLevel 着色が走る（[fabriq_evidence_manager__reference__excel_layout.md](fabriq_evidence_manager__reference__excel_layout.md) 参照）。

**Warning / Caution は基本的に Excel main シートに直接出ない**：監査人は判定列・各シートの着色から「赤くなっているセル」を見て同等の情報を得る設計。`WarningMessage` / `CautionMessage` 自体は管理者メモ列とは別に出ていない（将来要件として別列追加の余地あり）。

---

## エラー耐性

両判定とも **個別の例外を握りつぶす（catch して何もしない）** 設計：

```csharp
if (_hostlistService.IsLoaded)
{
    try { ... }
    catch { /* 判定失敗時はスキップ */ }
}
```

理由：「判定が走らない PC が 1 台あっても、**フリート全体の検収を止めない**」。例外発生は `Debug.WriteLine` でログだけ出し、ユーザーにダイアログは出さない。

例外発生時は対応する Caution メッセージが立たないだけで、PC は「該当判定なし」状態として表示される（黄色行ではなくなる）。これは「**沈黙のフォールバック**」で、ベテラン作業者向けにはトレードオフが許容される。改善案として「judging エラー」専用の caution メッセージを追加する余地はある（v3.8.0 では未実装）。

---

## 設計判断のまとめ

- **赤と黄の意味分離**: 赤 = 物理対処（再収集 / 機器修復）、黄 = 運用判断（人間の許容判定）
- **Office Partial の Caution 側回し**: ユーザーサインイン待ちは時間経過で解消する一時状態であり、認証失敗とは性質が違う
- **SerialNumber は CheckOption 不要**: 案件特性で OFF にできない監査の根本要件
- **ベースラインカテゴリ ON/OFF の即時反映**: 案件ごとに「Office 配備しない」「ドメイン参加なし」が違うため設定動的化が必要
- **両軸の同時表示（行色は赤、ステータス列は両方）**: 視認性と情報量のバランス、片方を隠さない
- **判定例外の silent 化**: フリート検収のスループット優先、1 件の判定エラーで作業フローを止めない

---

## 関連ドキュメント

- 階層構造（責務分離 + DI）: [fabriq_evidence_manager__architecture__01_layers.md](fabriq_evidence_manager__architecture__01_layers.md)
- 設定ダイアログ（CheckOptions / カテゴリ ON/OFF）: [fabriq_evidence_manager__apps__03_settings_window.md](fabriq_evidence_manager__apps__03_settings_window.md)
- フリート画面（DataGrid 行ハイライト）: [fabriq_evidence_manager__apps__01_main_window.md](fabriq_evidence_manager__apps__01_main_window.md)
- Office ライセンス評価分岐: [fabriq_evidence_manager__reference__office_license_evaluation.md](fabriq_evidence_manager__reference__office_license_evaluation.md)
- SerialNumber フォールバック判定: [fabriq_evidence_manager__reference__serial_number_logic.md](fabriq_evidence_manager__reference__serial_number_logic.md)
- hostlist 突合: [fabriq_evidence_manager__usage__02_hostlist_verification.md](fabriq_evidence_manager__usage__02_hostlist_verification.md)
- ベースライン突合: [fabriq_evidence_manager__usage__03_baseline.md](fabriq_evidence_manager__usage__03_baseline.md)
