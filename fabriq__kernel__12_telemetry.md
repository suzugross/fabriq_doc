# Telemetry レイヤ（AI 開発コーパス）

> **対象**: fabriq / kernel telemetry layer
> **対象バージョン**: kernel 3.2.5（取得元: `E:\fabriq\kernel\KERNEL_VERSION`、commit `fed181a`、2026-05-10）
> **ドキュメント更新日**: 2026-05-10

kernel 3.2.3 で追加された **AI 開発コーパス用** の構造化ロギング層。fabriq の運転中の挙動（モジュール envelope / Show-* / 例外 / CSV 構造メタ / セッションライフサイクル / cmdlet verbose 出力）を JSONL に蓄積し、後で AI が事後分析できるようにする。**運用上のオブザーバビリティ機能ではなく、エビデンス・監査チャネルでもない**（それらは引き続き `evidence/checklist*.html` と `manifest.json` が担う）。

> **重要 — 公開契約上の位置づけ**: 本機能は **`KERNEL_API.md` の公開サーフェス外**（dev/TELEMETRY_INTERNAL.md L3 が明記）。schemaVersion=1 の内側であればフィールドは予告なく追加・改名・削除される可能性がある。外部ツールから読み取る前提のチャネルではない。本書は実装の事実を記述するもので、契約を約束するものではない。

---

## 全体像

```
fabriq.exe 起動
   ↓
main.ps1 の Enable-FabriqVerboseCapture (kernel/json/verbose_capture.flag が存在すれば ON)
   ↓
モジュール 1 件実行（Invoke-SafeCommand 経由）
   ├─ Start-ModuleTelemetry        → envelope.start (JSONL 1 行)
   ├─ Show-Info / Show-Success ...   → show.info / show.success ... (各 1 行)
   ├─ Import-ModuleCsv が走る        → csv.load (1 行)
   ├─ 4>&1 redirect で cmdlet verbose → cmdlet.verbose (各 1 行)
   ├─ モジュール本体実行
   └─ Complete-ModuleTelemetry     → error イベント (catch 済み含む) + envelope.end
                                      ↓ append
logs/telemetry/{SessionID}/modules/{0001}_{module}.jsonl

セッションライフサイクル（Invoke-BatchExecution / Save-ResumeState など）
   └─ Write-KernelTelemetryEvent  → profile.start/end / restart.invoked / resume.consumed / finalize.start/end
                                      ↓ append
logs/telemetry/{SessionID}/_kernel.jsonl
```

実装の中心は `kernel/common.ps1` L280〜L733（telemetry レイヤ本体）と L1429〜L1443 / L1445〜L1578（Verbose capture と `Invoke-SafeCommand` への統合）。`kernel/main.ps1` L297-305 / L320-326 / L544 / L1461 で profile.start/end と verbose capture activation を呼ぶ。

---

## 出力先と SessionID

```
logs/telemetry/
└── {SessionID}/                        ← Get-Date "yyyyMMdd_HHmmss"（common.ps1 L22）
    ├── _meta.json                       ← 1 セッション 1 ファイル（_WriteTelemetryMeta）
    ├── _kernel.jsonl                    ← Write-KernelTelemetryEvent の出力先
    └── modules/
        ├── 0001_hostname_config.jsonl   ← {sequence:0000} + {module name sanitized}
        ├── 0002_ipaddress_config.jsonl
        └── ...
```

- `SessionID` は fabriq 起動時の `Get-Date -Format "yyyyMMdd_HHmmss"`（`$script:SessionID`）。再起動跨ぎでは新しい SessionID になる
- 先頭 4 桁は **時系列 sequence**（profile の `Order` ではない）。FlexProfile での再実行・順不同実行でもファイル名は monotonic
- `safeName` は `[^A-Za-z0-9_\-\.]` を `_` に置換（common.ps1 L632）

### `.gitignore` / log_uploader 除外

- `kernel/json/telemetry_salt.txt` は `.gitignore` 対象（site-specific salt をコミット禁止）
- `logs/telemetry/` は `log_uploader` v1.1.0+ が `/XD logs\telemetry` で除外する（顧客 PC 外への持ち出し禁止契約、kernel 3.2.3 で実装で担保）

---

## プライバシ契約（redaction policy "hash-and-redact-v1"）

`New-TelemetryRedactMap`（common.ps1 L350）が起動時の env vars / グローバルから redact map を構築。各 telemetry イベント書き込み前に `Invoke-TelemetryRedact`（L404、longest-first replacement）でメッセージ・スタックトレース・errorRecord を通す。

| ソース | 扱い | トークン |
|---|---|---|
| `$env:SELECTED_PIN` | **hard redact** | `[REDACTED]` |
| `$env:SELECTED_KANRI_NO` | salt + SHA-256（先頭 12 hex） | `KANRI:<hash>` |
| `$env:SELECTED_OLD_PCNAME` / `NEW_PCNAME` | 同上 | `PC:<hash>` |
| `$env:SELECTED_ETH_*` / `WIFI_*` / `DNS1-4` | 同上 | `IP:<hash>` |
| `$env:SELECTED_PRINTER_<N>_NAME/DRIVER/PORT`（N=1..10） | 同上 | `PRINTER:<hash>` |
| `$env:FABRIQ_WORKER_NAME` | 同上 | `WORKER:<hash>` |
| `$env:COMPUTERNAME` | 同上 | `HOST:<hash>` |
| `$global:FabriqUniqueId` | 同上 | `HW:<hash>` |

### Salt

- 場所: `kernel/json/telemetry_salt.txt`（256 bit、初回自動生成）
- 同一環境 → 同一 hash（cross-session correlation 可能）
- 異環境 → hash 衝突しない
- Salt 生成失敗時は **すべて `[REDACTED]` に倒す**（safe-by-default、common.ps1 L335）
- `_meta.json` には `saltDigest`（salt の SHA-256 先頭 4 byte = 8 hex）が記録される。salt そのものは外に出ない

### redact の限界（実装上の事実）

- 短すぎる値（3 文字未満）は redact map に入れない（誤検出回避、L390）
- モジュール内で `Write-Host` 直呼びされた値は **そもそも捕捉されない**（telemetry は Show-* 5 関数経由のみ）
- 子プロセス（winget / robocopy / dism）の stdout/stderr は対象外
- 値の中間計算結果（env に入っていない加工値）は redact 対象外

---

## モジュール envelope のスキーマ（modules/{seq}_{name}.jsonl）

### envelope.start

`Start-ModuleTelemetry`（common.ps1 L597）が `Invoke-SafeCommand` の入口で発行。

| フィールド | 出処 |
|---|---|
| `module`, `sequence`, `order`, `segment`, `errorMode`, `group`, `isAsync` | 呼び出し側引数 |
| `profileName`, `profileOrder`, `executionMode` | `$global:_FabriqCurrentProfileContext`（main.ps1 L320 で `Invoke-BatchExecution` がセット） |
| `prevModuleName`, `prevModuleStatus` | 同上（cross-module 依存検出用） |

profile context フィールドは **`Invoke-BatchExecution` 経由のときだけ存在**。Script Menu からの単発実行時は不在。

### show.\<function\>

`_TrackShowEvent`（L471）が `Show-Info` / `Show-Success` / `Show-Warning` / `Show-Error` / `Show-Skip` から呼ばれる。

`tag` は `_GetShowTag`（L454）で message 先頭の literal prefix から推測:

| Prefix | tag |
|---|---|
| `[APPLY]` | `apply` |
| `[SKIP]` / `[NOT FOUND]` | `skip` / `notFound` |
| `[VERIFIED]` / `[VERIFY FAILED]` | `verifyPass` / `verifyFail` |
| `[AUTOPILOT]` / `[ASYNC]` / `[RESTART]` | `autopilot` / `async` / `restart` |
| いずれでもない | 関数名（`info` / `success` 等） |

### csv.load

`Import-ModuleCsv`（L1086〜L1101）が正常 return 時に発行。**CSV の値は含まず構造メタのみ**: `fileName`, `path`, `totalRows`, `returnedRows`, `filterEnabled`, `segment`, `columns`。

### cmdlet.verbose

Verbose capture 有効時のみ。`Invoke-SafeCommand`（L1499）の `4>&1 | ForEach-Object` redirect が `[VerboseRecord]` をフィルタして発行。`msg` は redact map 通過。

### error

`Complete-ModuleTelemetry`（L678）が envelope 終了時に `$global:Error` 配列を順次走査して発行（**catch して握り潰した例外も含む** — L650 で envelope 開始時に `$Error.Clear()` してスコープを envelope に閉じている）。

フィールド: `errorType`, `hresult`, `msg`, `scriptStack`, `categoryInfo`, `targetObject`。`scriptStack` は redact 対象。

### envelope.end

`status`, `verified`, `message`, `duration_ms`, `errorCount`, `showCounts`（`info` / `success` / `warning` / `error` / `skip` の累積数）。`verified` は modulesが Post-Apply Verification 未実装の場合は `null`。

---

## セッションライフサイクル（_kernel.jsonl）

`Write-KernelTelemetryEvent`（common.ps1 L561）が以下を発行:

| イベント | 発火点 | キーフィールド |
|---|---|---|
| `profile.start` | `Invoke-BatchExecution`（main.ps1 L297） | `profileName`, `profilePath`, `executionMode`, `moduleCount`, `autoPilot`, `selectedOrders` |
| `profile.end` | 同（L544、natural completion 時のみ） | `modulesRun`, `successCount`, `errorCount`, `skippedCount`, `partialCount`, `cancelledCount`, `outcome` |
| `restart.invoked` | `Save-ResumeState`（common.ps1 L3410） | `profileName`, `executionMode`, `resumeAfterOrder`, `completedCount`, `schemaVersion` |
| `resume.consumed` | `Load-ResumeState`（common.ps1 L3431、複数回発火可能） | 同上系 |
| `finalize.start` / `finalize.end` | `Complete-ProfileExecution`（common.ps1 L3000 / L3095） | `profileName`, `mode` (Auto/Manual), `elapsedMs`, `definedModules`, `durationMs`, `checklistGenerated` |

`profile.end` は `__RESTART__` で early-exit したパスでは **発火しない**（再起動後の `resume.consumed` で対になる）。

---

## `_meta.json` スキーマ

`_WriteTelemetryMeta`（common.ps1 L527）がセッション最初の telemetry 書き込み時に **idempotent に 1 度だけ** 出力。

```json
{
  "telemetrySchemaVersion": 1,
  "sessionId": "20260510_143000",
  "kernelVersion": "3.2.5",
  "startedAt": "2026-05-10T14:30:00.123+09:00",
  "redactionPolicy": "hash-and-redact-v1",
  "saltDigest": "sha256:a3f2c891",
  "host": {
    "os":       { "caption": "...", "version": "...", "build": "..." },
    "hardware": { "manufacturer": "LENOVO", "model": "ThinkCentre M75q Gen 2", "ram_gb": 16 },
    "powershell": "5.1.26200.0"
  }
}
```

`host` は `Get-TelemetryHostInfo`（L492）が `Win32_OperatingSystem` + `Win32_ComputerSystem` を CIM で読む（1 セッション 1 回キャッシュ）。`manufacturer` / `model` は **フリート単位の識別子**（ThinkCentre / OptiPlex 等）として **redact せず raw 保存**（kernel 3.2.4 で追加）。シリアル / MAC / hostname など PC 個体識別子は別所で redact 済みで `_meta.json` には現れない。

---

## Verbose stream capture（cmdlet.verbose チャネル、kernel 3.2.4）

**標準配備、デフォルト ON**。`kernel/json/verbose_capture.flag` がリポジトリに git tracked で同梱されている。

### 仕組み

main.ps1 L1461 の `$null = Enable-FabriqVerboseCapture` が flag 存在チェックで `$global:FabriqVerboseCaptureActive=$true` をセット（common.ps1 L1429）。`Invoke-SafeCommand` がこの flag が真のとき以下を行う（L1474〜L1571）:

1. **save**: `$VerbosePreference` と `$global:PSDefaultParameterValues['*:Verbose']` を退避
2. **set**: `$VerbosePreference = 'Continue'` + `$global:PSDefaultParameterValues['*:Verbose'] = $true`
3. **invoke**: `& $ScriptBlock 4>&1 | ForEach-Object { ... }` で verbose ストリーム redirect。`[VerboseRecord]` 型は telemetry 行きにフィルタ（pass-through せず）、それ以外（ModuleResult 含む）は通常通り pass through
4. **finally**: 上記 2 つを必ず復元（finally で他より先）

なぜ `$VerbosePreference='Continue'` だけでなく `$PSDefaultParameterValues['*:Verbose']=$true` も必要か: 組み込み cmdlet の `ShouldProcess` が「Performing the operation...」verbose を発行するには **明示的な `-Verbose`** 相当が必要で、`$VerbosePreference` だけでは触発されないため（実装コメント L1471 が明記）。

### Opt-out

**flag を削除すれば pre-P5 の telemetry に戻る**（envelope + show.* + csv.load + kernel events のみ、`$VerbosePreference` 触らない、redirect なし）。これが唯一の opt-out。

### 取得対象と非対象（実装上の事実）

| 種類 | 捕捉 |
|---|---|
| 組み込み cmdlet の `ShouldProcess` 経由 verbose（`Set-ItemProperty`, `Rename-Computer`, `Add-Computer`, `Set-NetFirewallProfile`, `New-LocalUser`, ...） | ✅ |
| ユーザ定義関数の `Write-Verbose` | 🔶 該当関数が呼ばれた場合のみ |
| 外部プロセス（`winget`, `robocopy`, `dism`, `slmgr`, native binaries）の stdout/stderr | ❌ PowerShell verbose ストリームの圏外 |
| `Invoke-SafeCommandAsync` 経由（`__ASYNC__` モジュール） | ❌ Phase 1 では child runspace 内 redirect 未実装 |

### Trade-off（実装上の事実）

- **PC 永続トレースなし**: `$VerbosePreference` / `$PSDefaultParameterValues` は process-scoped。レジストリも Event Log も触らない
- **process restart 不要**: `$VerbosePreference` はランタイム評価のため即時有効（先行検討した Module Logging E1 trial が registry mid-session 書き換えで late-load module をカバーできなかった反省で本方式採用、TELEMETRY_INTERNAL.md §6.7「Rejected alternative: Module Logging (E1)」参照）
- **データボリューム**: モジュール envelope あたり ~50-200 events、フルプロファイル 1 セッションで ~1-3 MB（TELEMETRY_INTERNAL.md §6.7 の見積り）。ローテーション機構は Phase 1 では未実装（長期蓄積を避けたい運用は flag を削除する）

---

## reentrancy / 失敗隔離（実装上の契約）

- `$global:_TelemetryWriting` フラグ（L270 注釈）: JSONL append 中は true。Show-* と writer 自身がチェックして再帰防止（Show-Error が telemetry write 中に呼ばれてもループしない）
- すべての telemetry path は `try { } catch { }` で wrap、**失敗してもコンソール / operator にエラーを上げない**（L446 / L591 / L729）
- ディレクトリ生成失敗 → 当該 envelope だけ silent disable、kitting 続行（L626）
- Salt 生成失敗 → 全値 `[REDACTED]` （L335）
- envelope leak（caller が finally を忘れた）→ 次の `Start-ModuleTelemetry` が前 envelope を `Cancelled / "envelope superseded"` で閉じる（L608）

`Telemetry never affects kitting outcomes` が設計契約。

---

## 公開 API への昇格（未実施）

`KERNEL_API.md` の §1〜§5（公開関数 / グローバル / 環境変数 / Profile CSV / ModuleResult 契約）には telemetry レイヤは **載せていない**。理由は dev/TELEMETRY_INTERNAL.md §2 が明記する通り「外部 consumer がまだ存在しない」「schema を自由に変更できる余地を残す」。fabriq_studio や fabriq_evidence_manager 等が telemetry を読み始めた段階で `KERNEL_API.md §11` 新設で MINOR 昇格する想定（その時点で schema は frozen 化）。

それまでは: **モジュール開発者は telemetry の存在を意識せず Show-* / Import-ModuleCsv / `New-ModuleResult` だけを使う**。telemetry は副次副作用としてバックグラウンドで動く。
