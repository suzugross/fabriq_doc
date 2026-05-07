# Evidence と Quick Actions

> **対象**: fabriq / usage
> **対象バージョン**: kernel 3.2.2（取得元: `E:\fabriq\kernel\KERNEL_VERSION`）+ commit `e513cf1`（取得元: `git -C E:\fabriq rev-parse --short HEAD`、2026-05-06）
> **ドキュメント更新日**: 2026-05-07

profile / モジュール実行に伴う成果物（evidence）の出力場所と、ダッシュボード Settings タブの Quick Actions 群（CSV Editor / Windows Update / Refabriq / System Launcher / And More...）の使い方。

---

## 前提

- セッション確立済み（[fabriq__usage__02_session_setup.md](fabriq__usage__02_session_setup.md)）
- 1 度以上 profile 実行 or 個別実行を行うと evidence ディレクトリが生成される

---

## 1. Evidence ベースパス

`Initialize-EvidenceBasePath`（`common.ps1` L785）が session 開始時に **1 セッション = 1 ディレクトリ** で生成：

```
{SessionTimestamp}_{PCName}_{SerialNumber}_evidence
```

| 部分 | 出処 |
|---|---|
| `{SessionTimestamp}` | `$global:FabriqSessionTimestamp`（`yyyy_MM_dd_HHmmss` 形式、起動時に決定） |
| `{PCName}` | `$env:SELECTED_NEW_PCNAME`（hostlist で選んだ NewPCName）。空時は `Unknown_PC` fallback |
| `{SerialNumber}` | `$global:FabriqUniqueId`（BIOS Serial 由来、`Get-HardwareUniqueId` の結果）|

例: `2026_03_12_143025_NEW-PC-01_ABC123XYZ_evidence/`

このディレクトリは fabriq ルート直下の `evidence/` 配下に配置される：

```
{fabriq root}/
└── evidence/
    └── 2026_03_12_143025_NEW-PC-01_ABC123XYZ_evidence/
        ├── pc_information/         ← evidence_config モジュール出力
        │   ├── manifest.json
        │   ├── 01_SystemInfo.txt
        │   ├── ... §31 まで
        │   └── 31_HardwareIdentifiers.txt
        ├── auto_capture/           ← Capture-ScreenEvidence 出力（PNG）
        │   ├── 001_<ModuleName>_Success.png
        │   ├── 002_<ModuleName>_Error.png
        │   └── ...
        ├── bitlocker/              ← bitlocker_config モジュール出力
        │   └── {pcName}_C.txt
        ├── checklist/              ← Complete-ProfileExecution 出力
        │   └── checklist_<ts>.html
        └── export_history/         ← Export-ExecutionHistory 出力
            └── history_export_<ts>.csv
```

`pc_information/manifest.json` は kernel 公開契約 [fabriq__contracts__evidence_manifest_contract.md](fabriq__contracts__evidence_manifest_contract.md) の schemaVersion=1 として外部 consumer（fabriq_evidence_manager）が消費する。

### `$env:FABRIQ_EVIDENCE_BASE`

session 確立後にモジュール側から参照可能な環境変数。`evidence_config` 等のエビデンス生成系モジュールが本値を起点として書き込む。

詳細は [fabriq__contracts__host_environment.md](fabriq__contracts__host_environment.md) §「FABRIQ_*」を参照。

---

## 2. 自動エビデンス取得（モジュール実行ごと）

`Invoke-BatchExecution` の foreach ループ内で、各モジュール実行後に：

```powershell
Capture-ScreenEvidence -ModuleName $module.MenuName -Status $result.Status
```

が呼ばれる（`common.ps1` L4218）。**operator が何もしなくても各モジュールごとにスクリーンショット PNG が `auto_capture/` に保存される**。

ファイル名: `{連番}_{ModuleName}_{Status}.png`（例: `001_hostname_config_Success.png`）。連番は zerofill 3 桁、resume 跨ぎでも継続。

スクリーンショット失敗（GUI セッション無し / WTSDisconnected 等）時は silent に skip。`Show-Warning` も基本出ない（実行履歴へのノイズ防止）。

---

## 3. HTML チェックリスト

### 自動生成（Linear `[Execute Profile]`）

profile 完走で `Complete-ProfileExecution -Mode 'Auto'` が自動発火（`FinalizeOnComplete=true` の場合、Linear 既定）：

- `evidence/{ts}_{pc}_{sn}_evidence/checklist/checklist_<ts>.html` 生成
- `<table class="verify-table">` に hostlist 突合結果（PC 名 / IP / DNS / プリンタ等）
- `<div class="table-wrap">` に ModuleItems（実行モジュール各々の Status / Verified / Time / Message、kernel 2.x で Verified 列追加）
- `evidence_config` モジュールがあれば前後で manifest.json + §01〜§31 ファイル生成（[fabriq__contracts__evidence_manifest_contract.md](fabriq__contracts__evidence_manifest_contract.md)）

### 手動 finalize（FlexProfile `[Complete]`）

FlexProfile の `[Complete]` 押下で `Complete-ProfileExecution -Mode 'Manual'` が走る（[fabriq__usage__04_flexprofile_dashboard.md](fabriq__usage__04_flexprofile_dashboard.md) §「`[Complete]`」）。

### 手動再生成（Quick Actions [And More...] → Regenerate Checklist）

profile を 1 度以上実行した後、HTML を作り直す経路：

`[And More...]` → `Regenerate Checklist` を選ぶと dashboard が `Action="RegenerateChecklist"` を返し、`main.ps1` 側 dispatch（L1843）が：

```powershell
Complete-ProfileExecution `
    -ProfileName    $global:FabriqLastProfileName `
    -ProfilePath    $global:FabriqLastProfilePath `
    -DefinedModules $global:FabriqLastProfileModules `
    -Mode           'Manual'
```

**直前に実行した profile 名 / パス / モジュールリストを `$global:FabriqLastProfile*` から取得**するため、**1 度も profile を実行していない session では動作しない**：

```
Show-Warning "No profile has been executed in this session"
Wait-KeyPress
```

ResumeAfter / ResumeBefore のような時刻 filter は無く、**現 session の execution_history.csv を全件再読み込みして HTML を再生成**する。

---

## 4. 実行履歴のエクスポート

### Export History（And More... → Export History）

`Action="HistoryExport"` で `main.ps1` L1838：

```powershell
Export-ExecutionHistory
```

これは `evidence/{ts}_{pc}_{sn}_evidence/export_history/history_export_<ts>.csv` を生成する。`extended/log_uploader` の入力にもなる CSV。

通常は `Complete-ProfileExecution` の中で自動的に呼ばれるが、**profile 完走前に手動でエクスポートしたい場合**にこのボタンを使う。

### CSV 列

```csv
"Timestamp","KanriNo","PCName","ModuleName","Category","Status","Message","WindowsUser","Worker","MediaSerial","SessionID","Verified","Order"
```

**13 列**（kernel 3.1.3 で `Order` 列追加、より以前に `Verified` 列追加）。詳細は [fabriq__contracts__module_result.md](fabriq__contracts__module_result.md) と [fabriq_evidence_manager__reference__file_format__pc_information.md](fabriq_evidence_manager__reference__file_format__pc_information.md) §「export_history/ ディレクトリ」。

---

## 5. log_uploader 連動

`extended/log_uploader` モジュールは `kernel/csv/log_destinations.csv` の宛先（UNC 共有 / SFTP / ローカルパス等）に Transcript・実行履歴・evidence を一括アップロードする。

### 自動発火タイミング

- Linear `[Execute Profile]` 完走時の `Complete-ProfileExecution -Mode 'Auto'`
- FlexProfile `[Complete]` 押下時の `Complete-ProfileExecution -Mode 'Manual'`

ともに finalize の最後に `extended/log_uploader/log_uploader.ps1` を実行する。

`log_destinations.csv` が空 / 不在の場合は silent skip。

### 手動実行

`extended/log_uploader` を Modules タブから個別実行する経路もある（`[Execute]`）。完走後に追加でアップロード先を変更したいケース等で使用。

詳細は [fabriq__modules__log_uploader.md](fabriq__modules__log_uploader.md) を参照。

---

## 6. Quick Actions 5 件（Settings タブ前面）

`apps/fabriq_operator/lib/dashboard_form.ps1` の Settings タブ。**前面 5 件 + And More... サブダイアログ** の構成。

| ボタン | Action | 動作 |
|---|---|---|
| `[Open CSV Editor]` | `OpenCsvEditor` | `apps/csv_editor/csv_editor.ps1` を起動 |
| `[Windows Update]` | `WindowsUpdate` | `Invoke-WindowsUpdateLoop` を発火（再起動跨ぎループ） |
| `[Refabriq]` | `Refabriq` | Fabriq.exe プロセス再起動 |
| `[System Launcher]` | `SystemLauncher` | `apps/system_launcher/system_launcher.ps1` を起動 |
| `[And More...]` | （内部） | サブダイアログ表示（4 件選択肢、後述） |

### `[Open CSV Editor]`

`apps/csv_editor/csv_editor.ps1` をその場で `& $csvEditorScript` で実行（in-process）。閉じるとダッシュボードに戻る。

主な編集対象：`hostlist.csv` / `app_list.csv` / `reg_hkXm_list.csv` / `local_user_list.csv` / `bitlocker_list.csv` / `profiles/*.csv` 等 20 種以上。詳細は [fabriq__apps__03_other_apps.md](fabriq__apps__03_other_apps.md)。

**fabriq_studio が現場 PC に無い場合の代替経路**として位置づけられている。

### `[Windows Update]`

`Invoke-WindowsUpdateLoop`（`common.ps1` 内、長尺ループ）を発火：

1. `modules/standard/windows_update/wu_state.json` を作成して進捗管理
2. `Set-WindowsUpdateAutoLogon -MaxLoops 5` で AutoLogon 設定（CBS 後始末分の 2 倍カウント = 5×2=10）
3. PSWindowsUpdate モジュール / WUA で更新適用
4. 再起動 → AutoLogon → ループ復帰
5. 全更新完了で `wu_state.json` 削除 + `Clear-WindowsUpdateAutoLogon`

profile に `__WINDOWS_UPDATE__` のような特殊マーカーは無く、**WU は dashboard 経由で個別に走らせる**専用機能。

### `[Refabriq]`

```powershell
Stop-StatusMonitor -MonitorProcess $global:FabriqStatusMonitorProcess
Start-Process Fabriq.exe -WorkingDirectory $fabriqRoot
Stop-Transcript
exit 0
```

**Fabriq.exe プロセス自体を再起動する** 強制リフレッシュ手段。「PID 入れ替え相当の cold boot」になり、起動時のすべての初期化が再走する。

`Refabriq` と `New Session` の違いは [fabriq__usage__02_session_setup.md](fabriq__usage__02_session_setup.md) §「6. セッション切替（New Session）」の比較表参照。要約：

- **`[New Session]`**: 同 process で `Reset-FabriqState` を呼ぶ。state ファイル（execution_history.csv / session.json / resume_state.json）を **明示削除**。Transcript rotate。env vars リセット
- **`[Refabriq]`**: 別 process で `Fabriq.exe` 再起動。state ファイルは **明示削除しない**（Initialize-Session が読み戻す可能性）。新 PID

### `[System Launcher]`

`apps/system_launcher/system_launcher.ps1` を起動。Windows 設定系のショートカット 34 項目（`ms-settings:` URI / `*.cpl` / `*.msc` / shell:: GUID / cmd / powershell 等）を 1 画面に集約したパレット。

検索ボックスでフィルタしてダブルクリックで起動。Windows Search を経由しないため「**検索履歴を残さずに設定画面へ到達**」できるのが運用上のメリット（キッティング後の証跡をクリーンに保つ意図）。

---

## 7. And More... サブダイアログの 4 件

`[And More...]` 押下で `Show-AndMoreDialog`（`quickactions_dialog.ps1`）が表示される。サブダイアログの構成：

```
┌────────────────────────────────────────┐
│ And More                                │
├────────────────────────────────────────┤
│ #  │ Action                             │
│ 1  │ Restart PC                         │
│ 2  │ Export History                     │
│ 3  │ Regenerate Checklist               │
│ 4  │ FabriqApps                         │
├────────────────────────────────────────┤
│              [Close]      [Launch]      │
└────────────────────────────────────────┘
```

`[Launch]` または行ダブルクリックで該当 Action を選ぶ。`[Close]` は `Action="Cancel"` 返却（ダッシュボードに戻る）。

| 表示 | 返り Action | 主な処理 |
|---|---|---|
| `Restart PC` | `Restart` | `Confirm-Execution "Register RunOnce and restart the computer?"` → Yes なら `Register-FabriqRunOnce` + `Invoke-CountdownRestart` |
| `Export History` | `HistoryExport` | `Export-ExecutionHistory` で `history_export_<ts>.csv` 生成 |
| `Regenerate Checklist` | `RegenerateChecklist` | `$global:FabriqLastProfile*` から profile 情報を取得、`Complete-ProfileExecution -Mode 'Manual'` で HTML 再生成 |
| `FabriqApps` | `AppsMode` | `Show-AppsDialog` で 8 補助具一覧、選択 1 件を起動 |

---

## 8. FabriqApps（補助具 8 件）

`Show-AppsDialog`（`apps_dialog.ps1`、133 行）が `apps/<name>/<name>.ps1` 規約のスクリプトを自動列挙する。`fabriq_operator` 自身と `fabriq_ios` は除外（前者は本ダッシュボード、後者は別ランチャ `Fabriq_IOS.exe` で起動）。

| アプリ名 | 用途 |
|---|---|
| `csv_editor` | ジェネリック CSV エディタ（fabriq_studio 不在時の現場用） |
| `system_launcher` | Windows 設定パレット（Quick Actions 直で出るが Apps mode 経由でも可） |
| `bloatware_exporter` | HKLM/HKCU Uninstall キーを走査して `bloatware_list.csv` を編集 |
| `desktop_icon_backup_app` | デスクトップアイコン配置を `.reg` で採取 |
| `local_user_setup` | `local_user_list.csv` のプレースホルダーを Wizard 型 GUI で埋める |
| `storeapp_editor` | `Get-AppxPackage` 一覧と `storeapp_list.csv` を並べて編集 |
| `winget_gui` | `winget search` 非同期実行 + `app_list.csv` 編集 |

選択した行の `[Launch]` で `& <AppPath>` 同期実行。アプリが終了すると `Show-Success "App closed"` を表示してダッシュボードに戻る。

詳細は [fabriq__apps__03_other_apps.md](fabriq__apps__03_other_apps.md) と [fabriq__apps__04_dev_template_and_tooling.md](fabriq__apps__04_dev_template_and_tooling.md)。

---

## 9. Manifesto 表示

ダッシュボード Settings タブ末尾の `[Manifeste du Surkitinisme]` ボタン。`Action="Manifesto"` 返却で `main.ps1` L1897 が `Show-Manifesto` を呼ぶ。

`kernel/csv/manifesto.csv` に記録された宣言文を、`kernel/ps1/manifesto.ps1` の WinForms ビューアで表示する。**機能ではなく芸術部門 / fabriq の理念表明**であり、operator は読まなくても作業に支障はない。

CentreCOM 風の濃紺背景に Surkitinisme（シュルキティニスム）の宣言文がスクロール表示される。閉じるとダッシュボードに戻る。

---

## 10. Status Monitor（別プロセス）

`Start-StatusMonitor`（`common.ps1` L3782）が session 確立直後に起動する**別プロセスのサブモニタ**。

`Start-Process powershell.exe -ArgumentList "-File", "kernel/ps1/status_monitor.ps1"` で隔離プロセスとして起動し、PID を `$global:FabriqStatusMonitorProcess` に記録。

### 役割

- `kernel/json/status.json` を polling して進捗 / PC 情報 / ART pulse 等を別ウィンドウで可視化
- `__ASYNC__` 中のモジュールに対する `[Skip]` ボタン提供（`kernel/json/skip_request.flag` flag ファイル経由）
- `Write-StatusFile` が atomic write でステータス更新（fabriq main プロセス → status_monitor が読み取り）

### ART pulse

`kernel/json/art_pulse.txt` に `Show-*` 系関数が呼ばれるたびに +1 される動作鼓動カウンタ。Status Monitor がこれを polling して「fabriq が動いている」表示に変換する。`kernel/txt/silence.flag` を置くと演出抑制。

`Show-Manifesto` 経由で表示される `art_sentences.txt` のランダム一文も Status Monitor で表示される。

### 終了

`Stop-StatusMonitor` が `Stop-Process -Id $PID` で別プロセスを終了する：

- `[Refabriq]` 押下時
- `Exit-Fabriq`（main.ps1 終了処理）

Status Monitor は **fabriq main の補助モニタ**であり、本体機能には影響しない（無くても fabriq は動く）。`silence.flag` 置きで非表示化可能。

---

## トラブル対応

### Evidence ディレクトリが生成されない

- `Initialize-EvidenceBasePath` 実行前にモジュールを呼んだ → 通常は session 確立で自動的に走るので、このパスは normally 通過済み
- `$env:SELECTED_NEW_PCNAME` 空 → `Unknown_PC` として fallback で動作（致命的ではない）
- `$global:FabriqUniqueId` 空 → `Get-HardwareUniqueId` 失敗。BIOS Serial 取得不能機 → `UNKNOWN` fallback

### HTML チェックリストが見当たらない

- profile 完走前に dashboard を閉じた / `[Refabriq]` した → 自動 finalize 未発火
- FlexProfile で `[Complete]` 未押下 → 手動 finalize 未発火

**対処**: `[And More...] → Regenerate Checklist` で手動再生成。

### Regenerate Checklist が "No profile has been executed" で動かない

`$global:FabriqLastProfileName` が空。これは **当該 session で 1 度も profile 実行していない** ことを意味する：

- Modules タブの個別実行は `$global:FabriqLastProfile*` を更新しない（仕様）
- Refabriq / New Session 直後

**対処**: 何らかの profile を `[Execute Profile]` で 1 度実行する → `$global:FabriqLastProfile*` が立つ → 再生成可能になる。

### log_uploader が動かない / 宛先に届かない

- `kernel/csv/log_destinations.csv` 空 → silent skip（仕様）
- 宛先 UNC 共有の credential が hostlist 由来の `ENC:` 値の場合、復号失敗で skip される
- ネットワーク到達不能 → モジュール内でエラー、`Show-Warning` 表示

詳細は [fabriq__modules__log_uploader.md](fabriq__modules__log_uploader.md)。

### Status Monitor 別ウィンドウが落ちている

`Stop-StatusMonitor` が呼ばれた後、または起動時に失敗した。fabriq main の動作には影響しないため放置可能。再起動するなら：

```powershell
Start-Process powershell.exe -ArgumentList "-File", "kernel/ps1/status_monitor.ps1"
```

を手動で実行（PID 管理が外れるが、目視確認用途なら十分）。

### Refabriq 後に履歴が前回のまま見える

`[Refabriq]` は state ファイルを削除しないため、新 process が `Initialize-Session` で `session.json` を読み戻す → SessionID 系が前回値で復元される。**完全な白紙にしたいなら `[New Session]`** を使う（`Reset-FabriqState` で execution_history.csv 削除）。

---

## 関連ドキュメント

- セッション開始（New Session との差異）: [fabriq__usage__02_session_setup.md](fabriq__usage__02_session_setup.md)
- Linear 一括実行: [fabriq__usage__03_profile_execution_linear.md](fabriq__usage__03_profile_execution_linear.md)
- FlexProfile 部分実行: [fabriq__usage__04_flexprofile_dashboard.md](fabriq__usage__04_flexprofile_dashboard.md)
- ダッシュボード UI 仕様（Settings タブ詳細）: [fabriq__apps__01_fabriq_operator_dashboard.md](fabriq__apps__01_fabriq_operator_dashboard.md)
- 補助具 GUI 群: [fabriq__apps__03_other_apps.md](fabriq__apps__03_other_apps.md) / [fabriq__apps__04_dev_template_and_tooling.md](fabriq__apps__04_dev_template_and_tooling.md)
- Evidence Manifest 公開契約: [fabriq__contracts__evidence_manifest_contract.md](fabriq__contracts__evidence_manifest_contract.md)
- 環境変数公開契約: [fabriq__contracts__host_environment.md](fabriq__contracts__host_environment.md)
- evidence_config モジュール: [fabriq__modules__evidence_config.md](fabriq__modules__evidence_config.md)
- log_uploader モジュール: [fabriq__modules__log_uploader.md](fabriq__modules__log_uploader.md)
- カーネル ステータスモニタ: [fabriq__kernel__06_status_monitor.md](fabriq__kernel__06_status_monitor.md)
- カーネル エビデンス履歴: [fabriq__kernel__07_evidence_history.md](fabriq__kernel__07_evidence_history.md)
