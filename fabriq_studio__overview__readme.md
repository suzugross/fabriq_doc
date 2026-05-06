# fabriq_studio 全体像

> **対象**: fabriq_studio / 全体
> **対象バージョン**: commit `3897c6e`（取得元: `git -C E:\fabriq_studio rev-parse --short HEAD`、csproj に `<Version>` 未設定のため）
> **ドキュメント更新日**: 2026-05-06

## fabriq_studio とは

`E:\fabriq_studio` は fabriq フレームワーク本体（`E:\fabriq`）を **GUI で安全に編集・配備するためのデスクトップアプリ** である。fabriq 本体が PowerShell スクリプト + CSV 設定ファイルの組み合わせで構成されているため、CSV を直接編集する運用ではタイポや列順事故が起きやすい。fabriq_studio はそれらを型付き ViewModel でラップし、未保存変更ガード・パスフレーズ照合・SemVer 比較といった安全策を組み込んだ編集環境を提供する。

技術スタックは **.NET 8.0 / WPF / C# 12（NRT 有効）**。MVVM パターンを `CommunityToolkit.Mvvm` 8.3.2 で構築し、CSV 入出力は `CsvHelper` 33.0.1、DI は `Microsoft.Extensions.DependencyInjection` 8.0.1 を使う。ソリューションは単一プロジェクト構成（`FabriqStudio.sln` → `FabriqStudio/FabriqStudio.csproj`）。

## 中核概念 — ワークスペース

fabriq_studio の全機能は「**ワークスペース**」を起点として動作する。ワークスペースは `kernel/` と `modules/` 両方を含むディレクトリ（= fabriq フレームワークインスタンス）で、Studio はその中の CSV / JSON / PowerShell ファイルを安全に R/W する。ワークスペースが開かれていない状態では編集系メニューはすべて無効化される。

ワークスペース選択は永続化される（`<exe>\config\workspace.json`、ポータブル運用対応）。アプリ起動時に前回パスを silent 復元し、検証 NG なら未設定状態に戻る。

## 機能カテゴリ

| カテゴリ | 機能 | 関連 ViewModel / View |
|---|---|---|
| 編集 | 基本パラメータ（workers / categories / log_destinations 等） | `BasicParamsViewModel` / `BasicParamsView` |
| 編集 | 端末一覧（hostlist.csv の 43 列、IP / Wi-Fi / DNS / BitLocker PIN / プリンタ最大10台） | `HostListViewModel` → `HostDetailViewModel` |
| 編集 | モジュールマスター（`modules/{standard,extended}/<name>/module.csv`、カテゴリ・実行順序） | `ModuleEditViewModel` → `ModuleDetailViewModel` / `AppConfigViewModel` |
| 編集 | プロファイル（`profiles/*.csv`、Order 10 刻み振り直し付き） | `ProfileDetailViewModel` |
| ツール | レジストリ辞書（カタログ → workspace の reg_hklm/hkcu_list.csv にエクスポート） | `RegistryCollectionViewModel` / `RegistryPickerViewModel` |
| ツール | Pianist Profile Editor（5 タブ + 4 sub-tab DSL + テスト実行） | `PianistProfileEditorViewModel` |
| ツール | プリンタドライバ検出（INF 解析・7z/Zip 展開・hostlist 転記） | `PrinterDriverDetectorViewModel` |
| ツール | Script Looper（リトライ条件付きタスクの繰り返し実行モジュール生成） | `LooperEditorViewModel` |
| 配備 | fabriq バックアップ（PS1 等を除いたミラーコピー） | `FabriqBackupDialog` |
| 配備 | fabriq オーバーレイ更新（同梱テンプレートから本体を SemVer 比較で安全に上書き） | `FabriqUpdateDialogViewModel` |
| セキュリティ | パスフレーズ + AES-256-CBC 暗号化（PowerShell 側 `Unprotect-FabriqValue` と相互運用） | `CryptoService` / `PassphraseDialog` |
| 配備 | hostlist エクスポート（タイムスタンプ付きフォルダ、復号オプション） | `HostListExportDialog` |

## 規模感

| 区分 | 件数 |
|---|---|
| Views (XAML) | 29 |
| ViewModels | 16（うち 1 件は `IDirtyAwareViewModel` インターフェース） |
| Models | 37（うち 17 件が Pianist 専用） |
| Services | 16（インターフェース 16 / 実装 16、DI 登録 15） |
| Helpers | 10 |
| Converters | 13 |

## 開発の活発な領域

git log 上位 15 件のうち約半数が Pianist Profile Editor 関連（test_run / list_csv_editor / step_drag_drop_reorder / window_picker / profile_delete / key_presets_extend など）。次いで横断的な dirty_guard / unsaved_changes_guard が直近で追加されている。Pianist Profile Editor は v1.x 系として現在も拡張中の主要開発対象である。

## 配布・ビルド

- ビルド: `dotnet build FabriqStudio.sln`
- パブリッシュ（self-contained, single-file 化済み運用）:
  ```
  dotnet publish FabriqStudio/FabriqStudio.csproj -c Release -o "E:/publish_fabriq_studio" --self-contained true -r win-x64
  ```
- `.csproj` の `AfterTargets="Build" / "Publish"` ターゲットで `registry_collection/` と `template/template_fabriq/` を出力先に自動コピー（fabriq 本体実体は `.gitignore` で追跡対象外、別途配置必須）

## 関連ドキュメント

- アーキテクチャ: [fabriq_studio__architecture__01_layers.md](fabriq_studio__architecture__01_layers.md)
- ワークスペース定義: [fabriq_studio__architecture__02_workspace.md](fabriq_studio__architecture__02_workspace.md)
- メイン編集画面: [fabriq_studio__apps__01_main_pages.md](fabriq_studio__apps__01_main_pages.md)
- Pianist Profile Editor: [fabriq_studio__apps__02_pianist_profile_editor.md](fabriq_studio__apps__02_pianist_profile_editor.md)
- レジストリ辞書: [fabriq_studio__apps__03_registry_collection.md](fabriq_studio__apps__03_registry_collection.md)
- その他ツール: [fabriq_studio__apps__04_other_tools.md](fabriq_studio__apps__04_other_tools.md)
- 暗号化相互運用: [fabriq_studio__contracts__crypto_interop.md](fabriq_studio__contracts__crypto_interop.md)
- ワークスペースセットアップ手順: [fabriq_studio__usage__01_workspace_setup.md](fabriq_studio__usage__01_workspace_setup.md)
