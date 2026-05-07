# FlexProfile ダッシュボード操作

> **対象**: fabriq / usage
> **対象バージョン**: kernel 3.2.2（取得元: `E:\fabriq\kernel\KERNEL_VERSION`）+ commit `e513cf1`（取得元: `git -C E:\fabriq rev-parse --short HEAD`、2026-05-06）
> **ドキュメント更新日**: 2026-05-07

`[Execute (Flex)]` で開く **state-aware 部分実行ダッシュボード** の使い方。Linear（先頭から末尾まで自動進行）に対し、Flex は「**1 モジュール単位で再実行可能 + 完了タイミングは operator 判断**」の運用モデル。kernel 3.1.0 で導入、3.1.x 系で安定化、3.2.0 で `Group` 列追加。

---

## 前提

- セッション確立済み（[fabriq__usage__02_session_setup.md](fabriq__usage__02_session_setup.md)）
- `Profiles` タブで対象 profile を選択 → `[Execute (Flex)]`（緑ボタン）押下で本ダッシュボードが起動
- Linear 一括実行については [fabriq__usage__03_profile_execution_linear.md](fabriq__usage__03_profile_execution_linear.md)

---

## いつ Flex を使うか

| シナリオ | Flex が向く理由 |
|---|---|
| 失敗箇所だけ再実行したい | チェックボックスで失敗行のみピック → `[Run Selected]` |
| 段階的に進めたい（人間判断挟む） | Profile 全行を一気に流さず、Group 単位 / 1 行単位で進められる |
| 設定ミスを途中で気づいて止め直したい | `[Mark as Pending]`（右クリック）で 1 行だけ状態リセット → 再実行 |
| 完了処理（HTML 生成 / log_uploader）を確認後に実行したい | `[Complete]` 押下まで finalize を保留できる |
| 同 Profile 内で `Group` 単位の運用を分けたい | `[Run: <Group>]` で 1 クリック実行（kernel 3.2.0+）|

逆に **「Profile 先頭から末尾まで一気通貫で流す + 自動 finalize 」** が要件なら Linear（[fabriq__usage__03_profile_execution_linear.md](fabriq__usage__03_profile_execution_linear.md)）の方がシンプル。

---

## 実行モデル: 「**実行 = 常に AutoPilot / 完了 = 常に手動**」

kernel 3.1.5 以降の Flex は AutoPilot トグルを **持たない**。`[Run]` / `[Run Selected]` / `[Run: Group]` のいずれを押しても：

- **`AutoPilot = $true` を強制**（`Confirm-Execution` / `Wait-KeyPress` を自動 Y / Enter）
- **`FinalizeOnComplete = $false` を強制**（HTML 生成 + log_uploader を保留）

この設計が示す哲学：

- **「どのモジュールを動かすか」だけが operator の意思決定**
- 実行中の Y/N プロンプトは AutoPilot で消える
- 完了処理（HTML 出力・evidence upload）は **`[Complete]` 押下まで明示的に保留** されるため、operator が結果を確認してから finalize できる

`[Mark as Pending]` で行をリセットしたり、`[Restart Now]` で再起動したり、状態を作り変えながら段階的に進めて、最後に納得したら `[Complete]` する。

---

## ダッシュボード構成

ウィンドウサイズ 900 幅。`Group` 列の有無で高さが変わる（Groups bar 40 px 分）。

```
┌────────────────────────────────────────────────────────────────────────────┐
│ Header: FlexProfile: <ProfileName>     |  Last finalized: HH:MM:SS         │
│   または                                                                    │
│   FlexProfile: <ProfileName>           [赤バッジ: PENDING FINALIZE]        │
├────────────────────────────────────────────────────────────────────────────┤
│ Groups bar （CSV に Group 列があり値が入っている場合のみ表示、40 px）        │
│   [Run: NetworkBlock]  [Run: Tweaks]  [Run: Cleanup]    ...                 │
├────────────────────────────────────────────────────────────────────────────┤
│ DataGridView（高さ 470 px）                                                  │
│   ┌──────┬──┬──────┬──────────┬─────────┬─────────┬──────┐                │
│   │ Chk  │# │Group │ Module   │ Status  │Verified │ Run  │                │
│   ├──────┼──┼──────┼──────────┼─────────┼─────────┼──────┤                │
│   │ ☐    │10│      │__AUTO... │ Success │   -     │[Run] │                │
│   │ ☑    │20│Net   │hostname… │ Success │ ●PASS   │[Run] │                │
│   │ ☐    │30│Net   │ipaddres… │ Error   │ ●FAIL   │[Run] │                │
│   │ ☐    │40│      │__RESTART │ Success │   -     │[Run] │                │
│   │ ...                                                                     │
│   └──────┴──┴──────┴──────────┴─────────┴─────────┴──────┘                │
├────────────────────────────────────────────────────────────────────────────┤
│ Footer Top:                                                                 │
│   [Select All] [Clear All]    WaitSec [3]   [Run Selected (N)]  [Restart Now]│
│ Footer Bottom:                                                              │
│   [Complete]  または  [Complete with N issues] / [Complete (nothing run)]   │
│   [Back]                                                                    │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 1. グリッド構成

| 列 | 幅 | 内容 |
|---|---|---|
| `Checked` | 36 | 選択チェックボックス。**既定すべて未チェック**（白紙からピック） |
| `#` | Order の整数値 |  |
| `Group` | （`Group` 列がある場合のみ表示） |  |
| `Module` | MenuName |  |
| `Status` | 90 | Pending / Success / Partial / Error / Skipped / Cancelled。CellFormatting で **背景色付きバッジ** 描画 |
| `Verified` | 70 | `-` / `PASS` / `FAIL`。PASS=緑バッジ、FAIL=赤バッジ |
| `Run` | 56 | 行内 `[Run]` ボタン（kernel 3.1.8+。footer の "Run This: M" を置き換え） |

### Status バッジの色

| Status | 背景色 | 文字色 |
|---|---|---|
| `Success` | 薄緑 | 緑（太字）|
| `Partial` | 薄黄 | オレンジ（太字）|
| `Error` | 薄赤 | 赤（太字）|
| `Skipped` / `Cancelled` | 薄グレー | グレー |
| `Pending` | より薄いグレー | グレー（イタリック）|

### State precedence（状態の決定順）

`Show-FlexDashboard` が起動時に各行の Status を決定する優先順（後勝ち）：

1. **すべて Pending として seed**（profile 全行）
2. **`execution_history.csv` を上書き**（`SELECTED_KANRI_NO` + 現 `SessionID` で絞った最新エントリ）
   - 第 1 lookup: `Order` 一致（複数同一 MenuName 行を区別、kernel 3.1.7+）
   - 第 2 lookup（fallback）: MenuName 一致、ただし `Order=0`（マーカー / [RESTART NOW] 等）か Order が一致するときのみ採用（兄弟行漏出防止）
3. **`$script:LastBatchResults` を上書き**（直前 batch の in-process 記録、最新かつ history.csv 同期前の真値）

これで「履歴 CSV の遅れ」と「兄弟行同名の混同」を両立的に解決している。

### 右クリックメニュー: `[Mark as Pending (reset state)]`

任意の行を右クリック → 該当行を即時に Pending に戻す。`Action="ResetState"` 経由で：

- `Add-ExecutionResult -Operation <MenuName> -Status "Pending" -Message "Reset by operator" -Order <Order>`
- `Write-ExecutionHistory ...`（execution_history.csv 追記）
- `$pendingFinalize = $true`（finalize 提案バッジが立つ）

`Order` を渡すことで「同 MenuName の兄弟行」の片方だけをリセットできる。

---

## 2. Run の 3 経路

### 2-1. 行内 `[Run]` ボタン（RunSingle）

該当行 1 件のみを即時実行。

`main.ps1` `Invoke-FlexProfileLoop` の `"RunSingle"` 分岐：

```powershell
$global:AutoConfirmMode = $true     # Y/N + Press-Enter を自動短絡
try {
    Invoke-BatchExecution -SelectedModules @($tgt) `
        -ProfilePath $ProfilePath `
        -ProfileName $ProfileName `
        -FinalizeOnComplete:$false `
        -ExecutionMode 'Flex' `
        -SelectedOrders @([int]$tgt.Order) `
        -FullProfileModules $resolved.ValidModules
} finally {
    $global:AutoConfirmMode = $false
}
$pendingFinalize = $true
```

`AutoConfirmMode` は AutoPilot のサブセット動作（モジュール間 wait なし、Y/N と Press-Enter のみ自動）。1 モジュール単発実行に最適。

### 2-2. `[Run Selected (N)]`（RunBatch）

下部 footer 中央の大ボタン。チェックボックス ON 行を Order 昇順で一括実行。

```powershell
Invoke-BatchExecution -SelectedModules $batch `
    -AutoPilot:$true `              # 強制 AutoPilot
    -AutoPilotWaitSec $flex.AutoPilotWaitSec `   # WaitSec NumericUpDown
    -ProfilePath $ProfilePath `
    -ProfileName $ProfileName `
    -FinalizeOnComplete:$false `
    -ExecutionMode 'Flex' `
    -SelectedOrders @($flex.SelectedOrders) `
    -FullProfileModules $resolved.ValidModules
$pendingFinalize = $true
```

- 選択行 0 件で押すと `Show-Warning "FlexProfile: no modules matched the selected orders"`
- WaitSec NumericUpDown で AutoPilot のモジュール間 wait（既定 3 秒）を変更
- ボタン文言は動的に `[Run Selected (N)]`（N = チェック済み件数）

### 2-3. `[Run: <Group>]`（RunGroup、kernel 3.2.0+）

Profile CSV に `Group` 列があり値が入っている場合のみ Header 直下に表示される Groups bar。

```powershell
Invoke-BatchExecution -SelectedModules $batch `   # Group 値一致の行のみ
    -AutoPilot:$true `
    -AutoPilotWaitSec $flex.AutoPilotWaitSec `
    -ProfilePath $ProfilePath `
    -ProfileName $ProfileName `
    -FinalizeOnComplete:$false `
    -ExecutionMode 'Flex' `
    -SelectedOrders @($flex.SelectedOrders) `
    -FullProfileModules $resolved.ValidModules
$pendingFinalize = $true
```

ボタンは CSV の Group 値出現順、横幅 130px × N（最大 6 個程度を想定、auto-wrap 無し）。

### Group 跨ぎ `__RESTART__` の literal interpretation（kernel 3.2.0+）

profile CSV に `Group=Net` の行群と `Group=Tweaks` の行群があり、間に `__RESTART__`（Group 空）がある場合、`[Run: Net]` を押すと **`__RESTART__` は対象外（Group 不一致でスキップ）**。

```csv
10,standard/hostname_config/...,1,...,,,Net
20,standard/ipaddress_config/...,1,...,,,Net
30,__RESTART__,1,再起動,,,         ← Group 空、Net group には含まれない
40,standard/reg_hklm_config/...,1,...,,,Tweaks
```

`[Run: Net]` 押下時：10 と 20 のみ実行、`__RESTART__` は無視される。

operator は **`__RESTART__` を所属させたい Group に明示的に書く** 必要がある（literal interpretation）：

```csv
30,__RESTART__,1,再起動,,,Net    ← 明示的に Net 所属。Run: Net で発火する
```

---

## 3. Footer ボタン群

### 上段

| ボタン | 動作 |
|---|---|
| `[Select All]` | **CSV `Enabled=1` 行のみ** にチェック（マーカーや Disabled 行は除外） |
| `[Clear All]` | 全チェック解除 |
| `WaitSec` NumericUpDown | AutoPilot Wait 秒数（既定 3） |
| `[Run Selected (N)]` | 上記 RunBatch |
| `[Restart Now]` | profile 外から `__RESTART__` を発火（後述） |

### 下段

| ボタン | 動作 |
|---|---|
| `[Complete]` | finalize 発火（`Complete-ProfileExecution -Mode 'Manual'`）|
| `[Back]` | dashboard 終了、main loop に戻る |

---

## 4. `[Complete]` の動的表示

ボタン文言・色は status に応じて動的変化：

| 条件 | 文言 | 色 |
|---|---|---|
| Issue（Error / Partial / Pending で ✓ された行）が 1 件以上 | `Complete with N issues` | 黄色 |
| 何も実行していない（全 Pending）| `Complete (nothing executed)` | 黄色 |
| それ以外（クリーン完了） | `Complete` | 緑 |

押下時に Issue があれば Yes/No 確認ダイアログを出し、Yes ならフィナライズ進行：

1. `Complete-ProfileExecution -Mode 'Manual'`
2. HTML チェックリスト生成（viewer-before-upload）
3. `extended/log_uploader` 自動発火
4. `"Log Upload (cl)"` を実行履歴に追加（手動 finalize の trail）
5. `lastFinalizedAt = HH:MM:SS` で header 表示更新
6. `pendingFinalize = false`

`Wait-KeyPress` の後 dashboard 再オープン。再度 Run を押すと `pendingFinalize` は再び true になり、再 finalize が必要になる。

---

## 5. PendingFinalize 警告

`pendingFinalize=true` の状態（Run の後 / `[Mark as Pending]` の後）で：

- Header の "Last finalized:" の代わりに **赤バッジ "PENDING FINALIZE"** を表示
- `[Back]` または X-button で閉じようとすると **確認ダイアログ**：

```
You have run modules but not yet pressed [Complete].

If you exit now:
  - The HTML checklist will not be regenerated
  - Evidence will not be uploaded via log_uploader
  - Your work will be saved to history.csv but not delivered

Are you sure you want to exit?

[Yes (exit anyway)]  [No (return to dashboard)]
```

`Yes` で `Action="Close"` 返却、main.ps1 が main loop に戻る。

`No` ならダッシュボードに留まり、`[Complete]` を押せる。

---

## 6. `[Restart Now]`（プロファイル外 `__RESTART__`）

Profile 内の `__RESTART__` 行が無くても、operator が任意のタイミングで再起動 + 自動再開できる。

`Invoke-FlexProfileLoop` の `"RestartNow"` 分岐：

1. **現在の状態スナップショット構築**: profile 全行に対して `execution_history.csv` から最新 status を引いて `ModuleStates` ハッシュテーブルを作る（Pending 含む）
2. **`Save-ResumeState`** with `ResumeAfterOrder = -1` sentinel + `ExecutionMode = 'Flex'` + `ModuleStates`
3. **`Register-FabriqRunOnce`** 失敗時は `Remove-ResumeState` + 中断
4. `Add-ExecutionResult / Write-ExecutionHistory` に `[RESTART NOW]` を Order=0 で記録
5. **`Invoke-CountdownRestart`** （カウントダウン → Restart-Computer、process exit）

再起動後の挙動：

- main.ps1 Resume Detection が `resume_state.json` を読む（`schemaVersion=2 + ExecutionMode=Flex`）
- AutoPilot 中扱いで `Wait-SystemReady + Invoke-AutoResumeCountdown 60s`
- **Linear 風の "Order > N の残りモジュールを自動継続" は走らない**（`ResumeAfterOrder=-1` sentinel）
- `Invoke-FlexProfileLoop` が再オープン、operator が再度 Run / Complete を選ぶ

これにより「再起動後の状態を確認してから次の操作を選ぶ」運用が可能。Linear `__RESTART__` は再起動後に必ず次モジュールから自動継続するため、より「人間判断を挟む」性質が違う。

---

## 7. Linear と Flex の動作差分（5 観点）

| 観点 | Linear | Flex |
|---|---|---|
| `[AutoPilot]` トグル | あり（チェックボックスで Profile 全体に適用） | 無し（Run 系は常に AutoPilot 強制） |
| 完了タイミング | profile 完走で自動 finalize（`FinalizeOnComplete=true`） | `[Complete]` 押下まで保留（`FinalizeOnComplete=false`） |
| `Group` 列利用 | 無視 | `[Run: <Group>]` ボタン化 |
| `__RESTART__` 後再開 | 中断 Order の次から自動継続（Linear 続行） | dashboard 再オープン（残り自動継続なし）|
| ResumeState schema | `schemaVersion=1`（v1 互換） | `schemaVersion=2`（`SelectedOrders` + `ModuleStates` 拡張）|

---

## 8. Action 値の dispatch 表

`Show-FlexDashboard` が返す `Action` 値と main.ps1 の `Invoke-FlexProfileLoop` の処理：

| Action | UI トリガ | 主な処理 |
|---|---|---|
| `Close` | `[Back]` / X-button | `Remove-ResumeState`（defensive）+ loop exit |
| `RunSingle` | 行内 `[Run]` | 1 module + AutoConfirmMode、`pendingFinalize=true` |
| `RunBatch` | `[Run Selected (N)]` | 複数 module + AutoPilot 強制、`pendingFinalize=true` |
| `RunGroup` | `[Run: <Group>]` | Group 一致 module + AutoPilot 強制、`pendingFinalize=true` |
| `Complete` | `[Complete]` | `Complete-ProfileExecution -Mode 'Manual'`、HTML 生成 + log_uploader、`pendingFinalize=false` |
| `RestartNow` | `[Restart Now]` | `Save-ResumeState` + `Register-FabriqRunOnce` + `Invoke-CountdownRestart`（process exit）|
| `ResetState` | 行右クリック → `[Mark as Pending]` | 該当行を Pending に書き戻し、`pendingFinalize=true` |

---

## トラブル対応

### 全行 Pending に見える / 履歴が反映されない

state precedence の起点 `SELECTED_KANRI_NO` が空、または `SessionID` が違う：

- `Refabriq` / `New Session` 直後は SessionID が新規生成されるため履歴が見えない（**仕様**）
- `KanriNo` が空の hostlist 行を選んでセッション開始した場合、`Import-ExecutionHistory -FilterKanriNo` が空配列を返す
- 対処: hostlist の `AdminID` 列に値を入れて再選択

### `[Run: <Group>]` ボタンが表示されない

profile CSV の `Group` 列がすべて空、または列自体が無い：

- 古い profile（kernel 3.1.x 以前）には `Group` 列が無い → そのまま使える、Groups bar は非表示
- Group 列を追加するには profile CSV を編集（`Open CSV Editor` 経由）

### `[Run Selected (N)]` が `0` のまま

チェックボックス ON 行が 0 件。`[Select All]` を押すと **CSV `Enabled=1` 行のみ** チェックが入る。

### `[Complete]` を押し忘れて `[Back]` した

`pendingFinalize=true` のまま閉じた → 確認ダイアログで `Yes (exit anyway)` した場合：

- 履歴 CSV には status が残っている（次回起動時に Restore-ExecutionHistory で復元される）
- HTML チェックリストは生成されていない → ダッシュボード `[Regenerate Checklist]` で再生成可能（[fabriq__usage__05_evidence_and_quick_actions.md](fabriq__usage__05_evidence_and_quick_actions.md)）
- `extended/log_uploader` も発火していないため、evidence は手動アップロード or 次回 finalize 時に一括

### `[Restart Now]` 後に Flex に戻らない

- `resume_state.json` が壊れている / 削除された → main.ps1 が Fresh Start に流れる。手動で `[Execute (Flex)]` から再開
- RunOnce 登録失敗 → `Register-FabriqRunOnce` ログを確認、UAC や RunOnce レジストリ権限を点検

### Status が "Pending" のまま `[Mark as Pending]` で変わらない

`Order` が 0 の特殊行（`[RESTART NOW]` 等）は state map に乗らない。マーカー行は元々 Pending を持たない設計。

---

## 関連ドキュメント

- Linear 一括実行（同 Profile を別経路で）: [fabriq__usage__03_profile_execution_linear.md](fabriq__usage__03_profile_execution_linear.md)
- セッション開始: [fabriq__usage__02_session_setup.md](fabriq__usage__02_session_setup.md)
- Evidence と Quick Actions: [fabriq__usage__05_evidence_and_quick_actions.md](fabriq__usage__05_evidence_and_quick_actions.md)
- ダッシュボード UI 仕様（FlexProfile dashboard 部分）: [fabriq__apps__01_fabriq_operator_dashboard.md](fabriq__apps__01_fabriq_operator_dashboard.md)
- 特殊マーカー契約: [fabriq__contracts__special_markers.md](fabriq__contracts__special_markers.md)
- Profile CSV `Group` 列契約: [fabriq__contracts__profile_csv_schema.md](fabriq__contracts__profile_csv_schema.md)
- 再起動跨ぎ実装（resume_state.json schemaVersion=2）: [fabriq__kernel__05_resume_restart.md](fabriq__kernel__05_resume_restart.md)
