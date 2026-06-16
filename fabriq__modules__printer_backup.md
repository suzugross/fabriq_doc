# printer_backup (Extended)

> **対象**: fabriq / modules/extended/printer_backup
> **対象バージョン**: モジュール 0.5.0 / kernel 3.6.0（取得元: `E:\fabriq\modules\extended\printer_backup\VERSION` / `E:\fabriq\kernel\KERNEL_VERSION`、commit `0fca159`、2026-06-16）
> **ドキュメント更新日**: 2026-06-16

**カテゴリ**: Printer
**メニュー名**: Backup Printer Setup / Restore Printer Setup
**VERSION**: 0.5.0  / **REQUIRES_KERNEL**: 2.0.0
**Post-Apply Verification**: あり（両スクリプトとも実装、全件 PASS で `-Verified $true`）
**サブスクリプト**: `printer_backup.ps1`（採取, Order 34）, `printer_restore.ps1`（復元, Order 35）

## 目的
このPCのプリンタ環境（ドライバ / ポート / プリンタ / 印刷設定 / 既定プリンタ）の
バックアップと復元を担う統合モジュール。`desktop_icon_config` と同様に backup / restore の
2 スクリプトが 1 モジュール内に同居し、共有の制御 CSV（`printer_backup_config.csv`）を読む。
Backup は現環境を `manifest.json`（`schemaVersion=1` / `manifestType="fabriq-printer-backup"`）を
中心とした portable な `backup/<PC名>/<timestamp>/` フォルダに採取し、Restore はその manifest を
読み込んで依存順（driver → port → printer → print settings → default printer）に再生する。

`module.csv` は 2 行構成（`Backup Printer Setup,Printer,printer_backup.ps1,34,1` /
`Restore Printer Setup,Printer,printer_restore.ps1,35,1`）で、両スクリプトとも
`Category=Printer` / `Enabled=1`。

## 入力 (CSV)
両スクリプトが同一の `printer_backup_config.csv` を 1 行で読み、各スクリプトは自分が利用する列だけを
読み取り、他列は無視する。`Import-ModuleCsv -FilterEnabled` で読むため `Enabled=0` 行は
取り込み 0 件 → `Status=Skipped` を返す。

- `Enabled`: `0`=モジュール全体スキップ / `1`=実行（共通）
- `IncludeDriverBinaries`（Backup）: `1`=pnputil でドライバ payload も export / `0`=メタデータのみ
- `IncludePrintSettings`（Backup）: `1`=各プリンタの PrintConfiguration / Property / DEVMODE /
  HwConfig をキャプチャ / `0`=印刷設定を採取しない
- `SourcePcName`（Restore）: Restore 元のバックアップフォルダ名（明示 override）。空欄時は
  `$env:SELECTED_OLD_PCNAME` に自動フォールバック
- `BackupTimestamp`（Restore）: 空欄=指定 PC 配下の最新を auto-select / 値=その timestamp を指定
- `StrictOsVersion`（Restore）: `1`=osVersion 完全一致のみ許可 / `0`=不一致でも警告のみで続行
- `ReuseInboxDrivers`（Restore）: `1`=Microsoft 供給ドライバの payload を使わず Windows 内蔵を再利用 /
  `0`=backup の payload で強制再注入
- `OnConflict`（Restore）: `skip`=既存プリンタ温存 / `replace`=既存プリンタを削除してから再作成（破壊的）。
  上記以外の値は `Status=Error` で abort
- `RestoreDefaultPrinter`（Restore）: `1`=既定プリンタも復元 / `0`=触らない
- `SkipVirtualPrinters`（Restore）: `1`=仮想プリンタ（PDF/XPS/OneNote/Fax）をスキップ / `0`=復元
- `RestoreHardwareConfig`（Restore）: `1`=installable options（トレイ / フィニッシャ等）を復元 /
  `0`=ドライバ既定のみ（HwConfig をスキップ）
- `Description`: 表示用コメント列。両スクリプトとも読まない

同梱の `preset.csv` は `Column,Value,Label` 形式で、GUI ダッシュボードが各列の選択肢にラベルを
出すための定義（例: `OnConflict,replace,Replace existing printers (destructive)`）。スクリプト本体は
参照しない。

### 出力／入力ディレクトリ
backup / restore の双方が同じパスを使う（同一モジュール内完結）。

```
modules/extended/printer_backup/
  backup/
    <PCName>/                      ← $env:SELECTED_OLD_PCNAME（hostlist OldPCname）優先、
                                     未設定時は $env:COMPUTERNAME にフォールバック
      <yyyy_MM_dd_HHmmss>/         ← 実行時刻
        manifest.json              ← schemaVersion=1 単一の真実源
        printers.json
        ports.json
        drivers_registered.json
        drivers_inf_inventory.json
        _restore_notes.txt
        printsettings/<name>.xml             ← Get-PrintConfiguration の Export-Clixml
        printsettings/<name>.properties.json
        printsettings/<name>.devmode.b64     ← HKCU 側 per-user DEVMODE(Base64)
        printsettings/<name>.hwconfig.json   ← HKLM PrinterDriverData ダンプ
        drivers/<oemNN.inf>/                 ← pnputil /export-driver の出力
```

別 PC から移行する場合は、ソース PC の `<ComputerName>/` サブフォルダをターゲット PC の
`backup/` にコピーしてから Restore を実行する。

## 主要ステップ（Backup Printer Setup）
1. `printer_backup_config.csv` 読み込み（`IncludeDriverBinaries` / `IncludePrintSettings` 参照）
2. 前提チェック: 管理者権限 / `Get-Service Spooler` が `Running`
3. 環境スキャン + dry-run: `Get-Printer` / `Get-PrinterPort` / `Get-PrinterDriver` /
   `Get-WindowsDriver -Online`（`ClassName='Printer'` のみ）。プリンタ 0 件なら `Status=Skipped`
   - ドライバ分類: `Get-WindowsDriver -Online` に出るものを 3rd-party、出ないものを inbox と判定
     （`IsInboxDriver`）。InfPath の basename を突き合わせて `oemNN.inf` を解決
   - PC 名解決: `$env:SELECTED_OLD_PCNAME`（trim）優先、空なら `$env:COMPUTERNAME`
4. 実行確認 `Confirm-ModuleExecution`（AutoPilot は自動 Y）
5. `backup/<PC名>/<timestamp>/` を作成して採取
   - 5.1 `printers.json`（Name/DriverName/PortName/Shared/ShareName/Comment/Location/Published 等）
   - 5.2 `ports.json`（`PortMonitor` から `PortType` を TCPIP/LPR/WSD/Local/Bonjour/Other に分類。
     WSD ポートは warning に積む）
   - 5.3 `drivers_registered.json`（登録ドライバ単位の分類 + payload export 計画）
   - 5.4 `drivers_inf_inventory.json`（3rd-party INF インベントリ）
   - 5.5 印刷設定（`IncludePrintSettings=1` のとき）: プリンタごとに `Get-PrintConfiguration` を
     `Export-Clixml`、`Get-PrinterProperty` を JSON 化、`Resolve-HkcuRoot` で解決した HKCU の
     `Printers\DevModePerUser` から per-user DEVMODE を Base64 保存、HKLM
     `...\Print\Printers\<name>\PrinterDriverData` を型付きでダンプ
   - 5.6 ドライバ payload（`IncludeDriverBinaries=1` のとき）: `pnputil /export-driver <oemNN.inf>` を
     パッケージ単位で実行し `drivers/<oemNN.inf>/` に出力
   - 5.7 既定プリンタ: `Get-CimInstance Win32_Printer -Filter "Default=$true"`
   - 5.8 `manifest.json` を書き出し（`schemaVersion` / `manifestType` / `osArch` / `osVersion` /
     `defaultPrinter` / counts / sizes / includes / items.{printers,ports,drivers} / warnings）。
     書き出し後に総バイト数を再計測して `sizes.totalBytes` を patch
   - 5.9 `_restore_notes.txt`（人間可読の復元ヒント）
6. Post-Apply Verification（後述「検証」）
7. `New-ModuleResult` を返却（`failCount=0` かつ verified=true なら `Success`、それ以外は `Partial`）

## 主要ステップ（Restore Printer Setup）
1. `printer_backup_config.csv` 読み込み（`SourcePcName` / `BackupTimestamp` / `StrictOsVersion` /
   `ReuseInboxDrivers` / `OnConflict` / `RestoreDefaultPrinter` / `SkipVirtualPrinters` /
   `RestoreHardwareConfig` 参照）。`OnConflict` が skip/replace 以外なら `Status=Error`
2. 前提チェック: 管理者権限 / Spooler `Running`
3. ソースバックアップ特定（3a）
   - PC 名解決の優先順位: (a) CSV `SourcePcName` → (b) `$env:SELECTED_OLD_PCNAME`（hostlist
     OldPCname）→ (c) どちらも空なら `Status=Error` で abort（曖昧な「全 PC 走査」は拒否）
   - その PC 名サブフォルダ配下の timestamp フォルダ（`manifest.json` を持つもの）を列挙し、
     `BackupTimestamp` 指定があればフィルタ。最新は `manifest.collectedAt`（パース不能ならフォルダ
     mtime）で選択
4. manifest 検証（3b）: `manifestType="fabriq-printer-backup"` / `schemaVersion=1` 以外は abort
5. 互換性チェック（3c）: `osArch` 不一致=ハード fail で abort。`osVersion` 不一致は
   `StrictOsVersion=1` なら fail、`0` なら warning を出して続行（Win10→Win11 amd64 想定）
6. Restore plan 構築（3d）: RDP リダイレクト判定（`driverName='Remote Desktop Easy Print'` または
   `portName` が `^TS\d+$`）→常にスキップ。`SkipVirtualPrinters=1` のとき仮想プリンタ
   （Microsoft Print To PDF / XPS / Fax / OpenXPS / OneNote、ポート PORTPROMPT:/XPSPort:/FAX:/
   nul:/SHRFAX:/OneNote*）をスキップ。planned printers から参照される port / driver のみ ensure
   対象に絞り、既存との衝突を検出。plan 表示後、planned printers 0 件なら `Status=Skipped`
7. 実行確認 `Confirm-ModuleExecution`（AutoPilot は自動 Y）
8. Phase 1/4 ドライバ復元（5a）: payload フォルダ単位に `pnputil /add-driver <inf> /install`
   （exit 0 と 259 を成功扱い）→ DriverStore の `FileRepository\<base>.inf_<arch>_*` を解決し
   `Add-PrinterDriver -InfPath`。payload 無し / `ReuseInboxDrivers=1` かつ Microsoft 供給ドライバの
   場合は既存・inbox を使い `Add-PrinterDriver -Name` のみ。既登録ドライバは skip
9. Phase 2/4 ポート復元（5b）: 既存は skip。TCPIP は `Add-PrinterPort -PrinterHostAddress`
   （+ `PortNumber`）、LPR は `-LprHostName`/`-LprQueueName`。Local は Windows 標準として skip、
   WSD / Bonjour は復元不能として skip
10. Phase 3/4 プリンタ復元（5c）: 既存衝突は `OnConflict=skip` なら skip、`replace` なら
    `Remove-Printer` 後に再作成。`Add-Printer -DriverName -PortName` → 続いて Shared/ShareName/
    Comment/Location を best-effort で `Set-Printer`
11. Phase 4/4 印刷設定復元（5d）: 2 パス構成
    - パス 1: `Import-Clixml` の `PrintTicketXML` を `Set-PrintConfiguration -PrintTicketXml` で
      round-trip + Color/Collate/DuplexingMode/PaperSize/PaperSource/PrintQuality を明示
      フィールド上書き（各個別 try/catch）。`properties.json` を `Set-PrinterProperty` で best-effort
    - HwConfig / DEVMODE 書き込みが計画されていれば Spooler を `Restart-Service -Force`（2 秒待機）
    - パス 2（Spooler 再起動後）: `RestoreHardwareConfig=1` のとき `hwconfig.json` を HKLM
      `PrinterDriverData` へ型付き復元、per-user `devmode.b64` を HKCU `DevModePerUser` へ Binary 復元
      （driver の再 init に上書きされないよう最後に書く設計）
12. 既定プリンタ復元（5e）: `RestoreDefaultPrinter=1` かつ planned 集合に含まれる場合のみ
    `WScript.Network.SetDefaultPrinter`
13. Post-Apply Verification（後述）
14. `New-BatchResult` を返却（Success=プリンタ成功数、Skip=printer skip + RDP + virtual、
    Fail=printer+driver+port fail）

## 注意点・運用メモ
- 管理者権限必須（pnputil /export-driver, /add-driver / Get-WindowsDriver -Online /
  Add-PrinterDriver / Add-Printer / Add-PrinterPort / Set-PrintConfiguration）。Print Spooler が
  Running であること
- AutoPilot モードでは `Confirm-ModuleExecution` を skip して即実行。`__ASYNC__` 経路
  （kernel 3.3.0+ DefaultAsync）で Skip ボタン中断可能（Guide.txt 記載）
- 自動 SKIP 項目: RDP リダイレクトプリンタ（常時）/ Local ポート（LPT1: / COM1: 等の Windows 標準）/
  WSD・Bonjour ポート（動的発見前提で自動復元不能）/ inbox driver の payload（`ReuseInboxDrivers=1`
  のとき Windows 内蔵を再利用）
- HKCU 参照は `Resolve-HkcuRoot` で解決する。UAC 別アカウント昇格や SYSTEM コンテキスト
  （RunOnce / Scheduled Task）下では裸の `HKCU:` が誤ユーザを指すため、`HKU\<LoggedOnUserSid>` へ
  リダイレクトする（`reg_hkcu_config` / `desktop_icon_*` と同パターン）
- ドライババージョン一致の重要性（Guide.txt、2026-05-13 実機検証）: 用紙サイズ / 両面 / 部数等の
  公開 DM_* field やポート・プリンタ登録自体は version 不一致でも動くが、Canon iR-ADV 系の
  Color モード / Installable Options（カセット 2/3・フィニッシャ）/ ベンダ固有設定は driver private
  DEVMODE 拡張部・HKLM `PrinterDriverData` の vendor binary blob に格納され driver version 依存。
  fresh target への restore（同梱 payload を pnputil install）で version 一致を狙うのが標準運用
- v3 ドライバ（Ricoh RPCS 等）の Win10→Win11 移行は警告 / 部分動作の可能性あり。失敗時はベンダの
  Win11 用最新ドライバを再注入推奨（Guide.txt）
- `manifest.json` の構造は将来 `kernel/PRINTER_BACKUP_MANIFEST.md` として公開契約化する構想あり
  （`schemaVersion=1` は後方互換に拡張する方針、Guide.txt 記載）

## 検証
両スクリプトとも Post-Apply Verification を実装し、全件 PASS のとき `-Verified $true` を返す。

- Backup: 必須ファイル（`manifest.json` / `printers.json` / `ports.json` /
  `drivers_registered.json` / `drivers_inf_inventory.json` / `_restore_notes.txt`）の存在と
  サイズ非ゼロ、`manifest.json` が valid JSON でかつ `schemaVersion=1` /
  `manifestType="fabriq-printer-backup"`、`drivers/` のフォルダ数が export 数と一致、
  `printsettings/*.xml` の数が manifest 上の `printSettingsFile` 数と一致。1 件でも fail すると
  `verified=$false`、`Status` は `failCount` に応じ `Success` / `Partial`
- Restore: planned printers 各々に `Get-Printer` 存在確認 + `DriverName` / `PortName` が manifest と
  一致。`verifyFail=0` かつ `printerFail=0` で `verified=$true`
