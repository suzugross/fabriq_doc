# エビデンス・実行履歴・HTML チェックリスト

> **対象**: fabriq / kernel + evidence_config モジュール
> **対象バージョン**: kernel 3.2.2（取得元: `E:\fabriq\kernel\KERNEL_VERSION`）+ evidence_config 1.6.0（取得元: `E:\fabriq\modules\standard\evidence_config\VERSION`）+ commit `e513cf1`
> **ドキュメント更新日**: 2026-05-07

fabriq は「**やった証拠を残す**」ことを業務契約として担保する。すべてのモジュール実行はスクリーンショット PNG + 実行履歴 CSV 行 + HTML チェックリストへ反映される。

---

## エビデンスベースパス（unified evidence directory）

`Initialize-EvidenceBasePath` がセッション開始時に決定：

```
.\evidence\{Timestamp}_{PCName}_{Serial}_evidence\evidence\
```

例:
```
.\evidence\2026_05_06_103045_NEW-PC-01_T2NXCV06Y22208C_evidence\evidence\
```

中身（サブディレクトリ）:

```
{base}/evidence/
├── auto_capture/         ── Capture-ScreenEvidence の自動 PNG
├── gyotaku/              ── Save-Screenshot の手動 PNG（Status Monitor のボタン経由）
├── checklist/            ── Export-HtmlChecklist の HTML レポート
├── export_history/       ── Export-ExecutionHistory のフルダンプ CSV
└── pc_information/       ── evidence_config モジュールの収集結果（22 セクション）
    └── {date}_{uid}_{pc}/
        ├── 01_SystemInfo.txt
        ├── ...
        ├── 22_OfficeLicense.txt
        └── manifest.json    ── EVIDENCE_MANIFEST.md 公開契約に基づく
```

`SELECTED_NEW_PCNAME` と Hardware Unique ID（BIOS Serial Number 優先、fallback で MAC Address）から命名。Windows パス禁止文字は `-` に sanitize。

---

## Capture-ScreenEvidence（自動スクリーンショット）

モジュール実行ごとに `Invoke-KittingScript` / `Invoke-BatchExecution` が後置で発火：

```powershell
Capture-ScreenEvidence -ModuleName $module.MenuName -Status $result.Status
```

### 動作

1. `[DPIUtil]::SetProcessDPIAware()` で物理解像度を取得（スケーリング無視）
2. `[Screen]::PrimaryScreen.Bounds` のサイズで Bitmap 作成
3. `Graphics.CopyFromScreen` で Primary Screen を描画
4. `evidence/{base}/auto_capture/{ts}_{ModuleName}_{Status}_{PCName}.png` に保存

### 命名規則

```
2026_05_06_103055_HostnameConfiguration_Success_NEW-PC-01.png
```

タイムスタンプ + モジュール名（path-invalid 文字は `_`）+ Status + PC 名。

### Save-Screenshot との違い

| 関数 | トリガ | 保存先 | DPI |
|---|---|---|---|
| `Capture-ScreenEvidence` | モジュール実行直後（自動） | `auto_capture/` | `SetProcessDPIAware()` 呼ぶ |
| `Save-Screenshot` | Status Monitor ボタン（手動） | `gyotaku/` | DPI 変更しない（GUI 破壊回避） |

`SetProcessDPIAware()` は不可逆で、呼ぶと WinForms ウィンドウが scaling 環境で縮む副作用がある。Status Monitor は GUI プロセスなので Save-Screenshot では呼ばない。

---

## execution_history.csv（実行履歴）

`logs/history/execution_history.csv` が永続的な追記 CSV。

### スキーマ（13 列、kernel 3.1.3 で `Order` 列追加）

| 列 | 中身 |
|---|---|
| `Timestamp` | `yyyy-MM-dd HH:mm:ss` |
| `KanriNo` | `$env:SELECTED_KANRI_NO` |
| `PCName` | `$env:SELECTED_NEW_PCNAME` |
| `ModuleName` | `module.csv` の MenuName（`[seg:office]` 等のサフィックス込み） |
| `Category` | カテゴリ |
| `Status` | `Success` / `Error` / `Skipped` / `Cancelled` / `Partial` / `Pending`（Flex の Reset State） |
| `Message` | `New-ModuleResult` の `Message`（カンマ・改行は CSV escape） |
| `WindowsUser` | `WindowsIdentity.GetCurrent().Name` |
| `Worker` | session.json の `WorkerName` |
| `MediaSerial` | session.json の `MediaSerial` |
| `SessionID` | `$script:SessionID`（`yyyyMMdd_HHmmss`） |
| `Verified` | `True` / `False` / 空（Post-Apply Verification の結果） |
| `Order` | profile CSV の `Order`（マーカー / ad-hoc は空） |

### 関数群

| 関数 | 役割 |
|---|---|
| `Initialize-ExecutionHistory` | ディレクトリ生成 + 旧 path（`kernel/execution_history.csv`）からの一回切り migration + `.bak` 作成 |
| `Write-ExecutionHistory` | 1 行追記。書き込み失敗時は 100ms 間隔で 3 回リトライ |
| `Import-ExecutionHistory [-FilterKanriNo] [-Limit]` | 全行 read → KanriNo フィルタ → Timestamp 降順 → Limit 件 |
| `Restore-ExecutionHistory [-SessionIDFilter]` | `$script:ExecutionResults` を CSV から再構築（IsRestored=$true マーク） |
| `Show-ExecutionHistory` | コンソール表示用 |
| `Export-ExecutionHistory` | `logs/history/history_export_{ts}.csv` + `evidence/{base}/export_history/...csv` への dual export |

### Restore-ExecutionHistory の二モード

- **legacy mode**（無引数）: 同 KanriNo の最新 50 件 + `--- Current Session ---` separator を append
- **filter mode**（`-SessionIDFilter`）: 指定 SessionID のみ、件数無制限、separator なし。FlexProfile が batch 開始時に呼んで in-batch state refresh に使う

---

## Export-HtmlChecklist（HTML 監査レポート）

`Complete-ProfileExecution` が finalize 段で呼ぶ。`evidence/{base}/checklist/checklist_{ts}.html` を生成。

### 中身（セクション別）

#### 1. Meta Grid

- AdminID / OldPCName / NewPCName / Profile / Worker / Generated At / Elapsed
- HardwareUniqueId / MediaSerial

#### 2. Network Verification（期待値 vs 実際値）

`Get-CurrentPCInfo` で実 OS から取った値と SELECTED_* 期待値を **行ベースで比較**：

| 項目 | Expected | Actual | Match |
|---|---|---|---|
| PC Name | NEW-PC-01 | NEW-PC-01 | ✓ |
| Ethernet IP | 192.168.1.100 | 192.168.1.100 | ✓ |
| Eth Subnet | 255.255.255.0 | 255.255.255.0 | ✓ |
| Eth Gateway | 192.168.1.1 | 192.168.1.1 | ✓ |
| Wi-Fi IP | 192.168.5.20 | (none) | ✗ |
| DNS | 8.8.8.8, 8.8.4.4 | 8.8.8.8, 8.8.4.4 | ✓ |

DNS は **set-based 比較**（順序非依存、sort してから join）。

#### 3. Printer Cross-Check（3-way）

- **Expected**: hostlist.csv の `Printer<N>Name` 列
- **Actual**: `Get-Printer` の Network / IP port printers
- **Extra**: Actual にあるが Expected に無い（人間が手で入れた、または別経路で増えた）

#### 4. Windows License Status

WMI `SoftwareLicensingProduct` で `LicenseStatus` を取得し human-readable に変換：

```
0 = Unlicensed       (NG, red)
1 = Licensed         (OK, green)
2 = OOB Grace        (Partial, yellow)
3 = OOT Grace        (Partial, yellow)
4 = Non-Genuine Grace
5 = Notification
6 = Extended Grace
```

#### 5. BitLocker Status (C:)

`Get-BitLockerVolume -MountPoint C:` で `ProtectionStatus` / `VolumeStatus` を表示。

#### 6. Module Execution Checklist

profile CSV の `DefinedModules` 全件 vs `ExecutionResults` を Order でクロスマップ：

| Order | Description（CSV から） | MenuName | Status | Verified | Message |
|---|---|---|---|---|---|
| 10 | ホスト名設定 | Hostname Configuration | ✓ Success | ✓ PASS | Renamed to NEW-PC-01 |
| 20 | IP 設定 | IP Address Configuration | ✓ Success | ✓ PASS | DHCP→Static OK |
| 30 | ドメイン参加 | Domain Join | ⚠ Partial | — | 1 of 2 attempted |

`IsRestored=$true` の行も含めて選別（`Select-Object -Last 1`）し、再起動跨ぎでも最新結果が反映される。

### HTML 生成の特徴

- `System.Web.HttpUtility.HtmlEncode` で XSS 防御
- 純粋な inline CSS（外部依存なし、disconnected 環境でも閲覧可能）
- 印刷を意識した最小色彩 + 大きなフォント
- 色分け: Success=green / Partial=yellow / Skipped=gray / Cancelled=yellow / Error=red / Pending=blue / NotRun=gray

---

## evidence_config モジュール（pc_information 収集）

`modules/standard/evidence_config/` は fabriq 同梱の最大規模モジュール（**v1.6.0**）。**31 主セクション + サブセクション 1（id "8b"）= 計 32 件** のシステム情報を `pc_information/{date}_{uid}_{pc}/` 配下に出力し、最後に `manifest.json` を `EVIDENCE_MANIFEST.md` 契約に従って書く（2026-04-30 の inventory 拡張で §27〜§31 を追加）。

### 出力例（抜粋）

```
01_SystemInfo.txt              ── ComputerInfo, OSVersion, BIOS
02 ../*.csv                    ── Local Users / Local Groups / Group Members
05_DomainStatus.txt            ── Domain / Azure AD Status
06_*, 07_*.csv                 ── Network Settings / Printers
08_BitLocker.txt + 8b_*.csv    ── BitLocker + Disk & Partition Info
10_SerialNumber.txt
11_*.csv .. 14_*.csv           ── Installed Software / Firewall / Optional Features / Server Roles
15_PowerSettings.txt
16_WiFiProfiles.txt
17_*.csv                       ── Restore Points
18_DefenderStatus.txt
19_*.csv                       ── Windows Update History
20_TempBackup.txt
21_WindowsLicense.txt          ── slmgr /xpr
22_OfficeLicense.txt           ── Office インストールあれば、なければ Skipped
23_SecurityBaseline.txt
24_GroupPolicySummary.txt
25_*.csv .. 26_BatteryReport   ── Certificates / Battery Report
27_*.csv .. 30_*.csv           ── Environment Vars / Startup / Memory Slots / PnP（2026-04-30 追加）
31_HardwareIdentifiers.txt     ── （2026-04-30 追加）
manifest.json                  ── 全セクションの schemaVersion=1 manifest
```

セクション 14（Server Roles）は Server OS のみ（Client は Skipped）、§22（OfficeLicense）も Office 不在時は Skipped で扱われる。詳細は [fabriq__contracts__evidence_manifest_contract.md](fabriq__contracts__evidence_manifest_contract.md)。

---

## log_uploader モジュール（外部共有への転送）

`modules/extended/log_uploader/` が Linear `[Execute Profile]` の finalize で auto-fire（`silent` mode）。FlexProfile では `[Complete]` ボタンで明示発火。

### 配送先設定（kernel/csv/log_destinations.csv）

| 列 | 意味 |
|---|---|
| Enabled | `1` で有効 |
| Name | 表示名 |
| Path | UNC パス or ローカル絶対パス |
| Username | UNC 認証用（任意、`ENC:` 暗号化可） |
| Password | UNC 認証用（任意、`ENC:` 暗号化可） |

### 配送内容

- `evidence/{base}/` 全体（auto_capture / gyotaku / checklist / export_history / pc_information）
- `logs/{ts}_{uid}_{hostname}.log`（Transcript）
- `execution_history.csv`

robocopy で送る。失敗してもキッティング自体は成功扱い（log 配送はベストエフォート）。
