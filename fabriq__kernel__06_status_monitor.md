# ステータスモニタ + 演出 (ART pulse / silence / manifesto)

> **対象**: fabriq / kernel + status_monitor
> **対象バージョン**: kernel 3.2.5（取得元: `E:\fabriq\kernel\KERNEL_VERSION`）+ commit `fed181a`（取得元: `git -C E:\fabriq rev-parse --short HEAD`、2026-05-10）
> **ドキュメント更新日**: 2026-05-10

fabriq の二画面構成の左右の片方。メインダッシュボードとは独立した別プロセスで動き、`status.json` ファイルを polling する設計。これにより重い WinForms 処理がメイン実行をブロックしない。

---

## アーキテクチャ概観

```
[ Main Process: Fabriq.exe → main.ps1 ]
   ├── Show-* / Add-ExecutionResult が呼ばれる度に
   │   Write-StatusFile -Phase "executing"  ── status.json を atomic 書き込み
   │   Write-ArtPulse                          ── art_pulse.txt のカウンタを +1
   ↓ ファイルベース IPC ↓
[ Monitor Process: kernel/ps1/status_monitor.ps1（別 PowerShell プロセス）]
   ├── status.json を 500ms 間隔で polling
   ├── 変更検出時に WinForms TextBox / Label / Grid を更新
   └── 演出:
       ├── art_pulse.txt の数字が増えたら「鼓動」アニメ
       ├── art_sentences.txt から 1 行をランダム表示
       └── silence.flag が存在すれば演出を完全停止
```

---

## status.json スキーマ

`Write-StatusFile -Phase <idle|executing|complete>` が atomic write する：

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
    /* Get-CurrentPCInfo の結果 — 実 OS から取った現在値（照合用） */
    "ComputerName": "NEW-PC-01",
    "EthernetIP": "192.168.1.100",
    /* ... */
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
      },
      /* ... */
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

書き込み中の半端な JSON を monitor が読んで crash しないよう、tmp ファイルへ書いてから rename。失敗時は直接書きへフォールバック（best-effort で例外は飲む）。

### PCInfo vs CurrentPCInfo

- `PCInfo`: hostlist.csv 起源の **期待値**（SELECTED_* env vars）
- `CurrentPCInfo`: `Get-CurrentPCInfo` が `Get-NetAdapter` / `Get-Printer` で取った **現在値**

monitor 画面の左右に並べて差分ハイライト。HTML チェックリストでも同じ照合を行う。

---

## status_monitor.ps1 の構造

別プロセスで起動される独立 WinForms アプリ。

### 起動コマンド

```powershell
$argList = @(
    "-NoProfile", "-ExecutionPolicy", "Unrestricted",
    "-File", ".\kernel\ps1\status_monitor.ps1",
    "-StatusFilePath", $statusFileFullPath,
    "-PulseFilePath",  $pulseFileFullPath,
    "-SilenceFlagPath", $silenceFlagFullPath,
    "-DiagLogPath", $diagLogPath        # kernel 3.2.3 で追加
)
# -SentenceFilePath は art_sentences.txt が存在する場合のみ後追い追加
if (-not [string]::IsNullOrWhiteSpace($sentenceFileFullPath)) {
    $argList += @("-SentenceFilePath", $sentenceFileFullPath)
}
Start-Process powershell.exe -ArgumentList $argList -WindowStyle Hidden -PassThru
```

PID は `$global:FabriqStatusMonitorProcess` に格納し、`Stop-StatusMonitor` で `CloseMainWindow` → 2 秒待ち → 強制 Kill。

### 起動診断ログ（kernel 3.2.3 で追加）

`-WindowStyle Hidden` で起動した子プロセスは **uncaught exception を全部飲み込む**ため、子が即死／hung した場合の症状は「タスクバーに何も出ない、モニタウィンドウが現れない」だけになる。kernel 3.2.3 でこの diagnostic blackout を解消するための診断ログ機構を追加した。

**ファイル配置**: `Start-StatusMonitor`（`common.ps1` L4475〜）が 1 セッション 1 ファイルを生成し、子に `-DiagLogPath` 引数で渡す。

```
logs/status_monitor_<yyyyMMdd_HHmmss>.log
```

**子側のログ機構**: `status_monitor.ps1` L24〜L31 の `Write-DiagLog` ヘルパが `Add-Content -ErrorAction SilentlyContinue` で best-effort 追記（ロガー自身が落ちないよう全例外を握り潰す設計）。起動チェーンの主要点に `[init]` / `[asm]` / `[dpi]` / `[forms]` / `[console]` / `[paths]` / `[common]` / `[form]` / `[run]` タグでログ行を残す。

**Exit codes**:

| Exit code | 失敗箇所 |
|---|---|
| 11 | `System.Drawing` ロード失敗 |
| 12 | `System.Windows.Forms` ロード失敗 |
| 13 | `kernel/common.ps1` の dot-source 失敗 |
| 14 | `Application.Run` 中の uncaught exception（ScriptStackTrace を診断ログに残す）|

**ウィンドウ存在ポーリングによる成否判定**（同 L4537〜L4555）:

`HasExited` の即死チェックだけでは不十分なケース（App Control 配備済端末で子が後段で死亡 / hung するパターン）が 2026-05-09 に判明したため、判定の主シグナルを **「Fabriq タイトルのメインウィンドウが `$monitorProcess.MainWindowTitle` に現れたか」** に変更。最大 4 秒、200ms 間隔でポーリング。3 つの終了条件:

1. `MainWindowTitle` が `"Fabriq"` を含む → 成功（process を return）
2. `HasExited` 検出 → 失敗（早期死亡）
3. timeout、生存中、ウィンドウ未出現 → 失敗（起動中 hung）

**失敗時の operator 通知**（L4557〜以降）:

- ExitCode と診断ログパスを Show-Warning で surface
- 早期死亡 + 診断ログサイズ 0 → host security policy（WDAC / AppLocker / Defender ASR / 同等）の疑いとして operator に明示
- 早期死亡 + 診断ログあり → `Get-Content -Tail 1` で最終行を抜粋表示
- 生存中 hung → kill して orphan 防止 + 「ホストセキュリティ調査中の可能性」を表示

### 子側 defensive fallback（kernel 3.2.3 で追加）

`status_monitor.ps1` の起動チェーンには以下の局所的 fallback が組み込まれている。

**DpiX ゼロ／負値ガード**（L70〜L86）: `Graphics.FromHwnd([IntPtr]::Zero).DpiX` が病的に 0 / 負を返す環境で Form Size が `(0, 0)` になり「実行しているのに不可視」となる経路を遮断。`$rawDpi -le 0` で 96 fallback、`dpiScale<=0` を 1.0 にクランプ。

**`NoActivateForm` Add-Type 失敗時の fallback**（L118〜L159）: 動的 C# コンパイルが AMSI / `csc.exe` パス問題 / `%TEMP%` 書込権限欠如等で失敗した場合、通常の `System.Windows.Forms.Form` および `System.Windows.Forms.StatusStrip` で代用。`WS_EX_NOACTIVATE` のフォーカスを奪わない挙動は失うが、**ウィンドウが画面に出ること** を優先。`$script:useFallbackForm` フラグで分岐。

**`Application.Run` を try/catch でラップ**（L1348〜L1355）: 例外時は `ScriptStackTrace` を診断ログに残して exit 14。

### 表示要素（概念）

- **PC Info ペイン**: PCInfo（期待値）と CurrentPCInfo（実際値）を縦並び
- **Execution Summary**: TotalCount / Success / Error / Skipped / Partial の数字
- **Execution Details Grid**: Operation, Status（色付）, Verified バッジ, Timestamp
- **Skip ボタン**: async モジュール強制中断（`skip_request.flag` を生成）
- **Manual Screenshot ボタン**: `Save-Screenshot` を呼んで `evidence/{base}/gyotaku/` に PNG 保存
- **ART Pulse 鼓動**: `art_pulse.txt` の数字が増えるたびにアニメ
- **ART 言葉**: `art_sentences.txt` から 1 行をランダム表示

### Skip ボタンの実装

monitor が `skip_request.flag` を作成 → `Invoke-SafeCommandAsync` の polling loop が検出 → Runspace を `ps.Stop()` で強制終了 → Skip 結果として記録。詳細は §08_async_execution.md。

---

## ART Pulse / Sentences / silence.flag（演出機能）

fabriq には「**Manifeste du Surkitinisme**」という演出文化がある（README L1）。Status Monitor がこの演出を担う。

### art_pulse.txt

- 中身: 整数 1 行（カウンタ）
- 書き手: `Write-ArtPulse`（`Show-Info` / `Show-Success` / `Show-Warning` / `Show-Error` / `Show-Skip` のすべてが内部で呼ぶ）
- 読み手: `status_monitor.ps1` が polling し、増加分を「鼓動」アニメに変換
- 用途: モジュール実行が「生きている」ことを視覚化（プロセスがハングしてないか operator が一目で判断できる）

### art_sentences.txt

- 中身: アンドレ・ブルトン『シュルレアリスム宣言』をキッティング文脈にパロディ翻案した日本語の散文 150 行（1 行 = 1 段落、各段落が長文）。fabriq の演出哲学キーワード「**シュルキティニスム / Surkitinisme（超展開主義）**」がこの中で定義される
- 描画担当: `kernel/ps1/art_display.ps1`（status_monitor から `-SentenceFilePath` 引数で渡されつつ、別プロセスとして並走するタイピング演出ウィンドウ）。`Select-NextSentence` でランダム選択 → `ART_BURST_SPEED` (8ms/char) または `ART_IDLE_SPEED` (35ms/char) で 1 文字ずつタイピング描画。**大文字化はしない**（原文の日本語をそのまま表示）
- pulse counter（`art_pulse.txt`）の増分でタイピング速度がバースト切替する仕組み。モジュール実行が活発なとき高速、idle 時は低速

### silence.flag

- 存在するだけで monitor が ART 演出を完全停止
- 業務環境で演出が邪魔な場合の opt-out 機構
- ファイル中身は問わない（存在チェックのみ）

---

## manifesto.ps1（マニフェスト表示 GUI）

`kernel/ps1/manifesto.ps1` に WinForms 関数 `Show-Manifesto` を定義。`kernel/csv/manifesto.csv`（章ごとの本文）を読み、ダッシュボードの `[Manifeste du Surkitinisme]` ボタンから呼ばれる。

純粋に演出機能であり、運用上の意味は無い（fabriq の哲学を作業者に表示する）。

---

## view_report.ps1（HTML チェックリストビューア）

`kernel/ps1/view_report.ps1` は単体起動可能な HTML ビューアスクリプト。プロファイル完了時に `Complete-ProfileExecution` から自動起動され、最新の `evidence/{base}/checklist/checklist_*.html` を既定ブラウザで開く。

---

## Status Monitor のライフサイクル

```
fabriq 起動
   ↓
Initialize-Session 後
   ↓
Start-StatusMonitor
   ├── Write-StatusFile -Phase "idle" でスケルトン生成
   ├── Start-Process powershell.exe ... -WindowStyle Hidden -PassThru
   ├── PID を $global:FabriqStatusMonitorProcess に保存
   └── 1.2 秒待ってから Set-ConsoleForeground（メインコンソールを前面に戻す）
   ↓
... モジュール実行中、Add-ExecutionResult 等のたびに Write-StatusFile が更新 ...
   ↓
Exit-Fabriq（ユーザーが終了）
   ↓
Stop-StatusMonitor
   ├── CloseMainWindow → 2 秒で Wait→ 強制 Kill
   └── Remove-StatusFile（status.json + tmp + art_pulse.txt 削除）
```
