# EasyProfile — 軽量 AutoPilot ランナー

> **対象**: fabriq / profiles（`profiles/easy_template/`）
> **対象バージョン**: kernel 3.3.1（取得元: `E:\fabriq\kernel\KERNEL_VERSION`）+ commit `5525728`（取得元: `git -C E:\fabriq rev-parse --short HEAD`、2026-05-12）
> **ドキュメント更新日**: 2026-05-12

`profiles/easy_template/` 配下に置かれる「**fabriq の正規ダッシュボード（Linear / FlexProfile）を通さず、選択された数モジュールだけ即実行する 1-shot ランナー**」のテンプレート。`framework_overlay_rules.json` により `profiles/` ツリー全体が overlay 時に保持されるため、`profiles/easy_<案件名>/` にディレクトリごとコピーして案件単位の easyprofile を並列運用できる。

---

## 何が違うのか（正規プロファイル経路との比較）

| 観点 | 正規 Profile（Linear / FlexProfile） | EasyProfile |
|---|---|---|
| 起動経路 | `Fabriq.exe` → `main.ps1` → 操作ダッシュボード | `easyprofile.bat`（UAC 昇格）→ `easyprofile.ps1` 直叩き |
| CSV スキーマ | `Order,ScriptPath,Enabled,Description,Segment,ErrorMode,Group`（必須 3 列 + 任意） | `Enabled,Script,Description,Segment`（4 列、Order なし上から順実行） |
| Session 初期化 | あり（`Initialize-Session` で作業者選択・媒体シリアル取得・session.json 保存） | なし |
| 実行履歴 (`exec_history.csv`) | あり | **なし** |
| Evidence 収集（スクリーンショット） | あり | **なし** |
| HTML チェックリスト生成 | あり（`Export-HtmlChecklist`） | **なし** |
| AutoPilot | プロファイル内 `__AUTOPILOT__` で opt-in | **強制 ON**（`$global:AutoPilotMode=$true` を起動時にセット） |
| AutoPilot 間 wait | `WaitSec=N` 指定可 | **0 秒固定**（`$global:AutoPilotWaitSec=0`） |
| 再起動跨ぎ resume | あり（`__RESTART__` + `resume_state.json` + RunOnce） | なし（`__RESTART__` は通常モジュールとして扱われず Script 解決失敗で `[NOT FOUND]` 扱い） |
| 特殊マーカー対応 | `__AUTOPILOT__` / `__ASYNC__` / `__RESTART__` / `__REEXPLORER__` / `__AUTO_to_<User>__` の 5 種 | **対応なし** — マーカー解釈器を持たないため、書いても通常 ScriptPath として `Test-Path` で蹴られる |
| async / Runspace | kernel 3.3.0 以降は `async_config.json.DefaultAsync=true` で全モジュール async 既定 | **直接 `& $scriptPath` で同期実行**（`Invoke-SafeCommand` / `Invoke-SafeCommandAsync` を経由しない） |
| Segment フィルタ（per-row） | `Resolve-ProfileModules` が `$env:FABRIQ_SEGMENT` を per-row で set/clear | `easyprofile.ps1` が同じ env-var contract で per-row set/clear（kernel 3.3.1 期に追加） |

**位置付け**: Hostname 設定 / ローカルユーザー作成 / 削除のような単発タスクを、エビデンス無しで素早く流すユースケース。「正規キッティングフロー」ではなく「ポストキッティング小作業」「現場で 5 分だけ走らせたい一品料理」が想定運用。

---

## ファイル構成

```
profiles/easy_template/
├── easyprofile.bat       管理者昇格 + powershell -File easyprofile.ps1
├── easyprofile.ps1       軽量ランナー本体（255 行、kernel/common.ps1 を dot-source）
└── easyprofile.csv       実行リスト（4 列）
```

`easyprofile.bat` は次の処理を行う：

1. `net session` で管理者権限を確認、未昇格なら `Start-Process cmd -Verb RunAs` で自己再起動
2. CWD を fabriq root（`%~dp0..\..`）に変更
3. `conhost.exe powershell.exe -NoProfile -ExecutionPolicy Unrestricted -File easyprofile.ps1` で本体を起動

`easyprofile.ps1` は `kernel/common.ps1` を dot-source してから AutoPilot を強制 ON、`easyprofile.csv` を読んでループ実行。Session / Evidence / Checklist 系は **意図的に呼び出さない**。

---

## CSV スキーマ（kernel 3.3.1 期に Segment 列追加）

```csv
Enabled,Script,Description,Segment
0,modules\standard\hostname_config\hostname_config.ps1,Hostname Setting,
0,modules\standard\local_user_config\local_user_config.ps1,Local User Creation,
0,modules\standard\local_user_config\local_user_delete.ps1,Local User Deletion,
```

| 列 | 必須 | 用途 |
|---|---|---|
| `Enabled` | 必須 | `1`=実行 / `0`=スキップ。デフォルト 3 行はすべて `0`（安全側、誤実行防止） |
| `Script` | 必須 | fabriq root からの相対パス。`modules\standard\...` 形式が標準だが任意の .ps1 でも可（`Join-Path $fabriqRoot $entry.Script` で絶対化） |
| `Description` | 任意 | UI 表示用。空ならファイル名を fallback 表示 |
| `Segment` | 任意（kernel 3.3.1 期〜） | `$env:FABRIQ_SEGMENT` として該当モジュール実行直前に export、`finally` で `$null` クリア。Segment-aware モジュール（`reg_hklm_config` / `app_config` / `acl_config` / `test_harness_config` 等）の `_list.csv` フィルタとして消費される |

エンコーディングは UTF-8 / SJIS いずれも可（`Import-CsvSafe` 側で吸収）。改行 CRLF。

### Segment 列の意味（kernel 3.3.1 期に追加）

profile CSV の Segment 列が **行ごとに違う Segment** を意図して使われる場合（同じスクリプトを異なる `_list.csv` セグメントで呼び分けるパターン）への対応。

```csv
Enabled,Script,Description,Segment
1,modules\standard\reg_hklm_config\reg_hklm_config.ps1,Office レジストリ,office
1,modules\standard\reg_hklm_config\reg_hklm_config.ps1,Home レジストリ,home
1,modules\standard\app_config\app_config.ps1,基本アプリのみ,baseline
```

実行時:

1. 1 行目: `$env:FABRIQ_SEGMENT = "office"` → `reg_hklm_config.ps1` 実行（内部の `Import-ModuleCsv` が `reg_hklm_list.csv` の `Segment=office` 行のみ拾う）→ `finally` で `$env:FABRIQ_SEGMENT = $null`
2. 2 行目: `$env:FABRIQ_SEGMENT = "home"` → 同スクリプト実行（`Segment=home` 行のみ）→ クリア
3. 3 行目: `$env:FABRIQ_SEGMENT = "baseline"` → `app_config.ps1` 実行（`app_list.csv` の `Segment=baseline` 行のみ）→ クリア

`finally` で確実にクリアされるため、Segment 空欄の行が直後に来ても前行の値が漏れない。これは `kernel/main.ps1:441-465` の Linear path と同じ env-var contract（**fabriq の Segment 受け渡しは「行直前 export、行直後 clear」がカーネル横断契約**）。

### Segment 列のない既存 CSV は後方互換

`easyprofile.ps1` は実行時に `$entry.PSObject.Properties.Name -contains 'Segment'` ガードで Segment 列の存在を確認する。列がなければ Segment 文字列は空文字となり、`$env:FABRIQ_SEGMENT` は触らない（モジュール側からは「未設定」と見える）。kernel 3.3.0 以前から運用している `easy_<案件名>/easyprofile.csv` をそのまま流用しても無害。

---

## CSV ロード経路の設計判断 — なぜ `Import-CsvSafe` か

`easyprofile.ps1` の CSV ロードは：

```powershell
$allEntries = Import-CsvSafe -Path $csvPath -Description "easyprofile.csv"
# ... null / 空 / 必須列の検証 ...
$scriptList = @($allEntries | Where-Object { $_.Enabled -eq "1" })
```

つまり `Import-CsvSafe` + `Test-CsvColumns` + **手動の Enabled フィルタ** という 3 段構成。`Import-ModuleCsv -FilterEnabled` を使わない理由：

- `Import-ModuleCsv` は CSV 自身が `Segment` 列を持つ場合、`$env:FABRIQ_SEGMENT`（entry-time の値、通常空）と CSV の Segment 列を **strict-match で照合**してフィルタする（`kernel/common.ps1:1063-1080`）
- この挙動は `<module>_list.csv`（モジュール内データ）に対しては正しい（モジュール起動時の Segment で `_list.csv` を絞る）
- しかし profile CSV（行ごとに違う Segment を意図）に対しては誤動作。entry-time の `$env:FABRIQ_SEGMENT` が空のとき、Segment 列に値がある行が **黙って drop される**

これは `Resolve-ProfileModules` が Linear profile CSV を `Import-Csv` で **直接** 読んでいる（Import-ModuleCsv を通さない）のと同じ判断。EasyProfile も同じ路線で `Import-CsvSafe` を採用、Segment フィルタは ps1 側で per-row に export して再現する。

---

## 実行フロー詳細

```
[ easyprofile.bat ]
  1. net session で管理者確認、未昇格なら RunAs で自己再起動
  2. CWD を fabriq root へ
  3. powershell.exe -File easyprofile.ps1

[ easyprofile.ps1 ]
  1. $fabriqRoot = ../../ を絶対化
  2. kernel/common.ps1 を dot-source（Show-* / Import-CsvSafe / Test-CsvColumns etc）
  3. $global:AutoPilotMode = $true / $global:AutoPilotWaitSec = 0
  4. Set-ConsoleSize -Columns 75 -Lines 35 + Enable-SleepSuppression
  5. easyprofile.csv を Import-CsvSafe で読み込み
     - null → Press Enter to exit / exit 1
     - Count=0（ヘッダのみ） → Show-Warning 後、Press Enter to exit / exit 0
     - Test-CsvColumns で Enabled / Script 必須列を検証
     - $scriptList = @(... Enabled -eq "1" ...)
     - Enabled=1 が 0 行なら Show-Info "No enabled entries" → exit 0
  6. Pre-execution display: 各行を [n] DisplayName [seg:X] で列挙
     - $scriptPath が Test-Path 失敗なら "[NOT FOUND]" 表示（後続で skip）
  7. Execution loop: foreach ($entry in $scriptList)
     - $segment / $segLabel / $baseName / $displayName を算出
     - Script not found なら Show-Skip → $skipCount++
     - try {
         $env:FABRIQ_SEGMENT = $segment （あれば）
         $output = & $scriptPath
         ModuleResult を pipeline / $global:_LastModuleResult から検出
         Status 別に Show-Success / Error / Info / Skip / Warning + カウント
       } catch { Show-Error / $failCount++ }
       finally { $env:FABRIQ_SEGMENT = $null （セットしていた場合） }
  8. Final summary: Success / Skipped / Failed カウント表示
  9. Disable-SleepSuppression + Read-Host "Press Enter to exit"
```

---

## 制限事項（fabriq 正規経路との非互換）

- **`__ASYNC__` / `__AUTOPILOT__` / `__RESTART__` / `__REEXPLORER__` / `__AUTO_to_<User>__` の特殊マーカー非対応**: マーカー解釈器を持たないため `Test-Path` で蹴られて `[NOT FOUND]` 扱いになる。マーカー機能が必要な場面は正規 Profile（Linear / FlexProfile）を使う
- **再起動跨ぎ resume なし**: `Save-ResumeState` / `Register-FabriqRunOnce` を呼ばないため、モジュール内で OS 再起動が走ると単純にプロセスが終わる。再起動を含むシーケンスは正規 Profile + `__RESTART__` を使う
- **Evidence 収集なし**: `Capture-ScreenEvidence` / `Write-ExecutionHistory` を呼ばないため `evidence/` 配下に何も残らない。納品向け作業ではなく**現場の臨時作業**を前提とする
- **kernel 3.3.0 の `DefaultAsync` の対象外**: easyprofile.ps1 は `Invoke-SafeCommand` / `Invoke-SafeCommandAsync` を経由せず直接 `& $scriptPath` で実行するため、`async_config.json` の `DefaultAsync` / `Enabled` 設定は EasyProfile には影響しない。**Status Monitor の Skip ボタンも timeout 安全網も無し**。ハングしたら手動で PowerShell ウィンドウを閉じるしかない（→ ハングリスクのある winget_install / windows_update 等の長時間モジュールは EasyProfile では運用しない）
- **`__RESTART__` 等のマーカー行を CSV に書いてもエラーにはならない**: 通常 ScriptPath として `Test-Path $scriptPath` で蹴られて `Show-Skip "Script not found"` → `$skipCount++` で続行。明示的なエラー扱いではないため、誤ってマーカー混入しても EasyProfile は止まらない（ただし期待動作はしない）

---

## 運用ルール

- `easy_template/` 自体は配布物。**現場では `easy_<案件名>/` にディレクトリごとコピーして編集**する
- `framework_overlay_rules.json` により `profiles/` ツリー全体が overlay 時に保持されるため、フレームワーク更新で `easy_<案件名>/` が消える事故はない
- デフォルト 3 行（hostname_config / local_user_config / local_user_delete）はすべて `Enabled=0`。テンプレートをコピーした直後に勢いで `easyprofile.bat` を叩いても何も走らないようにする安全側
- 新規 EasyProfile を作る場合の最小構成: フォルダ作成 → `easyprofile.bat` / `.ps1` / `.csv` を `easy_template/` からコピー → CSV を編集して `Enabled=1` の行を入れる → `.bat` を叩く

---

## 関連ドキュメント

- 全体カタログ: [fabriq__profiles__00_profiles_overview.md](fabriq__profiles__00_profiles_overview.md)
- Profile CSV スキーマ（正規プロファイル経路の契約）: [fabriq__contracts__profile_csv_schema.md](fabriq__contracts__profile_csv_schema.md)
- Segment フィルタ契約（kernel 側の env-var contract）: [fabriq__contracts__profile_csv_schema.md#Segment フィルタ仕様](fabriq__contracts__profile_csv_schema.md)
- 非同期実行の全体像（EasyProfile は対象外だが kernel 3.3.0 の文脈把握用）: [fabriq__kernel__08_async_execution.md](fabriq__kernel__08_async_execution.md)
- overlay 保持ルール（`profiles/` ツリーが保持される根拠）: [fabriq__contracts__overlay_contract.md](fabriq__contracts__overlay_contract.md)
