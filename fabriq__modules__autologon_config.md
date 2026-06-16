# autologon_config (Standard)

> **対象**: fabriq / modules/standard/autologon_config
> **対象バージョン**: kernel 3.6.0 / commit 0fca159 — モジュール VERSION 1.1.0 / REQUIRES_KERNEL 2.0.0（取得元: `E:\fabriq\modules\standard\autologon_config\VERSION` / `REQUIRES_KERNEL`）
> **ドキュメント更新日**: 2026-06-16

**カテゴリ**: System
**メニュー名**: AutoLogon Config
**VERSION**: 1.1.0  / **REQUIRES_KERNEL**: 2.0.0
**Post-Apply Verification**: あり（Step 6.5 で Winlogon 5 値を読み返し `-Verified` を返却）
**サブスクリプト**: なし

## 目的
キッティング工程で「再起動を挟んでも一度だけ自動ログオンしたい」需要に応えるため、`HKLM\...\Winlogon` に `AutoAdminLogon` / `DefaultUserName` / `DefaultPassword` / `DefaultDomainName` / `AutoLogonCount=1` を書き込みます。`AutoLogonCount=1` を使うことで「次回 1 回限り」の挙動になり、ログオン後に Windows 自身が資格情報を自動消去するため、放置時のセキュリティリスクを最小化しています。Profile 実行時は `__AUTO_to_<User>__` マーカー経由で `$env:FABRIQ_AUTOLOGON_USER` が渡され、対象ユーザーが自動選択されます。

## 入力 (CSV)
`autologon_list.csv` の主な列:
- `Enabled` … 1=対象 / 0=スキップ
- `No` … 識別番号（対話 No 入力 / Profile 指定でも使用）
- `User` … ユーザー名
- `Password` … パスワード（**`ENC:` プレフィックス暗号化対応**。マスターパスフレーズで復号）
- `Domain` … ドメイン名（ローカルユーザーは空欄）
- `Description` … 表示ラベル
- `Segment` … Segment フィルタ

## 主要ステップ
1. `autologon_list.csv` を読み込み（`Import-ModuleCsv -FilterEnabled`）
2. `$env:FABRIQ_AUTOLOGON_USER` または対話 `No` 番号で対象ユーザー決定（Enabled=1 が 1 件なら自動選択）
3. 対象設定を表示（パスワードはマスク表示）
4. **冪等性チェック**: 現在のレジストリ値 5 要素（`AutoAdminLogon="1"` / `DefaultUserName` 一致 / `DefaultPassword` を `-ceq`（大文字小文字区別）で一致 / Domain 整合 / `AutoLogonCount>=1`）を読み返し、すべて一致なら `Skipped`（`-Verified $true`）で終了。Domain 整合は CSV の `Domain` が空欄ならレジストリ `DefaultDomainName` も空、CSV に値があれば一致を要求
5. 実行確認（AutoPilot 時は自動 Y）
6. レジストリ書き込み（`Set-ItemProperty -ErrorAction Stop` で個別設定し `$failCount` を計上）。`Domain` が空欄のローカルアカウント時は `Remove-ItemProperty` で stale な `DefaultDomainName` をクリア
7. **Post-Apply Verification**: 書き込んだ 5 値を読み返して `$verified` を判定（後述「検証」節）

## 注意点・運用メモ
- **管理者権限必須**（HKLM 書き込み。明示チェックはせず HKLM 書き込み失敗で検出）
- パスワードは `ENC:` プレフィックス対応。ENC 復号失敗時は対話モードで入力を求める／Error 終了
- `AutoLogonCount=1` を必ず付けるため、無人放置で何度も自動ログオンされる事故を防止
- ローカルアカウント運用時は `Domain` を空欄、ドメイン参加端末でドメインユーザー指定時は `Domain` を埋めること。`Domain` 空欄時は他構成が残した stale な `DefaultDomainName` を `Remove-ItemProperty` でクリアし、`STALEDOMAIN\user` での次回起動失敗を防止する
- Profile 連携時は profile 側に `__AUTO_to_<User>__` を書き、kernel ランナーが `$env:FABRIQ_AUTOLOGON_USER` を立ててからこのモジュールを起動する流れ

## 検証
Step 6.5 で **Post-Apply Verification を実装**しています。書き込み直後に `Get-ItemProperty -Path $WINLOGON_PATH` で Winlogon を一括取得し、以下 5 要素を照合して `$verified`（bool）を決定します:
1. `AutoAdminLogon` が `"1"`
2. `DefaultUserName` が CSV の `User` と一致
3. `DefaultPassword` が CSV の `Password` と `-ceq`（大文字小文字を区別）で一致
4. Domain 整合（CSV `Domain` 空欄ならレジストリ `DefaultDomainName` も空、値があれば一致）
5. `AutoLogonCount` が 1 以上

一致時は `[VERIFIED]`、不一致または読み返し失敗時は `[VERIFY FAILED]` を表示し、`$verified` は `$false` になります。判定結果は `New-ModuleResult ... -Verified $verified` で結果に乗ります（冪等 Skipped 経路では `-Verified $true`）。`$failCount > 0` のときは `Status="Partial"`、それ以外は `Status="Success"` を返します。

実際の自動ログオン挙動そのものは次回再起動時の動作で確認する運用です（fabriq 標準では `restart_config` 経由）。
