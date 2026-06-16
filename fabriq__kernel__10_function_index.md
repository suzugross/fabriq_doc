# common.ps1 + main.ps1 関数インデックス（105 関数）

> **対象**: fabriq / kernel
> **対象バージョン**: kernel 3.6.0（取得元: `E:\fabriq\kernel\KERNEL_VERSION`）+ commit `0fca159`（取得元: `git -C E:\fabriq rev-parse --short HEAD`、2026-06-16）
> **ドキュメント更新日**: 2026-06-16
>
> **行番号注意**: 本索引の行位置は kernel 3.6.0（commit `0fca159`）の `kernel/common.ps1` / `kernel/main.ps1` に対し `Grep '^function <Name>'` で再取得した実値です。telemetry レイヤ・破壊的削除ガード・`__GATE__` バリア等の追加で過去版から行位置が大きく drift しているため、別版を参照する際は同コマンドで再確認してください。

`kernel/common.ps1`（96 関数）+ `kernel/main.ps1`（9 関数）で定義されている全関数の一覧。**公開 API**（KERNEL_API.md §1〜§5 で宣言）と**内部実装**（PATCH バージョンでも変更されうる）を区別して記載。`_` 始まりの関数は kernel 内部 private helper（モジュール依存禁止）。

> KERNEL_API.md §9（更新オーバーレイ契約）/ §10（Evidence Manifest 契約）は外部ツール向けの **公開契約** であり、本索引の対象（モジュール向け関数 API）とは別レイヤ。本ファイルでは扱わない（→ それぞれ [fabriq__contracts__overlay_contract.md](fabriq__contracts__overlay_contract.md) / [fabriq__contracts__evidence_manifest_contract.md](fabriq__contracts__evidence_manifest_contract.md)）。

> **kernel 索引の対象外**: `Show-ExecutionToolbar` / `Hide-ExecutionToolbar` / `Update-ExecutionToolbar` および `Get-ModuleTelemetryLog` / `Show-ModuleLogViewer` は `apps/fabriq_operator` 側（in-process GUI）で定義され、kernel の関数索引には含めない。`Exit-Fabriq` は `Hide-ExecutionToolbar` を `Get-Command` 越しに best-effort 呼び出すのみ（common.ps1 4339）。

---

## A. 公開 API（モジュールから安全に依存可能）

KERNEL_API.md §1 で宣言された 18 関数。これらは MINOR/PATCH 昇格で互換が保証される。

### A.1 表示・通知（color-coded console output）

| 関数 | 行番号 | 用途 |
|---|---|---|
| `Show-Info` | 761 | シアン `[INFO]` |
| `Show-Success` | 768 | グリーン `[SUCCESS]` |
| `Show-Warning` | 775 | イエロー `[WARNING]` |
| `Show-Error` | 782 | レッド `[ERROR]` |
| `Show-Skip` | 789 | ダークグレー `[SKIP]` |
| `Show-Separator` | 739 | シアンの横線 |
| `Show-CategorySeparator` | 743 | `=== <Name> ===` |

### A.2 結果オブジェクト

| 関数 | 行番号 | 用途 |
|---|---|---|
| `New-ModuleResult` | 800 | ModuleResult 生成（_IsModuleResult, Status, Message, Details, Verified, Timestamp） |
| `New-BatchResult` | 830 | 集計表示 + 自動 Status 判定 + New-ModuleResult 呼び出し |
| `Confirm-ModuleExecution` | 876 | Y/N 確認（N で Cancelled の ModuleResult 返却） |

### A.3 CSV / 暗号化

| 関数 | 行番号 | 用途 |
|---|---|---|
| `Import-ModuleCsv` | 1017 | CSV 読み込み + 透過復号 + 必須列検証 + Enabled / Segment フィルタ |
| `Unprotect-FabriqValue` | 933 | AES-256-CBC + PBKDF2-SHA256 復号 |

### A.4 ユーザー確認・待機

| 関数 | 行番号 | 用途 |
|---|---|---|
| `Confirm-Execution` | 1128 | Y/N → bool（AutoPilot/AutoConfirm 自動 Y） |
| `Wait-KeyPress` | 1162 | Press-Enter 待機（AutoPilot/AutoConfirm スキップ） |
| `Wait-NetworkReady` | 1176 | Test-Connection で host 到達待ち |

### A.5 権限・環境

| 関数 | 行番号 | 用途 |
|---|---|---|
| `Test-AdminPrivilege` | 3833 | 管理者権限判定 (bool) |

### A.6 破壊的削除ガード（since 3.5.0）

| 関数 | 行番号 | 用途 |
|---|---|---|
| `Test-FabriqProtectedPath` | 3965 | 再帰削除対象パスの検証。保護ルート / 親 / 3 セグメント未満 / device・UNC・拡張長パスを fail-closed でブロック。`{IsSafe; Reason; NormalizedPath}` 返却 |
| `Test-FabriqSafePathComponent` | 3946 | 固定ベース配下に Join-Path される単一パス成分の検証（`.`/`..`・区切り文字・ワイルドカード等を拒否）。bool 返却 |

---

## B. 内部実装（PATCH で変更されうる、モジュール依存禁止）

### B.1 オーケストレーション

| 関数 | 場所 | 用途 |
|---|---|---|
| `Invoke-SafeCommand` | common 1443 | 同期実行 + ModuleResult 捕捉 + 例外吸収。kernel 3.3.0 で `DefaultAsync=true` shipped default 化後はこの経路を通るのは `__ASYNC__` 関連の Enabled=false / DefaultAsync=false のとき限定 |
| `Invoke-SafeCommandAsync` | common 1623 | Runspace 実行 + Skip flag / Timeout 監視。inject hashtable に `AutoConfirmMode = $global:AutoConfirmMode` を伝播（公開グローバルの child runspace 伝播） |
| `Get-FabriqAsyncConfig` | common 1593 | `async_config.json` 読み込み。返却 PSCustomObject に `Enabled` / `DefaultAsync`（kernel 3.3.0〜）/ `DefaultTimeoutSec` / `PollIntervalMs` / `SkipFlagPath` を含む。`DefaultAsync` 欠損時のフォールバックは `$false`（旧 config 互換、silent な async 化を防ぐ） |
| `Invoke-BatchExecution` | main 299 | プロファイル一括実行ループ（`__GATE__` 前進バリア・各種マーカー処理含む） |
| `Invoke-KittingScript` | main 211 | 単発モジュール実行 |
| `Invoke-FlexProfileLoop` | main 799 | FlexProfile sub-loop（dashboard 駆動） |
| `Invoke-WindowsUpdateLoop` | main 1127 | WU リブートループ（wu_state.json） |
| `Set-WindowsUpdateAutoLogon` | main 1068 | WU 用 AutoLogon 設定 |
| `Clear-WindowsUpdateAutoLogon` | main 1108 | WU 完了後の credential クリア |
| `Show-WindowsUpdateSummary` | main 1304 | WU 結果集計表示 |

### B.2 プロファイル解決

| 関数 | 場所 | 用途 |
|---|---|---|
| `Resolve-ProfileModules` | common 3570 | profile CSV → module list 変換（マーカー解釈含む）。`$asyncMode` は `$asyncEnabledGlobally -and [bool]$asyncCfg.DefaultAsync` で算出、`__ASYNC__` マーカーは ON-only no-op として後方互換保持 |
| `Get-FabriqGateBarrier` | common 3747 | `__GATE__` 前進バリア解決（TM t-0073、since 3.6.0）。直前ゲート〜マーカ窓に Error/Partial または Post-Apply Verification 失敗（Verified=$false）が残る間、最初の未充足ゲートの Order を返し以降をブロック。窓解消で $null（バリア無し） |
| `Initialize-ModuleSystem` | common 4377 | modules/{std,ext}/*/module.csv 自動検出 |
| `Build-CategoryMenu` | common 4356 | カテゴリ別グルーピング + 順序 |
| `Load-Profiles` | common 3528 | profiles/*.csv 一覧化 |
| `Create-DefaultProfiles` | common 3496 | Basic Setup / Full Setup の自動生成 |
| `Show-ProfileMenu` | common 3813 | コンソール用プロファイル一覧表示 |

### B.3 セッション・状態

| 関数 | 場所 | 用途 |
|---|---|---|
| `Initialize-Session` | common 3369 | worker / media serial / session.json |
| `Reset-FabriqState` | common 3236 | New Session / Refabriq の総リセット |
| `Get-VolumeSerial` | common 3356 | ドライブシリアル取得（media identification） |
| `Get-HardwareUniqueId` | common 1242 | BIOS Serial > MAC > UNKNOWN の優先順位 |
| `Initialize-EvidenceBasePath` | common 1288 | `{ts}_{pc}_{sn}_evidence` ディレクトリ命名 |

### B.4 Resume / Restart

| 関数 | 場所 | 用途 |
|---|---|---|
| `Save-ResumeState` | common 3071 | 環境変数 + Profile 状態 + DPAPI passphrase を json 保存 |
| `Load-ResumeState` | common 3190 | resume_state.json 読み込み |
| `Remove-ResumeState` | common 3230 | resume_state.json 削除 |
| `Restore-HostEnvironment` | common 3345 | json → env vars 復元 |
| `Register-FabriqRunOnce` | common 4460 | HKLM RunOnce に Fabriq.exe を登録 |
| `Register-FabriqActiveSetup` | common 4488 | Active Setup（GUID + StubPath）登録 |
| `Deploy-FabriqUserSetupLauncher` | common 4528 | C:\ProgramData\fabriq\fabriq_user_setup.ps1 配置 |
| `Deploy-FabriqStartupTrigger` | common 4604 | Default Profile Startup folder に .cmd 配置 |
| `Invoke-CountdownRestart` | common 4638 | 5 秒カウント → Restart-Computer -Force |
| `Invoke-AutoResumeCountdown` | common 4654 | AutoPilot 復帰確認（Enter/Y/Esc/N 制御） |
| `Invoke-CountdownSignout` | common 4839 | サインアウト用カウントダウン |
| `Wait-SystemReady` | common 1195 | services 起動待ち + network 到達確認 |

### B.5 履歴・エビデンス

| 関数 | 場所 | 用途 |
|---|---|---|
| `Initialize-ExecutionHistory` | common 2034 | 履歴 CSV ディレクトリ生成 + 旧 path migration |
| `Write-ExecutionHistory` | common 2096 | 1 行追記 + retry 3 回 |
| `Import-ExecutionHistory` | common 2168 | KanriNo フィルタ + Limit + Timestamp 降順 |
| `Restore-ExecutionHistory` | common 2224 | 過去履歴を $script:ExecutionResults へ復元 |
| `Export-ExecutionHistory` | common 2334 | logs/history + evidence/{base}/export_history への dual 出力 |
| `Export-HtmlChecklist` | common 2389 | HTML 監査レポート生成（Network / Printer 照合 + License + BitLocker + チェックリスト） |
| `Add-ExecutionResult` | common 1878 | $script:ExecutionResults に 1 件追加 + status.json 更新 |
| `Clear-ExecutionResults` | common 1905 | 結果配列クリア（IsRestored は保持） |
| `Show-ExecutionSummary` | common 1922 | 実行結果サマリ表示 |
| `Capture-ScreenEvidence` | common 4708 | 自動 PNG（auto_capture/、SetProcessDPIAware） |
| `Save-Screenshot` | common 4787 | 手動 PNG（gyotaku/、DPI 不変更） |
| `Complete-ProfileExecution` | common 2939 | finalize: Export-ExecutionHistory + Export-HtmlChecklist + log_uploader |

### B.6 テレメトリ（best-effort、kernel 挙動に影響しない）

| 関数 | 場所 | 用途 |
|---|---|---|
| `Get-TelemetrySalt` | common 288 | `kernel/json/telemetry_salt.txt` の 256-bit salt 取得 / 自動生成。digest を相関指標に使用、破損時は hash 無効化（[REDACTED]） |
| `_HashTelemetryValue` | common 331 | private: salt 付き値ハッシュ（salt 無効時 [REDACTED]） |
| `New-TelemetryRedactMap` | common 350 | redact マップ生成 |
| `Invoke-TelemetryRedact` | common 404 | 値の redact 適用 |
| `Write-TelemetryEvent` | common 420 | per-module envelope への event 追記 |
| `_GetShowTag` | common 454 | private: Show-* 種別タグ解決 |
| `_TrackShowEvent` | common 471 | private: Show-* 呼び出しのテレメトリ計上 |
| `Get-TelemetryHostInfo` | common 492 | ホスト識別子（hash 済み）取得 |
| `_WriteTelemetryMeta` | common 527 | private: session meta（salt digest 等）の書き出し |
| `Write-KernelTelemetryEvent` | common 561 | session-level event チャネル（`logs/telemetry/{SessionID}/_kernel.jsonl`、profile.start/end・restart.invoked 等） |
| `Start-ModuleTelemetry` | common 597 | per-module envelope 開始（ModuleName/Order/Segment/ErrorMode/Group/IsAsync） |
| `Complete-ModuleTelemetry` | common 678 | per-module envelope 完了（結果サマリ封入） |

### B.7 ステータスファイル・実 OS 状態

| 関数 | 場所 | 用途 |
|---|---|---|
| `Write-StatusFile` | common 4219 | status.json atomic write（PCInfo + CurrentPCInfo + Execution） |
| `Remove-StatusFile` | common 4304 | status.json + tmp + art_pulse.txt 削除（旧 Stop-StatusMonitor のクリーンアップ職務を継承） |
| `Write-ArtPulse` | common 750 | art_pulse.txt の counter +1 |
| `Get-CurrentPCInfo` | common 4131 | Get-NetAdapter / Get-Printer から実 OS 状態取得 |

> kernel 3.5.0 で out-of-process Status Monitor を廃止（`Start-StatusMonitor` / `Stop-StatusMonitor` / `Show-MonitorFailureDialog` 削除）。status.json / art_pulse.txt は in-process で書き出すのみ。

### B.8 暗号化補助

| 関数 | 場所 | 用途 |
|---|---|---|
| `Test-MasterPassphrase` | common 996 | passphrase_verify.txt を復号してマスターパスフレーズと一致するか検証 |
| `Protect-PassphraseForResume` | common 895 | DPAPI LocalMachine で passphrase を暗号化 → Base64 |
| `Unprotect-PassphraseFromResume` | common 912 | DPAPI で復号 |

### B.9 ユーザー権限・環境・パス検証

| 関数 | 場所 | 用途 |
|---|---|---|
| `_Resolve-LoggedOnUser` | common 3853 | private: UAC elevation 時に `Win32_ComputerSystem.UserName` から logged-on user を解決 |
| `Expand-UserEnvironmentVariables` | common 3884 | %USERPROFILE% / %LOCALAPPDATA% / %APPDATA% を logged-on user 視点で展開 |
| `Resolve-HkcuRoot` | common 4095 | HKCU root の解決（admin elevation 環境用） |
| `Remove-ZoneIdentifier` | common 3918 | NTFS Zone.Identifier ADS 削除（Internet zone block 解除） |

### B.10 コンソール制御

| 関数 | 場所 | 用途 |
|---|---|---|
| `Hide-ConsoleWindow` | common 92 | ShowWindow(SW_HIDE) |
| `Show-ConsoleWindow` | common 99 | ShowWindow(SW_SHOW) + SetForegroundWindow |
| `Set-ConsoleForeground` | common 145 | foreground にだけする |
| `Set-ConsoleSize` | common 217 | Window 75x35, Buffer 75x9999 |
| `Disable-QuickEditMode` | common 127 | QuickEdit 無効化（クリックフリーズ防止） |
| `Enable-SleepSuppression` | common 238 | SetThreadExecutionState で sleep 抑止 |
| `Disable-SleepSuppression` | common 246 | sleep 抑止解除 |

### B.11 通知・エラー

| 関数 | 場所 | 用途 |
|---|---|---|
| `Invoke-ErrorNotification` | common 158 | 3-tone beep（Error）/ 2-tone beep（Partial） + foreground |
| `Show-AutoPilotErrorDialog` | common 189 | WinForms MessageBox で Retry/Cancel |

### B.12 詳細ログ捕捉

| 関数 | 場所 | 用途 |
|---|---|---|
| `Enable-FabriqVerboseCapture` | common 1427 | `kernel/json/verbose_capture.flag` 存在時に cmdlet.verbose ストリーム捕捉を有効化（flag 削除で opt-out） |
| `Disable-FabriqVerboseCapture` | common 1438 | verbose 捕捉解除（Exit-Fabriq からも best-effort 呼び出し） |

### B.13 CSV / 表示補助

| 関数 | 場所 | 用途 |
|---|---|---|
| `Import-CsvSafe` | common 1345 | Import-Csv -Encoding Default + retry + 0 行警告 |
| `Test-CsvColumns` | common 1370 | 必須列の存在検証 |
| `Show-BatchConfirmation` | common 2008 | batch 実行前確認 |
| `Show-BatchProgress` | common 1114 | `=== N/Total : ItemName ===` |

### B.14 main.ps1 ローカル

| 関数 | 場所 | 用途 |
|---|---|---|
| `Load-HostList` | main 63 | hostlist.csv 読み込み（Default encoding） |
| `Set-SelectedHostEnvironment` | main 86 | hostlist 行 → SELECTED_* env vars（ENC 復号 + `__SELF__` 解決付き） |

### B.15 終了

| 関数 | 場所 | 用途 |
|---|---|---|
| `Exit-Fabriq` | common 4323 | Remove-StatusFile + Hide-ExecutionToolbar（best-effort）+ Disable-FabriqVerboseCapture + Disable-SleepSuppression + Stop-Transcript（idempotent guard） |

---

## 関数数の総計

- **公開 API（モジュール依存可）**: **18 関数**（KERNEL_API.md §1.1〜§1.6）
- **内部実装（PATCH で変更されうる）**: **87 関数**（うち private helper `_*` 5: `_HashTelemetryValue` / `_GetShowTag` / `_TrackShowEvent` / `_WriteTelemetryMeta` / `_Resolve-LoggedOnUser`）
- **合計**: **105 関数**（common.ps1 96 + main.ps1 9）

公開 API は全体の約 **17%**。残りは内部実装で、kernel 開発者の自由度を確保する設計。

> 3.5.0 で削除済み（本索引から除去）: `Start-StatusMonitor` / `Stop-StatusMonitor` / `Show-MonitorFailureDialog` / `Write-KitLog` / `Save-RollbackInfo` / `Clear-AllLogs` / `Clear-Evidence` / `Show-Progress` / `Parse-MenuSelection` / `Test-BatchInput` / `Show-ProfileConfirmation` / `Show-ExecutionHistory` / `Get-ModuleBasePath`。これらはソースに存在しないため依存しないこと。

---

## なぜこの分担が機能するか

公開 API を絞ることで:

1. **モジュール側の依存ポイントが少なく、リファクタが容易**
2. **`KERNEL_API.md` の真のソース性が成立**（記載されていれば公開、されていなければ内部）
3. **PATCH バージョン昇格でモジュール再配備が不要**（公開 API 不変だから）
4. **MINOR 昇格時もモジュールは opt-in で新機能を使うか選べる**（既存モジュールは無風）

`Resolve-ProfileModules` のような公開しない関数は、引数追加・戻り値追加・内部 logic の刷新を自由に行える。実際 kernel 3.x 系では `IncludeDisabled` スイッチや FlexProfile 用の戻り値拡張が PATCH で行われた。
