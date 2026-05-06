# fabriq_doc — Fabriqシリーズ統合ドキュメント管理プロジェクト

本プロジェクトは、Windowsキッティング・デプロイ用汎用フレームワーク「**Fabriqシリーズ**」5プロジェクトの技術仕様書・利用方法・スタートアップガイドを **一元管理・整備** するためのドキュメント専用リポジトリである。

**`e:\fabriq_doc\` 配下のみが書き込み可能**。配下5プロジェクトのソースは **完全 read-only** として扱う。

ファイル配置はトップレベル **完全フラット**。すべての md は `e:\fabriq_doc\` 直下に置き、ファイル名のプレフィックスでプロジェクトとカテゴリを判別する。NotebookLM 等のフラット投入を一次運用とするため、ディレクトリ階層には依存しない。

---

## 対象プロジェクト一覧

| プロジェクト | パス | 接頭辞 | 役割 | 言語 / 形態 | バージョン情報源 |
|---|---|---|---|---|---|
| **fabriq** | `E:\fabriq` | `fabriq__` | キッティング実行フレームワーク本体（カーネル + 75 モジュール + GUI ダッシュボード） | PowerShell + C# ランチャ | `kernel/KERNEL_VERSION` + per-module `VERSION` + `CHANGELOG.md` |
| **fabriq_evidence_manager** | `E:\fabriq_evidence_manager` | `fabriq_evidence_manager__` | エビデンス収集・整形・納品エクスポートツール | C# / WPF（.NET 8） | `FabriqEvidenceManager.csproj <Version>` |
| **fabriq_studio** | `E:\fabriq_studio` | `fabriq_studio__` | fabriq 本体の管理GUI・ホスト/モジュール/プロファイル編集・Pianist Profile Editor | C# / WPF（.NET 8） | `FabriqStudio.csproj <Version>` または `git log` 短縮ハッシュ |
| **tonebender** | `E:\tonebender` | `tonebender__` | WinPE 内ディスクイメージ取得/復元 GUI ツール | C++ / Win32 native | `git log` 短縮ハッシュ |
| **tonebender-controller** | `E:\tonebender-controller` | `tonebender_controller__` | WinPE ISO 自動ビルドフレームワーク（Tonebender を内包する起動環境を生成） | C# / WPF（.NET 8） + PowerShell | `ToneBenderController.csproj <Version>` または `git log` 短縮ハッシュ |

> **接頭辞ルール**: ソースリポジトリ名のハイフンはアンダースコアに変換する（`tonebender-controller` → `tonebender_controller__`）。区切り文字 `__` の意味を保つため。

---

## 絶対遵守事項（クリティカル・ルール）

### R1. ソースの改変は禁止（最優先）

- 5プロジェクト（`E:\fabriq`, `E:\fabriq_evidence_manager`, `E:\fabriq_studio`, `E:\tonebender`, `E:\tonebender-controller`）配下のファイルを **絶対に編集・追加・削除しない**。
- 5プロジェクト配下には **Read / Glob / Grep のみ** 使用する。Edit / Write / NotebookEdit / Bash の書き込み系操作は禁止。
- `git commit` / `git push` / ブランチ操作などソース側の状態を変える操作も禁止（`git log` / `git status` / `git diff` などの参照系のみ可）。
- ソースに修正提案がある場合は、本リポジトリ内の md に「提案」として記録するに留める。実適用は各プロジェクト側で別途行う。

### R2. ディレクトリは作らない（完全フラット構造）

- すべての md は **`e:\fabriq_doc\` 直下** に配置する。**プロジェクト別ディレクトリも切らない**。
- ディレクトリでのグルーピングは **ファイル名の `<project>__<category>__<name>.md` 形式で代替** する（後述 R4）。
- 階層が無いことで:
  - NotebookLM 等にフォルダ全体をフラット投入したとき、リポジトリと同じ basename で全ファイルが揃う
  - INDEX.md の相対パスリンクが両環境（VSCode / NotebookLM）で同一に機能する
  - `<project>__` プレフィックスで basename が全リポジトリに対しユニークになり、衝突しない
- 以前存在した `fabriq/` サブディレクトリは廃止済み。**復活させてはならない**。

### R3. 許容されるトップレベル構造

```
e:\fabriq_doc\
├── CLAUDE.md                              ── 本ファイル（運用ルール）
├── INDEX.md                               ── 全プロジェクト統合インデックス（必須）
├── README.md                              ── プロジェクト概要（任意・新規作成可）
│
├── fabriq__kernel__01_overview.md         ── ★以下、すべてフラットに並ぶ
├── fabriq__kernel__02_public_api.md
├── fabriq__modules__pianist.md
├── fabriq__contracts__module_result.md
├── fabriq_studio__apps__01_host_editor.md
├── fabriq_evidence_manager__architecture__01_layers.md
├── tonebender__usage__01_image_capture.md
├── tonebender_controller__usage__03_winpe_build.md
└── 【未完成けどギリギリ使える】Fabriqスタートアップガイド.docx  ── 既存資料
```

- `_archive/` `_drafts/` 等の補助ディレクトリも作らない。古い版は `git log` で追跡する。
- 既存の docx 等の非 md 資料はトップに置いてよい（フラット原則の対象は md のみ）。

### R4. ファイル命名規則 — `<project>__<category>__<name>.md`（3層、ダブルアンダースコア区切り）

#### 形式

```
<project>__<category>__NN_<name>.md     ← 順序が意味を持つもの（章立て、概要文書）
<project>__<category>__<name>.md        ← 順序を持たないもの（個別モジュール解説 等）
```

- **区切り文字はダブルアンダースコア `__`**（シングルではない）。階層 3 つを `__` で結ぶ。
- `<project>` 部はソースリポジトリ名から導出（ハイフンはアンダースコアに変換 / `<対象プロジェクト一覧>` の接頭辞列を参照）。
- `<category>` 部は次表の語彙から選ぶ。複数語は `_`（シングル）で結合。
- `<name>` 部にハイフン（`-`）は使わない。
- 章立てが必要な系列文書には `<name>` 先頭に `NN_`（ゼロパディング2桁）を付与（例: `fabriq__kernel__01_overview.md`）。

#### カテゴリ語彙表

| カテゴリ | 用途 | 例 |
|---|---|---|
| `overview` | プロジェクト全体像・README | `<project>__overview__readme.md` |
| `kernel` | コア / フレームワーク本体（fabriq 専用） | `fabriq__kernel__01_overview.md` |
| `modules` | 機能モジュール毎の解説（fabriq 専用） | `fabriq__modules__hostname_config.md` |
| `apps` | サブアプリ・GUI ツール解説 | `fabriq_studio__apps__01_host_editor.md` |
| `architecture` | 設計上の思想・レイヤ構成 | `fabriq_evidence_manager__architecture__01_layers.md` |
| `contracts` | 公開 API・スキーマ・契約 | `fabriq__contracts__module_result.md` |
| `profiles` | プロファイル / 設定ファイル仕様 | `fabriq__profiles__00_profiles_overview.md` |
| `usage` | 使い方・操作手順 | `tonebender__usage__01_image_capture.md` |
| `startup` | 導入・初期セットアップ | `<project>__startup__01_install.md` |
| `troubleshooting` | トラブル対応・既知の問題 | `<project>__troubleshooting__common_errors.md` |
| `reference` | 参照表・一覧（関数索引・列挙値など） | `fabriq__kernel__10_function_index.md` |
| `changelog` | 変更履歴のミラー / 注釈付き要約 | `<project>__changelog__history.md` |

新カテゴリを足す場合は本表と [INDEX.md](INDEX.md) のカテゴリ語彙セクションを **同時更新** すること。

### R5. バージョン追跡（ドキュメント先頭にスナップショットを明記）

ソース側のバージョンと本ドキュメントの整合を保つため、**全ドキュメントの先頭** に以下のメタブロックを置く：

```markdown
> **対象**: <project_name> / <area>
> **対象バージョン**: <semver or commit hash>（取得元: `<相対パス>` / `git rev-parse HEAD`）
> **ドキュメント更新日**: YYYY-MM-DD
```

#### 例

```markdown
> **対象**: fabriq / kernel
> **対象バージョン**: 3.2.2（取得元: `E:\fabriq\kernel\KERNEL_VERSION`）
> **ドキュメント更新日**: 2026-05-06
```

```markdown
> **対象**: fabriq_evidence_manager / 全体
> **対象バージョン**: 3.8.0（取得元: `E:\fabriq_evidence_manager\FabriqEvidenceManager\FabriqEvidenceManager.csproj` `<Version>`）
> **ドキュメント更新日**: 2026-05-06
```

```markdown
> **対象**: tonebender / 全体
> **対象バージョン**: commit fb86baa（取得元: `git -C E:\tonebender rev-parse --short HEAD`）
> **ドキュメント更新日**: 2026-05-06
```

#### バージョン情報源の優先順位

1. プロジェクトが正式 SemVer ファイル / プロパティを持つ場合はそれを採用（`KERNEL_VERSION` / csproj `<Version>` / モジュール `VERSION`）。
2. 持たない場合は **最新コミットの短縮ハッシュ**（`git rev-parse --short HEAD`）。
3. 加えて、明確な機能差分が日付ベースの場合は最新コミット日（`git log -1 --format=%cd --date=short`）も併記してよい。

### R6. ドキュメント新規作成・更新フロー

1. **読む**: 対象プロジェクトのソース・既存 md・git 状態を Read / Glob / Grep のみで読み取る。
2. **対象バージョンを特定**: R5 のメタブロック用に正確な版/ハッシュを取る。
3. **既存ドキュメントを優先更新**: 同テーマの既存 md があれば編集を検討。新規作成は重複を避ける。
4. **ファイル名を R4 規約で確定**（`<project>__<category>__<name>.md`、トップ直下に配置）。
5. **メタブロック付きで書く**。既存 fabriq__ 系ドキュメントの語り口・章構成（見出し階層・表形式）を踏襲する。
6. **[INDEX.md](INDEX.md) を更新**: 該当プロジェクトの該当カテゴリ節にエントリを追加。プロジェクト別ファイル数も更新。
7. **書き終えたら git で差分確認**。`fabriq_doc` 側のみが変更されているか必ず確認（`git status` で他プロジェクト側に変更が及んでいないこと）。

### R7. ドキュメントの記述スタイル

- **言語**: 既存の `fabriq__*` 系は日本語ベース + コード/識別子は英語混在。**この体裁を踏襲**する（tonebender 系もコード識別子は英語、説明は日本語で揃える）。
- **見出し階層**: `#` はファイルタイトル1個のみ。本文は `##` から開始。
- **表**: 「列が多い・レンダリング崩れの懸念」が出たら箇条書きへフォールバックする。
- **コードブロック**: PowerShell は ```` ```powershell ````、C# は ```` ```csharp ````、ディレクトリツリーは ```` ``` ```` のみ。
- **絵文字は使わない**。
- **絶対パス** は `E:\fabriq\...` のように Windows 表記で統一。コマンド例の中は文脈に従う。

### R8. リンクと INDEX

- すべての md がトップ直下にあるため、**md 内リンクは basename のみ** で書く: `[link](fabriq__kernel__01_overview.md)`。
- これにより:
  - VSCode / GitHub での **クリック可能な相対リンク** として機能
  - NotebookLM 等にフラット投入したときも **同じ basename** で参照解決される
- `INDEX.md` はトップ直下の単一ファイル。**新規 md を作るたびに必ず INDEX.md にエントリを追記** する（R6.6）。
- INDEX.md はプロジェクト別 + カテゴリ別の二段グルーピングで一覧化。サンプル構造は現状の [INDEX.md](INDEX.md) を参照。

---

## 既存 fabriq__\* ドキュメントの取り扱い

- すでに **100 ファイル** が `fabriq__<category>__<name>.md` 形式で整備済み（旧 `fabriq/` サブディレクトリから 2026-05-06 にフラット化リネーム済み、git 履歴は保持）。**新規プロジェクト分のドキュメントを書く際はここを必ず先に読み、文体・構成を踏襲する**こと。
- 代表的な踏襲対象:
  - 章立て型: `fabriq__kernel__01_overview.md` ～ `fabriq__kernel__11_directory_layout.md`
  - 概要型: `fabriq__modules__00_modules_overview.md`, `fabriq__apps__00_apps_overview.md`
  - 個別解説型: `fabriq__modules__pianist.md`, `fabriq__modules__bitlocker_config.md`
  - 契約定義型: `fabriq__contracts__module_result.md`, `fabriq__contracts__overlay_contract.md`
- 既存 md にメタブロック（R5）が付与されていない場合がある。**触ったときに最低限該当ファイル先頭にメタブロックを補完する**（既存内容は不要に編集しない）。

---

## 各プロジェクト別の推奨初期構成（実装ガイド）

すべてトップ直下に `<project>__` プレフィックス付きで配置する。

### fabriq_evidence_manager

- `fabriq_evidence_manager__overview__readme.md`
- `fabriq_evidence_manager__architecture__01_layers.md` — MVVM 構造 / DI / Service 層
- `fabriq_evidence_manager__architecture__02_evidence_input.md` — 入力となる evidence ディレクトリ構造
- `fabriq_evidence_manager__usage__01_import.md` — 取り込み手順
- `fabriq_evidence_manager__usage__02_export.md` — 納品エクスポート手順
- `fabriq_evidence_manager__reference__csv_schema.md` — manifest.csv / exec_history.csv の取り扱い
- `fabriq_evidence_manager__changelog__history.md` — csproj `<Version>` 履歴の追跡

### fabriq_studio

- `fabriq_studio__overview__readme.md`
- `fabriq_studio__architecture__01_layers.md`
- `fabriq_studio__apps__01_host_editor.md`
- `fabriq_studio__apps__02_module_editor.md`
- `fabriq_studio__apps__03_pianist_profile_editor.md`
- `fabriq_studio__apps__04_registry_dictionary.md`
- `fabriq_studio__usage__01_workspace_setup.md`
- `fabriq_studio__usage__02_overlay_update.md` — オーバーレイ更新（fabriq 本体への適用）
- `fabriq_studio__reference__appsettings_schema.md`

### tonebender

- `tonebender__overview__readme.md`
- `tonebender__architecture__01_winpe_runtime.md`
- `tonebender__usage__01_image_capture.md`
- `tonebender__usage__02_image_apply.md`
- `tonebender__usage__03_autopilot_recovery.md`
- `tonebender__reference__cli_options.md`

### tonebender-controller（接頭辞は `tonebender_controller__`）

- `tonebender_controller__overview__readme.md`
- `tonebender_controller__architecture__01_pipeline.md` — ADK → WinPE → ToneBender 同梱の流れ
- `tonebender_controller__usage__01_profile_json.md`
- `tonebender_controller__usage__02_usb_partitioning.md`
- `tonebender_controller__usage__03_winpe_build.md`
- `tonebender_controller__usage__04_oem_driver_injection.md`
- `tonebender_controller__reference__profile_schema.md`

これらは **目安** であり、調査の進捗に応じて R4 の規約内で増減する。

---

## 自動化補助（任意・参考）

バージョン取得を機械的に行うためのワンライナ集（読み取り専用）:

```powershell
# fabriq カーネル版
Get-Content E:\fabriq\kernel\KERNEL_VERSION

# fabriq モジュール毎の版（一覧）
Get-ChildItem E:\fabriq\modules -Recurse -Filter VERSION |
    ForEach-Object { "{0,-40} {1}" -f ($_.Directory.Name), (Get-Content $_.FullName) }

# csproj <Version> 抽出（fabriq_evidence_manager / fabriq_studio / tonebender-controller）
Select-String -Path E:\fabriq_evidence_manager\FabriqEvidenceManager\FabriqEvidenceManager.csproj -Pattern '<Version>'
Select-String -Path E:\fabriq_studio\FabriqStudio\FabriqStudio.csproj -Pattern '<Version>'
Select-String -Path E:\tonebender-controller\src\ToneBenderController\ToneBenderController.csproj -Pattern '<Version>'

# 各プロジェクトの最新コミット（短縮ハッシュ + 日付）
'E:\fabriq','E:\fabriq_evidence_manager','E:\fabriq_studio','E:\tonebender','E:\tonebender-controller' |
    ForEach-Object { "{0}: {1}" -f $_, (git -C $_ log -1 --format='%h %cd' --date=short) }
```

---

## 違反時の取り扱い

- R1（ソース改変禁止）違反は **致命的**。誤って書き込み系ツールを発火しそうになった場合は **即停止し、ユーザに確認** すること。
- R2（フラット構造）違反 = 新規ディレクトリ作成を検知したら、**作る前に提案して承認を得る**。例外を認める場合は本 CLAUDE.md に明記する。

---

## 参考: ライセンスとサードパーティ

各プロジェクトの LICENSE / THIRD_PARTY_NOTICES は **本リポジトリには複製しない**（ライセンス文の分散管理は崩壊しやすいため）。必要時は元プロジェクトを参照し、本リポジトリのドキュメントからは相対参照のみ行う。
