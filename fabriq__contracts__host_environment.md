# SELECTED_* / FABRIQ_* 環境変数契約

> **対象**: fabriq / 契約（環境変数 + 公開グローバル）
> **対象バージョン**: kernel 3.6.0（取得元: `E:\fabriq\kernel\KERNEL_VERSION`）+ commit `0fca159`（取得元: `git -C E:\fabriq rev-parse --short HEAD`、2026-06-16）
> **ドキュメント更新日**: 2026-06-16

`KERNEL_API.md §3` で公式宣言。fabriq の最重要 IPC（プロセス内コミュニケーション）。`hostlist.csv` の選択行から流れた値が、すべてのモジュールに共通して見える形で配信される。

---

## §3.1 ホスト情報変数（hostlist.csv 由来）

`Set-SelectedHostEnvironment` がホスト選択時点で `$env:` に流し込み。`ENC:` 暗号化フィールドは **この時点で透過復号** される。

### 識別系

| 環境変数 | hostlist.csv 列 | 用途 |
|---|---|---|
| `SELECTED_KANRI_NO` | `AdminID` | 管理 ID（実行履歴の一級識別子、Restore-ExecutionHistory のフィルタキー、HTML チェックリスト Meta） |
| `SELECTED_OLD_PCNAME` | `OldPCName` | 旧 PC 名（リネーム前の参照用） |
| `SELECTED_NEW_PCNAME` | `NewPCName` | 新 PC 名（hostname_config が適用先、エビデンスベースパス命名 / HTML / status.json で常時参照） |

### ネットワーク系

| 環境変数 | hostlist.csv 列 | 用途 |
|---|---|---|
| `SELECTED_ETH_IP` | `EthernetIP` | イーサネット IPv4 |
| `SELECTED_ETH_SUBNET` | `EthernetSubnet` | イーサネットサブネットマスク |
| `SELECTED_ETH_GATEWAY` | `EthernetGateway` | イーサネットゲートウェイ |
| `SELECTED_WIFI_IP` | `WifiIP` | Wi-Fi IPv4 |
| `SELECTED_WIFI_SUBNET` | `WifiSubnet` | Wi-Fi サブネットマスク |
| `SELECTED_WIFI_GATEWAY` | `WifiGateway` | Wi-Fi ゲートウェイ |
| `SELECTED_DNS1` 〜 `SELECTED_DNS4` | `DNS1`..`DNS4` | DNS サーバ最大 4 件 |

ipaddress_config / network_profile_config / DNS 系モジュールが消費。

### セキュリティ系

| 環境変数 | hostlist.csv 列 | 用途 |
|---|---|---|
| `SELECTED_PIN` | `Pin` | セットアップ時の PIN（cert_config / sysprep 等で参照、`ENC:` 暗号化推奨） |

### プリンタ系（10 スロット）

| 環境変数 | hostlist.csv 列 | 用途 |
|---|---|---|
| `SELECTED_PRINTER_<N>_NAME` | `Printer<N>Name` | プリンタ名（N=1..10） |
| `SELECTED_PRINTER_<N>_DRIVER` | `Printer<N>Driver` | ドライバ名 |
| `SELECTED_PRINTER_<N>_PORT` | `Printer<N>Port` | ポート（`IP_192.168.1.50` / `192.168.1.50` / TCPIP_*）|

`printer_driver_config` の register subroutine が消費。HTML チェックリストの Printer Cross-Check では Expected vs Actual で照合。

### 自己参照トークン `__SELF__`（kernel 3.5.0+）

hostlist.csv のセル値がリテラル文字列 `__SELF__` の場合、`Set-SelectedHostEnvironment` が **入室時（リスト選択 + パスフレーズ確定後）** に実行中 PC の live 値（`Get-CurrentPCInfo`）を **列の意味（列文脈）に従って解決** し、対応する `SELECTED_*` に流す。hostlist を持たないキッティングや端末調査（エビデンスファイル名に実 PC 名を載せたい等）を想定した「常に正しい」モード。

#### 解決メカニズム

```
ホスト選択 + パスフレーズ確定
   ↓
Set-SelectedHostEnvironment
   ├── SelectedHost の全列に __SELF__ が 1 つでもあれば Get-CurrentPCInfo を 1 回だけ実行
   │   （__SELF__ が無いプレーンな hostlist は live 取得をスキップ＝オーバーヘッド 0）
   ↓
Resolve-HostValue が各セルを解決
   ├── __SELF__  → その列の SelfKind に対応する live 値（解決不能なら空 + 警告）
   ├── ENC:...   → パスフレーズ読込済みなら透過復号
   └── それ以外  → リテラルそのまま
   ↓
解決後の具体値が SELECTED_* に baked される
```

- 解決は **入室時に 1 回だけ**。`__SELF__` トークン自体は resume state に到達せず、解決後の具体値だけが `SELECTED_*` に焼き付く（baked）。
- `Save-ResumeState` はこの baked 済み具体値をスナップショットし、`Restore-HostEnvironment` がそのまま（verbatim）復元する。したがって **`__RESTART__` 跨ぎや以降の PC 名 / IP 変更には再追従しない**（設計通り）。

#### 対応列（live 値の解決源あり）

| hostlist.csv 列 | 解決される SELECTED_* | live 解決源（SelfKind） |
|---|---|---|
| `OldPCName` | `SELECTED_OLD_PCNAME` | ComputerName（実行中 PC のホスト名） |
| `NewPCName` | `SELECTED_NEW_PCNAME` | ComputerName |
| `EthernetIP` / `EthernetSubnet` / `EthernetGateway` | `SELECTED_ETH_*` | 現 Ethernet アダプタの IP / Subnet / Gateway |
| `WifiIP` / `WifiSubnet` / `WifiGateway` | `SELECTED_WIFI_*` | 現 Wi-Fi アダプタの IP / Subnet / Gateway |
| `DNS1`..`DNS4` | `SELECTED_DNS1`..`SELECTED_DNS4` | live DNS サーバ配列のスロット 1..4 |

#### 非対応列・解決不能（いずれも空 + 警告）

- **Pin / Printer 系**（`Pin`, `Printer<N>Name/Driver/Port`）は live 解決源を持たないため `__SELF__` を置いても解決できず、**空文字列 + 警告**（`Resolve-HostValue` が SelfKind 無しで呼ばれるため）。`SELECTED_KANRI_NO`（AdminID）も同様に解決源なし。
- 対応列でも解決不能な場合（該当アダプタが存在しない・DNS サーバ数がスロット番号に満たない等）は **空文字列 + 警告**。

#### SELECTED_* 契約は不変（モジュール透過）

`SELECTED_*` の **名前 / 型 / 有無の契約は `__SELF__` の有無によって変わらない**。モジュール側は解決済みの具体値を読むだけで、トークンを意識する必要はない（カーネルが入室時に解決を完了させているため）。

---

## §3.2 プロファイル実行パラメータ

| 環境変数 | 由来 | 寿命 |
|---|---|---|
| `FABRIQ_SEGMENT` | profile CSV の `Segment` 列 | モジュール 1 件の実行スコープ（Invoke-BatchExecution が前後で setup/teardown） |
| `FABRIQ_AUTOLOGON_USER` | `__AUTO_to_<User>__` マーカー | 同上（モジュール実行直前に立て、終了後に `$null`） |
| `FABRIQ_WORKER_NAME` | session.json `WorkerName` | セッション全体（Initialize-Session で設定） |
| `FABRIQ_EVIDENCE_BASE` | `Initialize-EvidenceBasePath` | セッション全体（`$global:FabriqEvidenceBasePath` と同値、モジュール内 `Join-Path` 用） |

### `FABRIQ_SEGMENT` の動作詳細

profile CSV の `Segment=office` 行から呼ばれた場合：

```
Invoke-BatchExecution の loop:
   1. $env:FABRIQ_SEGMENT = "office"
   2. & $module.Script   ← この間中 $env:FABRIQ_SEGMENT が見える
   3. $env:FABRIQ_SEGMENT = $null
```

モジュール側 `Import-ModuleCsv` のデフォルト引数 `-Segment $env:FABRIQ_SEGMENT` で自動的にフィルタが効く。

### `FABRIQ_AUTOLOGON_USER` の動作詳細

`__AUTO_to_admin01__` マーカーから呼ばれた場合：

```
Invoke-BatchExecution の loop:
   1. $env:FABRIQ_AUTOLOGON_USER = "admin01"
   2. & $autologon_config.ps1   ← 内部で $env:FABRIQ_AUTOLOGON_USER を読み、autologon_list.csv から User=admin01 行を選択
   3. $env:FABRIQ_AUTOLOGON_USER = $null
```

---

## §2 公開グローバル変数（補足）

環境変数とは別に、`$global:` スコープの公開変数も存在：

| グローバル | 型 | 環境変数版 | 違い |
|---|---|---|---|
| `$global:FabriqMasterPassphrase` | string | (なし) | 環境変数化しないことで child process 漏洩を防ぐ。Runspace 注入時のみ明示転送 |
| `$global:AutoPilotMode` | bool | (なし) | プロセス内 flag（環境変数化すると spawn された script や子プロセスにも影響して混乱） |
| `$global:AutoPilotWaitSec` | int | (なし) | 同上 |
| `$global:AutoConfirmMode` | bool | (なし) | FlexProfile 単発実行用 (kernel 3.1.0+)。kernel 3.3.1 で `Invoke-SafeCommandAsync` の inject hashtable に追加され、child runspace 側でも値が正しく見えるようになった（3.3.0 で `DefaultAsync=true` 既定化により RunSingle が child runspace 経路に乗った際の Y/N プロンプト regression を解消） |
| `$global:FabriqEvidenceBasePath` | string | `FABRIQ_EVIDENCE_BASE` | 環境変数版も提供（モジュール内 `Join-Path` のため） |

---

## 環境変数のライフサイクル

### セッション全体

```
ホスト選択（Show-SessionSetupForm or resume の Restore-HostEnvironment）
   ↓
Set-SelectedHostEnvironment → 全 SELECTED_* 設定
   ↓
Initialize-Session → FABRIQ_WORKER_NAME 設定
   ↓
Initialize-EvidenceBasePath → FABRIQ_EVIDENCE_BASE 設定
   ↓
... モジュール実行中、SELECTED_* / FABRIQ_WORKER_NAME / FABRIQ_EVIDENCE_BASE は不変 ...
   ↓
Reset-FabriqState（New Session / Refabriq）
   ↓
すべての SELECTED_* / FABRIQ_AUTOLOGON_USER / FABRIQ_EVIDENCE_BASE を null 化
```

### モジュール 1 件のスコープ

```
Invoke-BatchExecution の foreach:
   1. _AutoLogonUser があれば $env:FABRIQ_AUTOLOGON_USER を立てる
   2. _Segment があれば $env:FABRIQ_SEGMENT を立てる
   3. & $module.Script
   4. $env:FABRIQ_AUTOLOGON_USER = $null（クリア）
   5. $env:FABRIQ_SEGMENT = $null
```

### 再起動跨ぎ

```
__RESTART__ 検出
   ↓
Save-ResumeState
   ├── 全 SELECTED_* を hash table 化（HostEnvironment fields）
   ├── EvidenceBasePath を json 保存
   └── ProtectedPassphrase（DPAPI 暗号化）も保存
   ↓ 再起動 ↓
   ↓
main.ps1 が Restore-HostEnvironment で全 SELECTED_* を json から戻す
   ↓
EvidenceBasePath / SessionID / ProtectedPassphrase（DPAPI 復号 → $global:FabriqMasterPassphrase）も復元
```

---

## 環境変数の Runspace 継承

profile 経路でモジュールが Runspace 実行（`Invoke-SafeCommandAsync`）に切り替わった場合：

- `$env:*` は Process スコープなので **Runspace に自動継承される**（child runspace が parent process と同じ environment block を共有）
- `$global:*` は Runspace のスコープが異なるため **明示注入が必要**

`Invoke-SafeCommandAsync` の `$inject` ハッシュテーブルが `$global:*` を Runspace の global スコープへ Set-Variable で注入する。

**Runspace 経路に乗る条件（kernel 3.3.0 以降）**: 従来は `__ASYNC__` マーカー以降のモジュールに限られたが、3.3.0 で `async_config.json` に `DefaultAsync` フィールドが追加され、shipped default は `true`。**既定環境では profile 1 行目から全モジュールが Runspace 経路に乗る**ため、本セクションの記述（明示注入の必要性）はマーカー有無に関わらず全 profile モジュールに該当する。`__ASYNC__` マーカーは ON-only no-op として後方互換保持。詳細は [fabriq__kernel__08_async_execution.md](fabriq__kernel__08_async_execution.md)。

---

## 環境変数の運用ルール

### 1. モジュールは SELECTED_* を読み取り専用扱いとする

```powershell
$pcName = $env:SELECTED_NEW_PCNAME   # OK: 読み取り
$env:SELECTED_NEW_PCNAME = "..."     # NG: モジュールから書き換えてはならない
```

理由: 同セッション内の他モジュールへ漏れて副作用となる。書き換えはカーネルの `Set-SelectedHostEnvironment` / `Restore-HostEnvironment` のみ許容。

### 2. ENC: 復号は kernel 経由で透過完了している

モジュールは `Unprotect-FabriqValue` を直接呼ぶ必要がない（hostlist.csv 由来は env 配信時、`_list.csv` 由来は `Import-ModuleCsv` で復号される）。

### 3. 機密値の env 配信を最小限に保つ

`SELECTED_PIN` / `SELECTED_PRINTER_<N>_*` 等の機密項目は env に出すが、**子プロセスを spawn する際は意識的に環境変数を絞る**（明示的な `Start-Process` 呼び出し時の Environment 引数で）。

### 4. fabriq_ios サブシェルからの参照

`apps/fabriq_ios` の `show ip interface` 等のコマンドは `$env:SELECTED_ETH_IP` 等を参照する。fabriq_ios も SELECTED_* 環境変数を「IOS のシステム情報」として表示する設計のため、契約破壊は fabriq_ios の挙動も壊す。
