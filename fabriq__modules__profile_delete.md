# profile_delete (Standard)

> **対象**: fabriq / modules（profile_delete）
> **対象バージョン**: kernel 3.6.0（取得元: `E:\fabriq\kernel\KERNEL_VERSION` / commit 0fca159） / モジュール VERSION 1.1.0・REQUIRES_KERNEL 3.5.0（取得元: `E:\fabriq\modules\standard\profile_delete\VERSION` / `REQUIRES_KERNEL`）
> **ドキュメント更新日**: 2026-06-16

**カテゴリ**: User Management
**メニュー名**: Delete User Profiles
**VERSION**: 1.1.0  / **REQUIRES_KERNEL**: 3.5.0
**Post-Apply Verification**: なし（実装推奨メモは Guide にあり、現状未実装）
**サブスクリプト**: `profile_delete.ps1`（メイン処理 1 本のみ）

## 目的
指定ユーザーのプロファイル（`%SystemDrive%\Users\<UserName>`）を削除するモジュール。
WMI（`Win32_UserProfile`）を使用して Windows のプロファイル登録を正しく解除し、
WMI レコードが存在しない孤児フォルダや WMI 削除後にフォルダだけが残ったケースに
備えて `Remove-Item -Recurse -Force` の物理削除フォールバックを持ちます。
キッティング後に testuser や defaultuser0 を一掃する用途が代表的。

## 入力 (CSV)
`profile_list.csv`:
- `Enabled`: 有効フラグ（1=削除対象, 0=スキップ）
- `UserName`: ユーザー名（`C:\Users\<UserName>` のフォルダ名と一致）
- `Description`: 表示用
- `Segment`: Segment フィルタ

## 主要ステップ
1. 管理者権限チェック（`Test-AdminPrivilege`）
2. `profile_list.csv` 読み込み（`Import-ModuleCsv -FilterEnabled`、必須列 `Enabled` / `UserName`）
3. `%SystemDrive%\Users` 存在確認（D: ドライブインストール等にも追従）
4. **破壊的パスガード**: 各エントリの `UserName` を `Test-FabriqSafePathComponent` で
   単一パスコンポーネントとして検証し、`[System.IO.Path]::GetFullPath(Join-Path $usersBase $UserName)`
   が `$usersBase + '\'` 配下に strict containment されることを確認。違反は
   `_GuardBlocked` フラグを立てて記録する（確認ゲートより前＝AutoPilot 自動承認下でも有効）。
5. ドライラン表示: ガード違反は `[BLOCKED]`（Fail 予告）、実フォルダ存在は `[APPLY]`、不在は `[SKIP]`
6. 実行確認（`Confirm-ModuleExecution`。AutoPilot は自動 Y）
7. 削除ループ:
   - `_GuardBlocked` のエントリは `Show-Error` で `[Blocked]` 表示し Fail 計上して continue
   - フォルダ不在は Skip
   - **Stage 1**: `Get-CimInstance Win32_UserProfile | Where { LocalPath -like "*\$userName" }` で
     WMI レコードを取得し `Remove-CimInstance` で正規登録解除
   - **Stage 2**: フォルダがまだ残っていれば `Remove-Item -Path -Recurse -Force` で物理削除
8. `New-BatchResult` で集計（`-Success` / `-Skip` / `-Fail`）

## 注意点・運用メモ
- 管理者権限必須（`Win32_UserProfile.Remove-CimInstance` 実行）
- 現在ログオン中のユーザープロファイルは削除不可（OS が拒否、Error 計上）
- WMI 削除のみではフォルダが残るケースがあり、Stage 2 のフォールバックは必須
- 削除されたユーザーの SID / プロファイル情報が Windows から登録解除される（レジストリも含む）
- パス検証（破壊的パスガード, fail-closed）: CSV の `UserName` を
  `Test-FabriqSafePathComponent`（空/空白・`.`/`..`・区切り文字・ワイルドカード・
  末尾ドット/スペース等を拒否）で検証し、さらに `GetFullPath` 解決後のパスが
  `C:\Users\` 配下に strict containment されることを確認する。違反エントリは
  削除を行わず `[BLOCKED]` 表示のうえ Fail として計上する（fail-closed）。
  空/空白の `UserName` は `Join-Path` が `C:\Users` 自体へ collapse し、
  WMI マッチ不成立で Stage 2 が Users ツリー全体を `Remove-Item -Recurse` する
  リスクがあるため、このガードがそれを遮断する。ガードは確認ゲートより前段に
  あり、AutoPilot 自動承認下でも有効（運用 CSV レビューに依存しない）。

## 検証
Post-Apply Verification は未実装。Guide には実装推奨メモがあり、
- `Test-Path "$env:SystemDrive\Users\<UserName>"` が `$false`
- `Get-CimInstance Win32_UserProfile` で該当 LocalPath 不在

の 2 条件で `-Verified $true` 返却が可能との設計指針が記載されているが、現状の
`New-BatchResult` には `-Verified` を渡していない。実行履歴の Verified 列は空欄。
