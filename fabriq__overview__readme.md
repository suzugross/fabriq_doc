# fabriq 全体像

> **対象**: fabriq / 全体
> **対象バージョン**: kernel 3.6.0（取得元: `E:\fabriq\kernel\KERNEL_VERSION`）+ commit `0fca159`（取得元: `git -C E:\fabriq rev-parse --short HEAD`、2026-06-16）
> **ドキュメント更新日**: 2026-06-16

## fabriq とは

`E:\fabriq` は **Windows 11 PC のキッティングを自動化する PowerShell + WinForms フレームワーク** である。本体 README.md 冒頭の副題「**Manifeste du Surkitinisme**」が示す通り、繰り返し作業の手作業排除を志向するシリーズの中核実装にあたる。技術スタックは **PowerShell 5.1 + WinForms GUI + C# ランチャ（`Fabriq.exe`、自動 UAC 昇格）**。

CSV 駆動 + モジュール型の設計を採り、コード改修なしに対象 PC・設定値・実行順序を切り替えられる。AES-256-CBC + PBKDF2-SHA256 の鍵導出による CSV 内機密値の透過暗号化、再起動跨ぎの RunOnce 自動再開、HTML チェックリスト出力、エビデンス自動収集などを内蔵する。

設計指針として掲げられている標語は「**堅牢・確実・共通化**」。すべてのモジュールは共通関数（`New-ModuleResult` / `Import-ModuleCsv` / `Show-Info` 系）を必ず使い、結果オブジェクトを統一スキーマで返却する。

本ドキュメントは **fabriq に初めて触れるエンジニアが本リポジトリ全 100+ 件の md を読み始める前に把握すべき骨格** と、**シリーズ内の位置づけ**、**読書順案内** を扱う。kernel 詳細は [fabriq__kernel__01_overview.md](fabriq__kernel__01_overview.md)、モジュール一覧は [fabriq__modules__00_modules_overview.md](fabriq__modules__00_modules_overview.md)、apps カタログは [fabriq__apps__00_apps_overview.md](fabriq__apps__00_apps_overview.md) を参照。

## 中核概念（4 軸）

fabriq の機能はすべて次の 4 つの抽象に分解される。

| 軸 | 定義 | 場所 | 件数 |
|---|---|---|---|
| **Kernel** | 共通基盤・公開 API・状態管理。すべてのモジュールが依存する単一の運転系 | `kernel/main.ps1` + `kernel/common.ps1`（90+ 関数）+ `kernel/KERNEL_API.md` 公開境界 | 1 セット（SemVer 3.6.0） |
| **Modules** | 個別の設定タスクを packaging した単位。独立 SemVer + `REQUIRES_KERNEL` で要求カーネル版宣言 | `modules/standard/` + `modules/extended/` | **standard 61 / extended 18（計 79）** |
| **Profiles** | モジュール実行順序を CSV で定義する宣言ファイル | `profiles/*.csv` | 10 ファイル（Master_Pre / Master_Config 系列 + sysprep + テンプレート、別途 `easy_template/easyprofile.csv`） |
| **AutoPilot / FlexProfile** | 実行モデル。Linear（先頭から末尾まで自動進行）と Flex（state-aware 部分実行）の 2 経路 | `apps/fabriq_operator/lib/dashboard_form.ps1` + `flex_dashboard.ps1` | 2 dashboard 並走 |

### Kernel 公開境界

`kernel/KERNEL_API.md` が `KERNEL_VERSION` と同コミット内で更新される **真の公開 API サーフェス**。§1〜§11 に分節され、§1 公開関数（表示・CSV・結果生成・確認待機・権限・暗号化）/ §2 グローバル変数 / §3 環境変数 / §4 Profile CSV スキーマ / §5 ModuleResult 契約 / §6 共通ライブラリ / §7 セッション / §8 オーケストレーション / §9 更新オーバーレイ / §10 Evidence Manifest / §11 公開ファイル群 を宣言する。

ここに **記載されていない `common.ps1` 関数は内部実装**であり、PATCH バージョンでも予告なく変更されうる。モジュール開発者は KERNEL_API のみに依存することを規約とする。

### Modules の構成パターン

各モジュールは次のファイル群から成る packaging：

| ファイル | 役割 |
|---|---|
| `module.csv` | メニュー名・カテゴリ・表示順・有効無効（1 モジュール内に複数エントリ可） |
| `<name>.ps1` | 実行スクリプト本体（`dev/template/_template_script.ps1` がベース） |
| `<name>_list.csv` | 設定データ（対象リスト等） |
| `Guide.txt` | 使い方ガイド（日本語） |
| `preset.csv` | 設定 CSV のドロップダウン UI 定義（Fabriq Studio が検出して ComboBox 化） |
| `VERSION` | モジュール SemVer（独立進化） |
| `REQUIRES_KERNEL` | 要求カーネル版（オーバーレイ更新時の整合性チェックに使用） |

`windows_update` のみ `module.csv` を持たず、`Fabriq.exe` ダッシュボードの専用ボタンから直接呼ばれる例外（このため Standard 件数 61 = `module.csv` あり 60 + `windows_update` 1）。

### Profiles と特殊マーカー

profile CSV は 7 列スキーマ（kernel 3.2.0 で `Group` 列追加）：

```csv
Order,ScriptPath,Enabled,Description,Segment,ErrorMode,Group
10,__AUTOPILOT__,1,WaitSec=3,,,
20,standard/hostname_config/hostname_config.ps1,1,ホスト名設定,,,Network
30,standard/ipaddress_config/ipaddress_config.ps1,1,IP アドレス設定,,retry,Network
40,__RESTART__,1,再起動,,,
```

`ScriptPath` 列に **特殊マーカー**を記載すると kernel 側が flow control として解釈する：

| マーカー | 動作 |
|---|---|
| `__AUTOPILOT__` | 以降を AutoPilot 化（`Description` の `WaitSec=N` でモジュール間ウェイト秒指定） |
| `__ASYNC__` | 以降のモジュールを監視付き Runspace で実行（Skip ボタン / `async_config.json` の `DefaultTimeoutSec` で強制中断可能、kernel 2.1.0 以降）。kernel 3.3.0 以降は `async_config.json` の `DefaultAsync` が shipped default `true` のため、マーカー有無に関わらず全モジュールが async 化され、本マーカーは idempotent な ON-only no-op として後方互換保持される（kill switch `Enabled=false` は引き続き優先） |
| `__RESTART__` | Windows 再起動 + RunOnce 経由で次モジュールから自動再開 |
| `__REEXPLORER__` | Explorer 再起動（レジストリ変更の即時反映） |
| `__GATE__` | 前進バリア（kernel 3.6.0 以降）。直前ゲート〜本マーカーの窓に `Error` / `Partial` または Post-Apply Verification 失敗（`Verified=False`）のモジュールが残る間、`Invoke-BatchExecution` が本マーカー以降の `Order` の実行を拒否する（動的評価）。FlexProfile dashboard では該当行をグレーアウト。窓が解消すると解除。`Verified=$null`（検証非対応）/ `Pending`（未実行）はブロックしない |
| `__AUTO_to_<User>__` | `autologon_config` の `autologon_list.csv` から `User` 列一致のエントリで自動ログオン |

詳細は [fabriq__contracts__special_markers.md](fabriq__contracts__special_markers.md) を参照。

### Linear と FlexProfile の使い分け

- **Linear** （`Execute Profile`）: プロファイル先頭から末尾まで一気通貫で実行。AutoPilot 中はエラー処理ポリシー（`ErrorMode = skip / retry`、retry は最大 5 回）に従う
- **FlexProfile** （`Execute (Flex)`）: kernel 3.1.0 で導入された state-aware 部分実行ダッシュボード。実行履歴から各モジュールの `Success / Partial / Error / Skipped / Pending / NotRun` を復元し、行単位 `[Run]` / `[Run Selected (N)]` / Group 単位 `[Run: <Group>]` で任意の組合せを段階的に進められる
  - 実行モデル: 「**実行 = 常に AutoPilot 挙動 / 完了 = 常に手動**」（`[Complete]` 押下まで HTML 生成 + `log_uploader` を保留、kernel 3.1.5 以降）

Linear は将来 FlexProfile が安定したのち撤去予定（README L196）。

## 起動フロー

```
operator が Fabriq.exe をダブルクリック
   │
   ▼
Fabriq.exe（C# ランチャ）が UAC 自動昇格
   │
   ▼
管理者権限の PowerShell 5.1 コンソール起動
   │
   ▼
kernel/main.ps1（dot-source）
   │  ・kernel/common.ps1 を読み込み（90+ 関数）
   │  ・kernel/ps1/manifesto.ps1 を読み込み（演出表示）
   │  ・apps/fabriq_operator/fabriq_operator.ps1 を読み込み（メインダッシュボード GUI）
   │  ・Enable-SleepSuppression（実行中スリープ抑制）
   │  ・Disable-QuickEditMode（誤フリーズ防止）
   │  ・Set-ConsoleSize 75x35
   │
   ▼
WinForms session form 表示
   │  ・マスターパスフレーズ入力（kernel/txt/passphrase_verify.txt の "surkitinisme" を復号して照合）
   │  ・hostlist.csv から対象 PC 選択
   │  ・workers.csv から作業者選択
   │  ・$env:SELECTED_* を設定（KERNEL_API §3.1）
   │
   ▼
WinForms operator dashboard 表示
   │  ・Execute Profile / Execute (Flex) / Execute（個別実行）
   │  ・FabriqApps（補助具 GUI 起動）/ Quick Actions / Refabriq 等
   │
   ▼
モジュール実行 → New-ModuleResult を pipeline 返却 → 実行履歴 CSV 追記 + スクリーンショット保存
```

**起動の必須前提**は `kernel/txt/passphrase_verify.txt` の存在。これは **Fabriq Studio でマスターパスフレーズを設定すると生成される検証トークン**で、トークン無しでは fabriq は起動できない（README L24 で明記）。

ダッシュボードの主要ボタンは README.md L120〜137 を参照。本リポジトリでは [fabriq__apps__01_fabriq_operator_dashboard.md](fabriq__apps__01_fabriq_operator_dashboard.md) で個別解説。

## 規模感

| 区分 | 件数 | 出処 |
|---|---|---|
| kernel 公開関数（`KERNEL_API.md §1`） | 表示 7 + CSV 1 + 結果 3 + 確認待機 3 + 権限 2 = **16+ 公開関数** | KERNEL_API.md |
| kernel 共通関数（`common.ps1` 全体） | 90+ 関数（内部実装含む、4,371 行） | kernel__01_overview.md |
| kernel json 状態ファイル | 6 種（status / session / resume_state / async_config / art_pulse / skip_request） | kernel/json/ |
| kernel txt | 3 種（passphrase_verify / art_sentences / silence.flag） | kernel/txt/ |
| kernel csv マスタ | 5 種（categories / hostlist / workers / log_destinations / manifesto） | kernel/csv/ |
| **Standard モジュール** | **61 件** | `modules/standard/*/` ディレクトリカウント（module.csv あり 60 + windows_update 1） |
| **Extended モジュール** | **18 件** | `modules/extended/*/` ディレクトリカウント |
| Profiles | 10 ファイル（+ `easy_template/easyprofile.csv`） | profiles/*.csv（top-level 10 + easy_template 1） |
| Apps（GUI ツール群） | 9 件（fabriq_operator + 8 補助具） | apps/ |
| Commands（ユーティリティ） | 6 件（gpupdate / temp / explore_restart / diag_crypto / get_evidence / system_launcher） | commands/ |
| 公開契約 doc（本リポジトリ） | 6 件（KERNEL_API は kernel.md 経由） | fabriq__contracts__*.md |

`fabriq_operator/lib/` は **8 .ps1 で構成**（theme / session_form / apps_dialog / quickactions_dialog / dashboard_form / flex_dashboard / execution_toolbar / log_viewer）。`fabriq_operator.ps1` 自身は薄いブートストラップで、これら 8 ファイルを dot-source する。`execution_toolbar.ps1` は kernel 3.4.0 で旧 Status Monitor（別プロセス）の後継として導入された in-process 浮遊ツールバー（Skip / Gyotaq）で、`Show-ExecutionToolbar` / `Hide-ExecutionToolbar` / `Update-ExecutionToolbar` の dedicated STA Runspace 実装を担う。

## 5 シリーズ内の位置づけ

fabriq シリーズは Windows キッティング・デプロイの全工程を 5 プロジェクトに分割している：

| プロジェクト | 言語 / 形態 | 役割 | 本 fabriq との関係 |
|---|---|---|---|
| **fabriq** | PowerShell + WinForms + C# ランチャ | キッティング実行フレームワーク本体（カーネル + 79 モジュール + GUI ダッシュボード） | （本ドキュメント対象） |
| **fabriq_studio** | C# / WPF / .NET 8 | 設定編集 GUI（hostlist 編集、モジュール `_list.csv` 編集、profiles 編集、レジストリ辞書） | **パスフレーズ検証トークン生成**（fabriq 起動の必須前提）+ CSV 編集経路を提供 |
| **fabriq_evidence_manager** | C# / WPF / .NET 8 | evidence 取り込み・突合・納品エクスポート | **fabriq の `evidence_config` モジュール出力を消費**（manifest schemaVersion=1 公開契約） |
| **tonebender** | C++ / Win32 native | WinPE 内ディスクイメージ取得・復元 GUI | キッティング前段（OS 配備）。fabriq 配備後の運用とは独立 |
| **tonebender-controller** | C# / WPF + PowerShell | WinPE ISO 自動ビルドフレームワーク（tonebender を内包する起動環境を生成） | tonebender の上位ビルダ |

fabriq 単体で起動するには、**事前に fabriq_studio でパスフレーズを設定して `passphrase_verify.txt` を生成**する必要がある。本体 fabriq は Studio のバージョン・機能には依存しないが、起動前提条件としての検証トークンは Studio の責任範囲。

evidence の流れ：

```
fabriq の evidence_config モジュール（standard）
   │
   ▼
{evidence_base}/{collection_dir}/pc_information/manifest.json + 30+ ファイル出力
   │
   ▼
extended/log_uploader モジュールが集約サーバへアップロード
   │
   ▼
fabriq_evidence_manager がフリート単位で取り込み → 突合 → 納品 Excel 出力
```

manifest.json の形式は kernel `EVIDENCE_MANIFEST.md schemaVersion=1` で公開契約化されており、本リポジトリ内では [fabriq__contracts__evidence_manifest_contract.md](fabriq__contracts__evidence_manifest_contract.md) と [fabriq_evidence_manager__contracts__manifest_schema.md](fabriq_evidence_manager__contracts__manifest_schema.md) が producer / consumer 両側を文書化済み。

## 公開契約（外部 consumer 向け）

fabriq が **本体外から依存可能と宣言する公開サーフェス** は以下 6 系統。これらの破壊的変更は kernel SemVer の MAJOR/MINOR 昇格を伴う。

| 契約 | doc | 内容 |
|---|---|---|
| kernel API | （`kernel/KERNEL_API.md` を本体に保持、本リポジトリでは [fabriq__kernel__02_public_api.md](fabriq__kernel__02_public_api.md) で要約） | モジュール開発者向け公開関数 / グローバル変数 / 環境変数 |
| Profile CSV スキーマ | [fabriq__contracts__profile_csv_schema.md](fabriq__contracts__profile_csv_schema.md) | 7 列定義 + 特殊マーカー |
| ModuleResult | [fabriq__contracts__module_result.md](fabriq__contracts__module_result.md) | モジュールが pipeline で返す結果オブジェクトの 5 フィールド |
| 特殊マーカー | [fabriq__contracts__special_markers.md](fabriq__contracts__special_markers.md) | `__AUTOPILOT__` / `__RESTART__` / `__ASYNC__` 他 |
| Host 環境変数 | [fabriq__contracts__host_environment.md](fabriq__contracts__host_environment.md) | `SELECTED_*` 群 + `FABRIQ_*` |
| 更新オーバーレイ | [fabriq__contracts__overlay_contract.md](fabriq__contracts__overlay_contract.md) | `dev/framework_overlay_rules.json` を介した本体テンプレート上書きポリシー |
| Evidence Manifest | [fabriq__contracts__evidence_manifest_contract.md](fabriq__contracts__evidence_manifest_contract.md) | `pc_information/manifest.json` schemaVersion=1（外部 consumer = fabriq_evidence_manager 向け） |

## ホット領域（2026 春の活発領域）

直近 2 週間（〜 2026-05-07）の git log で commit が集中している領域：

| 領域 | 内容 | 関連 doc |
|---|---|---|
| **extended/pianist** | 2026-05-02 v1.0.0 → CHANGELOG `[Unreleased]` v1.6.0、9 commits / 2 weeks。Phase 階層 + Stop/Pause/Speed 制御ボタンを追加中 | [fabriq__modules__pianist.md](fabriq__modules__pianist.md) |
| **kernel 3.2.x FlexProfile** | 3.2.0 で Profile CSV `Group` 列導入 + FlexProfile Groups バー（kernel 3.2.2 で cyan tint 撤去） | [fabriq__kernel__03_orchestration.md](fabriq__kernel__03_orchestration.md) |
| **evidence_config §27〜§31** | 2026-04-30 inventory 拡張（環境変数 / Startup / Memory / PnP / Hardware Identifiers）。consumer 側 fabriq_evidence_manager v3.8.0 で受領済み | [fabriq__modules__evidence_config.md](fabriq__modules__evidence_config.md) + [fabriq_evidence_manager__contracts__section_dispatch.md](fabriq_evidence_manager__contracts__section_dispatch.md) |
| **fabriq_ios** | apps/fabriq_ios v0.3.5（独立 SemVer）、芸術部門サブプロジェクト。? help / inline 表示の改善（2026-04-30） | [fabriq__apps__02_fabriq_ios.md](fabriq__apps__02_fabriq_ios.md) |
| **typo 修正** | `FrexProfile → FlexProfile` 改名（2026-05-04） | （既存 docs に旧表記が残っている可能性、点検対象） |

## このドキュメントセットの読み方

本リポジトリ `e:\fabriq_doc\` の `fabriq__` プレフィックス付き md は **100+ 件**。新規参画者向けの推奨読書順は：

| 順 | カテゴリ | 代表 doc | 目的 |
|---|---|---|---|
| 1 | overview | **本ドキュメント** | プロジェクト全体像 + 5 シリーズ位置づけ |
| 2 | kernel | [fabriq__kernel__01_overview.md](fabriq__kernel__01_overview.md) | カーネルとは何か（11 章立ての §01） |
| 3 | kernel | [fabriq__kernel__02_public_api.md](fabriq__kernel__02_public_api.md) | 公開 API（モジュール開発の前提） |
| 4 | kernel | [fabriq__kernel__03_orchestration.md](fabriq__kernel__03_orchestration.md) | 一括実行・FlexProfile・再起動跨ぎ |
| 5 | apps | [fabriq__apps__00_apps_overview.md](fabriq__apps__00_apps_overview.md) | GUI ツール群 9 件のカタログ |
| 6 | apps | [fabriq__apps__01_fabriq_operator_dashboard.md](fabriq__apps__01_fabriq_operator_dashboard.md) | メインダッシュボード詳細 |
| 7 | modules | [fabriq__modules__00_modules_overview.md](fabriq__modules__00_modules_overview.md) | 標準 61 + 拡張 18（計 79）のカテゴリ別一覧 |
| 8 | contracts | `fabriq__contracts__*` 6 件 | 開発時に守る公開契約 |
| 9 | profiles | [fabriq__profiles__00_profiles_overview.md](fabriq__profiles__00_profiles_overview.md) | 既存プロファイル 10 件（+ `easy_template/easyprofile.csv`）の用途 |
| 10 | apps（任意） | [fabriq__apps__02_fabriq_ios.md](fabriq__apps__02_fabriq_ios.md) など | 芸術部門サブプロジェクトの読み物 |

kernel は **11 章立て**（`fabriq__kernel__01_overview` 〜 `fabriq__kernel__11_directory_layout`）で、§04 csv_encryption、§05 resume_restart、§06 status_monitor、§07 evidence_history、§08 async_execution、§09 versioning、§10 function_index、§11 directory_layout が個別 doc になっている。

本リポジトリの modules doc は **80 件**（`fabriq__modules__00_modules_overview.md` + 個別モジュール doc）で、ソース側のモジュール総数 79（standard 61 + extended 18）に対応する。本リポジトリで個別解説を必要とするモジュールは個別 doc を作成する方針で、CSV 駆動の薄いラッパに留まるモジュールは overview のテーブル行のみで足る。

apps は **6 件**（`00_apps_overview` + `01_fabriq_operator_dashboard` + `02_fabriq_ios` + `03_other_apps` + `04_dev_template_and_tooling` + `05_commands`）。

## 配布・デプロイ

- **配布**: フォルダ `E:\fabriq\` を運搬媒体（USB / ネットワーク共有）に配置するだけで配布可能（self-contained）
- **デプロイ**: 運搬媒体から対象 PC へフォルダをコピーして配置する。かつて USB→対象 PC コピーを担っていた `Deploy.bat`（媒体上で完結する PowerShell 不要のヘルパ）は **kernel 3.6.0 で廃止・削除済み**（CHANGELOG `[3.6.0]` Removed、TM t-0042。運用で一度も使用しておらず不要と判断）。`source_media.id`（MediaSerial Priority 2）の唯一の生成元だったが、消費側は `Get-VolumeSerial` フォールバック付き（`common.ps1` / `main.ps1`）のため MediaSerial に影響なし
- **本体起動**: 対象 PC 上で `Fabriq.exe` をダブルクリック → UAC 自動昇格 → ダッシュボード表示
- **更新オーバーレイ**: `dev/framework_overlay_rules.json` を介した本体テンプレート部分上書き（fabriq_studio 側の `FabriqUpdateDialogViewModel` から呼ぶ。詳細は [fabriq__contracts__overlay_contract.md](fabriq__contracts__overlay_contract.md)）。なお `framework_overlay_rules.json` の kernel bundle `includePaths` には削除後も `Deploy.bat` の entry が残存している（ソース側の未整理。実ファイルは 3.6.0 で削除済み）

## サードパーティ

本リポジトリ `E:\fabriq` は以下のサードパーティ製ソフトウェアをバイナリ同梱：

- **7-Zip 25.01**（GNU LGPL v2.1+ ほか / Copyright © 1999-2025 Igor Pavlov / https://www.7-zip.org/）— `modules/standard/printer_driver_config/tools/` に `7z.exe` / `7z.dll` を同梱（プリンタドライバ展開用）

各コンポーネントの著作権・ライセンス条件は本体 `THIRD_PARTY_NOTICES.md` および `LICENSES/` を参照。本体 fabriq そのものは [MIT License](https://opensource.org/licenses/MIT)。

## 関連ドキュメント

### kernel / モジュール開発

- カーネル詳細: [fabriq__kernel__01_overview.md](fabriq__kernel__01_overview.md)（11 章立ての §01）
- 公開 API: [fabriq__kernel__02_public_api.md](fabriq__kernel__02_public_api.md)
- 関数索引: [fabriq__kernel__10_function_index.md](fabriq__kernel__10_function_index.md)
- ディレクトリ構成: [fabriq__kernel__11_directory_layout.md](fabriq__kernel__11_directory_layout.md)
- モジュール一覧: [fabriq__modules__00_modules_overview.md](fabriq__modules__00_modules_overview.md)

### 公開契約

- 6 contracts docs（`fabriq__contracts__*.md`）

### apps / 操作

- apps カタログ: [fabriq__apps__00_apps_overview.md](fabriq__apps__00_apps_overview.md)
- メインダッシュボード: [fabriq__apps__01_fabriq_operator_dashboard.md](fabriq__apps__01_fabriq_operator_dashboard.md)
- fabriq_ios: [fabriq__apps__02_fabriq_ios.md](fabriq__apps__02_fabriq_ios.md)

### profiles

- [fabriq__profiles__00_profiles_overview.md](fabriq__profiles__00_profiles_overview.md)

### シリーズ内他プロジェクト

- fabriq_studio: [fabriq_studio__overview__readme.md](fabriq_studio__overview__readme.md)
- fabriq_evidence_manager: [fabriq_evidence_manager__overview__readme.md](fabriq_evidence_manager__overview__readme.md)
