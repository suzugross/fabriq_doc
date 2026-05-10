# fabriq 変更履歴（注釈付き要約）

> **対象**: fabriq / 全体（kernel + 76 modules）
> **対象バージョン**: 3.2.5（取得元: `E:\fabriq\kernel\KERNEL_VERSION` + `E:\fabriq\CHANGELOG.md`、commit `fed181a`、2026-05-10）
> **ドキュメント更新日**: 2026-05-10

`E:\fabriq\CHANGELOG.md` の **公式変更履歴** を、ドキュメント読者向けに **テーマ別に整理した注釈付き要約** にしたもの。原典を置き換える意図はない（行数で 2500+ 行）。本書は**重要転換点を素早く把握する**ためのオーバービュー。

---

## バージョン体系

| プロジェクト | 版管理 | 値 |
|---|---|---|
| **kernel** | SemVer（`kernel/KERNEL_VERSION`） | 3.2.5（最新） |
| **各モジュール** | SemVer（`modules/<kind>/<name>/VERSION`） | 個別、kernel と独立 |
| **REQUIRES_KERNEL** | 各モジュールの最小要求 kernel 版 | `modules/<kind>/<name>/REQUIRES_KERNEL` |
| **CHANGELOG カテゴリ** | Keep a Changelog 1.1.0 準拠 | Added / Changed / Deprecated / Removed / Fixed / Security |

---

## 主要マイルストーン

### kernel 3.2.5 — Pester v5 テストスイート整備（2026-05-10）

**production code を一切触らず**、kernel ユニットテスト基盤を Phase 0〜4 で組み上げた純粋追加リリース。`KERNEL_API.md` 不変、公開 API 不変。

| ファイル | 内容 |
|---|---|
| `tests/_helpers/test_state.ps1` | `Get-FabriqRepoRoot` / `Set-FabriqTestState`（`$script:ResumeStatePath` 等を test から override 可能に） |
| `tests/_helpers/test_csv.ps1` | `New-TestProfileCsv` / `Remove-TestProfileCsv` / `New-MockModule`。一時 Profile CSV は `Resolve-ProfileModules` の `Import-Csv -Encoding Default` に合わせ ASCII 出力（JP locale CP932 環境の BOM 誤読を回避） |
| `tests/kernel/Resolve-ProfileModules.tests.ps1`（Phase 0/1） | 6 Context / 14 ケース。過去 regression（kernel 3.0.0 / 3.1.3 / 3.1.7 / 3.2.0 / 2.1.0）を pin |
| `tests/kernel/New-BatchResult.tests.ps1`（Phase 1b） | 4 Context / 18 ケース。Status 自動判定 5x3 マトリクス全分岐 |
| `tests/kernel/Import-ModuleCsv.tests.ps1`（Phase 1b） | 4 Context / 14 ケース。Segment 厳密マッチ 5 ケース、空集合 → `$null` の caller-observable 挙動を pin |
| `tests/kernel/ResumeState.tests.ps1`（Phase 2） | 4 Context / 15 ケース。Linear (v1) / Flex (v2) 出力契約 + round-trip + load 境界 |
| `tests/kernel/New-ModuleResult.tests.ps1`（Phase 3） | 5 Context / 19 ケース。`KERNEL_API.md §1.3 / §5` 直接 pin |
| `tests/kernel/Confirm-Execution.tests.ps1`（Phase 3） | 3 Describe / 14 ケース。AutoPilot + AutoConfirmMode 二段短絡契約 |
| `tests/kernel/Set-SelectedHostEnvironment.tests.ps1`（Phase 4） | 16 ケース |
| `dev/run_tests.ps1` | Pester v5+ runner（v3.4.0 OS 同梱版は不可、`-SkipPublisherCheck` 必須） |

**運用への影響**: 開発者向け。AI / 開発者が CHANGELOG に挙げた regression を未来検出できるようになった。本番動作には影響なし。

### kernel 3.2.4 — Verbose stream capture + Telemetry 拡張（2026-05-10）

Telemetry レイヤを実用化する 2 つの拡張を同日リリース。**`KERNEL_API.md` 公開 API 不変**だが、release sync 対象に `KERNEL_API.md` L3「Current Kernel Version」ヘッダを追加（過去に 2 release ぶん drift した実績への対応、`dev/check_version.ps1` で検証）。

**(1) Verbose stream capture**（`cmdlet.verbose` チャネル、デフォルト ON）:

- `kernel/json/verbose_capture.flag` を **git tracked で同梱**（標準配備でデフォルト有効）
- 起動時 `Enable-FabriqVerboseCapture` が flag 存在で `$global:FabriqVerboseCaptureActive=$true` をセット
- `Invoke-SafeCommand` がモジュール実行を `$VerbosePreference='Continue'` + `$PSDefaultParameterValues['*:Verbose']=$true` + `4>&1` redirect で wrap、`[VerboseRecord]` を `cmdlet.verbose` イベントに変換
- 取得対象: 組み込み cmdlet の `ShouldProcess` 経由 verbose（`Set-ItemProperty` 等の "Performing the operation..." 行）。外部プロセス（winget / robocopy / dism）と `Invoke-SafeCommandAsync` 経由は対象外
- **Process restart 不要**（先行検討した Module Logging E1 trial は registry 書き換えが late-load module をカバーできなかった反省、`dev/TELEMETRY_INTERNAL.md` §6.7 が経緯を明文化）
- Opt-out: flag 削除のみ（`logs/telemetry/` も `log_uploader` `/XD` 除外で外に出ない、local-only）

**(2) Telemetry coverage 拡張**:

- **`csv.load` イベント**: `Import-ModuleCsv` 正常 return 時に `{fileName, path, totalRows, returnedRows, filterEnabled, segment, columns}` を発行（CSV 値含まず）
- **Profile context フィールド**: `envelope.start` に `profileName`, `profileOrder`, `executionMode` (Linear/Flex), `prevModuleName`, `prevModuleStatus` 追加（`Invoke-BatchExecution` が `$global:_FabriqCurrentProfileContext` を per-module でセット）
- **`_meta.json` host info**: `Win32_OperatingSystem` + `Win32_ComputerSystem` を CIM で 1 セッション 1 回キャッシュ。`manufacturer` / `model` はフリート単位識別子として **redact せず raw 保存**
- **`_kernel.jsonl` チャネル**: `Write-KernelTelemetryEvent` 新設、session-lifecycle 5 イベント（`profile.start/end`, `restart.invoked`, `resume.consumed`, `finalize.start/end`）を発行

**運用への影響**: モジュール側無変更、副次副作用としてバックグラウンド telemetry が動く。詳細は [fabriq__kernel__12_telemetry.md](fabriq__kernel__12_telemetry.md)。

### kernel 3.2.3 — AI 開発テレメトリ層 + Status Monitor 起動診断（2026-05-09）

fabriq の挙動を構造化 JSONL に蓄積する **AI 開発コーパス機構** を新設。**KERNEL_API.md には未昇格**（dev/TELEMETRY_INTERNAL.md で内部設計を集約）。

**Telemetry レイヤ**:

- 新規関数群: `Get-TelemetrySalt`, `New-TelemetryRedactMap`, `Invoke-TelemetryRedact`, `Write-TelemetryEvent`, `Start-ModuleTelemetry`, `Complete-ModuleTelemetry`, `_GetShowTag`, `_TrackShowEvent`, `_HashTelemetryValue`
- Show-* 5 関数（Show-Info / Success / Warning / Error / Skip）に副次副作用追加（戻り値・出力ストリーム・コンソール表示は不変）
- `Invoke-SafeCommand` / `Invoke-SafeCommandAsync` の entry / exit に envelope hook
- プライバシ契約: salt 付き SHA-256（先頭 12 hex）でハッシュ化、`SELECTED_PIN` のみ `[REDACTED]` 固定。Salt は `kernel/json/telemetry_salt.txt`（256 bit、初回自動生成、`.gitignore`、site-specific）
- Scope 設計: telemetry 内部状態は `$global:` 保持（`$script:` だと Show-* がモジュール側 script scope を参照する PowerShell 動的 scope 仕様の罠を回避）
- 子 runspace（`__ASYNC__`）でも envelope 共有（inject hashtable 経由）
- `log_uploader` v1.0.0 → v1.1.0（MINOR）: robocopy `/XD logs\telemetry` 除外を追加し「テレメトリは AI 開発専用、PC 外持ち出し禁止」契約を実装で担保
- 設計詳細は `dev/TELEMETRY_INTERNAL.md`（schemaVersion=1）

**Status Monitor 起動診断ログ**:

- 親側（`Start-StatusMonitor`）が `logs/status_monitor_<ts>.log` を 1 セッション 1 ファイル生成、`-DiagLogPath` 引数で子に渡す
- 子側（`status_monitor.ps1`）が `Write-DiagLog` ヘルパで起動チェーン全段（DPI / WinForms / common.ps1 dot-source / Form 生成 / Application.Run）にログ
- Exit codes 11〜14 で失敗箇所を識別（System.Drawing / WinForms / common.ps1 / Application.Run）
- **`NoActivateForm` Add-Type 失敗時の plain Form fallback**（AMSI / `csc.exe` パス問題等）
- **DpiX 病的値ガード**（0 / 負を 96 fallback 化、`dpiScale<=0` を 1.0 にクランプ）
- **ウィンドウ存在ポーリング判定**（最大 4 秒、200ms 間隔。`MainWindowTitle` に `"Fabriq"` が現れたかで主判定。App Control 配備済端末で `HasExited` 即死チェックが取りこぼす事例への対応）

**併走モジュール変更**: `sysprep_config` v1.0.0 → v1.1.0（MINOR、generalize-pass driver 設定の variabilize、詳細は [fabriq__modules__sysprep_config.md](fabriq__modules__sysprep_config.md)）

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
