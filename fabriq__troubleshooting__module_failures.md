# モジュール実行失敗のトラブルシューティング

> **対象**: fabriq / 全モジュール（kernel + standard + extended）
> **対象バージョン**: 3.2.2（取得元: `E:\fabriq\kernel\KERNEL_VERSION`）
> **ドキュメント更新日**: 2026-05-07

モジュールが `Status=Error` または `Status=Partial` を返したとき、**何が起きていて、どこから調べればよいか**をまとめた運用者向けガイド。AutoPilot 中の Retry/Skip ダイアログ、`ErrorMode` 列、ログ・履歴の見方を解説する。

> **対象読者**: fabriq を実機で走らせるオペレータ。コードを読まずに復旧したい想定。

---

## 失敗が報告される 4 つの場面

| 場面 | 観測点 | UI |
|---|---|---|
| **モジュール実行直後** | コンソールバナー（赤色） | `Show-Banner` で `Status: Error` 表示 |
| **AutoPilot 中** | OS のメッセージダイアログ | `Show-AutoPilotErrorDialog`（後述） |
| **Profile 終了時** | サマリ画面 | `Show-ExecutionSummary` で件数集計 |
| **後追い参照** | `evidence/<host>/exec_history.csv` | テキストエディタ / fabriq_evidence_manager |

これら 4 つはすべて **同じ `ModuleResult`** を観測している。観測点が違うだけ。

---

## ModuleResult の 4 状態

`New-ModuleResult` が返す PSCustomObject の `Status` フィールド（[fabriq__contracts__module_result.md](fabriq__contracts__module_result.md) も参照）:

| Status | 意味 | 自動判定 |
|---|---|---|
| `Success` | すべて正常完了。Verified が True なら Post-Apply Verification も合格 | 緑バナー |
| `Partial` | 一部成功・一部失敗。両方が起きた事実を残す | **Error と同じ扱い**（AutoPilot ダイアログ発火） |
| `Error` | モジュール実行失敗 | 赤バナー、AutoPilot ダイアログ発火 |
| `Cancelled` | ユーザがキャンセル / `Confirm-ModuleExecution` 拒否 | グレー、AutoPilot ダイアログ **発火しない**（user 意思を尊重） |

`Partial` は監査上「事実保存」を優先する状態であり、**Error と同等に扱う** べき。Successと混同しないこと。

---

## AutoPilot 中の Retry/Skip ダイアログ

### 発火条件（[main.ps1:442-475](file:///E:/fabriq/kernel/main.ps1)）

```powershell
if ($result.Status -eq "Error" -or $result.Status -eq "Partial") {
    Invoke-ErrorNotification -ModuleName $module.MenuName -Status $result.Status

    if ($global:AutoPilotMode) {
        $errorMode = if ($module._ErrorMode) { $module._ErrorMode.ToLower() } else { "" }
        if ($errorMode -eq "skip")        { ... }
        elseif ($errorMode -eq "retry")   { ... }
        else                              { Show-AutoPilotErrorDialog ... }
    }
}
```

### `_ErrorMode` 列（profile CSV）の挙動

profile CSV の **`ErrorMode`** 列で各モジュールの失敗時挙動を **事前に固定**できる:

| 値 | 動作 | UX |
|---|---|---|
| (空欄) | **Ask**: 対話ダイアログ表示（Retry/Skip） | 操作者の判断を要求 |
| `Skip` | エラーを記録して **次のモジュールへ** | 完全無人。失敗は事後参照 |
| `Retry` | 自動で **`AutoPilotMaxRetry` 回まで再実行** | 一過性の失敗（NW、ロック）に有効 |
| `Ask` | 空欄と同じ | 明示的に対話を要求 |

`AutoPilotMaxRetry` の既定値は kernel で定義（`$script:AutoPilotMaxRetry`）。

### Retry/Skip ダイアログの表示内容

[common.ps1:189-215](file:///E:/fabriq/kernel/common.ps1) の `Show-AutoPilotErrorDialog`:

```
┌── fabriq - AutoPilot Error ──────────────────────┐
│ ⚠  Module: <ModuleName>                          │
│    Status: Error                                  │
│    Detail: <Message from New-ModuleResult>        │
│                                                   │
│    Retry  = Re-execute this module                │
│    Cancel = Skip and continue                     │
│                              [ Retry ] [ Cancel ] │
└──────────────────────────────────────────────────┘
```

- Retry = モジュールを **同じ CSV 行で再実行**。1 段の `do-while` で再走される。
- Cancel = `Status=Error/Partial` のまま `exec_history.csv` に記録して次へ。
- ダイアログが出ない場合（`_ErrorMode` 設定済み）は、上記ロジックがバックグラウンドで判定。

### 「Retry してもまた同じ失敗」が出るとき

Retry は **CSV 値・PC 状態・環境変数を変えずに再実行**するだけ。本質的な原因（DNS 切れ、権限不足、CSV 値が誤り）が解消しないと **無限に同じエラー**が出る。次の Triage 表でカテゴリを判定して、Retry する前に環境を直すこと。

---

## エラーカテゴリ別 Triage チェックリスト

`Status=Error` 報告時、Detail メッセージから以下のカテゴリを判定する:

### A. ネットワーク / DNS / 接続系

| 兆候（Detail メッセージ含む） | 典型原因 | 復旧手順 |
|---|---|---|
| `Failed to resolve hostname`, `DNS request timed out` | DNS 未設定 / DNS サーバ到達不可 | (1) `ipconfig /all` で DNS サーバ確認 / (2) `nslookup <controller>` で疎通確認 |
| `Could not contact domain controller` | LAN ケーブル抜け / VLAN 不一致 / ipaddress_config 失敗 | (1) `Get-NetAdapter` で接続状態 / (2) `ping -4 <gateway>` で L3 確認 |
| `Connection refused`, `timed out` | プロキシ要求 / FW ブロック | (1) IE プロキシ設定確認 / (2) `Test-NetConnection -Port 443 -ComputerName <host>` |
| `WinRM service` 関連 | Remote PowerShell 経路の前提崩れ | 通常 fabriq では使わない。再起動で復旧することがある |

**該当モジュール例**: `domain_join`, `azure_ad_join_check`, `windows_update`, `winget_install`, `office_update`

### B. 権限 / 管理者 / UAC 系

| 兆候 | 典型原因 | 復旧手順 |
|---|---|---|
| `Access is denied`, `アクセスが拒否されました` | 非管理者権限で実行中 / 対象ファイルが ReadOnly | (1) Fabriq.exe を **管理者として実行** /(2) `Get-Acl <path>` で ACL 確認 |
| `RegistryWriteAccess`, `Cannot write to HKLM` | UAC 越しでない非管理プロセス | 同上 |
| `Operation requires elevation` | 直接昇格要求 | Fabriq.exe を「管理者として実行」で再起動。`Refabriq` だけでは権限は変わらない |

**該当モジュール例**: ほぼすべて。特に `bitlocker_config`, `cert_config`, `acl_config`, `hostname_config`

### C. ファイルロック / 占有系

| 兆候 | 典型原因 | 復旧手順 |
|---|---|---|
| `used by another process`, `別のプロセス` | 設定 CSV / ログを Excel 等で開いたまま | (1) Excel を閉じる / (2) `handle.exe <path>` で占有プロセス特定 |
| `cannot delete because it is being used` | プロファイル削除時、まだ Office Outlook 起動中 | 関係プロセスを停止して Retry |
| `Process cannot access the file ...exec_history.csv` | 別ターミナルで fabriq を **二重起動** している | 片方を `q` で終了 |

**該当モジュール例**: `printer_delete`, `process_killer`, `directory_cleaner`, `evidence_config`

### D. CSV 値の誤り（Profile / Hostlist 由来）

| 兆候 | 典型原因 | 復旧手順 |
|---|---|---|
| `Cannot find module CSV row for hostname` | hostlist.csv に **OldPCName / NewPCName が未登録** | hostlist.csv を確認、現在の `$env:COMPUTERNAME` で検索 |
| `Required field missing: Pin` | `bitlocker_config` で hostlist の Pin 欄が空 | hostlist.csv の Pin 列を確認、ENC: で暗号化済みなら復号テスト |
| `Invalid IP/subnet`, `Subnet prefix out of range` | `ipaddress_config` の SubnetPrefix が "24" ではなく "255.255.255.0" など書式違い | `ConvertFrom-SubnetMaskToPrefix` の入力形式に揃える |
| `ENC: ... could not be decrypted` | パスフレーズ間違い / 暗号化時と異なる kernel | [fabriq__troubleshooting__resume_and_state.md](fabriq__troubleshooting__resume_and_state.md) の暗号化セクション参照 |

**Triage の鉄則**: CSV の値間違いは **Retry してもまた失敗**するため、**先に CSV を直す** こと。

### E. モジュール固有の前提条件

| モジュール | 主な前提 | 失敗時の最初の確認 |
|---|---|---|
| `domain_join` | DNS が DC を解決可能、ネットワーク到達、認証情報が有効 | `nslookup <domain>` / `Test-NetConnection -Port 389 -ComputerName <dc>` |
| `windows_update` | `windows_update_list.csv` の `MaxRebootLoops` 残あり、AutoLogon 設定可能 | `wu_state.json` 残存有無 |
| `bitlocker_config` | TPM 有効、回復キー保管先指定 | `manage-bde -status` |
| `winget_install` | winget コマンド導入、インターネット到達 | `winget --version` |
| `cert_config` | 対象 CER/PFX ファイルが存在 | `Test-Path` で確認 |
| `pianist` | 対象アプリがインストール済み、Profile 設定が正しい | `procedure.csv` 構造、`pianist.json` schema |

各モジュールの詳細は `fabriq__modules__*.md` を参照。

### F. システム状態前提（再起動・冪等性）

| 兆候 | 典型原因 | 復旧手順 |
|---|---|---|
| `Restart required`, `Reboot pending` | 前モジュールの再起動指示が未消化 | `restart_config` で再起動 → 再開 |
| `Operation completed but reboot pending` | Sysprep / 機能インストール直後 | 再起動して Refabriq |
| `Module already applied`（Verified=True） | 冪等再実行で No-op 確認済み | **エラーではない**。Status=Success なら問題なし |

**冪等性の原則**: fabriq モジュールは原則 **idempotent**。同じ CSV で 2 回走らせて 2 回目が `Success`（Verified=True）になるのが正しい挙動。2 回目で **Error** が出る場合、モジュール側のバグか、状態前提の崩れ。

---

## ログ・履歴の見方

### 順序: 直近 → 過去

1. **コンソール出力**（バナー赤）— 最直近のメッセージ
2. **`evidence/<host>/exec_history.csv`** — 1 セッション内の全モジュール結果（CSV 形式）
3. **`logs/transcript_<timestamp>.txt`** — PowerShell トランスクリプト（フル詳細）
4. **モジュール固有ログ**（例: `wu_state.json`, `wu_completed.json`, モジュール内 `.log` ファイル）

### `exec_history.csv` の重要列

| 列 | 役割 | 失敗時にチェックすべきこと |
|---|---|---|
| `Order` | profile 内実行順 | 同 Order の重複（再実行の痕跡）を確認 |
| `MenuName` | モジュール名 | profile CSV の `MenuName` と一致 |
| `Status` | Success/Partial/Error/Cancelled | 連続 Error/Cancelled 行は要注意 |
| `Verified` | True/False/(空) | False なら **Post-Apply Verification** で否認 |
| `Message` | New-ModuleResult から | 復旧の起点。可能ならエラーをそのまま検索 |
| `Timestamp` | 開始時刻 | 直前モジュールとの時間差で再起動を疑う |

`Verified=True` でも Status=Error の組み合わせは **稀だが起きる**（部分成功で検証だけ通るパターン）。Message を必ず確認。

### Transcript（`logs/transcript_*.txt`）

PowerShell `Start-Transcript` 由来。**実行コマンドそのものとその stdout/stderr** が時系列で残る。`exec_history.csv` の Message が短すぎて原因不明な時、必ずこちらを当たる。

### モジュール固有のログ場所（代表例）

- `windows_update`: `wu_state.json`（loop state）/ `wu_completed.json`（finalize）
- `pianist`: `modules/extended/pianist/profiles/<name>/last_run/`（実行ごとに上書き）
- `evidence_config`: `evidence/<host>/<section>/`（収集物そのもの）
- `restart_config`: 再起動後の RunOnce 経由で `kernel/json/resume_state.json` に状態が一旦退避

---

## 「再現させない」ための予防策

### A. profile CSV に `ErrorMode` を明示

無人化したいモジュールには `ErrorMode=Skip`、一過性失敗が見込まれるモジュールには `ErrorMode=Retry` を設定。空欄＋AutoPilot は **対話ダイアログ表示**となり、無人化が崩れる。

### B. `__RESTART__` マーカーの活用

profile CSV に `__RESTART__` 行を入れると、**そこで PC を再起動 → 再ログオン後に同 profile を resume** する。再起動が必須なモジュール（hostname_config、domain_join、windows feature install）は **直前に `__RESTART__` を置く**こと。詳細は [fabriq__kernel__05_resume_restart.md](fabriq__kernel__05_resume_restart.md)。

### C. Pre-flight チェックモジュールを上位に置く

profile の先頭に検査専用モジュール（自作 PS1）を 1 つ置き、ネットワーク・権限・前提パッケージ等を **明示的にエラーを発生させて事前に止める** ことで、本番モジュールが半端な状態で失敗する事故を防ぐ。

### D. hostlist.csv の事前検証

`fabriq_evidence_manager` の **ホストリスト整合性検証** で、profile 実行前に各 PC の必須列（Pin, NewPCName, IP 等）が埋まっているかを一括確認できる（[fabriq_evidence_manager__usage__02_hostlist_verification.md](fabriq_evidence_manager__usage__02_hostlist_verification.md) 参照）。

### E. モジュールの `Verified` を見る

`Status=Success` だけで満足せず **`Verified=True`** も確認する。`Verified=False` は「実行は成功したが、適用後の読み戻しチェックで一致していない」状態を意味し、Post-Apply Verification を実装したモジュールでは **真の合否**を表す。

---

## 異常状態の早期検知

### Refabriq が必要なケース

`exec_history.csv` の `Verified` 列がほぼ全行 `(空)` の場合、以前のセッション状態が混入している可能性。**メインメニューから `r` (Refabriq)** で再起動すると、プロセスは再ロードされるが状態ファイルは残るため、`NewSession` が必要なケースもある（詳細は [fabriq__troubleshooting__resume_and_state.md](fabriq__troubleshooting__resume_and_state.md) 参照）。

### 連続 Error に対しての判断基準

- 同モジュールで **2 回連続 Error** → CSV 値の誤りを疑う（Retry を **やめて** CSV を直す）
- 異モジュールで **連続 Error** → 共通前提の崩れ（権限失効・ネットワーク断・ディスク満杯）。`Get-PSDrive` / `Get-NetAdapter` / `whoami /priv` を確認

### 「Cancelled の連発」が起きる場合

`Cancelled` は `Confirm-ModuleExecution` 拒否 = 操作者の意思。**AutoPilot 下で連発する場合**は AutoPilotMode が `[Confirm-ModuleExecution]` をモック化していない（テスト環境設定漏れ）か、profile CSV にデバッグ用の Cancel 強制が残っている可能性。

---

## 関連ドキュメント

| ドキュメント | 関係 |
|---|---|
| [fabriq__contracts__module_result.md](fabriq__contracts__module_result.md) | `New-ModuleResult` の正式契約（Status/Verified/Message） |
| [fabriq__troubleshooting__resume_and_state.md](fabriq__troubleshooting__resume_and_state.md) | resume_state.json / session.json の異常時復旧（本書の姉妹ドキュメント） |
| [fabriq__usage__03_profile_execution_linear.md](fabriq__usage__03_profile_execution_linear.md) | Linear モードでの正常時の挙動 |
| [fabriq__usage__04_flexprofile_dashboard.md](fabriq__usage__04_flexprofile_dashboard.md) | FlexProfile ダッシュボードでの再実行・状態表示 |
| [fabriq__kernel__05_resume_restart.md](fabriq__kernel__05_resume_restart.md) | __RESTART__ + resume の正常フロー |
| [fabriq__contracts__profile_csv_schema.md](fabriq__contracts__profile_csv_schema.md) | `ErrorMode` 列の正式仕様 |

---

## 変更履歴

- 2026-05-07 初版作成（kernel 3.2.2 commit `e513cf1` を対象、AutoPilot Retry ダイアログ + ErrorMode 4 値 + Triage 6 カテゴリ + 予防策 5 項目を網羅）
