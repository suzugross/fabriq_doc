# オーケストレーション層

> **対象**: fabriq / kernel
> **対象バージョン**: kernel 3.6.0（取得元: `E:\fabriq\kernel\KERNEL_VERSION`）+ commit `0fca159`（取得元: `git -C E:\fabriq rev-parse --short HEAD`、2026-06-16）
> **ドキュメント更新日**: 2026-06-16

カーネルがモジュールをどう順序付けて呼び出し、結果を集約するか。`main.ps1` 内の `Invoke-BatchExecution` / `Invoke-FlexProfileLoop` / `Invoke-WindowsUpdateLoop` がコア（`common.ps1` ではなく `main.ps1` に集約されている）。

---

## main.ps1 起動フロー

```
Fabriq.exe（C# ランチャ、UAC 自動昇格）
   ↓
kernel/main.ps1
   ├── . kernel/common.ps1            ── 共通関数読み込み
   ├── . kernel/ps1/manifesto.ps1     ── マニフェスト GUI 関数
   ├── . apps/fabriq_operator/fabriq_operator.ps1
   │                                  ── ダッシュボード GUI
   ├── Enable-SleepSuppression        ── SetThreadExecutionState で sleep 抑止
   ├── Disable-QuickEditMode          ── マウスクリックでフリーズしないよう QuickEdit 無効化
   ├── Set-ConsoleSize -Columns 75 -Lines 35
   ├── Start-Transcript               ── logs/{timestamp}_{uid}_{hostname}.log
   │
   ├── ── Resume Detection ──
   ├── Test-Path wu_state.json → Invoke-WindowsUpdateLoop（WU resume 専用、最優先）
   ├── Load-ResumeState → resume_state 読み込み
   │   ├── schemaVersion>=2 + ExecutionMode='Flex' → FlexProfile resume ルート
   │   ├── AutoPilot=true → Wait-SystemReady + Invoke-AutoResumeCountdown 60s
   │   └── 通常 → Confirm-Execution "Resume profile execution?"
   │   shouldResume なら:
   │     ├── Restore-HostEnvironment   ── SELECTED_* 環境変数を json から戻す
   │     ├── DPAPI 復号で master passphrase 復元 or 手動再入力（最大 3 回）
   │     └── EvidenceBasePath / SessionID 復元
   │
   ├── ── Fresh Start（resume なしの場合）──
   ├── Test-Path passphrase_verify.txt （無いと exit 1）
   ├── Load-HostList                  ── kernel/csv/hostlist.csv
   ├── Show-SessionSetupForm          ── WinForms で worker / host / passphrase を一括選択
   │   （3 つを 1 ダイアログで取る。GUI モード必須、CLI モードは廃止）
   ├── Set-SelectedHostEnvironment    ── SELECTED_* env vars を立てる
   ├── Initialize-EvidenceBasePath    ── evidence/{ts}_{pc}_{sn}_evidence/ を作る
   ├── Initialize-Session             ── workers / media serial / session.json
   ├── Initialize-ExecutionHistory    ── logs/history/execution_history.csv 作成 + 旧パスからの migrate
   ├── Initialize-ModuleSystem        ── modules/{standard,extended}/*/module.csv を再帰検出
   ├── Load-Profiles                  ── profiles/*.csv を一覧化（無ければ Basic Setup / Full Setup を生成）
   ├── Start-StatusMonitor            ── 別プロセスで status_monitor.ps1 を起動（PID 記録）
   │
   └── Show-Operator-Dashboard        ── apps/fabriq_operator のメインダッシュボード（無限ループ）
```

メインダッシュボードがイベントを受けるたびに、対応するアクション（個別モジュール実行・プロファイル実行・FlexProfile 起動・履歴表示・apps 起動・新セッション・終了）を呼び分ける。

---

## Invoke-BatchExecution（プロファイル一括実行の中核）

`main.ps1` 内、L223〜（次の `Invoke-FlexProfileLoop` が L561 なので約 340 行規模）のメイン関数。Linear `Execute Profile` も FlexProfile の `RunBatch` / `RunSingle` / `RunGroup` も最終的にここを通る。

### パラメータ

```powershell
Invoke-BatchExecution
    -SelectedModules <array>           # Resolve-ProfileModules が返した module オブジェクト配列
    [-AutoPilot]                       # AutoPilot モードか
    [-AutoPilotWaitSec <int>]          # モジュール間スリープ秒（デフォ 3）
    [-ProfilePath <string>]            # CSV パス（restart 時の resume 用）
    [-ProfileName <string>]            # 表示名
    [-FullProfileModules <array>]      # チェックリスト用フル module list
    [-ProfileStartTime <datetime>]     # 経過時間計算の絶対起点
    [-FinalizeOnComplete <bool>]       # Linear=true（HTML 自動生成）/ Flex=false（手動）
    [-ExecutionMode <Linear|Flex>]     # resume_state.json の schemaVersion を切替
    [-SelectedOrders <int[]>]          # FlexProfile が ResumeState に書く Order 配列
    [-ModuleStates <hashtable>]        # FlexProfile が ResumeState に書く状態マップ
```

### モジュール 1 件の処理ループ

```
（ループ前）gate コンテキストを用意:
   - $gateRows        = FullProfileModules ?? SelectedModules（gate 位置/窓を sparse 選択でも知るためフル profile）
   - $statusByOrder / $verifiedByOrder = 当該セッション history を Order→Status / Order→Verified で seed
     （prior run + __RESTART__ 前に完了した分を捕捉、MenuName 一致で別 profile 行を誤 seed しない安全弁）
   - $gateBlockedOrders = @()

foreach $module in $SelectedModules:
   1. progress 表示（Show-BatchProgress）
   2. 特殊マーカー分岐:
      ├── _IsGate           → "[GATE] checkpoint at Order N" を表示し continue（実行しない。位置だけ barrier 計算に効く）
      ├── _IsRestart        → Save-ResumeState + Register-FabriqRunOnce + Invoke-CountdownRestart（return）
      └── _IsReexplorer     → Stop-Process explorer + 復活待ち → Add-ExecutionResult
   2.5 __GATE__ admission control:
       $gateBarrier = Get-FabriqGateBarrier -Rows $gateRows -StatusMap $statusByOrder -VerifiedMap $verifiedByOrder
       if barrier != $null && module.Order >= barrier:
         - "[GATE] Blocked: Order N (name) is beyond the unsatisfied gate at Order M ..." を warning 表示
         - $gateBlockedOrders += module.Order
         - 当該モジュールは Pending 据え置き（新ステータス記録なし）で continue
   3. AutoPilot inter-module wait（current > 1 のみ）
   4. retry loop（do/while $retryModule）:
      a. _AutoLogonUser があれば $env:FABRIQ_AUTOLOGON_USER を立てる
      b. _Segment があれば $env:FABRIQ_SEGMENT を立てる
      c. _IsAsync && async_config.Enabled なら Invoke-SafeCommandAsync、
         さもなくば Invoke-SafeCommand
         （kernel 3.3.0 以降は async_config.DefaultAsync=true が shipped default のため、
          _IsAsync は Resolve-ProfileModules が profile 1 行目から全モジュールに attach する
          → 既定で全行が Invoke-SafeCommandAsync 経路を通る）
      d. 上記 env を解除
      e. result.Status が Error/Partial の場合:
         - Invoke-ErrorNotification（3-tone beep + console foreground）
         - AutoPilot 中なら _ErrorMode で分岐:
           ├── "skip"  → 記録して続行
           ├── "retry" → autoRetryCount++（最大 5 回）→ retryModule=true で再実行
           └── ""/"ask" → Show-AutoPilotErrorDialog（Retry / Skip）
   5. Add-ExecutionResult + Write-ExecutionHistory + Capture-ScreenEvidence
   6. $completedResults に Order/MenuName/Status/Verified/Message を追加
   6.5 $statusByOrder[Order]=result.Status / $verifiedByOrder[Order]=result.Verified を更新
       （直後の Order の barrier 判定に即反映 → フォワード実行中の失敗もゲートで止まる）
end foreach

7. if $gateBlockedOrders.Count > 0:
   "[GATE] N module(s) blocked by an unsatisfied gate (Orders: ...)" を warning でまとめ表示
8. Remove-ResumeState（再起動なしで完走できた場合の resume クリア）
9. Show-ExecutionSummary
10. Linear かつ ProfileName が指定されていれば:
    Complete-ProfileExecution（HTML チェックリスト生成 + log_uploader 起動 + view_report）
```

`finally` 節で必ず `$global:AutoPilotMode = $false` にリセット（Profile スコープ保証）+ `$script:LastBatchResults = $completedResults` を publish（FlexProfile dashboard が polling して状態反映）。

### ErrorMode マトリクス

| `_ErrorMode` 値 | AutoPilot=on | AutoPilot=off |
|---|---|---|
| 空文字 / `ask` | `Show-AutoPilotErrorDialog`（Retry/Skip 選択） | エラー記録のみ、続行 |
| `skip` | warning 出力 → エラー記録のまま続行 | 同左（off でも適用） |
| `retry` | autoRetryCount を最大 5 回まで増やして再実行、超えたらエラー記録 | 同左 |

---

## __GATE__ 前進バリア（forward barrier, kernel 3.6.0〜）

`__GATE__` は profile CSV に置く checkpoint マーカー（TM t-0073）。「ここまでの区間が健全に通っていなければ、この先のモジュールを実行させない」という前進バリアを敷く。`ModuleResult` 契約は不変で、`__GATE__` を含まない既存 profile の挙動は完全に不変。

### マーカー登録（Resolve-ProfileModules）

`common.ps1` L3675-3679 の `specialMarkers` ハッシュに `'__GATE__' = @{ MenuName='[GATE]'; Flag='_IsGate' }` を登録。`__RESTART__` / `__REEXPLORER__` と同列の扱いで、専用 PSCustomObject を生成し `_IsGate=$true` を立てる。`__GATE__` 行自体は実行されない（後述）。

### 判定ヘルパ Get-FabriqGateBarrier（純関数）

`common.ps1` L3747-3811 の内部ヘルパ。副作用を持たず、入力（Rows / StatusMap / VerifiedMap）から barrier の Order を返すだけの純関数。

```powershell
Get-FabriqGateBarrier
    [-Rows <array>]            # profile 全行（module/gate 混在）。デフォルト @()
    [-StatusMap <hashtable>]   # Order -> Status（'Error'/'Partial'/'Success'/...）
    [-VerifiedMap <hashtable>] # Order -> $true / $false / $null（PASS / FAIL / 未検証）
```

- Rows を **Order 昇順** に走査。各 `__GATE__` は「直前ゲート（または profile 開始）〜自身」の窓を守る。
- 窓内のモジュールが次のいずれかなら、その窓は失敗（`$windowFailed=$true`）:
  - `StatusMap[Order]` が `Error` または `Partial`
  - Post-Apply Verification 失敗（`VerifiedMap[Order] -eq $false`）。Success だが readback が一致しなかった = 設定が効いていない、という意味なので下流を走らせない。
- `Success` / `Skipped` / `Cancelled` / `Pending`（未実行・StatusMap に不在）は Status ではブロックしない。`Verified` が `$null`（未検証）/ `$true` もブロックしない（`$false` のみ）。ゲートは「失敗」を守るもので「未実施」は守らない。
- 走査中に失敗窓を抱えたままゲートに到達したら、その時点でゲートを **unsatisfied** と判定し、その `Order` を barrier として即 return。最初の unsatisfied gate の Order を返す。1 つも無ければ `$null`。
- 決定的ソートのため、Order 同値では二次キー「非ゲート(0) → ゲート(1)」を使う（PS 5.1 の `Sort-Object` は非安定）。同 Order のモジュールはそのゲートの **窓の内側** に数える（保守的）。

### admission control（Invoke-BatchExecution）

`main.ps1` のモジュール実行ループが各モジュール実行直前に barrier を評価する:

- **`__GATE__` 行のスキップ**（main.ps1 L458-462）: `module._IsGate` が真なら `"[GATE] checkpoint at Order N"` を表示して `continue`。何も実行しない。位置だけが barrier 計算に効く。
- **ブロック判定**（main.ps1 L464-476）: `Get-FabriqGateBarrier` を `$gateRows` / `$statusByOrder` / `$verifiedByOrder` で評価。`barrier != $null` かつ `module.Order >= barrier` なら、`"[GATE] Blocked: Order N (name) is beyond the unsatisfied gate at Order M. Resolve the failure(s) above and re-run."` を warning 表示し、当該 Order を `$gateBlockedOrders` に記録して `continue`。**ブロックされたモジュールは Pending 据え置き（新ステータスを記録しない）**。
- **完走時まとめ警告**（main.ps1 L709）: ループ終了後、`$gateBlockedOrders.Count > 0` なら `"[GATE] N module(s) blocked by an unsatisfied gate (Orders: ...). Resolve the upstream failure(s) and re-run."` を 1 回だけ表示し、ゲートで止まったことを operator に通知。

### 動的評価（フォワード実行中の失敗も止める）

`$gateRows` は `FullProfileModules ?? SelectedModules`（sparse な Flex 選択でもゲート位置/窓構成が判るようフル profile を使う、main.ps1 L401）。`$statusByOrder` / `$verifiedByOrder` は判定時点の状態:

- 当該セッションの execution history から seed（prior run、および `__RESTART__` の前に完了したモジュールを history 経由で捕捉、main.ps1 L419-438）。`__RESTART__` 跨ぎでも状態が保持される。
- history seed は Order だけでなく **MenuName 一致** で当該 profile 行に束縛する（同一セッションで Order が重なる別 profile run の行で誤った barrier を seed しない安全弁、main.ps1 L403-409）。
- 各モジュール完了直後に `$statusByOrder` / `$verifiedByOrder` を live 更新（main.ps1 L696-697）するため、本 run のフォワード実行中に起きた失敗も次の Order の判定で即座にゲートに反映され、そこで止まる。

### FlexProfile dashboard のグレーアウト（反映のみ）

enforcement の権威は kernel（Invoke-BatchExecution）にあり、UI は同じ規則を operator 向けに反映するだけ（`flex_dashboard.ps1` L139-141）。

- dashboard が同じ `Get-FabriqGateBarrier` で barrier を計算（`flex_dashboard.ps1` L148、`$stateMap` から組んだ StatusMap/VerifiedMap を渡す）。
- gate 行（`_IsGate`）: 青タイント `ARGB(214,224,240)` + `Checked` セル `ReadOnly` + tooltip「Gate checkpoint: blocks Orders at/after it while the prior window has Error/Partial.」（L453-456）。
- barrier 以降の blocked 行（`Order >= barrier`、L428）: `Checked=false` + `ReadOnly`、`MenuName` / `Order` セルを灰色 `ARGB(150,150,150)` 表示（L463-468）。`[Select All]` からも除外（gate 行・blocked 行は bulk 選択されない、L704-708）。
- ただし `Status` / `Verified` バッジ色と `[Log]` ボタンは有効のまま（ブロックの原因となった失敗を operator が確認できるよう、L460-462）。

---

## Invoke-FlexProfileLoop（FlexProfile sub-loop）

`main.ps1` 内、L617〜。FlexProfile dashboard と Invoke-BatchExecution の橋渡し。

### イベント駆動アクション一覧

dashboard が intent を返すたびに loop が dispatch する：

| Action | 動作 |
|---|---|
| `RunSingle` | 1 行だけ `Invoke-BatchExecution` で実行（`AutoConfirmMode=$true`、Y/N と Press-Enter のみ短絡） |
| `RunGroup` | `Group` 列で集約された Order 群を `AutoPilot=$true` + `FinalizeOnComplete:$false` で実行 |
| `RunBatch` | チェックボックスで選択された Order 群を `AutoPilot=$true` + `FinalizeOnComplete:$false` で実行 |
| `Complete` | `Complete-ProfileExecution -Mode 'Manual'`（HTML 生成 + log_uploader 発火） |
| `RestartNow` | プロファイル外 RESTART。`ResumeAfterOrder=-1` sentinel で resume_state を書き、`Register-FabriqRunOnce` + `Invoke-CountdownRestart` |
| `ResetState` | 該当 Order の状態を `Pending` に戻す（再実行候補） |
| `Close` | `Remove-ResumeState`（防御的）+ loop 脱出 |

### PendingFinalize フラグ

`RunBatch` / `RunSingle` / `RunGroup` / `ResetState` が走ったら `$pendingFinalize = $true`。dashboard は赤 PENDING FINALIZE バッジを点灯し、`Back` / `X` 押下時に確認ダイアログで警告。`Complete` で `$false` にリセット。

### 実行モデルの統一（kernel 3.1.5〜）

> **実行 = 常に AutoPilot 挙動 / 完了 = 常に手動**

AutoPilot トグルは UI から消した。operator の意思決定は「どのモジュールを動かすか」と「いつ Complete するか」の 2 軸のみ。これにより成果物確認 → 明示 finalize の運用に最適化されている。

---

## Resolve-ProfileModules（profile CSV → モジュールリスト変換）

`Invoke-BatchExecution` に渡す前段。約 170 行。

### 入力 → 出力

```
profile CSV (Order, ScriptPath, Enabled, Description, Segment, ErrorMode, Group)
   ↓
[ValidModules: array of module objects with attached _IsAsync / _Segment / _ErrorMode / _Group / _AutoLogonUser / _IsRestart / _IsReexplorer / _IsGate / _IsCheckedDefault / Order]
[InvalidPaths: array of unresolved ScriptPath strings]
[AutoPilot: bool, set if __AUTOPILOT__ row was Enabled=1]
[AutoPilotWaitSec: int, parsed from "WaitSec=N" in __AUTOPILOT__'s Description]
```

### 特殊マーカー処理

- `__AUTOPILOT__`: ValidModules には入れず、戻り値の `AutoPilot` フラグだけ立てる
- `__ASYNC__`: 同様に list には入れず、以降のモジュールに `_IsAsync=$true` を打つ sticky フラグ（プロファイル末尾まで継続）。**kernel 3.3.0 以降**: `async_config.json.DefaultAsync=true`（shipped default）のとき `$asyncMode` 初期値が profile 1 行目から `$true` のため、マーカーは idempotent ON-only no-op として動作（既存 profile への後方互換）
- `__AUTO_to_<User>__`: `autologon_config` モジュールを参照し、`_AutoLogonUser` を attach、MenuName を `[AUTO:User] AutoLogon Configuration` に書き換え
- `__RESTART__` / `__REEXPLORER__` / `__GATE__`: `specialMarkers` ハッシュ（common.ps1 L3675-3679）に登録された 3 種。専用 PSCustomObject を生成し、それぞれ `_IsRestart` / `_IsReexplorer` / `_IsGate` を立てる（MenuName は `[RESTART]` / `[REEXPLORER]` / `[GATE]`）。`__GATE__` 行は実行されない checkpoint で、Order 位置のみが前進バリア判定に効く（後述「__GATE__ 前進バリア」）
- 通常 ScriptPath: AllModules リストから `RelativePath` 完全一致で探索、見つかれば copy + Order/Segment/ErrorMode/Group を attach

### IncludeDisabled スイッチ（FlexProfile 専用）

- 通常: `Enabled=1` 行のみ返す
- `-IncludeDisabled`: 全行返し、各 module に `_IsCheckedDefault`（Enabled の元値）を attach。dashboard はこれを見て初期チェックボックス状態を決定

`__AUTOPILOT__` / `__ASYNC__` マーカーは `IncludeDisabled` でも `Enabled=1` のときのみ効果を発揮する（disabled marker 行が global state を勝手に flip しない安全弁）。

---

## Invoke-WindowsUpdateLoop（独立した再起動ループ）

`main.ps1` L872。Windows Update のリブート跨ぎを `__RESTART__` と同じ仕組みで自動化。

### 制御の輪

```
windows_update.ps1（1 pass で WU を 1 サイクル回し RebootRequired を返す）
   ↓
wu_state.json（LoopCount, MaxLoops, InstalledKBs, FailedKBs, StartTime, RebootSec, AutoLogon）
   ↓ RebootRequired && InstalledCount > 0 && LoopCount < MaxLoops:
   Set-WindowsUpdateAutoLogon（autologon_list.csv から該当 user を引いて
                              AutoLogonCount = MaxLoops*2 で書く。CBS 消費分）
   Register-FabriqRunOnce（Profile __RESTART__ と同じ entry）
   Invoke-CountdownRestart
   ↓ 再起動 → RunOnce → Fabriq.exe 自動起動
   ↓ main.ps1 冒頭で Test-Path wu_state.json → Invoke-WindowsUpdateLoop 再入
   ↓ LoopCount++ で次の pass、繰り返し
   ↓ RebootRequired が消える or LoopCount >= MaxLoops:
   Clear-WindowsUpdateAutoLogon（registry 残留 password / count を消す）
   Show-WindowsUpdateSummary
   wu_completed.json（finalize 用 metadata）を出力
```

**Phantom KB 検出**: `InstalledKBs` で同じ KB が 3 回以上現れたら `skipKBs` に入れて以後インストール対象から除外（再出現する不良 KB 対策）。

`MaxRebootLoops` / `RebootCountdownSeconds` / `AutoLogonEnabled` は `windows_update_list.csv` から読む（デフォ 5 / 15 / true）。

---

## Complete-ProfileExecution（finalize パイプライン）

profile 実行末尾の集約処理。Linear と FlexProfile `[Complete]` の両方で呼ばれる。

```
Mode='Auto' or 'Manual'
   ↓
1. Export-ExecutionHistory       ── execution_history.csv → logs/history/history_export_*.csv
                                  + evidence/{base}/export_history/ にもコピー
2. Export-HtmlChecklist           ── evidence/{base}/checklist/checklist_*.html を生成
                                    プロファイル定義 vs 実行結果のクロスマップ
                                    ネットワーク照合 / プリンタ照合 / Windows License /
                                    BitLocker ステータスを含む完全な audit レポート
3. Mode='Auto'（Linear）:
   - log_uploader を silent mode で発火（バックグラウンド送信）
   - view_report.ps1 で HTML を最後に開く
   Mode='Manual'（Flex [Complete]）:
   - 同上だが UI 順序が異なる（operator が成果物確認後に発火するため）
```

HTML チェックリストの中身は §07_evidence_history.md に詳述。
