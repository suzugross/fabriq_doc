# 状態ファイルとレジューム異常のトラブルシューティング

> **対象**: fabriq / kernel + windows_update + extended/pianist
> **対象バージョン**: 3.6.0（取得元: `E:\fabriq\kernel\KERNEL_VERSION`、commit `0fca159`、2026-06-16）
> **ドキュメント更新日**: 2026-06-16

fabriq は **複数の状態ファイル**で実行コンテキストを跨いだ継続性を担保している。これらのファイルが想定外の状態（残存・破損・古いバージョン）になったとき、何が観測され、どう正規化するかをまとめた運用ガイド。

> **対象読者**: kitting 中に「再起動したら fabriq の挙動がおかしい」「Refabriq したのに前のセッションが残っている」「Failed to decrypt が連発する」と感じたオペレータ。

---

## 状態ファイル一覧

すべて `kernel/json/` 配下に配置される。`*.flag` / `*.txt` 系も含む。

| ファイル | 役割 | 寿命 | 削除タイミング |
|---|---|---|---|
| `resume_state.json` | profile 実行中断状態（v1=Linear / v2=Flex 兼用） | 数分〜数時間 | profile 完走時 / `Remove-ResumeState` 明示 |
| `session.json` | 作業者・媒体・ホスト情報のセッション固有メタ | セッション期間 | `Reset-FabriqState`（NewSession 時） |
| `status.json` | Execution Toolbar の ART/ステータス同期が読む現在 phase（書き手は引き続き kernel の `Write-StatusFile`） | 常時更新 | `Write-StatusFile -Phase "idle"` で待機状態に |
| `wu_state.json` | Windows Update reboot loop の往復状態 | WU loop 中のみ | WU 完了で削除 |
| `wu_completed.json` | WU loop 終了サマリ。次回起動時 main.ps1 が消費 | 1 サイクル | main.ps1 import 後に消費削除 |
| `art_pulse.txt` | Execution Toolbar の ART パネルが読む鼓動カウンタ（`Write-ArtPulse` が増分） | 数秒 | `Remove-StatusFile` が `status.json` と併せて削除 |
| `skip_request.flag` | Execution Toolbar の `[Skip]` ボタン → kernel への async 中断要求 | 検知されるまで | `Invoke-SafeCommandAsync` の polling が読んだ瞬間に削除 |
| `silence.flag` | サウンド抑制フラグ | 設定中のみ | 手動 |
| `async_config.json` | async 経路の設定（`Enabled` kill switch / `DefaultAsync` 既定 ON 化 / `DefaultTimeoutSec` / `PollIntervalMs` / `SkipFlagPath`）。git tracked、shipped default は `Enabled=true` + `DefaultAsync=true` | 永続（git tracked） | 通常は触らない。緊急時に `Enabled=false` で全モジュール同期実行に降格 |

加えて:

| 場所 | ファイル | 役割 |
|---|---|---|
| `evidence/<host>/exec_history.csv` | 履歴 CSV | profile 実行の per-row 結果 |
| `logs/transcript_<ts>.txt` | トランスクリプト | PowerShell の stdout/stderr フル記録 |
| `kernel/txt/passphrase_verify.txt` | パスフレーズ照合用トークン | 暗号化セッション開始時のキー検証 |

---

## 「リセット」の 2 種類: Refabriq vs NewSession

メインメニューには似て非なる 2 つのリセット操作がある。**動作が大きく違う**ため誤解しないこと。

### Refabriq（プロセス再起動）

[main.ps1:2200-2215](file:///E:/fabriq/kernel/main.ps1):

```powershell
"Refabriq" {
    Show-Info "Restarting Fabriq..."
    try { Hide-ExecutionToolbar } catch { }
    Remove-StatusFile
    $fabriqRoot = (Resolve-Path ".").Path
    $fabriqExe = Join-Path $fabriqRoot "Fabriq.exe"
    if (Test-Path $fabriqExe) {
        Start-Process $fabriqExe -WorkingDirectory $fabriqRoot
    } else {
        Show-Error "Fabriq.exe not found: $fabriqExe"
        Wait-KeyPress
        continue
    }
    try { Stop-Transcript | Out-Null } catch { }
    exit 0
}
```

- **fabriq プロセス自体を再起動**。
- Execution Toolbar を `Hide-ExecutionToolbar` で閉じ → `Remove-StatusFile` で `status.json` / `art_pulse.txt` を後始末 → `Fabriq.exe` を新プロセスで起動 → 自プロセスは `exit 0`。
- **状態ファイルは削除しない**: `resume_state.json` / `session.json` / `wu_state.json` などは **そのまま残る**。
- **用途**: PowerShell 環境がおかしくなった、変数が汚染された、特定の関数を再ロードしたい、コマンド履歴を切りたい等。
- **注意**: 「まっさら」にはならない。レジューム状態が残っていれば、再起動後の fabriq は **そこから再開**する。

### NewSession（状態リセット）

[main.ps1:1693-1696](file:///E:/fabriq/kernel/main.ps1):

```powershell
"NewSession" {
    Show-ConsoleWindow
    Reset-FabriqState
    ...   # 以後セッション情報を再選択 (作業者 / 媒体 / 端末)
}
```

`Reset-FabriqState`（[common.ps1:2781-2879](file:///E:/fabriq/kernel/common.ps1)）の **10 カテゴリリセット内容**:

| # | 対象 | 動作 |
|---|---|---|
| 1 | Transcript | 既存を `Stop-Transcript` → 新ファイル `<timestamp>_<uid>_<host>.log` で `Start-Transcript -Append` |
| 2 | In-memory results | `$script:ExecutionResults`/`$script:LastBatchResults` クリア、`$script:SessionID` 再生成 |
| 2b | History CSV | **`exec_history.csv` と `.bak` を削除**（次回 `Restore-ExecutionHistory` がクリーン状態を見る） |
| 3 | Session info | `$script:SessionInfo = $null`、**`session.json` 削除** |
| 4 | Global flags | `$global:AutoPilotMode = $false`、`$global:_LastModuleResult = $null` |
| 5 | Evidence path | `$global:FabriqEvidenceBasePath` / `FabriqEvidenceRootPath` を null。環境変数 `FABRIQ_EVIDENCE_BASE` も解除 |
| 6 | Host env vars | `SELECTED_*`（管理 No / NewPCName / IP × 多数 / Pin / Printer × 10 など）を Process スコープで解除 |
| 7 | Resume state | `Remove-ResumeState`（`resume_state.json` 削除） |
| 7 | Status file | `Write-StatusFile -Phase "idle"` |
| - | **削除されないもの** | `evidence/` 配下のファイル（過去の収集物）、`wu_state.json` / `wu_completed.json`、`logs/` 過去分、master passphrase（in-memory のため Refabriq でも消える） |

NewSession 後は **作業者選択ダイアログから再スタート**する（`Show-SessionSetupForm`）。ホスト一覧再ロード、master passphrase の再入力が要求される。

### 比較表

| 観点 | Refabriq | NewSession |
|---|---|---|
| プロセス | 再起動 | 同一プロセス継続 |
| Transcript | 旧停止 → 新規作成 | 旧停止 → **同名ファイルに -Append** |
| `resume_state.json` | **残る** | **削除** |
| `session.json` | **残る** | **削除** |
| `exec_history.csv` | **残る** | **削除** |
| Host 環境変数 | 残る | 解除 |
| AutoPilot フラグ | 残る | false に戻る |
| Master passphrase | 失われる（プロセス再起動） | 失われる（NewSession 内で削除はしないが UI が再入力を要求） |
| Execution Toolbar | `Hide-ExecutionToolbar` で閉じてプロセス再起動時に再生成 | そのまま（idle に戻すだけ） |
| 用途 | PS 環境のリフレッシュ | 別端末を新規キッティング開始する |

> **典型的な誤運用**: 「リセットしたつもりが Refabriq だった」→ 前 PC の `resume_state.json` から再開してしまい、**現在の PC に異なる Profile を適用しようとする**事故。NewSession を選ぶこと。

---

## resume_state.json の異常パターン

### schemaVersion v1 と v2

`resume_state.json` は 2 つのスキーマを持つ:

| schemaVersion | 用途 | 主要フィールド |
|---|---|---|
| 未設定 (= v1) | Linear profile の中断 | `ResumeAfterOrder`, `CompletedModules` |
| `2` | FlexProfile の中断 | `ExecutionMode='Flex'`, `SelectedOrders`, `ModuleStates` (per-Order) |

[common.ps1:2756-2761](file:///E:/fabriq/kernel/common.ps1):

```powershell
if ($ExecutionMode -eq 'Flex') {
    $state['schemaVersion']  = 2
    $state['ExecutionMode']  = 'Flex'
    $state['SelectedOrders'] = @($SelectedOrders)
    $state['ModuleStates']   = $ModuleStates
}
```

v1 と v2 は **byte-for-byte 互換** で、Linear callers は schemaVersion を出さない（pre-P4 と互換）。

### よくある異常: 起動時に「再開しますか？」が出続ける

**原因**: 完了したのに `Remove-ResumeState` が呼ばれず `resume_state.json` が残ったまま。

**検証**:

```powershell
Test-Path .\kernel\json\resume_state.json
Get-Content .\kernel\json\resume_state.json | ConvertFrom-Json | Select-Object ProfileName, ResumeAfterOrder
```

**復旧**:

```powershell
Remove-Item .\kernel\json\resume_state.json -Force
```

または fabriq メニューから **NewSession** で全部リセット。Refabriq では消えない点に注意。

### よくある異常: 「Failed to parse resume_state.json」警告

[main.ps1:1448](file:///E:/fabriq/kernel/main.ps1) で発生する警告。**手動編集で JSON が壊れた / Windows がクラッシュ中に書き込みが途切れた** 等。

**復旧**: ファイルを **削除** すれば、fabriq は「resume なし」で起動する。データ損失は**直前の profile の途中状態のみ**（`exec_history.csv` には正常まで残っている）。

### Legacy resume_state.json without ProfileStartTime

[main.ps1:1452](file:///E:/fabriq/kernel/main.ps1):

```
Legacy resume_state.json without ProfileStartTime — elapsed time will be measured from now (pre-restart duration not included).
```

**意味**: pre-3.0 の fabriq で書かれた古いフォーマットで再開。実害は「経過時間表示が再起動以後の差分のみ」となるだけ。**動作には影響しない**。気になるなら NewSession で消す。

### FlexProfile 再開時の「どのモジュールが既に走ったか」が分かりにくい

[main.ps1:1493+](file:///E:/fabriq/kernel/main.ps1) の Flex resume パス。`ModuleStates` ハッシュテーブル（Order をキー）から `Show-FlexDashboard` が状態マップを再構成する。

**判別の基本**:

1. **`exec_history.csv`** を最新セッションに絞り、`Order` 列で時系列を確認
2. FlexDashboard 上部の **Order 別バッジ**（Success/Error/NotRun）が現状
3. 不安なら、未実行と分かっているモジュールだけを再選択して `[Run]` ボタンで個別再実行

> kernel 3.1.7（[CHANGELOG](file:///E:/fabriq/CHANGELOG.md)）で **MenuName 衝突による sibling-row leak バグ**は修正済み。古い fabriq から resume した場合、**3.1.7 以降の fabriq に**揃えてから再開することを推奨。

---

## session.json の異常パターン

### NewSession 後にすぐ「セッション情報がありません」が出る

`session.json` は `Reset-FabriqState` で削除され、`Show-SessionSetupForm` で再生成される。**ダイアログでキャンセル**したまま放置すると `session.json` 不在のまま操作することになる。

**復旧**: 再度 NewSession を選択し、ダイアログを **完了させる** （Cancel ではなく OK）。

### Worker 再選択を毎回求められる

これは正常動作。`Reset-FabriqState` 後はあえて作業者を再選択させる設計（端末別に作業者を切り替える運用に対応）。

### session.json の手動編集

可能だが、`SessionInfo` の構造（`WorkerID`/`WorkerName`/`MediaSerial`/`StartTime`/`WindowsUser`/`ComputerName`）を **完全に揃える必要がある**。1 フィールド欠損で起動時 parse 失敗 → **Worker 選択ダイアログから再スタート**になるため、修正したいなら NewSession の方が確実。

---

## wu_state.json / wu_completed.json — Windows Update loop

[modules/standard/windows_update/Guide.txt:84-86](file:///E:/fabriq/modules/standard/windows_update/Guide.txt):

```
[Runtime Files]
- wu_state.json     : Loop state (created during reboot loop, deleted on completion)
- wu_completed.json : Completion summary (created on finish, consumed by main.ps1)
```

### `wu_state.json` が残ったまま fabriq を起動した場合

WU loop 完了前に PC を強制終了 / fabriq を `q` で抜けた等で `wu_state.json` が残存することがある。**新しい kitting セッションでこれを引き継ぐと予期せぬ再 scan**が走る。

**判別**:

```powershell
$wu = Get-Content .\modules\standard\windows_update\wu_state.json | ConvertFrom-Json
$wu | Format-List LoopCount, MaxRebootLoops, LastRebootTime
```

**復旧**:

- WU を完走させたいなら、メインメニューから `[wu]` を再投入し loop を再開させる
- やり直したいなら **wu_state.json を削除** + RunOnce レジストリ（`HKCU:\Software\Microsoft\Windows\CurrentVersion\RunOnce` の `FabriqWindowsUpdate`）も削除

### MaxRebootLoops に達して停止した場合

`MaxRebootLoops`（既定 5）はデッドロック防止の **最終安全弁**。これに達した場合は:

1. ホットフィックスがそもそも入らないバグ持ち更新を踏んでいる可能性が高い
2. `windows_update_list.csv` の `MaxRebootLoops` を増やすのは原因不明のまま回数を増やすだけで非推奨
3. `wu_log.txt` または Windows Update ログ（`Get-WindowsUpdateLog`）で **失敗しているKB番号を特定** し、`SkipKBs` で除外して再実行

### AutoLogon 経由 reboot で fabriq が起動しない

`autologon_config/autologon_list.csv` に **現在ユーザーの行が無い** / Pin 暗号化が解けない / RunOnce が消えている。`runas /trustlevel:0x40000 fabriq.exe` で手動起動して状態を確認する。

---

## ENC: 暗号化セルの復号失敗

hostlist.csv / module CSV にある `ENC:<Base64>` プレフィクスのセルが復号できないとき:

### 兆候

- `Failed to decrypt` 警告がログに連発
- `Resolve-HostValue` が `ENC:...` の文字列を **そのまま PIN として返す** → bitlocker_config が `Error: PIN length out of range`（実際は ENC 文字列の長さ）
- `bitlocker_config` Guide 記述: `"復号に失敗した ENC: 値は PIN として使えず、モジュールは Error で停止"`

### 原因の優先度

1. **Master passphrase が間違っている**: NewSession ダイアログで入力したパスワードが正しくない。`passphrase_verify.txt` の `surkitinisme` トークンが復号できないと **dialog 上で OK が押せない** はずだが、何らかの理由で先に進めてしまった可能性。
2. **hostlist.csv の暗号化された後にパスフレーズを変えた**: 暗号化時のパスワードと復号時のパスワードが不一致。**この場合データ復旧は不可能**（暗号化前の値を別途保管してあれば再暗号化）。
3. **kernel/CryptoHelper のバージョンミスマッチ**: `kernel/encryption.ps1` と fabriq_studio 側の `CryptoHelper` の AES パラメータ（KDF iteration、salt）の解釈が違う（理論上ありえないが、kernel 1.x → 2.x で変更があった場合は確実に発生）。3.x 同士なら互換。
4. **マシン依存暗号化を使っていた**: かつての DPAPI ベース暗号化と混同。fabriq 3.0+ は **AES-256-CBC + PBKDF2** で **マシン非依存**になっている。1.x の値が混ざっていれば失敗。

### 検証手順

1. `kernel/txt/passphrase_verify.txt` を直接確認:

   ```powershell
   $verify = Get-Content .\kernel\txt\passphrase_verify.txt -Raw
   # 期待値: ENC:<base64> 形式 (1 行)
   ```

2. fabriq_studio で同じ workspace を開き、**同じ master passphrase** で hostlist 1 セルが復号できるか試す
3. 1 セルでも復号できれば **個別データの問題**。すべて失敗なら **passphrase の問題**

### 復旧

- 正しい passphrase が分かる場合: `Refabriq` してダイアログで再入力
- 分からない / 再現不能の場合: **hostlist.csv を再構築**するしかない。`fabriq_evidence_manager` の export 機能で過去 export を引き出せる場合がある（[fabriq_evidence_manager__usage__04_export_delivery.md](fabriq_evidence_manager__usage__04_export_delivery.md) 参照）。
- 暗号化を完全リセットしたい場合: hostlist.csv の `ENC:` セルを手動で削除して再入力 → fabriq 再起動 → 再暗号化

> **暗号化の原則**: 暗号化された値は **passphrase なしには復元不可能**（AES-256 は一方向の意味で安全）。passphrase 紛失は **データ紛失と同義**。

---

## status.json / Execution Toolbar の異常

> kernel 3.4.0 で別プロセス型 Status Monitor は in-process な **Execution Toolbar** に置き換わり、`status_monitor.ps1` / `art_display.ps1` および `Start-StatusMonitor` / `Stop-StatusMonitor` / `Show-MonitorFailureDialog` は 3.5.0 で物理削除された（取得元: `E:\fabriq\kernel\KERNEL_API.md` §8 [3.5.0]）。Execution Toolbar は kernel の `powershell.exe` 内の専用 STA Runspace で動くため、旧方式で「Defender / ASR が hidden 子プロセスをブロックしてモニタが出ない」という障害モードは存在しない。詳細は [fabriq__kernel__06_status_monitor.md](fabriq__kernel__06_status_monitor.md)。

### Execution Toolbar が表示されない / ART パネルが動かない

`art_pulse.txt` が更新されていない / `status.json` が壊れている。

**復旧**:

1. Refabriq でプロセスを再起動する（`Hide-ExecutionToolbar` → `Remove-StatusFile` → 新プロセス起動）。
2. 起動時に `Show-ExecutionToolbar`（[main.ps1:1702](file:///E:/fabriq/kernel/main.ps1)）が STA Runspace ごとツールバーを再生成する。

### `status.json` の Phase が「実行中」のまま固まる

kernel が `Write-StatusFile -Phase "idle"` を呼ばずに死んだ可能性。

**復旧**: NewSession で idle に戻す。または手動で:

```powershell
@{ Phase = "idle"; Module = ""; Timestamp = (Get-Date).ToString("o") } |
    ConvertTo-Json | Out-File .\kernel\json\status.json -Encoding UTF8
```

### `silence.flag` がいつまでも消えない

サウンド抑制が効きっぱなし。**手動削除**で復活:

```powershell
Remove-Item .\kernel\txt\silence.flag -Force
```

---

## 緊急時のクリーンアップ（最終手段）

すべての状態を捨ててやり直したい場合:

```powershell
# 1. fabriq を終了（q または ウィンドウ閉じ）
# 2. 状態ファイル一式を削除
Remove-Item .\kernel\json\resume_state.json -Force -ErrorAction SilentlyContinue
Remove-Item .\kernel\json\session.json      -Force -ErrorAction SilentlyContinue
Remove-Item .\kernel\json\wu_state.json     -Force -ErrorAction SilentlyContinue
Remove-Item .\kernel\json\wu_completed.json -Force -ErrorAction SilentlyContinue
Remove-Item .\kernel\json\status.json       -Force -ErrorAction SilentlyContinue
Remove-Item .\kernel\json\art_pulse.txt     -Force -ErrorAction SilentlyContinue
Remove-Item .\kernel\json\skip_request.flag -Force -ErrorAction SilentlyContinue
Remove-Item .\kernel\txt\silence.flag        -Force -ErrorAction SilentlyContinue
# 3. fabriq を起動 → NewSession で正常開始
.\Fabriq.exe
```

> **絶対に削除しないもの**: `evidence/` 配下、`logs/` 配下、`hostlist.csv`、`profiles/`、`kernel/txt/passphrase_verify.txt`。これらは **kitting の証拠 / 設計値** であり、消すと顧客との約束が果たせなくなる。

---

## 参考: 状態の関係図

```
┌── fabriq.exe プロセス ─────────────────────────────────┐
│ in-memory:                                            │
│   $script:SessionID                                   │
│   $script:ExecutionResults                            │
│   $script:LastBatchResults                            │
│   $global:FabriqMasterPassphrase  ◀── NewSession で再入力
│   $global:AutoPilotMode                               │
│                                                       │
│   ┌─ kernel/json/ ──────────────────────────────────┐ │
│   │ resume_state.json   ◀── profile 中断中のみ      │ │
│   │ session.json        ◀── NewSession で再生成     │ │
│   │ status.json         ◀── kernel が常時更新       │ │
│   │ wu_state.json       ◀── WU loop 中のみ          │ │
│   │ skip_request.flag   ◀── Toolbar[Skip] → kernel  │ │
│   │ art_pulse.txt       ◀── 一時 metric             │ │
│   └─────────────────────────────────────────────────┘ │
│                                                       │
│   ┌─ evidence/<host>/ ──────────────────────────────┐ │
│   │ exec_history.csv    ◀── profile 結果（Reset で削除）
│   │ <module>/<section>/ ◀── 永続（Reset で削除されない）
│   └─────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────┘
```

---

## 関連ドキュメント

| ドキュメント | 関係 |
|---|---|
| [fabriq__kernel__05_resume_restart.md](fabriq__kernel__05_resume_restart.md) | resume の正常フロー（`__RESTART__` マーカー → 再起動 → 再開） |
| [fabriq__kernel__04_csv_encryption.md](fabriq__kernel__04_csv_encryption.md) | ENC: 暗号化スキームの正式仕様（AES-256-CBC + PBKDF2） |
| [fabriq__usage__02_session_setup.md](fabriq__usage__02_session_setup.md) | NewSession の正常時セットアップフロー |
| [fabriq__troubleshooting__module_failures.md](fabriq__troubleshooting__module_failures.md) | モジュール単体の失敗トラブルシューティング（本書の姉妹ドキュメント） |
| [fabriq_evidence_manager__usage__02_hostlist_verification.md](fabriq_evidence_manager__usage__02_hostlist_verification.md) | hostlist.csv の事前整合性検証 |

---

## 変更履歴

- 2026-05-07 初版作成（kernel 3.2.2 commit `e513cf1`、Refabriq vs NewSession の 10 カテゴリ比較・resume_state.json v1/v2 異常・wu_state ライフサイクル・ENC: 暗号化復旧・最終クリーンアップ手順を網羅）
- 2026-06-16 kernel 3.6.0 commit `0fca159` へ同期。旧 Status Monitor 記述を Execution Toolbar へ是正（状態ファイル表の status.json / art_pulse.txt / skip_request.flag、Refabriq コードブロックの `Stop-StatusMonitor` 引用、比較表、状態関係図、専用節「status.json / Execution Toolbar の異常」）。`status_monitor.ps1` / `art_display.ps1` / `Start-StatusMonitor` / `Stop-StatusMonitor` / `Show-MonitorFailureDialog` の参照を除去
