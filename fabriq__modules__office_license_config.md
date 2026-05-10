# office_license_config (Standard)

> **対象**: fabriq / modules/standard/office_license_config
> **対象バージョン**: モジュール 1.0.1 / kernel 3.2.5（取得元: `E:\fabriq\modules\standard\office_license_config\VERSION` / `E:\fabriq\kernel\KERNEL_VERSION`、commit `fed181a`、2026-05-10）
> **ドキュメント更新日**: 2026-05-10

**カテゴリ**: Security
**メニュー名**: Install Office Product Key / Activate Office License
**VERSION**: 1.0.1  / **REQUIRES_KERNEL**: 2.0.0
**Post-Apply Verification**: install=なし／auth=スクリプト内で `/dstatus` 再解析（事実上の検証、`-Verified` は未渡し）
**サブスクリプト**: `office_license_install.ps1`（プロダクトキー登録）, `office_license_auth.ps1`（ライセンス認証）

## 目的
Microsoft Office のプロダクトキー登録（`OSPP.vbs /inpkey`）と
ライセンス認証（`OSPP.vbs /act`）を行う 2 スクリプト構成のモジュール。
OSPP.vbs パスは C2R 64/32bit、MSI 64/32bit の 4 候補を優先順で自動検出し、
特殊環境向けに CSV から OsppPath を強制指定可能なハイブリッド方式です。
ENC: プレフィクスによる暗号化キーの保管に対応し、複数製品（Office 本体 + Visio +
Project）を 1 度の認証で一括処理します。

## v1.0.1 の変更（セキュリティ強化、v1.0.0 からの差分）

### 変更 1: ProductKey の transcript マスク表示（A3-T1-A、2 箇所）

| 場所 | v1.0.0 | v1.0.1 |
|---|---|---|
| L118 事前一覧表示 | `Write-Host "    Key:  $($item.ProductKey)"` 生表示 | `Get-MaskedKey` 経由マスク表示 |
| L177 形式不正時 | `Show-Skip "Invalid key format: $($item.ProductKey)"` | 生キー値を除去、形式テンプレのみ表示 |

`Get-MaskedKey` は script 先頭にローカル定義（windows_license_config v1.0.1 と同実装、末尾 5 文字のみ可視化・dash 維持）。

### 変更 2: Guide.txt に Security Note を新設（cscript の構造的制約明記）

`office_license_install.ps1` の `cscript OSPP.vbs /inpkey:KEY` は ProductKey を **cscript.exe の起動引数として OS layer に展開** するため、以下のログ機構に記録される:

- `Win32_Process.CommandLine`（cscript.exe の lifetime 中、live snapshot）
- Windows Security Log Event ID 4688（`Audit Process Creation` + `IncludeCommandLine` 有効環境で永続記録 → SIEM 転送）
- PowerShell Module Logging（有効環境）

これは fabriq 側で **回避できない Microsoft プラットフォームの構造的制約**:

- OSPP.vbs は `WScript.Arguments` 経由で `/inpkey:KEY` を positional 引数として受け付ける設計
- stdin / file / 環境変数経由の代替インターフェイスが存在しない
- Office C2R 2019/2021/365 では `OfficeSoftwareProtectionService` WMI provider が deprecated/削除済で、CIM 直叩きの clean 経路が存在しない（MSI Office 2016 以前なら CIM 経由可能だが、現代の B2G 環境ではほぼ C2R）
- 自前 wrapper VBS / `Process.Start` native API 等の代替案も、最終的に OSPP.vbs に `/inpkey:` を引数として渡す経路を回避できない

**B2G / 厳格監査環境での運用推奨**（Audit Process Creation + IncludeCommandLine が site policy で強制有効、SIEM 転送ありの環境）:

1. **Office Customization Tool (OCT)** で生成した XML 設定に ProductKey を埋め込み、Office インストール時に同時に key 登録（install 時 1 回の暴露で post-install の再 key install 不要に）
2. **Microsoft Volume Licensing Service Center で AD 連携 KMS / ADBA** を構成し、ProductKey の手動登録を完全に省略
3. 上記が不可能な場合、本モジュールの実行時間帯のみ `IncludeCommandLine` を一時的に無効化（GPO ロールバック付き）

### fabriq の他モジュールとの対比

| モジュール | ProductKey/秘密の OS-layer 露出 | 対策状況 |
|---|---|---|
| **windows_license_config** v1.0.1+ | 露出なし（`SoftwareLicensingService` CIM 経由、ProductKey は CIM パラメータ） | 構造的に不要 |
| **robocopy_config** v1.0.1+ | UNC AuthPass を stdin pipe 経由で net.exe に渡し回避（v1.0.0 では暴露） | 修正済（kernel 3.2.5 期） |
| **office_license_config** | OSPP.vbs `/inpkey:` の構造的制約により対策不可 | **accept & document**（本セクション） |

詳細は `project_crypto_security_review.md` メモリの A3 T1-B office セクションを参照。Microsoft platform 改善（Office C2R に新しい license API 追加 等）があれば再評価対象。

公開 API 不変、`REQUIRES_KERNEL` 据え置き 2.0.0、CSV スキーマ不変。

## 入力 (CSV)
`office_key.csv`:
- `Enabled`: 有効フラグ
- `ProductKey`: `XXXXX-XXXXX-XXXXX-XXXXX-XXXXX` 形式（ENC: プレフィクス対応）
- `ActivationType`: `MAK` / `KMS`（疎通チェック分岐に使用）
- `OsppPath`: OSPP.vbs フルパス（空欄=自動検出）
- `Description`, `Segment`

## 主要ステップ（install）
1. 管理者権限チェック → 失敗で Error 終了
2. `office_key.csv` 読み込み（Enabled=1 のみ）
3. OSPP.vbs 自動検出。失敗時に CSV の OsppPath が皆無なら Error 終了
4. ドライラン表示（[APPLY] / [INVALID KEY] / [ENC ERROR] / [OSPP NOT FOUND] でマーキング）
5. 実行確認（AutoPilot は自動 Y）
6. 各キーごとに ENC 残存ガード → 形式バリデーション → `cscript //Nologo OSPP.vbs /inpkey:<KEY>` 実行
7. `New-BatchResult` で集計

## 主要ステップ（auth）
1. 管理者権限チェック
2. OSPP.vbs 検出
3. CSV から ActivationType を判定し MAK の場合のみ疎通チェック
   - 第一段: `Test-NetConnection activation.sls.microsoft.com:443`
   - 失敗時: `Test-Connection 8.8.8.8` フォールバック（ping 成功なら Warning で続行＝プロキシ環境想定）
   - 両方失敗で Error 中止
4. `/dstatus` で現状ステータス取得 + 全製品の LICENSE STATUS 解析
5. 全製品が LICENSED なら冪等 Skip（複数製品ロバスト判定）
6. 実行確認 → `cscript /act` 実行
7. 3 秒待機後 `/dstatus` 再実行 → 検証 → Status / Partial / Error 判定

## 注意点・運用メモ
- 管理者権限必須
- MAK: インターネット必須（プロキシ環境は ping フォールバックで継続判断）
- KMS: 社内 KMS サーバー到達性のみで OK、180 日サイクル自動再認証
- ENC: プレフィクスのまま到達した場合は passphrase 未入力検出として Error 計上
- `office_license_install.ps1` の `Find-OsppVbs` 関数は両スクリプトでローカル定義
  （`_office_common.ps1` に切り出す案は guide にコメントあるが現状未統合）

## 検証
- install: 検証は実装せず、cscript の ExitCode のみで判定。`-Verified` は未渡し
- auth: スクリプト内で `/dstatus` を再実行し全製品の LICENSE STATUS を再解析、
  Status / Message で結果を表現するが `New-ModuleResult -Verified` は未使用。
  事実上の検証はスクリプト内で完結しており、
  Verified カラム連携が必要な場合は今後 `-Verified` 引数追加で対応可能
