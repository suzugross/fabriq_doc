# userdata_backup (Extended)

> **対象**: fabriq / modules/extended/userdata_backup
> **対象バージョン**: モジュール 0.1.1 / kernel 3.6.0（取得元: `E:\fabriq\modules\extended\userdata_backup\VERSION` / `E:\fabriq\kernel\KERNEL_VERSION`、commit `0fca159`、2026-06-16）
> **ドキュメント更新日**: 2026-06-16

**カテゴリ**: Maintenance
**メニュー名**: Backup Userdata / Restore Userdata
**VERSION**: 0.1.1  / **REQUIRES_KERNEL**: 2.0.0
**Post-Apply Verification**: あり（Backup: manifest/notes の存在・サイズ・schema 検証 + 各 entry の data/ 存在、Restore: 各 resolvedPath の Test-Path）
**サブスクリプト**: `userdata_backup.ps1`（Backup Userdata, Order 91）, `userdata_restore.ps1`（Restore Userdata, Order 92）

## 目的
任意のファイル / ディレクトリを portable な backup フォルダへ複製し、companion の restore
モジュールで同一 PC または別 PC へ再展開する **試作モジュール**。`printer_backup` と同じ
backup フォルダ規約・manifest 中心設計・hostlist 駆動の解決ロジックを踏襲する。

backup フォルダ規約は `backup/<OldPCname>/<yyyy_MM_dd_HHmmss>/` で、`<OldPCname>` は
`$env:SELECTED_OLD_PCNAME` を優先、未設定時は `$env:COMPUTERNAME`。`userdata_backup_list.csv`
の enabled 各行が backup 内の 1 entry（`entries/<NN>/data/`）になり、`manifest.json`
（`fabriq-userdata-backup` schemaVersion=1）が restore モジュールが参照する単一の真実源となる。

将来、複数の backup モジュール（`printer_backup` / `userdata_backup` / 今後追加分）を
**単一統合 backup モジュール**へまとめる計画があるため、manifest 構造を `printer_backup` と
parallel に保つ設計（`manifestType` と `items` 内キーのみがモジュール固有）。

## 入力 (CSV)

### userdata_backup_list.csv（N 行、backup 対象を 1 行 1 entry で指定）
- `Enabled`: 0=この行を skip / 1=有効
- `SourcePath`: バックアップ対象パス。`%USERPROFILE%` などの環境変数を含めて可
  （backup 実行時に `ExpandEnvironmentVariables` で展開し manifest に絶対パスで記録）
- `Recurse`: 1=サブディレクトリ含めて再帰コピー（robocopy `/E`）/ 0=トップのみ（`/LEV:1`）。
  ファイル指定の場合は無視
- `ExcludePattern`: セミコロン区切りの除外パターン。末尾 `/` または `\` はディレクトリ名
  （robocopy `/XD`）、それ以外はファイル名（`/XF`）に振り分け。例: `*.tmp;Cache/;~$*`
- `OnConflict`: restore 時の挙動。`skip` / `overwrite` / `rename`（rename = 既存を
  `.bak_yyyyMMdd_HHmmss` にリネーム）。空欄時は `skip` を既定とする
- `IncludeAcl`: 1=ACL（NTFS 権限）も保存（robocopy `/COPYALL`）/ 0=データ・属性・タイムスタンプ
  のみ（`/COPY:DAT`）。既定 0
- `Description`: 表示用メモ（任意）

`RequiredColumns` は `Enabled` / `SourcePath` / `Recurse` / `ExcludePattern` / `OnConflict` /
`IncludeAcl` の 6 列（`Description` は必須列ではない）。

### userdata_backup_config.csv（1 行、restore 時の override）
- `Enabled`: 0=モジュール skip / 1=有効
- `SourcePcName`: 空欄=`$env:SELECTED_OLD_PCNAME` 駆動。値=その PC 名を強制（明示 override）
- `BackupTimestamp`: 空欄=最新を自動選択。値=その timestamp フォルダを指定
- `Description`: 表示用メモ

`RequiredColumns` は `Enabled` / `SourcePcName` / `BackupTimestamp` の 3 列。

### preset.csv（GUI ラベル定義）
列は `Column,Value,Label`。`Enabled`（1/0）、`Recurse`（1=Recurse subdirs / 0=Top level only）、
`OnConflict`（skip / overwrite / rename）、`IncludeAcl`（1=Preserve ACLs `/COPYALL` /
0=Data only `/COPY:DAT`）のドロップダウン候補とラベルを定義する。

## 主要ステップ（Backup / userdata_backup.ps1）
1. `userdata_backup_list.csv` を `Import-ModuleCsv -FilterEnabled` で読み込み。null=Error、
   0 件=Skipped で return
2. 前提チェック: 管理者権限（`Test-AdminPrivilege`、robocopy `/B` backup mode のため必須）+
   `robocopy.exe` の存在
3. Resolve & Scan: PC 名解決（`SELECTED_OLD_PCNAME` 優先）、各 entry の `SourcePath` を環境変数
   展開し dir/file/missing を判定、`ExcludePattern` を file/dir に分解。Backup Plan を表示
   （`[DIR ]`/`[FILE]`/`[MISSING]` マーカー、`/ACL` タグ、missing 件数の警告）
4. 実行確認（`Confirm-ModuleExecution`。AutoPilot は自動 Y）
5. 実行: `backup/<PcName>/<ts>/entries/<NN>/data/` へ robocopy で複製
   - missing source は Skipped として manifest に記録しスキップ
   - dir: `/E`（再帰）または `/LEV:1`（トップのみ）、共通フラグ `/B /R:1 /W:1 /NP`
   - file: parent dir 指定 + filename フィルタ + `/NDL /NS /NC`
   - `IncludeAcl=1`: `/COPYALL`、それ以外 `/COPY:DAT`
   - `ExcludePattern`: file は `/XF`、dir は `/XD`
   - robocopy exit code 判定: `>=8`=Failed、`>=4`=Partial（success にカウント、warning）、
     それ未満=Success。コピー後 `Get-ChildItem` で file/dir/byte 数を集計
6. `manifest.json`（`ConvertTo-Json -Depth 8`, UTF8）+ `_restore_notes.txt` を書き出し。
   各 entry の robocopy ログは `entries/<NN>/entry_log.txt`
7. Post-Apply Verification（後述）
8. `New-BatchResult`（`-Verified` 付き）で返却

## 主要ステップ（Restore / userdata_restore.ps1）
1. `userdata_backup_config.csv` 読み込み（`SourcePcName` / `BackupTimestamp` の override 取得）。
   0 件（Enabled 行なし）=Skipped
2. 前提チェック: 管理者権限 + `robocopy.exe` 存在
3. ソース backup の特定（解決優先順位は後述）。`backup/<PcName>/<ts>/` 直下に `manifest.json`
   を持つ timestamp フォルダを候補化し、CSV 指定 or `manifest.collectedAt` 最新を選択
4. manifest 検証: `manifestType="fabriq-userdata-backup"` かつ `schemaVersion=1` でなければ Error
5. Restore Plan 構築（`status='Skipped'` かつ `backupSubpath` 空のエントリを除外）→ 実行確認
6. 各 entry を robocopy で逆方向に復元
   - OnConflict（manifest の per-entry 値）: `skip`=既存ターゲット温存、`overwrite`=上書き、
     `rename`=既存を `.bak_yyyyMMdd_HHmmss` にリネーム。未知値は skip + fail
   - dir / file で型不一致（dir 期待だが file 等）は fail
   - 復元先の parent dir / target dir を必要に応じて作成
   - dir: `/E`、file: filename フィルタ + `/NDL /NS /NC`。`includeAcl=1` は `/COPYALL`
7. Post-Apply Verification: 各 `resolvedPath` を `Test-Path` で存在確認
8. `New-BatchResult`（`-Verified` 付き）で返却

### Restore 時のソース解決優先順位
PcName:
1. CSV `SourcePcName` が空でなければ → それを使用（明示 override）
2. `$env:SELECTED_OLD_PCNAME` が空でなければ → それを使用（hostlist 駆動）
3. どちらも空 → Error abort

Timestamp:
1. CSV `BackupTimestamp` が値ありならそれを使用
2. 空なら指定 PcName 配下の `manifest.collectedAt` 最新を auto-select

## manifest スキーマ（fabriq-userdata-backup schemaVersion=1）
トップレベル: `schemaVersion` / `manifestType` / `backupVersion`（モジュール VERSION）/
`fabriqKernelVersion` / `collectedAt` / `computerName`（=PcName）/ `hardwareUniqueId`
（`Win32_ComputerSystemProduct.UUID`）/ `osVersion` / `osArch`（arm64 / amd64 / x86）/
`counts {entry, file, dir, missingSource}` / `sizes {totalBytes}` /
`items {entries: [...]}` / `warnings []`。

各 entry: `id` / `sourcePath` / `resolvedPath` / `isDirectory` / `recurse` /
`excludePattern` / `onConflict` / `includeAcl` / `fileCount` / `dirCount` / `byteCount` /
`backupSubpath`（例 `entries/01/data`）/ `robocopyExitCode` / `status`
（Success / Partial / Skipped / Failed）/ `reason`。

`printer_backup` の manifest と意図的に parallel な構造で、`manifestType` と `items` 内の
キーのみがモジュール固有。

## 注意点・運用メモ
- **管理者権限必須**（robocopy `/B` backup mode、ACL 操作）。robocopy.exe は Windows Vista
  以降は常に存在
- **エンジンは robocopy 依存**（long path / locked file / リトライ処理を委譲）
- **Cross-user 復元は out of scope**（同一ユーザ前提）。`%USERPROFILE%` は backup 実行時の
  ユーザに展開されて manifest に絶対パスで記録される。restore 時のユーザが異なる場合は
  手動でパス調整が必要
- **ACL 保存は既定 OFF**。cross-PC で SID が異なるため、復元後の権限が意図と違う可能性。
  要件があればエントリ単位で `IncludeAcl=1`
- **EFS 暗号化ファイル**は robocopy 既定で復号できないものは skip される（`/EFSRAW` opt-in
  列の追加余地あり / 未実装）
- **シンボリックリンク / Junction** は robocopy 既定挙動（target を辿る）。リンクとして保存
  したい場合は `/SL` or `/SJ` の追加が必要（未実装）
- **ロック中のファイル**は robocopy `/B`（バックアップモード）で読み取り試行するが、それでも
  開けないファイルは skip + warning に降格し、他 entry の処理を阻害しない
- AutoPilot モードでは `Confirm-ModuleExecution` を skip して即実行（Guide.txt 記載）

## 検証
両スクリプトとも Post-Apply Verification を実装し、`New-BatchResult -Verified` で結果を渡す。

**Backup**:
- `manifest.json` / `_restore_notes.txt` の存在 + サイズ非ゼロ
- `manifest.json` を再読し `schemaVersion=1` / `manifestType="fabriq-userdata-backup"` を確認
- 非 Skipped 各 entry の `data/` フォルダ存在
- **何もバックアップしなかった backup は PASS にしない**: 非 Skipped entry が 0 件
  （全 source path が missing / skip）の場合は VERIFY FAILED。一部 missing は routine として
  PASS のまま（warnings / manifest `missingSource` / Skip 数で可視化される）

**Restore**:
- 各 entry の `resolvedPath` が target PC 上に存在することを `Test-Path` で確認。
  `verified = (verifyFail -eq 0 -and failCount -eq 0)`
