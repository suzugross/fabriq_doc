# driver_config (Standard)

> **対象**: fabriq / modules/standard/driver_config
> **対象バージョン**: モジュール 1.0.1 / kernel 3.2.5（取得元: `E:\fabriq\modules\standard\driver_config\VERSION` / `E:\fabriq\kernel\KERNEL_VERSION`、commit `fed181a`、2026-05-10）
> **ドキュメント更新日**: 2026-05-10

**カテゴリ**: System
**メニュー名**: Driver Export / Driver Import
**VERSION**: 1.0.1  / **REQUIRES_KERNEL**: 2.0.0
**Post-Apply Verification**: なし（限定的な検査として `*.inf` カウントと `pnputil` ExitCode は判定）
**サブスクリプト**:
- `driver_export_config.ps1` … `dism.exe /online /export-driver` で現 OS のサードパーティドライバを `driver/{model}/` に書き出し（v1.0.1 で `Export-WindowsDriver` cmdlet から `dism.exe` 直叩きに変更）
- `driver_import_config.ps1` … `pnputil /add-driver "*.inf" /subdirs /install` で対象 PC に展開

## v1.0.1 の変更（v1.0.0 からの差分）

エクスポート実装を **`Export-WindowsDriver` cmdlet → `dism.exe /online /export-driver` 直接呼び出し** に変更。

**理由**: `Export-WindowsDriver` PowerShell cmdlet は **Windows Server 2022 で「SafeHandle を Null にすることはできません」エラー**を発生させる既知の不具合がある。同一の DismApi を直接呼ぶ `dism.exe` で迂回。

| 観点 | v1.0.0 | v1.0.1 |
|---|---|---|
| Export 実装 | `Export-WindowsDriver -Online -Destination ...` | `dism.exe /online /export-driver /destination:"<path>"` |
| Server 2022 動作 | SafeHandle エラーで失敗 | 成功 |
| 出力フォルダ構造 | （cmdlet 版） | cmdlet 版と等価（同じ DismApi） |
| ExitCode 判定 | cmdlet 例外 | `$LASTEXITCODE != 0` で例外を throw、最後の意味のある出力行を error message に含める |

**運用への影響なし**（出力フォルダ構造が cmdlet 版と等価のため、Import 側は無変更）。

## 目的
キッティング工程で「マスター PC からドライバ一式をエクスポート → 同モデルの量産機にインポート」という典型ワークフローを CSV 1 行で表現するモジュールです。Export では現 OS にインストールされたサードパーティドライバ（OS 標準ドライバを除く）を `driver/{モデル名}/` 配下に展開し、Import では `pnputil` でそれを再投入します。`model` 列を空欄にすると **`Win32_ComputerSystem.Model` から自動取得** するため、ホスト名ベースではなく機種ベースで自動振り分けが可能です（例: ThinkPad X1 → `ThinkPad_X1` フォルダ）。

## 入力 (CSV)
`driver.csv` の主な列:
- `Enabled` … 1=実行 / 0=スキップ
- `Id` … 識別連番
- `model` … ドライバフォルダ名の明示指定（空欄ならホストの `Win32_ComputerSystem.Model` を自動採用、サニタイズ: 空白→`_`、`\/:*?"<>|` 除去、80 文字上限）
- `Segment` … Segment フィルタ

## 主要ステップ
（Export / Import で同形）
1. `driver.csv` 読み込み
2. Pre-flight: 管理者権限確認 / `driver/` ディレクトリ存在確認
3. Dry-run 表示
4. 実行確認（AutoPilot 時は自動 Y）
5. 適用ループ（Export: `dism.exe /online /export-driver /destination:"<path>"`、Import: `pnputil /add-driver`）
6. 結果集計

## 注意点・運用メモ
- **管理者権限必須**（両スクリプトとも）
- Export はサードパーティドライバのみ対象（OS 標準ドライバは含まれない）
- Export 時、既存の `driver/{model}/` フォルダは中身がクリアされてから再エクスポート（差分追記ではなく完全置換）
- Import 後に再起動が必要な場合あり（`pnputil` 終了コード 3010）
- Export 時はドライバ容量分のディスク空き容量が必要

## 検証
Post-Apply Verification は **限定的**。`-Verified` フラグは渡しません。
- Export: 出力先フォルダの `*.inf` 数をカウント表示し、0 件ならエラー扱い
- Import: 実行前後の `*.inf` カウント参照 + `pnputil` ExitCode（0/3010=Success）

OS に組み込まれたドライバのバージョン比較等の厳密検証は行いません。必要に応じ `pnputil /enum-drivers` で手動確認する運用です。
