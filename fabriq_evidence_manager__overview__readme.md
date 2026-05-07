# fabriq_evidence_manager 全体像

> **対象**: fabriq_evidence_manager / 全体
> **対象バージョン**: 3.8.0（取得元: `E:\fabriq_evidence_manager\FabriqEvidenceManager\FabriqEvidenceManager.csproj` `<Version>`）
> **ドキュメント更新日**: 2026-05-07

## fabriq_evidence_manager とは

`E:\fabriq_evidence_manager` は、fabriq フレームワーク本体（`E:\fabriq`）が**キッティング実行時に出力した evidence ディレクトリを read-only で取り込み、フリート単位で確認・突合・納品整形する** WPF デスクトップアプリである。fabriq 側は実機 1 台ごとにスクリーンショット・ログ・CSV・manifest.json を `log_uploader` で集約サーバへアップロードするが、複数台分が並列に積み上がった現場では「どの PC のキッティングが正常終了したか」「hostlist の期待値と一致しているか」を人手で照合するのは非現実的である。本ツールは **30+ 種類のエビデンスファイルを構造化パースし、欠損 (Warning) と要確認 (Caution) の 2 軸でフリート全体を可視化** することで、納品前検収を機械的に実施するための専用フロントエンドを提供する。

技術スタックは **.NET 8.0 / WPF / C# 12（NRT 有効）**。MVVM パターンを `CommunityToolkit.Mvvm` 8.4.0 で構築し、Excel 出力は `ClosedXML` 0.105.0、DI は `Microsoft.Extensions.DependencyInjection` 10.0.3 を使う。ソリューションは本体プロジェクト `FabriqEvidenceManager/FabriqEvidenceManager.csproj` と xUnit テストプロジェクト `FabriqEvidenceManager.Tests/FabriqEvidenceManager.Tests.csproj` の 2 構成（`fabriq_evidence_manager.sln`）。

## 中核概念 — manifest 駆動取り込み

fabriq_evidence_manager の全機能は `pc_information/manifest.json`（fabriq kernel 公開契約 [EVIDENCE_MANIFEST.md](fabriq__contracts__evidence_manifest_contract.md) `schemaVersion=1`）を起点に動作する。manifest が宣言する `sections[].id` と `files[]` を見て、対応するサブパーサを呼び出して `PcEvidence` 集約モデルに型付きで充填する。

設計の鍵は次の 3 点：

- **公開契約のみに依存**: `manifestType="fabriq-evidence-manifest"` と `schemaVersion=1` の合致を厳格チェックし、未対応 schema は silent fallback せず例外で停止する（`UnsupportedManifestSchemaException`）
- **未知 section の前方互換捕捉**: `EvidenceConstants.KnownSections` に無い ID（fabriq 側の将来追加 §xx 等）は `UnknownSection` に raw 保持し、Excel に専用シートを動的生成して監査人が中身を確認できる状態を維持
- **ソース不変**: 取り込み対象ディレクトリには一切の書き込みを行わず、納品出力時もコピーのみ。本ツールが書き込むのは PC ルート直下の `manager_memo.json`（管理者メモ）だけ

## 機能カテゴリ

| カテゴリ | 機能 | 関連 Service / ViewModel |
|---|---|---|
| 取り込み | evidence ルート走査・PC 単位集約・manifest 読み込み | `NestedEvidenceDiscoveryService` / `ManifestReaderService` |
| 取り込み | §01〜§31 セクションの構造化パース | `EvidenceParserService` + 11 サブパーサ |
| 取り込み | 30 秒間隔の自動差分マージリロード | `MainWindowViewModel.RefreshEvidenceAsync` |
| 検証 | hostlist.csv 期待値との突合（ホスト名・IP・DNS・プリンタ・BitLocker） | `HostlistService` / `EvidenceVerificationService` |
| 検証 | チェックリスト HTML / 実行履歴 CSV のパース | `ChecklistParserService` |
| 検証 | パイロット PC をベースラインに据えたフリート整合性突合（6 カテゴリ） | `BaselineService` + 6 件の `IBaselineComparator` |
| 検証 | 欠損警告（赤）／要確認（黄）の 2 軸ハイライト | `EvidenceCheckOptions` / `MainWindowViewModel.EvaluateCaution` |
| 表示 | フリート DataGrid（PC 名・SN・MAC・BitLocker・ライセンス状態） | `MainWindow` / `MainWindowViewModel` |
| 表示 | PC 個別詳細ウィンドウ（複数 PC を別窓で並列比較可） | `PcDetailWindow` / `PcDetailViewModel` |
| 表示 | 設定ダイアログ（取得チェック ON/OFF + ベースラインカテゴリ ON/OFF） | `SettingsWindow` / `SettingsViewModel` |
| 永続化 | PC 単位メモ（`manager_memo.json`） | `PcMemoService` |
| 出力 | 納品ディレクトリ生成・全エビデンスの再帰コピー・SHA-256 マニフェスト生成 | `EvidenceCollectorService` |
| 出力 | Excel 台帳（30+ シート、PC 個別詳細サブブック、案件メタデータヘッダ） | `ExcelExportService` |

## 取り込み対象ファイル

fabriq 側 `evidence_config` モジュール（および `log_uploader`）が出力する 1 PC あたりのディレクトリツリーをそのまま受け付ける：

```
{yyyy_MM_dd_HHmmss}_{PCName}_{SerialNumber}_evidence/
├── evidence/
│   ├── pc_information/        ── manifest.json + 30+ ファイル（§01〜§31）
│   ├── auto_capture/          ── デジタル魚拓 PNG/JPG（モジュール実行ごと）
│   ├── bitlocker/             ── {pcName}_{drive}.txt（回復キー）
│   ├── checklist/             ── checklist_*.html（チェックリスト HTML）
│   └── export_history/        ── history_export_*.csv（実行履歴）
└── manager_memo.json          ── 本アプリが保存する管理者メモ
```

詳細なファイル形式・命名規則・manifest セクション ID の対応は [fabriq_evidence_manager__architecture__02_evidence_input.md](fabriq_evidence_manager__architecture__02_evidence_input.md) を参照。

## 規模感

| 区分 | 件数 |
|---|---|
| Views (XAML) | 4（App.xaml + MainWindow / PcDetailWindow / SettingsWindow） |
| ViewModels | 3（MainWindow / PcDetail / Settings） |
| Models | 60+（集約ルート `PcEvidence` + 各セクションの DTO 群） |
| Services | 26（インターフェース 22 / 実装 24、DI 登録 24 — 詳細は [layers](fabriq_evidence_manager__architecture__01_layers.md)） |
| Baseline Comparators | 6（`IBaselineComparator` のプラグイン式実装） |
| Helpers | 3（`EvidenceConstants` / `EvidencePathParser` / `LicenseStatusToBackgroundConverter`） |
| Tests | 22 ファイル（xUnit、`TestData/` フィクスチャ駆動） |

## fabriq 側との接続点

- **入力**: fabriq kernel 2.2.2+ / `evidence_config` v1.3.0+ 出力の `pc_information/manifest.json`（`schemaVersion=1`）
- **既知セクション ID**: §01〜§31（kernel 3.0.0 / `evidence_config` v1.6.0 時点）。詳細マップは `EvidenceConstants.KnownSections`
- **fabriq 側に提案がある場合**: 本リポジトリ内 md に「提案」として記録するに留める（CLAUDE.md R1）

## 配布・ビルド

- ビルド: `dotnet build` （`fabriq_evidence_manager.sln`）
- パブリッシュ（self-contained, single-file 化）:

```
dotnet publish FabriqEvidenceManager/FabriqEvidenceManager.csproj `
  -c Release -r win-x64 --self-contained true `
  -p:PublishSingleFile=true -p:IncludeNativeLibrariesForSelfExtract=true `
  -o E:\publish_fabriq_evidence_manager
```

- 出力先は固定で `E:\publish_fabriq_evidence_manager\`

## 関連ドキュメント

- 階層構造（View / ViewModel / Service / Model）: [fabriq_evidence_manager__architecture__01_layers.md](fabriq_evidence_manager__architecture__01_layers.md)
- 入力 evidence ディレクトリ構造と manifest dispatch: [fabriq_evidence_manager__architecture__02_evidence_input.md](fabriq_evidence_manager__architecture__02_evidence_input.md)
- fabriq kernel 側 manifest 公開契約: [fabriq__contracts__evidence_manifest_contract.md](fabriq__contracts__evidence_manifest_contract.md)
- fabriq kernel 側 evidence_config モジュール: [fabriq__modules__evidence_config.md](fabriq__modules__evidence_config.md)
