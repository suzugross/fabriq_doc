# カーネル全体像

> **対象**: fabriq / kernel
> **対象バージョン**: kernel 3.6.0（取得元: `E:\fabriq\kernel\KERNEL_VERSION`）+ commit `0fca159`（取得元: `git -C E:\fabriq rev-parse --short HEAD`、2026-06-16）
> **ドキュメント更新日**: 2026-06-16

**現行版**: `kernel/KERNEL_VERSION` = `3.6.0`

## カーネルとは何か

`kernel/` は fabriq フレームワークの**運転系**である。モジュールを動かす実行基盤・GUI・状態管理・暗号化・ロギング・エビデンス取得・再起動跨ぎ・更新オーバーレイ契約のすべてを抱え込む層であり、モジュール側からは「公開 API」（`KERNEL_API.md`）で宣言されたサーフェスのみが利用可能。

**役割の分担**:

```
Fabriq.exe (C#ランチャ)
   ↓ UAC 自動昇格
   ↓ 管理者権限の PowerShell コンソールを開く
kernel/main.ps1
   ↓ . source
kernel/common.ps1（90+ 関数の共通ライブラリ）
   ↓ dot-source
apps/fabriq_operator/fabriq_operator.ps1（WinForms ダッシュボード）
   ↓ 操作
modules/{standard,extended}/<name>/<name>.ps1（実モジュール）
```

## カーネルが担う責務（カテゴリ別）

| カテゴリ | 責務 | 主な関数・ファイル |
|---|---|---|
| **エントリポイント** | UAC 昇格 → 管理者権限取得 → main.ps1 起動 → GUI ダッシュボードへ受け渡し | `Fabriq.exe`（C#）, `kernel/main.ps1` |
| **共通ライブラリ** | 表示・確認ダイアログ・CSV 読み込み・結果オブジェクト生成・暗号化・履歴・エビデンス取得・スリープ抑制・コンソール制御の共通関数群（90+） | `kernel/common.ps1` |
| **モジュールシステム** | `modules/{standard,extended}/<name>/module.csv` の自動検出・カテゴリ別グルーピング・カテゴリ順序管理 | `Initialize-ModuleSystem`, `Build-CategoryMenu`, `kernel/csv/categories.csv` |
| **プロファイル解決** | profile CSV の `Order` / `ScriptPath` / `Enabled` / `Segment` / `ErrorMode` / `Group` を読み込み、特殊マーカーを解釈してモジュールリストへ展開 | `Resolve-ProfileModules`, `Load-Profiles` |
| **オーケストレーション** | プロファイル一括実行・モジュール単発実行・AutoPilot ループ・FlexProfile sub-loop・Windows Update リブートループ | `Invoke-BatchExecution`, `Invoke-KittingScript`, `Invoke-FlexProfileLoop`, `Invoke-WindowsUpdateLoop` |
| **エラー制御** | モジュール例外を吸収して `ModuleResult` を取り出し、AutoPilot 中の `ErrorMode` (skip/retry/ask) を分岐 | `Invoke-SafeCommand`, `Invoke-SafeCommandAsync`, `Show-AutoPilotErrorDialog` |
| **CSV 読み込み + 暗号化透過復号** | `Import-CsvSafe` → 必須列検証 → `ENC:` 値の AES-256-CBC 復号 → `Segment` フィルタ → `Enabled` フィルタ | `Import-ModuleCsv`, `Unprotect-FabriqValue`, `Test-MasterPassphrase` |
| **再起動跨ぎ** | `__RESTART__` 検出 → 状態保存（`resume_state.json`）→ RunOnce 登録 → 再起動 → 復帰時に `Wait-SystemReady` → `Invoke-AutoResumeCountdown` → 環境復元 → 残モジュール継続 | `Save-ResumeState`, `Load-ResumeState`, `Register-FabriqRunOnce`, `Invoke-CountdownRestart`, `Invoke-AutoResumeCountdown`, `Restore-HostEnvironment` |
| **エビデンス自動収集** | モジュール実行ごとにスクリーンショット PNG 保存 + 実行履歴 CSV 追記 + プロファイル完了時に HTML チェックリスト生成 | `Capture-ScreenEvidence`, `Save-Screenshot`, `Write-ExecutionHistory`, `Export-HtmlChecklist`, `Initialize-EvidenceBasePath` |
| **実行ツールバー**（旧ステータスモニタの後継、in-process、since 3.4.0） | kernel の `powershell.exe` 内の専用 STA Runspace に浮遊ツールバーを生成し、実行状態・PC 情報照合・ART pulse を可視化（`[Skip]` / `[Gyotaq]` 操作を含む）。別プロセスを spawn しないため Defender / ASR の子プロセス制限を受けない。旧 out-of-process Status Monitor（`status_monitor.ps1`）は 3.4.0 で非推奨化・3.5.0 で物理削除済み。`status.json` は `Write-StatusFile` が今も書き続けるが、PC Info 照合は `status.json` ではなく `TargetHostInfo`（`SELECTED_*`）+ live OS クエリへ移行 | `Show-ExecutionToolbar`, `Hide-ExecutionToolbar`, `Update-ExecutionToolbar`, `Save-Screenshot`, `Write-StatusFile`, `Write-ArtPulse`, `apps/fabriq_operator/lib/execution_toolbar.ps1` |
| **AutoPilot / AutoConfirm** | プロファイル一括実行で Y/N 確認を自動承認、モジュール間ウェイトを設定 / FlexProfile 単発実行では Y/N と Press-Enter のみ短絡 | `$global:AutoPilotMode`, `$global:AutoConfirmMode`, `Confirm-Execution`, `Wait-KeyPress` |
| **セッション管理** | 作業者選択・媒体シリアル取得・`session.json` 保存・パスフレーズ検証 | `Initialize-Session`, `Test-MasterPassphrase`, `Reset-FabriqState` |
| **公開契約** | カーネル公開 API（§1〜§5）/ 更新オーバーレイ契約（§9）/ Evidence Manifest 契約（§10） | `KERNEL_API.md`, `dev/framework_overlay_rules.json`, `kernel/EVIDENCE_MANIFEST.md` |
| **Telemetry**（kernel 3.2.3+、内部実装） | AI 開発コーパス用の構造化ロギング層。モジュール envelope / Show-* / csv.load / cmdlet.verbose / セッションライフサイクル を JSONL に蓄積（公開 API 外、外部 consumer 非想定） | `Start-ModuleTelemetry`, `Complete-ModuleTelemetry`, `Write-TelemetryEvent`, `Write-KernelTelemetryEvent`, `Enable-FabriqVerboseCapture`, `dev/TELEMETRY_INTERNAL.md`, `kernel/json/{telemetry_salt.txt, verbose_capture.flag}`, `logs/telemetry/{SessionID}/` |

## 保管場所マップ（kernel 配下）

```
kernel/
├── KERNEL_VERSION          ── 3.6.0（カーネル API SemVer の真のソース）
├── KERNEL_API.md           ── 公開 API サーフェスの明文化（§1〜§11）。L3 に "Current Kernel Version" header（release sync 対象、check_version.ps1 が検証）
├── EVIDENCE_MANIFEST.md    ── manifest.json 公開契約（外部 evidence consumer 向け）
├── common.ps1              ── 共通ライブラリ（4862 行、telemetry レイヤ + verbose capture を含む）
├── main.ps1                ── エントリスクリプト・FlexProfile sub-loop・Windows Update ループ（2233 行）
├── csv/
│   ├── categories.csv      ── カテゴリと表示順マスタ
│   ├── hostlist.csv        ── 対象 PC マスタ（暗号化フィールド対応）
│   ├── workers.csv         ── 作業者マスタ
│   ├── log_destinations.csv── ログ配送先マスタ（log_uploader 用）
│   └── manifesto.csv       ── マニフェスト本文（演出機能）
├── json/                       （runtime / framework 混在 — 静的に存在するのは下記 ★ の 4 件）
│   ├── status.json         ── (runtime) ライブ状態（`Write-StatusFile` が atomic write、`Remove-StatusFile` が削除）。実行ツールバーの ART パネルが直接 polling。PC Info 照合には不使用
│   ├── session.json        ── (runtime) 現セッション情報（worker, media serial, start time）
│   ├── resume_state.json   ── (runtime) 再起動跨ぎ時の状態スナップショット（v1/v2 schema）
│   ├── async_config.json ★ ── (framework) __ASYNC__ Runspace 制御パラメータ
│   ├── verbose_capture.flag ★ ── (framework) cmdlet verbose capture を有効化する空ファイル（kernel 3.2.4、git tracked、削除で opt-out）
│   ├── telemetry_salt.txt    ── (runtime) AI 開発テレメトリの site-specific salt（kernel 3.2.3、初回自動生成、`.gitignore`）
│   ├── art_pulse.txt     ★ ── (runtime/framework) 動作鼓動カウンタ（演出用、Show-* で +1。空状態でも commit）
│   └── skip_request.flag   ── (runtime) async モジュール強制スキップ要求の flag ファイル
├── ps1/                       （旧 status_monitor.ps1 / art_display.ps1 は 3.4.0 で非推奨化・3.5.0 で削除済み。後継は in-process 実行ツールバー `apps/fabriq_operator/lib/execution_toolbar.ps1`）
│   ├── view_report.ps1     ── HTML チェックリストの単体ビューア
│   └── manifesto.ps1       ── マニフェスト表示 GUI
└── txt/
    ├── passphrase_verify.txt ── パスフレーズ検証トークン（Studio で生成、起動必須）
    ├── art_sentences.txt   ── ART pulse で表示する一文集
    └── silence.flag        ── 演出抑制 flag（存在すれば ART を黙らせる）
```

## カーネル API のポリシー

- **真のソースは `kernel/KERNEL_VERSION`**。`README.md` L1 / `common.ps1` L2 / `main.ps1` L3 の版表記は `X.Y` 桁で同期。
- **モジュールから利用する公開 API は `KERNEL_API.md` の §1〜§5**（関数・グローバル・環境変数・Profile CSV スキーマ・ModuleResult 契約）。これに記載のない `common.ps1` 関数は内部実装で、PATCH バージョンでも予告なく変更されうる。**外部ツールは加えて §9（更新オーバーレイ契約）/ §10（Evidence Manifest 契約）にも依存可**（モジュール向けではなく fabriq_studio / fabriq_evidence_manager 等の consumer 向け）。
- **モジュール側は `REQUIRES_KERNEL` ファイル（1 行 `X.Y.Z`）で要求カーネル版を宣言**。更新オーバーレイ時に `REQUIRES_KERNEL > 現行 KERNEL_VERSION` ならカーネル先行更新を強制。
- **API 変更は `KERNEL_API.md` の同コミット更新が必須**（CLAUDE.md ルール G）。MINOR/MAJOR 昇格に必ず追従。
