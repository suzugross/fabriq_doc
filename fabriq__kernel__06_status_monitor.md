# Execution Toolbar（旧 Status Monitor の後継）+ 演出 (ART pulse / silence / manifesto)

> **対象**: fabriq / kernel + apps/fabriq_operator（Execution Toolbar）
> **対象バージョン**: kernel 3.6.0（取得元: `E:\fabriq\kernel\KERNEL_VERSION`）+ commit `0fca159`（取得元: `git -C E:\fabriq rev-parse --short HEAD`、2026-06-16）
> **ドキュメント更新日**: 2026-06-16

fabriq の実行中ステータス表示は、kernel 3.4.0 で **別プロセス型 Status Monitor から in-process な Execution Toolbar へ全面移行**した。本ドキュメントは現行の Execution Toolbar を中心に記述する。

---

## 位置づけと履歴（重要）

- 旧 **Status Monitor**（`kernel/ps1/status_monitor.ps1` を `-WindowStyle Hidden` の別 PowerShell プロセスで起動し `status.json` を polling する方式）と、そのタイピング演出 `kernel/ps1/art_display.ps1` は kernel **3.4.0 で非推奨化・3.5.0 で物理削除**された。関連関数 `Start-StatusMonitor` / `Stop-StatusMonitor` / `Show-MonitorFailureDialog` も 3.5.0 で削除済み（取得元: `kernel/KERNEL_API.md` §8 [3.4.0]/[3.5.0]。両 `.ps1` はソースに存在しない）。
- 後継は **Execution Toolbar**（実装: `apps/fabriq_operator/lib/execution_toolbar.ps1`）。kernel の `powershell.exe` 内の **専用 STA Runspace** で動く in-process フローティングツールバー。別プロセスを spawn しないため、Defender / ASR の子プロセス制限（旧方式で hidden 子プロセスがブロックされ「モニタが出ない」障害になっていた問題）を受けない（取得元: `execution_toolbar.ps1` 冒頭コメント L1-25）。

旧 Status Monitor を直接呼んでいた手順・関数参照（`Start-StatusMonitor` / `Stop-StatusMonitor` / `Show-MonitorFailureDialog`、`-WindowStyle Hidden` 子プロセス起動、起動診断ログ `-DiagLogPath`、Exit codes 11-14、ウィンドウ存在ポーリングによる成否判定）は **すべて無効**。in-process Runspace では「子プロセス起動失敗」という失敗モード自体が存在しないため、それらの診断機構も廃止された。

---

## 公開/内部 API（since 3.4.0）

`kernel/KERNEL_API.md` では §6（内部実装）に分類される。**モジュールが依存してよい公開 API ではない**（取得元: `kernel/KERNEL_API.md` §6 / §8 [3.4.0]）。

- `Show-ExecutionToolbar` — 起動時にツールバーを開く（STA Runspace を生成）。
- `Hide-ExecutionToolbar` — 終了 / `__RESTART__` / 中断時にツールバーを閉じる。
- `Update-ExecutionToolbar [-ExecutionState 'Idle'|'Running'] [-ModuleName <s>] [-TargetHostInfo <hashtable>]` — 状態を共有 synchronized ハッシュテーブルへ push する。直接 UI を触らず、ツールバー Runspace 側の 1 秒 Timer が次 tick で反映する（`execution_toolbar.ps1` の `Update-ExecutionToolbar` L1269-1309）。
- `Save-Screenshot -BaseDir <string>`（since 3.4.0、公開化なし）— 任意タイミングで PNG を保存。`[Gyotaq]` ボタンが呼ぶ（`common.ps1` の `Save-Screenshot` L4787）。

---

## ライフサイクル（kernel/main.ps1 が駆動）

```
fabriq 起動
   ↓
Show-ExecutionToolbar                                         (main.ps1 L1702)  ── STA Runspace でツールバー生成
   ↓
ホスト選択時
   Update-ExecutionToolbar -TargetHostInfo (Get-FabriqHostInfoFromEnv)   (main.ps1 L203-204)
   ↓
各モジュール実行直前
   Update-ExecutionToolbar -ExecutionState 'Running' -ModuleName <名>     (main.ps1 L364, L583)
   ↓
モジュール間 / バッチ完了
   Update-ExecutionToolbar -ExecutionState 'Idle'                        (main.ps1 L766)
   ↓
Exit-Fabriq / __RESTART__ / 中断
   Hide-ExecutionToolbar                          (main.ps1 L530 / L1006 / L1268 / L2190 / L2202)
```

`Exit-Fabriq`（`common.ps1` L4323）は `Remove-StatusFile` → `Hide-ExecutionToolbar` の順でクリーンアップする（ソースコメントに「formerly Stop-StatusMonitor's job; the out-of-process Status Monitor was removed in 3.5.0」と明記）。

---

## ツールバーの構成要素

- **PC Info ペイン** — 期待値（hostlist 由来の `SELECTED_*`）と現在値を縦並びで照合表示する。
  - 期待値: `Update-ExecutionToolbar -TargetHostInfo` で push される `Get-FabriqHostInfoFromEnv` の戻り（Hostname / KanriNo / Pin / Eth{IP,Subnet,Gateway} / Wifi{IP,Subnet,Gateway} / DNS / Printers）。
  - 現在値: `Get-NetIPAddress` / `Get-Printer` 等の live OS クエリ。
  - **旧来の `status.json` の `PCInfo` / `CurrentPCInfo` ブロックには依存しない**（`execution_toolbar.ps1` 冒頭 L11-14、`Get-FabriqHostInfoFromEnv` L36-78）。
- **`[Skip]` ボタン** — async モジュールの強制中断要求。`Get-FabriqAsyncConfig` の `SkipFlagPath`（既定 `.\kernel\json\skip_request.flag`、`common.ps1` L1604）に「`requested at <ts>`」を書き込む。`Invoke-SafeCommandAsync` の polling loop がこのフラグを検出して Runspace を停止する。**`__ASYNC__` マーカ以降のモジュールにのみ有効**（ボタン ToolTip / `execution_toolbar.ps1` L1042-1066）。詳細は [fabriq__kernel__08_async_execution.md](fabriq__kernel__08_async_execution.md)。
- **`[Gyotaq]` ボタン** — 旧「Manual Screenshot」の後継。フォームを 300ms 退避 → `Save-Screenshot -BaseDir <EvidenceBasePath>\gyotaku`（`EvidenceBasePath` 未設定時は `<fabriqRoot>\evidence\gyotaku`）→ フォーム復帰（`execution_toolbar.ps1` L1068-1112）。
- **ART 演出パネル** — Surkitinisme 演出（後述）。`kernel/json/art_pulse.txt` / `kernel/txt/art_sentences.txt` / `kernel/txt/silence.flag` / `kernel/json/status.json` をディスクから直接読む（`execution_toolbar.ps1` L123-128, L745-776）。
- **ボタン活性制御** — `[Skip]` / `[Gyotaq]` は `ExecutionState='Running'` のときのみ活性化、`Idle` で無効化される（`execution_toolbar.ps1` L1160-1161、`Update-ExecutionToolbar` の `.PARAMETER ExecutionState` L1279-1280）。

---

## status.json スキーマ

`Write-StatusFile -Phase <idle|executing|complete>`（`common.ps1` L4219）が atomic write する。スキーマ自体は移行前後で不変（書き手は引き続き kernel。変わったのは読み手が「別プロセス monitor」から「in-process toolbar」になった点）：

```json
{
  "UpdatedAt": "2026-05-06 10:32:15",
  "WorkerName": "suzuki",
  "PCInfo": {
    "AdminID": "1",
    "OldPCName": "OLD-PC-01",
    "NewPCName": "NEW-PC-01",
    "EthernetIP": "192.168.1.100",
    "EthernetSubnet": "255.255.255.0",
    "EthernetGateway": "192.168.1.1",
    "WifiIP": "",
    "WifiSubnet": "",
    "WifiGateway": "",
    "DNS": ["8.8.8.8", "8.8.4.4"],
    "Printers": [
      { "Name": "Office", "Driver": "Canon Generic Plus PCL6", "Port": "192.168.1.50" }
    ]
  },
  "CurrentPCInfo": {
    "ComputerName": "NEW-PC-01",
    "EthernetIP": "192.168.1.100",
    "Printers": [{ "Name": "Office", "Port": "192.168.1.50" }]
  },
  "Execution": {
    "Phase": "executing",
    "TotalCount": 12,
    "SuccessCount": 9,
    "ErrorCount": 0,
    "SkippedCount": 1,
    "CancelledCount": 0,
    "PartialCount": 1,
    "WarningCount": 0,
    "Details": [
      {
        "Operation": "Hostname Configuration",
        "Status": "Success",
        "Message": "Renamed to NEW-PC-01 (pending reboot)",
        "Timestamp": "2026-05-06 10:30:55",
        "IsRestored": false,
        "Verified": true
      }
    ]
  }
}
```

### Atomic Write

```powershell
$tempPath = "$($script:StatusFilePath).tmp"
$statusData | ConvertTo-Json -Depth 5 | Out-File -FilePath $tempPath -Encoding UTF8 -Force
Move-Item -Path $tempPath -Destination $script:StatusFilePath -Force
```

書き込み中の半端な JSON を読み手が読んで crash しないよう、tmp ファイルへ書いてから rename。`Move-Item` 失敗時は直接書きへフォールバック（best-effort で例外は飲む。`common.ps1` L4287-4301）。

### status.json の現在の役割

- **書き手**: `Write-StatusFile`（`common.ps1` L4219）。`Show-*` / `Add-ExecutionResult` 等が実行のたびに呼ぶ。
- **読み手**: Execution Toolbar の ART/ステータス同期。`execution_toolbar.ps1` L745-776 が `Execution.Phase` と最新 `Execution.Details` を読み取る。
- **PC Info 照合**は `status.json` ではなく `TargetHostInfo`（`SELECTED_*`）+ live クエリへ移行済み（上記「PC Info ペイン」参照）。
- **後始末**: `Remove-StatusFile`（`common.ps1` L4304）が `status.json` + `.tmp` + `art_pulse.txt` を削除。`Exit-Fabriq` 経由で実行される。

---

## ART Pulse / Sentences / silence.flag（演出機能）

fabriq には「**Manifeste du Surkitinisme**」という演出文化がある。旧方式では Status Monitor / `art_display.ps1` がこの演出を担っていたが、`art_display.ps1` 削除後はタイピング描画も含め **Execution Toolbar の ART パネルに移植**されている。

### art_pulse.txt

- 配置: `kernel/json/art_pulse.txt`（`$global:ArtPulseFilePath` = `.\kernel\json\art_pulse.txt`、`common.ps1` L30）
- 中身: 整数 1 行（カウンタ）
- 書き手: `Write-ArtPulse`（`Show-Info` / `Show-Success` / `Show-Warning` / `Show-Error` / `Show-Skip` のすべてが内部で呼ぶ）
- 読み手: Execution Toolbar の ART パネルが polling し、増加分を「鼓動」アニメに変換
- 用途: モジュール実行が「生きている」ことを視覚化（プロセスがハングしていないか operator が一目で判断できる）

### art_sentences.txt

- 配置: `kernel/txt/art_sentences.txt`
- 中身: キッティング文脈の演出散文 150 行（1 行 = 1 段落）。fabriq の演出哲学キーワード「**シュルキティニスム / Surkitinisme（超展開主義）**」がこの中で語られる
- 描画担当: Execution Toolbar の ART パネル（旧 `kernel/ps1/art_display.ps1` は削除済み。タイピング演出は `execution_toolbar.ps1` に移植）。`art_pulse.txt` の増分でタイピング速度がバースト切替する

### silence.flag

- 配置: `kernel/txt/silence.flag`
- 存在するだけで ART 演出を完全停止
- 業務環境で演出が邪魔な場合の opt-out 機構（ファイル中身は問わない。存在チェックのみ）

---

## manifesto.ps1（マニフェスト表示 GUI）

`kernel/ps1/manifesto.ps1` に WinForms 関数 `Show-Manifesto` を定義（現存）。`kernel/csv/manifesto.csv`（章ごとの本文）を読み、ダッシュボードの `[Manifeste du Surkitinisme]` ボタンから呼ばれる。純粋に演出機能であり運用上の意味は無い。

---

## view_report.ps1（HTML チェックリストビューア）

`kernel/ps1/view_report.ps1` は単体起動可能な HTML ビューアスクリプト（現存）。プロファイル完了時に `Complete-ProfileExecution` から自動起動され、最新の `evidence/{base}/checklist/checklist_*.html` を既定ブラウザで開く。
