# ベースライン突合の使い方

> **対象**: fabriq_evidence_manager / usage
> **対象バージョン**: 3.8.0（取得元: `E:\fabriq_evidence_manager\FabriqEvidenceManager\FabriqEvidenceManager.csproj` `<Version>`）
> **ドキュメント更新日**: 2026-05-07

フリート 1 台を「ベースライン PC」として選び、他の全 PC が同じ posture（OS バージョン / ライセンス / モジュール実行成否 / インストール済みアプリ等）を持っているかを **PC 内部の差分ではなく PC 間の差分** として検証する機能。hostlist 突合（個別 PC × hostlist の期待値）とは直交する第 2 の検証軸。

---

## いつ使うか

| ユースケース | 適性 |
|---|---|
| パイロット 1 台 → 残り 49 台展開後の整合性チェック | ◎ |
| 同じ案件で日付違いに収集された 1 PC を 2 回比較する | ◎ |
| 異なる案件 / 異なる kernel バージョンの PC 同士を比較する | △（バージョン差で全項目 Mismatch になる） |
| OS 種別が混在するフリート（Win10 + Win11） | △（SystemInfo カテゴリで OSName 全件 Mismatch） |
| 案件が 1 PC のみ | × |

---

## 6 カテゴリの内容

`IBaselineComparator` プラグイン 6 件が並列に動く（`App.xaml.cs` で 6 件すべて DI 登録済み）：

| 表示名 | CategoryId | 比較対象（baseline ⇔ target） | 既定 |
|---|---|---|---|
| 実行サマリ | `ExecutionSummary` | `export_history.csv` の `ModuleName × Status`（不一致 / Missing / Extra を別タグで計上） | 有効 |
| SystemInfo | `SystemInfo` | `01_SystemInfo.txt` の OS 名 / OS バージョン / CPU / メモリ（4 項目） | 有効 |
| チェックリスト | `Checklist` | チェックリスト HTML の `OverallStatus + VerifyItems`（PASS/FAIL の項目別比較） | 有効 |
| インストール済みアプリ | `InstalledApps` | `11_DesktopApps.csv` + `11_StoreApps.csv` 統合の `Name × Version`。一致は集計のみ、差分のみ列挙 | 有効 |
| ライセンス | `License` | §21 Windows License + §22 Office License（typed model 直比較）、Win 4 + Office 4 = 最大 8 項目 | 有効 |
| ドメイン参加状態 | `DomainStatus` | §05 DomainStatus（CurrentUser を除く 7 項目: ドメイン参加 / ドメイン名 / ロール / Azure AD 参加 / AD 参加 / AD 名 / テナント名） | 有効 |

カテゴリは設定ダイアログ § 「ベースライン突合カテゴリ」で個別 ON/OFF 可（[fabriq_evidence_manager__apps__03_settings_window.md](fabriq_evidence_manager__apps__03_settings_window.md) §「セクション 2」）。OFF にしたカテゴリは `CompareAll` でスキップされ、Caution 件数集計と Excel 出力からも外れる。

---

## 操作手順

### 1. evidence を読み込む

[usage__01_import](fabriq_evidence_manager__usage__01_import.md) の手順に従ってフリート全 PC を DataGrid に並べる。**ベースライン PC も target PC もすべて同じ evidence ルートに含まれている前提**。

### 2. ベースライン PC を選ぶ

`MainWindow` Row 1 の `Baseline PC:` 行：

1. DataGrid でパイロット PC（基準にしたい 1 台）の行をクリックして選択
2. `選択PCをベースラインに設定` ボタン

`IBaselineService.LoadFromPc(SelectedPc)` が走り、6 Comparator 全部に `CacheBaseline(baselinePc)` が発火 → 各カテゴリのスナップショットがメモリに乗る。

ステータスバー: `ベースラインPCを設定しました: {PcName}`、`BaselineStatusText="ベースラインPC: {PcName}"`、全 PC の Caution 判定が再実行される。

### 3. 結果を見る

| 場所 | 内容 |
|---|---|
| DataGrid 行ハイライト（黄） | `CautionMessage` に `"ベースライン差異(N件)"` |
| PC 詳細ウィンドウの `BASELINE — ...` セクション群 | 6 カテゴリのサブセクションが個別に展開 |
| Excel 台帳の `ベースライン_*` シート | 6 カテゴリ × フリート全 PC の差分 |

ベースライン PC 自身を target にした場合は理論上全項目 Match になる（`SystemInfoComparator` 等が「両側空文字列も Match に倒す」防御的判定を入れているため、Office 未インストール機などの空フィールドが false-positive にならない）。

### 4. カテゴリを絞る

設定 → ベースライン突合カテゴリ で不要なカテゴリを OFF：

- 案件で Office 配備しない → `ライセンス` を OFF（Win/Office まとめてスキップになる点に注意）
- WORKGROUP 機混在 → `ドメイン参加状態` を OFF
- 実行履歴の差分が現場運用上想定内（特定モジュールを後で個別実行する等） → `実行サマリ` を OFF

OFF にしてもキャッシュ済みベースラインデータは破棄されない。再 ON で `CompareAll` の対象に戻る。

### 5. クリアする

`Baseline PC:` 行の `クリア` ボタン → `IBaselineService.Clear()` で全 6 Comparator の `ClearBaseline()` を発火。`BaselineStatusText="未設定"`、Caution 判定が再実行されてベースライン関連のオレンジ警告が消える。

---

## カテゴリ別の差分セマンティクス

### 実行サマリ（ExecutionSummary）

`export_history.csv` のモジュール（`ModuleName` 単位）を `BaselineComparisonItem` に展開：

| MatchStatus | 意味 |
|---|---|
| `Match` | baseline と target で `Status` 一致 |
| `Mismatch` | 同モジュールで `Status` 違い（例: baseline=Success / target=Error） |
| `MissingInActual` | baseline にあるが target 未実行（パイロットでは走ったが他では走っていない） |
| `ExtraInActual` | target だけにある（パイロット以後に追加されたモジュール、または手動実行） |

case-insensitive で比較。同名モジュールが複数回出ている場合は **最後勝ち**（`Dictionary` の上書き挙動）。

### SystemInfo

固定 4 項目（OS 名 / OS バージョン / CPU / メモリ）を `01_SystemInfo.txt` から再パースして比較。`baseline=actual=空文字列` のときは `Match` に倒す（パイロット自身を target にしたときに「空フィールド ≠ 空フィールド」で false-positive にしない）。

### チェックリスト（Checklist）

baseline と target の両方からチェックリスト HTML をパースし：

- `OverallStatus` を baseline と target で並列保持（`ChecklistComparisonResult.BaselineOverallStatus` / `ActualOverallStatus`）
- `VerifyItems[].ItemName` を辞書化して PASS/FAIL の項目別比較
- target に該当 ItemName が無い場合は `NoActual`（`(未検出)`）

VerifyItems の同名項目が複数あるときは最後勝ち。

### インストール済みアプリ（InstalledApps）

`11_DesktopApps.csv` + `11_StoreApps.csv` を統合し、アプリ `Name`（`OrdinalIgnoreCase`）でキーにして突き合わせる：

| 状態 | 出力 |
|---|---|
| 両方に同名同バージョン | 集計のみ（`MatchCount++`、`Items` には積まない） |
| 両方に同名違うバージョン | `Items` に `Mismatch`（Expected = baseline ver / Actual = target ver） |
| baseline にだけある | `Items` に `NoActual`（"(未検出)"） |
| target にだけある | `Items` に `NoExpected`（"(ベースラインなし)"） |

`InstalledAppsComparisonResult` は `TotalBaselineCount / TotalActualCount / MatchCount / Items` を保持し、Excel 出力で「総数 N 件、一致 M 件、差分 K 件」のサマリ + 差分行を生成する。

### ライセンス（License）

PcEvidence の `WindowsLicense` / `OfficeLicense` を typed model のまま比較（再パース不要）：

**Windows 比較**（`Products[0]` ベース）:

- ライセンスファミリ（例: `Professional`）
- プロダクトキーチャネル（例: `Volume:GVLK`）
- ライセンス状態（"認証済 (1)" 等の `LicenseStatusText (LicenseStatusCode)` 表記）
- KMS Machine（`Service.KeyManagementServiceMachine`）

**Office 比較**:

- インストール状態（`インストール済み` / `未インストール`）
- C2R `ProductReleaseIds`（例: `O365ProPlusRetail`）
- C2R `UpdateChannel`（例: `MonthlyEnterprise`）
- ライセンス状態（OSPP `Products[0].LicenseStatusText`）

baseline と target の両側が null の項目は出力しない（fleet 全機 Office 未インストールなら License カテゴリは Windows 4 項目のみ）。

### ドメイン参加状態（DomainStatus）

`DomainStatusData` の主要 7 項目を比較。`CurrentUser` のみ **PC 固有のため比較対象外**（キッティング作業者が PC ごとに違うのは正常）。各 `bool` フィールド（`PartOfDomain` / `AzureAdJoined` / `DomainJoined`）は `参加 / 未参加` の文字列で比較する。

---

## バックエンド: backward compat 経路

`BaselineService` には旧経路として **CSV から baseline を直接ロードする `Load(csvFilePath)` メソッド** が残っている：

```csharp
public int Load(string csvFilePath)
{
    var executor = _comparators.OfType<ExecutionSummaryComparator>().FirstOrDefault();
    return executor?.LoadFromCsv(csvFilePath) ?? 0;
}
```

`ExecutionSummaryComparator.LoadFromCsv` のみが `_baselineMap` を埋める形。`SystemInfo / Checklist / InstalledApps / License / DomainStatus` の 5 つは PC 全体（`PcEvidence` 参照）を要求するため、CSV 単体ロードでは無効化される。

現行 UI では本経路は表に出ていない（**`MainWindowViewModel` 側に CSV ベースラインロード用のコマンドは存在しない**）。`BaselineService.Compare(pcName, actualHistory)` も古い shape のまま残っているが、`CompareAll(targetPc)` への置き換えが完了している。

将来 UI 上で「CSV から baseline を作る」モードを復活させる場合は、`MainWindowViewModel` に CSV 選択コマンドと `_baselineService.Load(...)` 呼び出しを足すだけで動く形は維持されている。

---

## 結果の見方

### DataGrid（フリート画面）

ベースライン設定後、`MainWindowViewModel.EvaluateCaution` が各 PC で `_baselineService.CompareAll(pc)` を実行し `DifferenceCount > 0` で `CautionMessage += "ベースライン差異(N件)"`。**1 件でも差分があれば黄色行**。

`DifferenceCount` の集計式（`BaselineComparisonReport`）：

```
DifferenceCount =
    Items.Count(MatchStatus != Match)                 // 実行サマリ
  + (SystemInfoComparison?.MismatchCount ?? 0)
  + (ChecklistComparison?.MismatchCount ?? 0)
  + (InstalledAppsComparison?.MismatchCount ?? 0)
  + (LicenseComparison?.MismatchCount ?? 0)
  + (DomainStatusComparison?.MismatchCount ?? 0)
```

各サブの `MismatchCount` は「`Status != Match` の件数」（`NoExpected` / `NoActual` も含む）。

### PC 詳細ウィンドウ

`BASELINE — ...` の 6 セクションが個別表示（一致時はサマリ行 1 つに圧縮、差分時のみ詳細列挙）。詳細は [fabriq_evidence_manager__apps__02_pc_detail_window.md](fabriq_evidence_manager__apps__02_pc_detail_window.md) §「ベースライン突合（6 サブセクション）」。

### Excel 台帳

納品データ出力時、6 カテゴリそれぞれが **専用シート**として出る：

- `ベースライン_実行サマリ`
- `ベースライン_SystemInfo`
- `ベースライン_チェックリスト`
- `ベースライン_アプリ`
- `ベースライン_ドメイン`
- `ベースライン_ライセンス`

各シートで baseline PC が列方向の基準、target PC が行（または逆）として配置される。詳細は将来の `fabriq_evidence_manager__reference__excel_layout.md` 計画。

---

## 運用パターン

### パターン A: パイロット → 量産

1. パイロット 1 台を fabriq でキッティング → 検収 OK の状態で evidence を確保
2. 量産 PC（49 台）を順次キッティング → 各 PC の evidence を集約
3. 本アプリで evidence ルートを開き、パイロット PC をベースラインに設定
4. 全 49 台の `BASELINE — ...` シートが揃ったら納品データ出力

### パターン B: 日付差での再収集確認

1. 1 PC を kernel 3.0 → 3.2 にアップデート前後で 2 回 evidence 収集
2. 同じ PC のフォルダが 2 つ並ぶ（`CollectionDate` が異なるので別エントリ扱い）
3. 古い方をベースラインに設定
4. 新しい方が `BASELINE — ライセンス` で差分なしか確認

### パターン C: 部分検収

設定で `ライセンス` のみ ON にして他をすべて OFF：

- ライセンス情報の整合性のみ確認したい場合
- フリート全機が「Volume チャネル / KMS Machine 一致」を持っているかを単独で検証

---

## トラブル対応

### ベースライン設定時にエラー

`MessageBox.Show("ベースラインPCの設定に失敗しました")` + 例外メッセージ：

- **`ExecutionSummaryComparator.CacheBaseline` が CSV 不在で失敗** → 既存挙動として silent 維持（throw しない）。問題なくクリアされない
- baseline PC が `pc_information` 未取得 → `SystemInfoComparator` / `InstalledAppsComparator` の `_baseline` が null になり、対応カテゴリは silent skip

### 全 PC で同じ項目が Mismatch

- baseline と target で kernel バージョンが大きく違う（manifest.fabriqKernelVersion を比較）→ kernel を揃える
- baseline 自身が evidence 不完全 → 別の PC をベースラインに切り替える

### Caution の件数が想定外に多い

設定 → ベースライン突合カテゴリ で関連カテゴリを OFF。または baseline PC を切り替えて「より代表的な 1 台」を選ぶ。

---

## 関連ドキュメント

- 取り込み手順（前段）: [fabriq_evidence_manager__usage__01_import.md](fabriq_evidence_manager__usage__01_import.md)
- hostlist 突合（直交する別軸）: [fabriq_evidence_manager__usage__02_hostlist_verification.md](fabriq_evidence_manager__usage__02_hostlist_verification.md)
- 設定ダイアログ: [fabriq_evidence_manager__apps__03_settings_window.md](fabriq_evidence_manager__apps__03_settings_window.md)
- PC 詳細ウィンドウ（BASELINE セクション 6 件）: [fabriq_evidence_manager__apps__02_pc_detail_window.md](fabriq_evidence_manager__apps__02_pc_detail_window.md)
- 階層構造（BaselineService + 6 Comparator 詳細）: [fabriq_evidence_manager__architecture__01_layers.md](fabriq_evidence_manager__architecture__01_layers.md)
