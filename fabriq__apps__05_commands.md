# commands/ — 手動操作コマンド集

> **対象**: fabriq / apps (commands)
> **対象バージョン**: 3.6.0（取得元: `E:\fabriq\kernel\KERNEL_VERSION` / commit 0fca159）
> **ドキュメント更新日**: 2026-06-16

`E:\fabriq\commands\` は、操作員が任意のタイミングで **手動で呼び出す** ユーティリティスクリプト群です。README のディレクトリツリーでも「ユーティリティコマンド（diag_crypto, get_evidence, gpupdate 等）」と説明されています（`E:\fabriq\README.md` L76）。Profile 経由で自動実行されるモジュールとは異なり、ここに置かれるスクリプトは「操作員が必要だと判断したとき」に走らせる手元の道具箱としての位置付けで、エビデンス採取や履歴記録の対象外であることが多いのが特徴です。

これらのスクリプトは、現行カーネル／Operator GUI のいずれの自動起動経路にも直接は配線されていません（kernel/main.ps1・operator の dashboard_form.ps1 / apps_dialog.ps1 / quickactions_dialog.ps1 のいずれからも `commands/*.ps1` を呼ぶ参照は確認できない）。操作員がファイルシステム上から直接実行する想定です。なお、旧 out-of-process Status Monitor（`Start-StatusMonitor` 等）は kernel 3.4.0 で非推奨化・3.5.0 で物理削除されており、これらのコマンドを Status Monitor の手動ボタンから呼ぶ経路はもう存在しません。Operator GUI 側の常駐 UI は in-process の Execution Toolbar（`apps/fabriq_operator/lib/execution_toolbar.ps1`）に置き換わっていますが、Execution Toolbar は `[Skip]` / `[Gyotaq]` の 2 ボタンのみを提供し、本ディレクトリのコマンド群を起動するボタンは持ちません。

唯一 GUI と接点を持つのは `system_launcher.ps1` で、これは `apps/system_launcher/system_launcher.ps1` と同一実装のコピーです。ただし Operator ダッシュボードの「System Launcher」ボタン（dashboard_form.ps1 L320 / L508-509）から起動されるのは **apps/ 側のコピー**（kernel/main.ps1 L2091-2093 `.\apps\system_launcher\system_launcher.ps1`）であり、commands/ 側のコピーではありません。

## ファイル一覧

| ファイル | 役割 | 起動経路 |
|---|---|---|
| `gpupdate_command.ps1` | グループポリシー強制更新 | 操作員が手動実行 |
| `temp_command.ps1` | カスタム差し込み用テンプレート (空) | 操作員が手動実行 |
| `explore_restart_command.ps1` | Explorer 再起動 | 操作員が手動実行 |
| `diag_crypto.ps1` | 暗号化 / passphrase 状態の診断 | 操作員 / 開発者が手動実行 |
| `get_evidence.ps1` | PC 情報の手動採取 (split log 形式) | 操作員が手動実行 |
| `system_launcher.ps1` | Windows 設定ショートカットパレット (apps/system_launcher と同一実装) | 操作員が手動実行（GUI は apps/ 側コピーを起動） |

---

## gpupdate_command.ps1

### Role
`gpupdate /force` を実行してグループポリシーを即時適用する。ドメイン参加直後に GPO ベースのポリシーを反映させたい場合に手動で呼ぶ。

### 中身
- `gpupdate /force` を try/catch で実行
- 結果に応じて [SUCCESS] / [ERROR] を色付き出力
- 末尾 `pause` でユーザーが結果を確認してから閉じる

### 起動コンテキスト
操作員がファイルシステム上から手動実行する。Profile に組み込むケースは少ない (組み込むなら専用モジュール化される)。

---

## temp_command.ps1

### Role
**空のテンプレート**。中身は `# ========================================` の枠 + `# template` の 1 行のみ。現場ごとに必要な手作業をその場で書き込んで使う「使い捨てスロット」。

### 中身
```
# ========================================
# template
# ========================================
```

### 起動コンテキスト
操作員がファイルシステム上から手動実行する。ここに案件固有の臨時処理を書き込んで実行する想定。

---

## explore_restart_command.ps1

### Role
`explorer.exe` を停止 → Windows の自動再起動を 15 秒まで待機 → 起動したか確認するコマンド。レジストリ変更の即時反映、Start レイアウト適用後のタスクバー / デスクトップ再描画などに使う。

### 中身
- `Confirm-Execution` で Y/N 確認 (common.ps1 経由)
- `Stop-Process -Name explorer -Force`
- 1 秒間隔で 15 秒まで起動を polling
- 復帰しなかった場合は警告

### 起動コンテキスト
操作員がファイルシステム上から手動実行する。デスクトップやタスクバーが一時的に消える操作なので、`Confirm-Execution` によるユーザー確認必須。

---

## diag_crypto.ps1

### Role
fabriq の暗号化機能 (`Unprotect-FabriqValue` / DPAPI / passphrase) が正しく動いているかを **診断**する。CSV 内に `ENC:` プレフィックスの暗号化値があるかを scan し、復号成否を試す。

### 中身
1. `$global:FabriqMasterPassphrase` の有無確認
2. `Unprotect-FabriqValue`, `Import-ModuleCsv`, `Import-CsvSafe` 関数の存在確認
3. fabriq 配下の `*.csv` を再帰スキャンして `ENC:` パターンを含むファイルを列挙
4. 各 ENC 値に対して復号試行、成否を表示

### 起動コンテキスト
- 開発者 / 操作員がトラブルシュート用に手動実行する (パスフレーズ未投入で `[ENCRYPTED]` のまま動いてしまう問題の切り分け)

CLAUDE.md memory: `project_crypto_security_review.md` で挙げた懸念の確認用。

---

## get_evidence.ps1

### Role
PC 情報を手動採取して `evidence/pc_information/` 配下にテキストログとして保存する。`evidence_config` モジュールが定型化したエビデンス採取とは別経路の **緊急時 / 個別採取**用。

### 中身
- 出力先: `$global:FabriqEvidenceBasePath\pc_information` (未設定時は `evidence/pc_information/<COMPUTERNAME>_<yyyyMMdd_HHmmss>/`)
- マスターログ `_ALL_<COMPUTERNAME>_Log.txt` + セクション別 split log (例: `01_SystemInfo.txt`)
- `Out-Log` 関数で「Console / Master / Split」3 箇所同時出力
- `Start-Section -Title -FileName` でセクション切替
- 採取対象: 基本情報 (hostname / OS / spec)、その他 PC 構成

### 起動コンテキスト
操作員がファイルシステム上から手動実行する。「evidence_config が動かなかったとき / その前後で個別採取したいとき」用の個別採取手段。FabriqApps ダイアログ (apps_dialog.ps1) は `apps/` 配下のサブアプリのみを列挙し、本スクリプトは一覧に現れない。

---

## system_launcher.ps1

### Role
`apps/system_launcher/system_launcher.ps1` と **同一実装** (バイナリ一致)。commands/ にも複製が置かれている。ただし Operator ダッシュボードの「System Launcher」ボタンが起動するのは apps/ 側のコピー (kernel/main.ps1 L2091-2093) であり、commands/ 側コピーを自動起動する経路は確認できない。

### 中身
- 34 項目の Windows ツール (ms-settings: 系 / *.cpl / *.msc / shell:::{GUID} / cmd / powershell / runas)
- `Invoke-Tool` で Type に応じた Start-Process 起動分岐

### 起動コンテキスト
操作員がファイルシステム上から手動実行する。機能は apps/ 側と同一。GUI からの起動 (ダッシュボードの「System Launcher」ボタン) は apps/ 側コピーを参照する。
