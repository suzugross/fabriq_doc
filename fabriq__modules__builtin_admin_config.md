# builtin_admin_config (Extended)

> **対象**: fabriq / modules（extended/builtin_admin_config）
> **対象バージョン**: kernel 3.6.0 / commit 0fca159（取得元: `E:\fabriq\kernel\KERNEL_VERSION` / `git rev-parse --short HEAD`）。モジュール VERSION 1.0.1 / REQUIRES_KERNEL 2.0.0（取得元: `E:\fabriq\modules\extended\builtin_admin_config\VERSION` / `REQUIRES_KERNEL`）
> **ドキュメント更新日**: 2026-06-16

**カテゴリ**: User Management
**メニュー名**: Built-in Admin Config
**VERSION**: 1.0.1  / **REQUIRES_KERNEL**: 2.0.0
**Post-Apply Verification**: なし（パスワード値が `Get-LocalUser` で読み返せず Verification 全体を意図的に非実装）
**サブスクリプト**: なし

## 目的
ビルトイン Administrator アカウントの有効化・パスワード設定・パスワード無期限フラグ・説明文を一括設定するモジュール（**有効化専用**。無効化機能は持たない）。
CSV 読み込みは `Import-ModuleCsv -Path $csvPath -FilterEnabled` を使用し、`Enabled=0` の行は除外される。有効な行（`Enabled=1`）の先頭 1 行のみを使用し、全行が無効（または 0 件）の場合は `Status=Skipped`（`Message="No enabled entries"`）で終了する。`Enabled` は標準どおり `1`=実行 / `0`=スキップで、無効化（Disable）分岐は持たない（v1.0.1 で、`-FilterEnabled` により決して到達しなかった旧 Disable 分岐を除去し実態化）。

## 入力 (CSV)
`builtin_admin.csv`:
- **Enabled**: 実行有効化フラグ。`1`=この行を処理対象として読み込み、Administrator を有効化＋パスワード設定。`0`=`Import-ModuleCsv -FilterEnabled` により読み込み段階で除外（スキップ）
- **Password**: 設定パスワード（`ENC:` プレフィックス暗号化対応）
- **PasswordNeverExpires**: 1=無期限 / 0=期限あり
- **Description**: アカウント説明文
- **Segment**: Segment フィルタ（`Import-ModuleCsv` が暗黙参照）

## 主要ステップ
1. CSV 読み込み（`Import-ModuleCsv -FilterEnabled`。読み込み失敗時は Error、有効行 0 件のときは `Skipped`「No enabled entries」、それ以外は有効行の先頭 1 行のみ使用）
2. Password 空欄チェック
3. `Get-LocalUser` で Administrator 存在確認
4. 設定内容（Enable, パスワード期限, Description）を画面表示
5. `Confirm-ModuleExecution` で実行確認
6. `Enable-LocalUser` → `Set-LocalUser` でパスワード → PasswordNeverExpires → Description の順に適用

## 注意点・運用メモ
- パスワード値は冪等性チェックなしで毎回 `Set-LocalUser` を呼ぶ（`Get-LocalUser` ではパスワードを取得できないため構造的に比較不可）
- 何度実行しても結果は同じだが、書き込みは毎回発生する
- Administrator アカウントが存在しない環境（一部のクリーンインストール直後）では Error 終了
- 管理者権限必須（`Set-LocalUser` の前提）

## 検証
パスワード読み返し不可のため Verification 全体を未実装。Enabled / PasswordExpires は技術的には検証可能だが、肝心のパスワード検証ができないため整合性を保つ目的で全体非対応。`-Verified` 未指定で Verified 列は空欄。
