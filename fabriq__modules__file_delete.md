# file_delete (Standard)

> **対象**: fabriq / modules（standard / file_delete）
> **対象バージョン**: kernel 3.6.0（取得元: `E:\fabriq\kernel\KERNEL_VERSION` / commit 0fca159）／ module VERSION 1.1.1・REQUIRES_KERNEL 3.5.0（取得元: `E:\fabriq\modules\standard\file_delete\VERSION` / `REQUIRES_KERNEL`）
> **ドキュメント更新日**: 2026-06-16

**カテゴリ**: Maintenance
**メニュー名**: File Delete
**VERSION**: 1.1.1  / **REQUIRES_KERNEL**: 3.5.0
**Post-Apply Verification**: 実装あり（`Test-Path` で削除後の非存在を検証）
**破壊的削除ガード**: 実装あり（`Test-FabriqProtectedPath` で保護パス・その親・浅階層を `[BLOCKED]` 表示し Fail 計上・削除しない）
**サブスクリプト**: なし

## 目的
キッティング工程で「マスターイメージから引き継いだ不要ファイル」「テスト用一時ファイル」「アプリケーションのキャッシュフォルダ」などを CSV ベースで一括削除するモジュールです。`%TEMP%` / `%LOCALAPPDATA%` などの環境変数展開に対応し、`IfNotFound=Skip` / `IfNotFound=Error` で「存在しなくても OK」「絶対あるべきもの」を行単位に区別できる設計のため、冪等な再実行と「必須ファイルの取りこぼし検出」の両立が可能です。

## 入力 (CSV)
`delete_list.csv` の主な列:
- `Enabled` … 1=実行 / 0=スキップ
- `Description` … 表示ラベル
- `TargetPath` … 削除対象パス（環境変数展開対応）
- `IfNotFound` … `Skip`（既定: 不在でも正常終了）/ `Error`（不在ならエラーとして記録）
- `Segment` … Segment フィルタ

## 主要ステップ
1. `delete_list.csv` 読み込み + 環境変数展開 + **各行に破壊的削除ガードを 1 回適用**（`Test-FabriqProtectedPath`。ガードは確認ゲートの外側で評価されるため AutoPilot の自動 Y でも有効）
2. 各パスの存在状況表示。ガードに引っかかった行は `[BLOCKED]` マーカー + 理由（`... - will be recorded as Fail`）を赤表示
3. 実行確認（AutoPilot 時は自動 Y）
4. 各対象を順次処理:
   - ガードでブロックされた行は削除せず `Show-Error` で「Blocked protected path」を表示し **Fail に計上**（`continue`）
   - 存在しない行は `IfNotFound=Skip` ならスキップ、`Error` なら Fail
   - それ以外は `Remove-Item -Force -Recurse` で削除（フォルダは再帰削除）
5. **Post-Apply Verification**: 削除後に `Test-Path` で対象パスの非存在を全件検証（ブロック行は削除していないため検証対象外＝`[SKIPPED]`）
6. `New-BatchResult ... -Verified $verified` で返却

## 注意点・運用メモ
- 管理者権限は **状況依存**（`Program Files` 等の保護領域なら必須、ユーザー領域なら不要）
- フォルダ指定時は `Remove-Item -Recurse` による再帰削除。ただし保護パス・その親・浅階層（3 セグメント未満）は **破壊的削除ガード** が `[BLOCKED]` として弾き、削除せず Fail 計上するため、`C:\Users` や `%USERPROFILE%`、`C:\Program Files` 等を誤って書いても連鎖削除には至らない（ガードに依存せず CSV 自体を正しく書くこと）
- ブロックされた行は「設定エラー」として Fail に計上される。ガード起因の Fail はリザルトを失敗扱いにするため、想定外の `[BLOCKED]` が出たら CSV を見直す
- `IfNotFound=Skip` を使えば「テストで一時ファイルを置いた → 本番マスターでは存在しない」状況でも冪等
- `IfNotFound=Error` は「絶対消すべきファイル（個人情報の残骸など）」の検出に使う

## 検証
Post-Apply Verification は **実装あり**。Step 5 で `Test-Path` を使って対象パスの非存在を検証し、全件 PASS のとき `-Verified $true`、1 件でも残存していれば `$false` を `New-BatchResult` に渡して返却。ガードでブロックされた行は削除しておらず（既に Fail 計上済み）、ここで「まだ存在する」と二重に減点しないよう `[SKIPPED]` 扱いで検証から除外されます。実行履歴の Verified 列に結果が記録され、Evidence Manager 側で削除完了が突合可能です。`copyfile_config` の Verification 非実装と対照的に、削除側は「不在の検証」が `Test-Path` で確実に判定できるため実装している、という設計判断です。
