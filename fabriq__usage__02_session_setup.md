# セッション開始フォームの操作

> **対象**: fabriq / usage
> **対象バージョン**: kernel 3.6.0（取得元: `E:\fabriq\kernel\KERNEL_VERSION`）+ commit `0fca159`（取得元: `git -C E:\fabriq rev-parse --short HEAD`、2026-06-16）
> **ドキュメント更新日**: 2026-06-16

`Fabriq.exe` 起動 → resume 検出を抜けた fresh start で表示される **`Show-SessionSetupForm`**（`apps/fabriq_operator/lib/session_form.ps1`）の操作手順。Worker / Target Host / Master Passphrase の 3 入力を 1 ダイアログで取り、起動の前提条件である環境変数群を確定させる。

---

## 前提

- 初回起動が成功していること（[fabriq__usage__01_install_and_first_boot.md](fabriq__usage__01_install_and_first_boot.md) §「3. 初回起動」）
- `kernel/csv/hostlist.csv` に 1 行以上のエントリがあること
- `kernel/csv/workers.csv` に作業者リストがあれば DataGridView から選択可能（無い場合は手入力フォールバック）

---

## フォーム構成

ウィンドウサイズ 620 x 630 の WinForms フォーム。縦に 3 セクション：

```
┌──────────────────────────────────────────────────────────┐
│ fabriq operator (タイトル、32pt)                          │
├──────────────────────────────────────────────────────────┤
│ Worker                                                    │
│   ┌──────────────────────────────────────────────────┐   │
│   │ ID (80px) │ Name (Fill)                          │   │
│   │ ─────────────────────────────────────────────    │   │
│   │ ...（workers.csv の各行）                        │   │
│   └──────────────────────────────────────────────────┘   │
│   or enter manually: [____________________]              │
├──────────────────────────────────────────────────────────┤
│ Target Host                                               │
│   Search: [_________________]              X / Y         │
│   ┌──────────────────────────────────────────────────┐   │
│   │ ID │ OldPC │ NewPC │ IP │ Pin                    │   │
│   │ ─────────────────────────────────────────────    │   │
│   │ ...                                              │   │
│   └──────────────────────────────────────────────────┘   │
│   * Auto-detected: <CurrentPCName> (matches current PC)  │
├──────────────────────────────────────────────────────────┤
│ Master Passphrase                                         │
│   [●●●●●●●●●●●●●●●_________________]                     │
│   <validation message label>                              │
│                                                          │
│              [ Quit ]    [ Start Session ]                │
└──────────────────────────────────────────────────────────┘
```

このフォームは fabriq の中で **マウスショートカット例外**（他は基本マウスオンリー設計）。検索効率のため一部キーボード操作が許容されている。

---

## 1. Worker 選択

`kernel/csv/workers.csv` から作業者を選ぶ。**DataGridView 選択** と **手入力テキストボックス** が排他で動作する。

### DataGridView

| 列 | 幅 | sort |
|---|---|---|
| `ID` | 80px 固定 | 有効（ヘッダクリック） |
| `Name` | Fill | 有効 |

初回表示で先頭行が自動選択される（`Rows[0].Selected = true`）。ヘッダクリックで sort 可能、行クリックで選択。

### 手入力テキストボックス

ラベル `or enter manually:`、260px 幅。一覧外の作業者用（飛び込み作業者など）。

### 排他動作

- **手入力 BOX に文字を入れた瞬間**: `workerGrid.ClearSelection()` で grid 側の選択を解除
- **grid のセルクリック**: `manualWorkerBox.Text = ""` で手入力 BOX をクリア

両方を同時に持たない設計。Start Session 押下時は **手入力に値があれば優先**、なければ grid 選択を採用：

```
if (手入力 BOX に値あり)
    WorkerID   = "MANUAL"
    WorkerName = 手入力値（trim 済み）
elseif (grid に選択行あり)
    WorkerID   = grid 行の ID 列
    WorkerName = grid 行の Name 列
else
    "Please select a worker or enter a name manually." メッセージ表示、Start Session 中断
```

`WorkerName` が空白のみだと `"Worker name cannot be empty."` でも中断する。

---

## 2. Target Host 選択

`kernel/csv/hostlist.csv` 全行から対象 PC を選ぶ。ライブ検索 + auto-detect + 件数表示の 3 機能を備える。

### DataGridView 列

| 列 | 幅 | 内容 |
|---|---|---|
| `AdminID` | 50px | 管理 ID |
| `OldPCName` | 140px | 旧 PC 名（出荷時名・ベンダ名等） |
| `NewPCName` | Fill | 新 PC 名（突合キー、最終的に PC に設定する名前） |
| `EthernetIP` | 130px | 静的 IP（暗号化 `ENC:` 値の場合は復号後に表示されない、生のまま） |
| `Pin` | 80px | BitLocker PIN |

各列はヘッダクリックで sort。再生成時にも `Row.Tag` に元 host object を保持しているため、選択は sort / filter で行が並び替わっても消えない。

### Search ライブフィルタ

検索ボックス（380px 幅）で **AdminID + NewPCName のみ** が対象（case-insensitive substring）。

意図的に **OldPCName / EthernetIP / Pin は検索対象外**：

| 除外列 | 除外理由 |
|---|---|
| `OldPCName` | 出荷時名は operator が記憶していないことが多く、検索のシグナルとしてノイズ |
| `EthernetIP` | サイトごとに IP レンジが違い、複数 hit しやすい |
| `Pin` | 機密情報で部分検索の対象として不適 |

入力ごとに `Rows.Clear()` → 該当行のみ再構築。再構築後の選択ロジック：

```
if (検索 BOX 空 + auto_selected_host あり)
    auto_selected_host を再選択（Row.Tag 比較で物理位置に依らず特定）
else
    先頭行を選択
```

### 件数ラベル

`X / Y` 形式（`X` = フィルタ後件数、`Y` = 全件）。検索無しなら `Y / Y`。

### Auto-detect（緑ハイライト）

`$env:COMPUTERNAME` と一致する `NewPCName` がリストにあれば、起動時にその行を自動選択し、grid 直下に：

```
* Auto-detected: <CurrentPCName> (matches current PC)
```

の緑色ヒントラベル（RGB `46, 125, 50`）を表示する。一致なしならラベルは生成されない。

このマッチングは object reference で保持される（`autoSelectedHost` 変数）。検索フィルタで該当行が消えても変数は残り、検索を空に戻すと再度自動選択される。

### キーボードショートカット

| キー | 動作 |
|---|---|
| 検索 BOX で `Esc` | 検索テキストをクリア（フィルタ解除） |
| 検索 BOX で `Enter` | passphrase BOX へフォーカス移動 |

---

## 3. Master Passphrase 入力

400px 幅のテキストボックス。`UseSystemPasswordChar = true` で `●` 表示。検証は `Test-MasterPassphrase`（`common.ps1` L508）：

```
1. kernel/txt/passphrase_verify.txt を読み込み（先頭が "ENC:" であることを確認）
2. 入力パスフレーズで ENC: 値を Unprotect-FabriqValue で復号
3. 復号結果が "surkitinisme" と一致するか比較
4. 一致 → return true、それ以外（復号失敗含む） → return false
```

### 検証失敗時の挙動

`"Passphrase verification failed. Please try again."` を赤文字 message label に表示し：

- `ppBox.SelectAll()` で入力済み全文を選択状態に（次の打鍵で上書きされる）
- `ppBox.Focus()` でフォーカスをパスフレーズ BOX に戻す

何度でも再試行可（リトライ回数の制限なし）。

### キーボードショートカット

| キー | 動作 |
|---|---|
| パスフレーズ BOX で `Enter` | `[Start Session]` ボタンを `PerformClick()`（クリック相当） |

---

## 4. [Start Session] / [Quit]

| ボタン | 位置 | 動作 |
|---|---|---|
| `[Start Session]` | 右、140x34、aqua アクセント色、bold | `Cancelled = false` で form 閉じる、result hashtable を返す |
| `[Quit]` | その左、120x34 | `Cancelled = true` で form 閉じる |

`[Start Session]` の検証順：

1. Worker 確定（手入力優先 → grid 選択 → なければエラー）
2. Worker 名 trim 後空でないこと
3. Host 選択行が存在すること（`SelectedRows.Count > 0`）
4. 選択行の `Tag` が host object として有効であること
5. Passphrase 空でないこと
6. `passphrase_verify.txt` 存在時は `Test-MasterPassphrase` で検証

すべて通れば `result.Cancelled = false` で次フェーズへ。

### Quit を選んだ場合

`Cancelled = true` の result が `kernel/main.ps1` 側に返り、main.ps1 は `Show-Info "Session setup cancelled. Exiting."` 表示後に通常終了する。

---

## 5. セッション確定後に走る処理

`Show-SessionSetupForm` から `Cancelled=false` で戻ると、`kernel/main.ps1` は次の関数を順に呼ぶ：

```
1. Set-SelectedHostEnvironment $SelectedHost
   ─ 選択 host から SELECTED_* env vars を確定（後述）

2. Initialize-EvidenceBasePath
   ─ {SessionTimestamp}_{PCName}_{SerialNumber}_evidence/ ディレクトリ作成
   ─ $global:FabriqEvidenceBasePath / $env:FABRIQ_EVIDENCE_BASE をセット

3. Initialize-Session
   ─ $env:FABRIQ_WORKER_NAME = WorkerName
   ─ MediaSerial 確定（session.json 既存 → source_media.id → "UNKNOWN" の優先順）
   ─ kernel/json/session.json 書き込み

4. Initialize-ExecutionHistory
   ─ logs/history/execution_history.csv 作成 + 旧パスからの migrate

5. Initialize-ModuleSystem
   ─ modules/{standard,extended}/*/module.csv を再帰検出 + categories.csv 順序適用

6. Load-Profiles
   ─ profiles/*.csv 一覧化（無ければ Basic Setup / Full Setup を生成）

7. Show-ExecutionToolbar
   ─ kernel の powershell.exe 内の専用 STA Runspace で in-process Execution Toolbar を表示
   ─ 旧 out-of-process Status Monitor（status_monitor.ps1 / Start-StatusMonitor）は
     3.4.0 で非推奨化・3.5.0 で削除済み。別プロセスを spawn しないため Defender/ASR の
     子プロセス制限を受けない

8. Show-OperatorDashboard
   ─ メインダッシュボード表示（無限ループ）
```

### Set-SelectedHostEnvironment が立てる SELECTED_* env vars

`hostlist.csv` の 1 行から計 30+ の環境変数を生成する：

| 変数 | 出処列 | 備考 |
|---|---|---|
| `SELECTED_KANRI_NO` | `AdminID` | 管理 ID |
| `SELECTED_OLD_PCNAME` | `OldPCName` | 旧 PC 名 |
| `SELECTED_NEW_PCNAME` | `NewPCName` | **新 PC 名（突合キー）** |
| `SELECTED_ETH_IP / _SUBNET / _GATEWAY` | `EthernetIP / EthernetSubnet / EthernetGateway` | 有線 LAN 設定（3 件） |
| `SELECTED_WIFI_IP / _SUBNET / _GATEWAY` | `WifiIP / WifiSubnet / WifiGateway` | 無線 LAN 設定（3 件） |
| `SELECTED_DNS1` 〜 `SELECTED_DNS4` | `DNS1` 〜 `DNS4` | DNS 4 件 |
| `SELECTED_PIN` | `Pin` | BitLocker PIN |
| `SELECTED_PRINTER_<N>_NAME / _DRIVER / _PORT`（N=1..10） | `Printer{N}Name / Printer{N}Driver / Printer{N}Port` | プリンタ 30 列 |

### `ENC:` 値の透過復号

`Set-SelectedHostEnvironment` 内の `Resolve-HostValue` ヘルパが、値が `ENC:` で始まる場合に `$global:FabriqMasterPassphrase` で `Unprotect-FabriqValue` を呼んで復号する：

```powershell
function Resolve-HostValue {
    param([string]$Value)
    if ($Value.StartsWith('ENC:') -and -not [string]::IsNullOrWhiteSpace($global:FabriqMasterPassphrase)) {
        return (Unprotect-FabriqValue -EncryptedValue $Value -Passphrase $global:FabriqMasterPassphrase)
    }
    return $Value
}
```

復号失敗時は `Show-Warning "Failed to decrypt host value: ..."` をログ出力後、暗号文のまま env var に入れる。**モジュール側はパスフレーズを意識せず env var を読むだけ**で、機密値が透過的に手に入る。

---

## 6. セッション切替（New Session）

ダッシュボードに到達後、Settings タブ末尾の `[New Session]` ボタンで再度この form を呼べる。`main.ps1` のディスパッチで **`Reset-FabriqState` が呼ばれてから** session form が再表示されるため、フォームを 1 度通すごとに実質的に「セッションのフルリセット + 再構築」が起きる。

`Reset-FabriqState`（`common.ps1` L2781）の作用範囲：

| カテゴリ | 動作 |
|---|---|
| Transcript | `Stop-Transcript` 後に新ファイル `logs/{newTs}_{uid}_{hn}.log` で `Start-Transcript` |
| 実行結果 in-memory | `$script:ExecutionResults / LastBatchResults` を空配列、`SessionID` を新規生成 |
| 実行履歴 CSV | `logs/history/execution_history.csv` と `.bak` を削除 |
| `session.json` | 削除（worker 再選択を強制） |
| グローバルフラグ | `AutoPilotMode = false` / `AutoPilotWaitSec = 3` / `_LastModuleResult = null` |
| Evidence Base Path | `$global:FabriqEvidenceBasePath / FabriqEvidenceRootPath / $env:FABRIQ_EVIDENCE_BASE` を null |
| Host 環境変数 | `SELECTED_*` 14 種 + `FABRIQ_AUTOLOGON_USER` + `SELECTED_PRINTER_<N>_NAME/DRIVER/PORT`（N=1..10）を null |
| Resume State | `Remove-ResumeState` で `kernel/json/resume_state.json` 削除 |
| Status File | `Write-StatusFile -Phase "idle"` |

**Evidence ファイル本体（PNG / HTML / 既出力ディレクトリ）は削除されない**。`Reset-FabriqState` は in-memory + 設定ファイルのみを対象にする。

ダッシュボード Quick Actions の **`[Refabriq]` ボタン**と紛らわしいが、両者は別動作：

| ボタン | 実装 | 性質 |
|---|---|---|
| `[New Session]` | `Reset-FabriqState` を同 process で呼ぶ → session form 再表示 | 中強度（PID 維持、state を明示クリア） |
| `[Refabriq]` | `Hide-ExecutionToolbar` + `Remove-StatusFile` + `Start-Process Fabriq.exe` + `Stop-Transcript` + `exit 0` | 高強度（PID 入れ替え、cold boot 相当） |

`[Refabriq]` は state ファイル（`session.json` / `execution_history.csv` / `resume_state.json`）を **明示削除しない**。新 process が起動時の Resume Detection や `Initialize-Session` の優先順位に従ってそれらを再利用する可能性がある。`[New Session]` の方が **意図的に履歴を白紙にする** 用途に合致する。詳細は [fabriq__usage__05_evidence_and_quick_actions.md](fabriq__usage__05_evidence_and_quick_actions.md) §「Refabriq」。

---

## トラブル対応

### Auto-detect が効かない（緑ラベルが出ない）

`$env:COMPUTERNAME` が `hostlist.csv` の `NewPCName` 列のいずれとも一致していない。

**ありうる原因**:

- 対象 PC が rename 済みで、現在の名前が hostlist の `NewPCName` ではなく `OldPCName` と一致している（出荷時名の状態 → fabriq でこれから rename する想定）
- `NewPCName` 列に typo / 余分なスペース / 全角文字が混入

**対処**: 検索 BOX で AdminID または NewPCName の一部を入力して手動選択する。auto-detect は補助機能であり、無くても操作上の支障はない。

### Worker 一覧が空

`kernel/csv/workers.csv` が見つからない / 中身が空。

**対処**: 手入力 BOX で作業者名を入れる（`WorkerID = "MANUAL"` で記録される）。

### Host 一覧が空

`hostlist.csv` が見つからない場合は **そもそも fresh start ルートに到達しない**（`main.ps1` で exit 1）。中身は空でないが特定行が見えない場合：

- フィルタ BOX に文字が残っていないか確認
- `hostlist.csv` の文字コード（BOM 付き UTF-8 推奨）

### Passphrase が何度入れても通らない

[fabriq__usage__01_install_and_first_boot.md](fabriq__usage__01_install_and_first_boot.md) §「パスフレーズ入力で何度も拒否される」を参照。

### Pin 列に `ENC:abc...` が出ている

これは仕様。passphrase で復号せずに表示しているため、暗号文のまま見える。`SELECTED_PIN` 環境変数には Set-SelectedHostEnvironment 内で復号後の値が入る（モジュール側は透過的に平文を扱える）。

---

## 関連ドキュメント

- 初回起動とインストール: [fabriq__usage__01_install_and_first_boot.md](fabriq__usage__01_install_and_first_boot.md)
- Profile Linear 実行: [fabriq__usage__03_profile_execution_linear.md](fabriq__usage__03_profile_execution_linear.md)
- FlexProfile ダッシュボード: [fabriq__usage__04_flexprofile_dashboard.md](fabriq__usage__04_flexprofile_dashboard.md)
- Refabriq / Quick Actions: [fabriq__usage__05_evidence_and_quick_actions.md](fabriq__usage__05_evidence_and_quick_actions.md)
- ダッシュボード UI 仕様: [fabriq__apps__01_fabriq_operator_dashboard.md](fabriq__apps__01_fabriq_operator_dashboard.md)
- ホスト環境変数の公開契約: [fabriq__contracts__host_environment.md](fabriq__contracts__host_environment.md)
- カーネル全体像: [fabriq__kernel__01_overview.md](fabriq__kernel__01_overview.md)
