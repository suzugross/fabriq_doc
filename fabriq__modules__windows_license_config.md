# windows_license_config (Standard)

> **対象**: fabriq / modules/standard/windows_license_config
> **対象バージョン**: モジュール 1.1.0 / kernel 3.6.0（取得元: `E:\fabriq\modules\standard\windows_license_config\VERSION` / `E:\fabriq\kernel\KERNEL_VERSION`、commit `0fca159`、2026-06-16）
> **ドキュメント更新日**: 2026-06-16

**カテゴリ**: Security
**メニュー名**: Install Product Key / Activate Windows License
**VERSION**: 1.1.0  / **REQUIRES_KERNEL**: 2.0.0
**Post-Apply Verification**: Install 側 = 投入キー末尾5桁の read-back 照合（`-Verified` 返却あり）、Activate 側 = `LicenseStatus` 再取得確認（`-Verified` 引数は未使用）
**サブスクリプト**:
- `windows_license_install.ps1` … プロダクトキー投入
- `windows_license_auth.ps1` … オンライン認証

## 目的
Windows プロダクトキーのインストールとライセンス認証を 2 つのスクリプトで分離して扱うモジュール。
キー投入は `SoftwareLicensingService.InstallProductKey`（CIM 経由）で行い、認証は `SoftwareLicensingService.RefreshLicenseStatus`
を起動して結果を `SoftwareLicensingProduct` から再取得することで状態確認する。インターネット未接続
環境では Activate がスキップされる運用設計。

## v1.0.1 の変更（セキュリティ強化、v1.0.0 からの差分）

`windows_license_install.ps1` に **ProductKey の transcript 漏洩マスク**を実装（A3-T1-A、3 箇所）。共通 redact-map 経路（`Invoke-TelemetryRedact`）はキー値そのものを redact map に持たないため適用不能で、表示時マスクが正解。

| 場所 | v1.0.0 | v1.0.1 |
|---|---|---|
| L59 CSV 読み込み後 | `Write-Host "  Key: $productKey"` 生表示 | `Get-MaskedKey` 経由マスク表示 |
| L90 形式不正時 | `Show-Error "Invalid key format: $productKey"` | 生キー値を除去、形式テンプレ（`XXXXX-XXXXX-XXXXX-XXXXX-XXXXX`）のみ表示 |
| L118 確認ダイアログ前 | `Write-Host "New Key: $productKey ..."` 生表示 | `Get-MaskedKey` 経由マスク表示 |

`Get-MaskedKey` ヘルパは script 先頭にローカル定義（autologon_config L87-91 のマスク手法と同精神。`common.ps1` の `Invoke-TelemetryRedact` は redact-map 型で要件不一致のため独自実装）。**末尾 5 文字のみ可視化、dash は維持、他 24 文字を `*` に置換**（`*****-*****-*****-*****-XXXXX`）。

公開 API 不変、`REQUIRES_KERNEL` 据え置き 2.0.0、CSV スキーマ不変。

## v1.1.0 の変更（Install 側に Post-Apply Verification を追加）

`windows_license_install.ps1` に **投入キーの read-back 照合（Step 6）** を実装。キー投入後、`SoftwareLicensingProduct` を再取得し、**投入キー末尾 5 桁が `PartialProductKey` と一致するか**を照合する。`Get-CimInstance` のリフレッシュ遅延を吸収するため **2 秒待機 × 最大 3 回**リトライし、一致した時点で打ち切る（`windows_license_install.ps1` L180-198）。

「Windows 製品が `PartialProductKey` を持っている」という事実だけでは検証にならない（旧キーが残留していても同じ見え方になる）ため、末尾 5 桁照合で投入キーそのものを確認する。末尾 5 桁は `Get-MaskedKey` が設計上可視化している部分のため、新たな漏洩は生じない（コメント L182-185）。

照合結果に応じて `New-ModuleResult` の `-Verified` を返却する:

| 条件 | 戻り値 |
|---|---|
| 末尾 5 桁が一致する製品あり | `Status=Success` / `-Verified $true`（`windows_license_install.ps1` L205-213） |
| Windows 製品は読めるが末尾 5 桁が一致しない（旧キー残留） | `Status=Error` "Key mismatch after install"（fail closed）（L215-223） |
| Windows ライセンス製品が一切読み出せない | `Status=Success` / `-Verified $false`（read-back failed、チェックリストで surface 用）（L225-231） |

Activate 側（`windows_license_auth.ps1`）は本変更の対象外で、`-Verified` 引数は引き続き未使用。

### `Get-MaskedKey` の挙動

```
入力:    XXXXX-XXXXX-XXXXX-XXXXX-Y2P3Z
出力:    *****-*****-*****-*****-Y2P3Z
```

- ハイフンは保持（5-5-5-5-5 layout が認識可能なまま）
- 末尾 5 文字のみ平文
- 5 文字以下の入力は length-only マスク（全部 `*`）にフォールバック

### Office 側との対比

`office_license_config` も v1.0.1 で同等のマスクヘルパを導入（同実装）。Windows License は `SoftwareLicensingService` CIM 経由で実装されているため ProductKey は CIM パラメータとして渡り、**Win32_Process.CommandLine には現れない**（Office の OSPP.vbs cscript 経路が抱える構造的制約は本モジュールには無い）。詳細は [fabriq__modules__office_license_config.md](fabriq__modules__office_license_config.md) の Security Note セクションを参照。

## 入力 (CSV)
`license_key.csv`（Install 側で使用）
- `Enabled`: 1=実行 / 0=スキップ
- `ProductKey`: プロダクトキー (`XXXXX-XXXXX-XXXXX-XXXXX-XXXXX`)
- `Description`: 説明（例: "Windows 11 Pro Volume License"）
- `Segment`: Segment フィルタ（任意）

CSV にキーが無い・無効なら手動入力にフォールバック。Activate 側は CSV を持たない。

## 主要ステップ
**[Install Product Key]**
1. `license_key.csv` からキー読み込み（無ければ手動入力）
2. キー形式検証（`XXXXX-XXXXX-XXXXX-XXXXX-XXXXX`、不一致なら Error）
3. 現在のライセンス状態を `SoftwareLicensingProduct` から取得・表示
4. 実行確認（AutoPilot 自動 Y）
5. 既存キーがあれば `UninstallProductKey`（失敗しても警告のみで続行）→ `SoftwareLicensingService.InstallProductKey` でキー投入。投入で例外が出れば Error 返却
6. 投入キー末尾 5 桁を `PartialProductKey` と照合（2 秒 × 最大 3 回リトライ）→ 一致なら `Success` / `-Verified $true`、Windows 製品が読めて不一致なら `Error`、製品が読めなければ `Success` / `-Verified $false` を `New-ModuleResult` で返却

**[Activate Windows License]**
1. `SoftwareLicensingProduct` で現状表示
2. 冪等性チェック（`LicenseStatus=1`（Licensed）なら Skip）
3. 実行確認（AutoPilot 自動 Y）
4. `SoftwareLicensingService.RefreshLicenseStatus` 起動
5. 3 秒待機 → `SoftwareLicensingProduct` 再取得 → `LicenseStatus` 確認 →
   1 なら Success、それ以外なら Error

## 注意点・運用メモ
- 管理者権限必須（両スクリプトとも）
- Activate はインターネット接続必須。プロキシ環境では `slmgr.vbs /skms` 等の事前設定が必要
- Volume License (KMS) の場合は KMS サーバ到達性が前提
- License key は CSV 上は平文（Sysprep 用 `unattend.xml` 内の Key も同様）。
  公開リポジトリで誤コミットしないよう運用注意（ENC: 対応は本モジュールでは未実装）
- Order=91/92 で並んでおり、Install → Activate の順で Profile に組み込む想定
- Activate Skip 時のメッセージで「Already Licensed」を明示するため運用ログ上は分かりやすい

## 検証
- **Install Product Key**: Step 6 が Post-Apply Verification（v1.1.0 で追加）。
  キー投入後に `SoftwareLicensingProduct` を再取得し、**投入キー末尾 5 桁を
  `PartialProductKey` と照合**する（2 秒待機 × 最大 3 回リトライ、`windows_license_install.ps1` L186-198）。
  単に「Windows 製品が `PartialProductKey` を持つ」だけでは旧キー残留と区別できないため、
  末尾 5 桁の一致を確認する。結果に応じ `New-ModuleResult` の `-Verified` を返却する:
  - 一致 → `Success` / `-Verified $true`（L212）
  - Windows 製品は読めるが末尾 5 桁不一致（旧キー残留）→ `Error`（fail closed、L222）
  - Windows ライセンス製品が読めない → `Success` / `-Verified $false`（read-back failed、L231）
- **Activate Windows License**: Step 5 が事実上の Post-Apply Verification。
  `RefreshLicenseStatus` 呼び出し後 3 秒待機 → `SoftwareLicensingProduct` を再取得して
  `LicenseStatus=1` を確認する（`windows_license_auth.ps1` L96-131）。ただし Activate 側は
  `New-ModuleResult` の `-Verified` パラメータを使用しておらず、履歴の Verified 列は空欄。
