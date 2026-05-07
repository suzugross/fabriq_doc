# fabriq 変更履歴（注釈付き要約）

> **対象**: fabriq / 全体（kernel + 75 modules）
> **対象バージョン**: 3.2.2（取得元: `E:\fabriq\kernel\KERNEL_VERSION` + `E:\fabriq\CHANGELOG.md`）
> **ドキュメント更新日**: 2026-05-07

`E:\fabriq\CHANGELOG.md` の **公式変更履歴** を、ドキュメント読者向けに **テーマ別に整理した注釈付き要約** にしたもの。原典を置き換える意図はない（行数で 2200+ 行）。本書は**重要転換点を素早く把握する**ためのオーバービュー。

---

## バージョン体系

| プロジェクト | 版管理 | 値 |
|---|---|---|
| **kernel** | SemVer（`kernel/KERNEL_VERSION`） | 3.2.2（最新） |
| **各モジュール** | SemVer（`modules/<kind>/<name>/VERSION`） | 個別、kernel と独立 |
| **REQUIRES_KERNEL** | 各モジュールの最小要求 kernel 版 | `modules/<kind>/<name>/REQUIRES_KERNEL` |
| **CHANGELOG カテゴリ** | Keep a Changelog 1.1.0 準拠 | Added / Changed / Deprecated / Removed / Fixed / Security |

---

## 主要マイルストーン

### kernel 3.2.x — Profile CSV Group 列の導入（2026-05-02）

**FlexProfile に「グループ単位の一括実行」概念を追加**したマイナー系列。

| 版 | リリース | 概要 |
|---|---|---|
| 3.2.0 | 2026-05-02 | Profile CSV `Group` 列追加。FlexProfile dashboard 上部に **Groups バー**（`[Run: <GroupName>]` ボタン群）を追加 |
| 3.2.1 | 2026-05-02 | Group 列の **見え方を厳格化**（空欄列はバー非表示）、フォーム高さ調整 |
| 3.2.2 | 2026-05-02 | Group 列の **シアン色 tint を撤去**（視覚ノイズ削減） |

**運用への影響**: profile CSV に `Group` 列がある場合、FlexProfile dashboard で **同じ Group 値の行を 1 ボタンでまとめて実行**できる。Linear path はこの列を無視するため後方互換が保たれている。

### kernel 3.1.x — FlexProfile 系列の確立（2026-05-01〜05-02）

**state-aware execution dashboard** を導入し、Linear に加えて Flex 実行モードを正式採用したシリーズ。

| 版 | リリース | 概要 |
|---|---|---|
| 3.1.0 | 2026-05-01 | **FlexProfile 初期版**。dashboard 上で個別モジュールを再実行・スキップ可 |
| 3.1.1 | 2026-05-01 | minor fixes |
| 3.1.2 | 2026-05-02 | **state-aware execution dashboard**（実行履歴を取り込んでステータスバッジ表示） |
| 3.1.3 | 2026-05-02 | per-Order tracking + single-rerun NotRun fix（同名モジュールの状態 leak 修正） |
| 3.1.4 | 2026-05-02 | `[Complete]` ボタンの判定ロジック修正（unchecked 行の Error/Partial カウント） |
| 3.1.5 | 2026-05-02 | FlexProfile execution simplification |
| 3.1.6 | 2026-05-02 | Frex Complete ボタン: 空実行警告 |
| 3.1.7 | 2026-05-02 | **MenuName fallback の厳格化**（sibling-row leak バグ修正、`fabriq_studio` も該当） |
| 3.1.8 | 2026-05-02 | per-row `[Run]` ボタン追加（dashboard で個別再実行が容易に） |
| 3.1.9 | 2026-05-02 | Status/Verified セルの **色付きバッジ化**（Error → 赤背景、Success → 緑バッジ） |

**注意**: `Frex` という綴りは 3.1.x の途中で使われた typo。**正式名は `FlexProfile`**（commit `9e787d1` で typo 修正）。

### kernel 3.0.0 — 暗号化方式の刷新（2026-04-29）

**fabriq 全体で暗号化スキームを統一**した最重要メジャー版。

主要変更:

- **`ENC:<Base64>` インライン prefix に統一**（旧 `Encrypted` 列方式を廃止）
- **AES-256-CBC + PBKDF2-HMAC-SHA256** で**マシン非依存**の暗号化
- `kernel/txt/passphrase_verify.txt` に `surkitinisme` トークン埋込（パスフレーズ照合用）
- `Resolve-HostValue` 透過復号、`Protect-FabriqValue`/`Unprotect-FabriqValue` 公開 API

**運用への影響**: 2.x 以前の `Encrypted` 列を使った hostlist は **3.0.0 では復号できない**。マイグレーションが必要（fabriq_studio で再暗号化）。

**詳細**: [fabriq__kernel__04_csv_encryption.md](fabriq__kernel__04_csv_encryption.md) / [fabriq__troubleshooting__resume_and_state.md](fabriq__troubleshooting__resume_and_state.md) の暗号化セクション。

### kernel 2.2.x — 安定運用期（2026-04-22〜04-25）

3.0 への準備期。`ipaddress_config` の post-apply verification、profile_csv schema の固定化、各モジュールの evidence 連携強化など、**運用品質の底上げ** が中心。

### kernel 2.1.x — 並列実行 / async dispatch（2026-04-23）

profile CSV の `__ASYNC__` マーカー導入、segment 分離機能、kernel `_IsAsync` フラグの追加。**長時間実行モジュールを並列で走らせる**基盤が確立。

---

## モジュール別の主要進化

### modules/extended/pianist — RPA + 手順書ハイブリッド（独自進化）

kernel と独立したペースで **半年弱で 1.0 → 1.6** に到達した最も活発なモジュール。

| 版 | フェーズ | 概要 |
|---|---|---|
| 1.0.0 | 初版 | GUI configuration maestro。procedure.csv の Step 列挙 + Run Phase |
| 1.1.0 | wide-format | values.csv の **per-host wide format** 導入（変数を端末別に切替） |
| 1.1.1 | パッチ | Open with unquoted spaced paths 修正 |
| 1.2.0 | Phase A | **Copy Values ダイアログ**追加（Phase 内変数のクリップボードコピー） |
| 1.3.0 | Phase A 拡張 | Show-all トグル（参照外の values.csv 列も列挙）+ FlowLayoutPanel refactor |
| 1.4.0 | Phase B | **section marker DSL** 導入（`[RPA]`/`[Manual]`/`[Variables]`/`[Samples]`）+ TabControl 化 |
| 1.5.0 | Phase C | **Samples タブ** + モードレス画像ビューワ。`procedure.csv` Screenshot 列を撤去（8 列に） |
| 1.6.0 | 開発中 | **Run 中の Stop / Pause / Speed 制御ボタン** 3 種追加（`[Unreleased]` セクション、E:\fabriq では現在 WIP） |

**Pianist プロファイルの相互運用**: fabriq_studio の Pianist Profile Editor が profile を編集する。詳細は [fabriq_studio__reference__pianist_profile_schema.md](fabriq_studio__reference__pianist_profile_schema.md)。

### modules/standard 主要モジュール

- **`evidence_config`**: 3.0.0 で `kernel 3.0.0+` 対応、§27-§31 inventory sections 追加（Environment / Startup / Memory / PnP / 等）
- **`ipaddress_config`**: 2.2.1 で **Post-Apply Verification** 標準化（PrefixLength / AddressState=Preferred 等）
- **`bitlocker_config`**: 3.0.0 で ENC: PIN への切替、復号失敗時 Error
- **`windows_update`**: 標準 vs standalone 分離、reboot loop の per-CSV 設定（MaxRebootLoops / SkipKBs）
- **`autologon_config`**: WU loop と連携、autologon_list.csv で per-host 設定

各モジュールの詳細仕様は `fabriq__modules__<name>.md` を参照。

---

## 後方互換性ルール

### 破壊的変更が許される場面

- **メジャー版繰り上がり時のみ**（kernel 2.x → 3.x の暗号化変更が代表例）
- マイナー版・パッチ版では **破壊的変更を避ける**（追加・修正のみ）

### 破壊的変更時の扱い

- CHANGELOG に **`Removed`** カテゴリ + **`Security`** カテゴリで明示
- 該当モジュールの `REQUIRES_KERNEL` を更新
- migration guide を `dev/` 配下に配置（kernel 2.x → 3.x の `dev/migration_to_3_0.md` 等）

### schemaVersion 規約

主要状態ファイルが採用:

- **`resume_state.json`**: schemaVersion なし (= v1) / 2 (Flex)
- **`framework_overlay_rules.json`**: schemaVersion=1（Studio が消費、不一致時 拒否）
- **`evidence` manifest.json**: schemaVersion 別途管理（[fabriq__contracts__evidence_manifest_contract.md](fabriq__contracts__evidence_manifest_contract.md)）

---

## 直近の修正で覚えておくべき項目

`E:\fabriq\CHANGELOG.md` から特に **運用者の挙動が変わる** 修正をピックアップ:

| 版 | 修正 | 影響 |
|---|---|---|
| 3.1.7 | MenuName fallback の厳格化 | **同名モジュールが複数ある profile** での状態混線を防止 |
| 3.1.4 | Frex Complete ボタンの判定 | unchecked 行の Error も **必ずカウント**（事実保存） |
| 3.1.3 | per-Order tracking + single-rerun NotRun fix | 同 Order の再実行で旧結果を上書き |
| 3.0.0 | `Encrypted` 列廃止 → `ENC:` インライン | 2.x の encrypted hostlist は **再暗号化必須** |
| 2.2.0 | profile_csv schema 固定化 | 列名の自由度低下、ただし将来の互換性確保 |
| 2.1.0 | `__ASYNC__` マーカー導入 | profile CSV で `__ASYNC__` 行以下が並列実行される |

---

## 各種ガイドへのインデックス

| 観点 | ドキュメント |
|---|---|
| 公式 changelog 原本 | `E:\fabriq\CHANGELOG.md`（読み取り専用） |
| カーネル API | [fabriq__kernel__02_public_api.md](fabriq__kernel__02_public_api.md) |
| モジュール契約 | [fabriq__contracts__module_result.md](fabriq__contracts__module_result.md) |
| 暗号化仕様 | [fabriq__kernel__04_csv_encryption.md](fabriq__kernel__04_csv_encryption.md) |
| FlexProfile UI | [fabriq__usage__04_flexprofile_dashboard.md](fabriq__usage__04_flexprofile_dashboard.md) |
| Resume / Restart | [fabriq__kernel__05_resume_restart.md](fabriq__kernel__05_resume_restart.md) |
| Versioning 設計 | [fabriq__kernel__09_versioning.md](fabriq__kernel__09_versioning.md) |
| Pianist ホストエディタ | [fabriq_studio__reference__pianist_profile_schema.md](fabriq_studio__reference__pianist_profile_schema.md) |

---

## 変更履歴

- 2026-05-07 初版作成（kernel 3.2.2 commit `e513cf1` を対象、CHANGELOG.md 全期間（2.x 系〜3.2.2 + Pianist 1.0〜1.6 + extended/pianist Unreleased セクション）をテーマ別に注釈）
