# history_destroyer (Extended)

> **対象**: fabriq / modules（extended/history_destroyer）
> **対象バージョン**: kernel 3.6.0（取得元: `E:\fabriq\kernel\KERNEL_VERSION` / commit 0fca159）/ module VERSION 1.2.1（取得元: `E:\fabriq\modules\extended\history_destroyer\VERSION`）/ REQUIRES_KERNEL 3.5.0（取得元: `E:\fabriq\modules\extended\history_destroyer\REQUIRES_KERNEL`）
> **ドキュメント更新日**: 2026-06-16

**カテゴリ**: Maintenance
**メニュー名**: History Destroyer
**VERSION**: 1.2.1  / **REQUIRES_KERNEL**: 3.5.0
**Post-Apply Verification**: なし（キャッシュ系は OS が即時に再生成し得るため false FAIL リスクから意図的に非対応）
**サブスクリプト**: なし（CSV ドリブンで 7 種類の Special ハンドラを内包）

## 目的
キッティング完了直前の最終クリーンアップとして、Windows の各種履歴・キャッシュ・一時ファイル・ブラウザデータ・Wi-Fi プロファイル等を一括破壊するモジュール。
CSV ドリブン設計で、`ActionType` 列により `DeletePath` / `ClearRegistry` / `Command`（`[scriptblock]::Create` で生成したスクリプトブロックを `$ErrorActionPreference = 'Stop'` 配下で実行。`Invoke-Expression` は使わず CSV 文字列の再展開を防ぐ）/ `Special`（専用ハンドラ）の 4 種類に分岐。Special には `clear-all-eventlogs` / `recycle-bin`（Shell32.dll P/Invoke の `SHEmptyRecycleBin`）/ `office-mru` / `edge-cleanup` / `chrome-cleanup` / `search-index` / `wifi-ssid` の 7 種類が用意される。

## 入力 (CSV)
`destroy_list.csv` — 削除ターゲット定義（Enabled, GroupName, TargetName, ActionType, TargetPath, IfNotFound, Description, Segment）。
`ssid_list.csv` — `wifi-ssid` ハンドラが参照する Wi-Fi プロファイル削除リスト（Enabled, SSID, Description, Segment）。

## 主要ステップ
1. P/Invoke (`SHEmptyRecycleBin`) を `Add-Type` で取り込み
2. CSV 読み込み（`Import-ModuleCsv -FilterEnabled`） + 環境変数展開（DeletePath 行のみ、`Expand-UserEnvironmentVariables`）
3. **破壊的パス二重ゲート**: DeletePath 行を `Test-FabriqProtectedPath` で 1 行ごとに事前判定し、`_GuardBlocked` / `_GuardReason` を各アイテムに付与。判定は確認ゲート（`Confirm-ModuleExecution`）の外側で行うため AutoPilot 自動承認でも保護が効く
4. ドライラン表示（GroupName でグルーピング表示。ブロック対象は `[BLOCKED]` 赤表示 + 「will be recorded as Fail」を明示）
5. `Confirm-ModuleExecution`
6. **Explorer プロセス停止**（ファイルロック解除）。確認の後に実行するため、CSV 読込失敗・ゼロ件スキップ・オペレータキャンセル時は Explorer に手を付けない
7. メイン処理ループ: ActionType で switch 分岐、Special は `Invoke-DestroyHandler` でルーティング。`_GuardBlocked` の DeletePath 行は削除せず Fail に計上（config error 扱い）
8. **最終ステップ**: Explorer 自動再起動を最大 15 秒待機、復活しなければ `Start-Process explorer.exe` を強制実行
9. `New-BatchResult` 集計

## 注意点・運用メモ
- 13 系統のクリーンアップ: Explorer 履歴 / イベントログ / IME / Temp / クリップボード / DNS / ごみ箱 / Office MRU / Edge / Chrome / Search Index / サムネイル/アイコンキャッシュ / Wi-Fi
- 標準で `WSUS PingID/AccountDomainSid/SusClientId/SusClientIDValidation` の削除も含まれる（マスター展開後の重複検出回避）
- 管理者権限必須（イベントログ消去・Prefetch 削除・Wi-Fi プロファイル削除）
- DeletePath 行は `Test-FabriqProtectedPath` の保護パスゲートを通過しなければ削除されない。ブロックされた行は削除を行わず Fail に計上される（プレビュー段階で `[BLOCKED]` 表示）
- ClearRegistry は値削除後に読み返し検証（fail-closed）を行う。`(default)` 値を除いた残存値が 1 つでもあれば、または読み返し自体が失敗（アクセス拒否等）すれば Error として Fail 計上。残存ゼロを確認できた行のみ Success
- Edge/Chrome のキャッシュ削除時は対応プロセスを `Stop-Process -Force`、その後 13 種類のターゲット（Cache, Code Cache, GPUCache, History, Cookies, Cookies-journal, Top Sites, Top Sites-journal, Visited Links, Web Data, Web Data-journal, Session Storage, Local Storage）を全プロファイル（Default / Profile N）分削除。既存ターゲットが全てロックで 1 件も消せなかった場合は throw して当該行を Fail にする
- 作業中は GUI が一時的に消える（Explorer 停止のため）

## 検証
キャッシュ系は OS が即時再生成するため読み返し検証で false FAIL を起こしやすく、設計上意図的に Verification 非対応。`-Verified` 未渡しで履歴 Verified 列は空欄。
