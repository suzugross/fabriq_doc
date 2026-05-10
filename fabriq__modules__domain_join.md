# domain_join (Standard)

> **対象**: fabriq / modules/standard/domain_join
> **対象バージョン**: モジュール 2.0.0 / kernel 3.2.5（取得元: `E:\fabriq\modules\standard\domain_join\VERSION` / `E:\fabriq\kernel\KERNEL_VERSION`、commit `fed181a`、2026-05-10）
> **ドキュメント更新日**: 2026-05-10

**カテゴリ**: Network
**メニュー名**: Domain Join
**VERSION**: 2.0.0  / **REQUIRES_KERNEL**: 2.0.0
**Post-Apply Verification**: なし（再起動後に反映されるため、再起動前の検証では情報量が足りない）
**サブスクリプト**: なし

## 目的
PC を Active Directory ドメインに参加させるモジュール。`Add-Computer` の薄いラッパで、**v2.0.0 で挙動が大きく変わった**: 内部でのリトライループ／GUI ダイアログを廃止し、失敗時は即座に `Status=Error` で返す **fail-fast 構造**に再設計された。リトライ・スキップ・ダイアログ表示の制御は profile CSV の `ErrorMode` 列および FlexProfile dashboard に **完全委譲** する。

## v2.0.0 の破壊的変更（v1.x からの移行）

| 観点 | v1.x | v2.0.0 |
|---|---|---|
| **失敗時の挙動** | 内部 GUI ダイアログ + 無限リトライループ（`adminstop` 入力で中断） | 失敗即 `Status=Error` 返却（リトライしない） |
| **DNS 不到達時** | （v1.x の挙動は失敗時ダイアログに同じ） | Ping 2 回プローブで bounded fail-fast。不到達なら `Add-Computer` を試みず即 Error（`"DNS unreachable: <ip>"`） |
| **リトライ責任** | モジュール内部 | profile CSV の `ErrorMode` / FlexProfile dashboard |
| **AutoPilot 適合性** | リトライダイアログが立ち上がるため無人運用に不向き | `ErrorMode=retry` 等と組み合わせて完全無人化可能 |
| **`$ErrorActionPreference`** | （旧実装） | local Stop preference を設定し、`Add-Computer` の non-terminating エラー（DNS 解決失敗 / auth 失敗 / DC 不到達）を catch ブロックに集約 |

**運用への影響**: AutoPilot を使っている profile では `ErrorMode` の設定が事実上必須になる（後述）。

## 入力 (CSV)
`domain.csv` の主な列:
- `Enabled`: 1=実行 / 0=スキップ
- `domain`: ドメイン名（例: `example.local`）
- `user`: ドメイン参加用アカウント
- `pass`: パスワード（**`ENC:` プレフィックス暗号化対応**）
- `dns`: DNS サーバ IP（接続確認 Ping 用）
- `Description` / `Segment`

## 主要ステップ
1. `domain.csv` から有効エントリ読み込み（最初の 1 行のみ使用）
2. **DNS 接続事前チェック**: `Test-Connection -Count 2` で DNS サーバを bounded プローブ。不到達なら `Add-Computer` を試みずに `Status=Error` 即返却（診断メッセージ `"DNS unreachable: <ip>"` が dashboard に表示される）
3. `Add-Computer -DomainName <DOMAIN> -Credential <PSCredential> -Force` 実行
4. `Try/Catch` で例外を捕捉。成功なら `Status=Success`、例外なら `Status=Error`（即返却）

## ErrorMode による失敗時挙動の制御

profile CSV の `ErrorMode` 列で失敗時挙動を宣言する。

| ErrorMode | 挙動 |
|---|---|
| `retry` | 失敗時に最大 5 回まで自動再試行（DNS 復旧待ち等の一過性障害向け、kernel 既定 `AutoPilotMaxRetry=5`） |
| `skip` | 失敗をログに残して次モジュールへ自動継続 |
| 空白 | `Show-AutoPilotErrorDialog` で operator が Retry / Skip を意思決定 |

FlexProfile では失敗しても dashboard に戻るため Status バッジで Error が視認でき、`[Run]` ボタンで domain_join のみ単発再試行することも可能。

## 注意点・運用メモ

- **管理者権限必須**（`Add-Computer` 実行のため）
- DNS サーバへのネットワーク接続が必須（Ping 失敗で **`Add-Computer` を試みない**）
- ENC: 暗号化パスワードを使う場合は Fabriq 起動時のマスターパスフレーズ入力が必要
- 反映は **再起動後**（Windows 仕様）。本モジュールは再起動はトリガーしない（後続 `restart_config` で集約再起動）
- v2.0.0 から内部 GUI ダイアログが消えたため、**operator 介入のキューポイントは ErrorMode 空欄時の `Show-AutoPilotErrorDialog` に集約**された

## 検証
Post-Apply Verification は **未実装**。理由は、ドメイン参加状態は `Add-Computer` 実行直後ではなく **次回再起動後** に `(Get-WmiObject Win32_ComputerSystem).PartOfDomain = $true` として反映される Windows 仕様のため、再起動前のメモリ上で検証しても判定が曖昧になるため（`project_verification_exclusions.md` 整合）。`-Verified` は未渡しで Verified 列は空欄。

`Add-Computer` の成否で結果判定し、再起動後の確認は手動または `evidence_config` の収集レポート（`§05 Domain / Azure AD Status`）で行う運用。
