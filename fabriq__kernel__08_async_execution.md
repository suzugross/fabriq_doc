# 非同期モジュール実行（__ASYNC__ + Runspace + Skip）

> **対象**: fabriq / kernel
> **対象バージョン**: kernel 3.3.1（取得元: `E:\fabriq\kernel\KERNEL_VERSION`）+ commit `5525728`（取得元: `git -C E:\fabriq rev-parse --short HEAD`、2026-05-12）
> **ドキュメント更新日**: 2026-05-12

モジュールを別 PowerShell Runspace で実行し、Status Monitor からの **Skip ボタン** または **タイムアウト** で強制中断できる仕組み。kernel **2.1.0** で `__ASYNC__` マーカー as opt-in として追加、kernel **3.3.0** で `async_config.json.DefaultAsync=true` を shipped default に切り替え、**全モジュールが既定で監視付き Runspace 経路を通る**ようになった（`__ASYNC__` マーカーは idempotent な ON-only no-op として後方互換保持）。

---

## なぜ async 実行が必要か

通常の `Invoke-SafeCommand` は同期実行で、モジュールが内部でハングすると fabriq 全体がブロックされる。長時間処理（domain join, winget install など）でこのリスクが顕在化したため、Runspace 実行 + Skip flag/timeout 監視のレイヤを導入した。

**設計史**:

- **kernel 2.1.0**: `__ASYNC__` マーカーで「プロファイル末尾までの async」を opt-in する設計。Runspace 起動コスト・デバッグ性低下・ホスト依存 API（`PromptForChoice` 等）の STA Runspace 互換性問題から、初期は「ハング懸念のある一部のモジュールに対して opt-in」が落とし所だった。
- **kernel 3.3.0**: 上記コスト群が運用上で許容範囲と判明したため、`async_config.json` に `DefaultAsync` フィールドを新設し shipped default を `true` に切り替えた。profile 内のマーカー有無に関わらず、全モジュールが既定で監視付き Runspace 経路を通る。動機は「マーカー書き忘れで Skip ボタン / timeout の安全網が無効になる事故の常時防止」で、疲弊した作業者の認知負荷軽減を目的とした defensive default 化。

---

## kernel/json/async_config.json

```json
{
  "Enabled": true,
  "DefaultAsync": true,
  "DefaultTimeoutSec": 0,
  "PollIntervalMs": 500,
  "SkipFlagPath": ".\\kernel\\json\\skip_request.flag"
}
```

| キー | 用途 |
|---|---|
| `Enabled` | `false` で全 async 動作を kill。`__ASYNC__` マーカーも `DefaultAsync` も silent ignore、全モジュールが同期経路へ降格（最終 kill switch） |
| `DefaultAsync`（kernel 3.3.0〜） | `true` でマーカー有無に関わらず全モジュールを Runspace 経路で実行。shipped default は `true`。`false` だと 2.x 互換挙動（`__ASYNC__` マーカー必須）。config 欠損時のフォールバックは `$false`（silent な async 化を防ぐ安全側） |
| `DefaultTimeoutSec` | `0`=無制限。`> 0` なら全 async モジュールにこの timeout を適用 |
| `PollIntervalMs` | Skip flag / completion / timeout チェックの polling 間隔（最小 50ms に clamp） |
| `SkipFlagPath` | Status Monitor が touch する flag ファイル絶対パス |

**優先順位**: `Enabled=false` > `DefaultAsync` > `__ASYNC__` マーカー。`Enabled=false` の kill switch を kernel 側に置いた理由: 緊急時に運用環境でも 1 つのフィールド書き換えで全モジュールが従来の同期実行へ自動 fallback できる。

### `Get-FabriqAsyncConfig` の返却

`common.ps1` L1591。`async_config.json` を `ConvertFrom-Json` して上記 5 キーを `[PSCustomObject]` で返す。各フィールドが欠損/不正値の場合のフォールバックは：

- `Enabled` → `$true`
- `DefaultAsync` → `$false`（config 欠損時に silent な async 化を防ぐ）
- `DefaultTimeoutSec` → `0`
- `PollIntervalMs` → `500`
- `SkipFlagPath` → `.\kernel\json\skip_request.flag`

「config 同梱 = `DefaultAsync=true`、config 欠損 = `false`」という非対称設計は意図的で、shipped 配布版は async 既定でも、何らかの事故で config が壊れた / 旧版 config を持ち込んだ環境では従来挙動に倒れる。

---

## Resolve-ProfileModules での解釈

`common.ps1` L3781。マーカー解釈と `_IsAsync` 属性 attach は `async_config.json` の 2 軸（`Enabled` / `DefaultAsync`）で次のように分岐：

```powershell
$asyncCfg              = Get-FabriqAsyncConfig
$asyncEnabledGlobally  = [bool]$asyncCfg.Enabled
$asyncMode             = $asyncEnabledGlobally -and [bool]$asyncCfg.DefaultAsync
```

`$asyncMode` 初期値が profile 1 行目から有効になる点が 3.3.0 の本質的変化。3.2.x 以前は `$asyncMode = $false` 固定で、`__ASYNC__` 行通過時に `$true` へ flip する設計だった。

### モード別の動作マトリクス

| `Enabled` | `DefaultAsync` | `__ASYNC__` マーカー | 結果 |
|---|---|---|---|
| `true` | `true` | あり / なし | 全モジュール `_IsAsync=$true`（マーカーは ON-only no-op、idempotent） |
| `true` | `false` | あり | マーカー以降のモジュールに `_IsAsync=$true`（2.x 互換挙動） |
| `true` | `false` | なし | 全モジュール同期実行（`_IsAsync` 未 attach） |
| `false` | 任意 | 任意 | 全モジュール同期実行（kill switch、マーカーは silent ignore） |

### profile 例（shipped default 環境）

```
profile CSV:
   Order, ScriptPath
   10,    standard/hostname_config/hostname_config.ps1   ← _IsAsync=$true
   20,    __ASYNC__                                       ← no-op（既に async）、ValidModules には入らない
   30,    standard/winget_install/winget_install.ps1     ← _IsAsync=$true
   40,    standard/app_config/app_config.ps1              ← _IsAsync=$true
   50,    __RESTART__
   60,    standard/reg_hklm_config/reg_hklm_config.ps1   ← _IsAsync=$true
```

`__ASYNC__` はプロファイル末尾までの sticky フラグ。途中で sync に戻すマーカーは無い（`DefaultAsync=true` 環境で特定モジュールだけ sync にしたい場合は profile 分割か `Enabled=false` への切替が必要）。

---

## Invoke-SafeCommandAsync の実装

`common.ps1` L1621、約 220 行。

### Runspace 立ち上げ

```powershell
$runspace = [runspacefactory]::CreateRunspace($Host)
$runspace.ApartmentState = "STA"           # WinForms 互換のため
$runspace.ThreadOptions  = "ReuseThread"
$runspace.Open()
$runspace.SessionStateProxy.Path.SetLocation($fabriqRoot)  # 相対パス整合
```

### Globals 注入

```powershell
$inject = @{
    FabriqMasterPassphrase = $global:FabriqMasterPassphrase
    AutoPilotMode          = $global:AutoPilotMode
    AutoConfirmMode        = $global:AutoConfirmMode   # since kernel 3.3.1
    AutoPilotWaitSec       = $global:AutoPilotWaitSec
    FabriqTranscriptPath   = $global:FabriqTranscriptPath
    FabriqUniqueId         = $global:FabriqUniqueId
    FabriqSessionTimestamp = $global:FabriqSessionTimestamp
    FabriqEvidenceBasePath = $global:FabriqEvidenceBasePath
    FabriqEvidenceRootPath = $global:FabriqEvidenceRootPath
}
```

env vars は Process スコープなので Runspace に自動継承される。`$global:*` だけ明示注入。

**`AutoConfirmMode` 追加の経緯（kernel 3.3.1）**: kernel 3.1.0 で `$global:AutoConfirmMode` が `KERNEL_API.md §2` に公開グローバルとして宣言されたが、`Invoke-SafeCommandAsync` の inject リストには追加されていなかった。3.2.x 以前は FlexProfile `[Run This]`（RunSingle 経路）が sync 実行のため child runspace を経由せず影響が出なかったが、kernel 3.3.0 で `DefaultAsync` 既定 ON 化により全モジュールが child runspace 経由となり、`AutoConfirmMode` が `$false`（common.ps1 init 既定値）のまま残って `Confirm-ModuleExecution` の Y/N プロンプトが Read-Host へ落ちる regression が顕在化（`reg_hkcu_config` の "Apply the above registry changes?" 等、`Confirm-ModuleExecution` を呼ぶ 108 モジュールが対象）。kernel 3.3.1 で inject に 1 行追加して修正済み。`Run Batch` / `Run Group` / Linear は `AutoPilotMode` が inject 済みのため無影響。

### Wrapper Script Block

```powershell
$ps.AddScript({
    param($CommonPath, $ModuleScript, $Inject, $FabriqRoot)
    Set-Location -Path $FabriqRoot
    . $CommonPath                          # Show-Info etc を再ロード
    foreach ($key in $Inject.Keys) {
        Set-Variable -Name $key -Value $Inject[$key] -Scope Global -Force
    }
    $global:_LastModuleResult = $null
    $output = & $ModuleScript
    [PSCustomObject]@{
        _AsyncWrapper = $true
        Output        = $output
        LastResult    = $global:_LastModuleResult
    }
})
```

`_LastModuleResult` も pipeline と並列で回収するため、wrapper PSCustomObject 経由で親 Runspace に運ぶ。

### Monitor Loop

```powershell
$asyncHandle = $ps.BeginInvoke()
while (-not $asyncHandle.IsCompleted) {
    Start-Sleep -Milliseconds $pollMs

    if (Test-Path $skipFlagPath) {
        Remove-Item $skipFlagPath -Force
        $interrupted = $true
        $interruptReason = "Skip"
        try { $ps.Stop() } catch { }       # Runspace 強制停止
        break
    }

    if ($TimeoutSec -gt 0 -and ((Get-Date) - $startTime).TotalSeconds -ge $TimeoutSec) {
        $interrupted = $true
        $interruptReason = "Timeout"
        try { $ps.Stop() } catch { }
        break
    }
}
```

中断時の result.Message:

- Skip: `"Module skipped by operator (async runspace stopped; system state may be incomplete)"`
- Timeout: `"Module exceeded timeout of {N}s (async runspace stopped; system state may be incomplete)"`

「system state may be incomplete」を文字列に含めることで、Skip 後の状態が部分適用である可能性を operator に明示する。

### 完走時の捕捉ロジック

```powershell
$wrapper = $wrappedOutput | Where { $_._AsyncWrapper -eq $true } | Select -First 1
$moduleResult = $wrapper.Output | Where { $_._IsModuleResult -eq $true } | Select -First 1
if (-not $moduleResult -and $null -ne $wrapper.LastResult) {
    $moduleResult = $wrapper.LastResult     # フォールバック
}
```

`Invoke-SafeCommand` と同じセマンティクスを保つ。

### クリーンアップ

```powershell
finally {
    $result.Duration = (Get-Date) - $startTime
    if ($null -ne $ps)       { try { $ps.Dispose() } catch { } }
    if ($null -ne $runspace) {
        try { $runspace.Close() } catch { }
        try { $runspace.Dispose() } catch { }
    }
}
```

Skip / Timeout / 例外いずれの経路でも Runspace は確実に解放される。

---

## Skip Flag Path Normalization

`SkipFlagPath` を絶対パスに正規化してから polling する：

```powershell
if (-not [System.IO.Path]::IsPathRooted($skipFlagPath)) {
    $skipFlagPath = Join-Path (Get-Location).Path $skipFlagPath
}
$skipFlagPath = [System.IO.Path]::GetFullPath($skipFlagPath)
```

理由: Status Monitor は別プロセスで cwd が異なる可能性があり、絶対パスに揃えないと Skip 要求を取りこぼす。長時間 polling の途中で fabriq 側 cwd が変わるケースもあるため、loop 開始前に固定する。

---

## Skip フローの全体像

```
[ Status Monitor process ]
    operator が [Skip] ボタンを押す
       ↓
    New-Item -Path .\kernel\json\skip_request.flag -Force
       ↓
[ Main process / Invoke-SafeCommandAsync ]
    polling loop が Test-Path で検出
       ↓
    Remove-Item で flag を消す（次の async 実行のために）
       ↓
    $ps.Stop() で Runspace を強制終了
       ↓
    result.Status = "Error" + 上記の Message
       ↓
    Add-ExecutionResult / Write-ExecutionHistory で記録
       ↓
    AutoPilot 中なら ErrorMode 分岐（skip / retry / ask）
```

---

## 設計上の注意

### 1. Default が opt-in から default-on に切り替わった経緯（kernel 3.3.0）

kernel 2.1.x 初期は次の懸念から「同期実行 default、`__ASYNC__` で opt-in」の設計：

- Runspace 起動オーバーヘッド（数十〜数百 ms / module）
- デバッグ性低下（Runspace 内例外のスタックトレースが分かりにくい）
- 互換性のリスク（一部モジュールが `$Host.UI.PromptForChoice` などホスト依存 API を使うと STA Runspace で崩れる）

3.2.x までの運用で上記懸念が実害として顕在化しなかった一方、「`__ASYNC__` を profile に書き忘れたために Skip ボタン / timeout の安全網が無効化されていた」インシデント側の重みが上回ったため、kernel **3.3.0** で `DefaultAsync=true` を shipped default 化。マーカーが必要だった profile も書き換え不要のまま async 化される後方互換拡張（マーカー自体は ON-only no-op として残置）。

`DefaultAsync=false` への opt-out は、async 経路で誤動作するモジュールを抱えた現場・デバッグ目的で「全モジュールを sync で走らせたい」場合のみ。

### 2. AutoPilot skip/timeout 機能の rejected 経緯

過去に「AutoPilot 全体に skip/timeout を適用する」案が検討されたが、Runspace refactor が広範すぎる vs ハング occurrence が稀という比較で **rejected**（2026-04-22、project memory `project_autopilot_skip_rejected`）。`__ASYNC__` の選択的適用が現行の落とし所。

### 3. Skip した状態の不完全性

Skip された Runspace は強制停止のため、モジュールが書きかけたレジストリ・ファイル・ネットワーク設定が中途半端に残る可能性がある。fabriq はこれを「Error として記録、operator に通知、続行は ErrorMode 次第」と扱う。後続モジュールが影響を受ける場合は profile 設計時に skip 不可能な順序を組む（domain_join の前に hostname_config を絶対に通すなど）。
