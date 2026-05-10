# server_feature_config (Extended)

> **対象**: fabriq / modules/extended/server_feature_config
> **対象バージョン**: モジュール 0.1.0 / kernel 3.2.5（取得元: `E:\fabriq\modules\extended\server_feature_config\VERSION` / `E:\fabriq\kernel\KERNEL_VERSION`、commit `fed181a`、2026-05-10）
> **ドキュメント更新日**: 2026-05-10

**カテゴリ**: System
**メニュー名**: Server Feature Install
**VERSION**: 0.1.0  / **REQUIRES_KERNEL**: 2.0.0
**Post-Apply Verification**: 実装あり（`Get-WindowsFeature` で `InstallState` を読み戻し、`Installed` または `InstallPending` なら PASS）
**サブスクリプト**: なし

## 目的
Windows Server の **役割・役割サービス・機能** を CSV マニフェストに従ってオンラインインストールするモジュール。`Install-WindowsFeature`（ServerManager モジュール）の薄いラッパで、稼働中 Windows Server に対するオンライン操作のみを行う（オフライン VHD への注入や `ConfigurationFilePath` ベースのウィザード取り込みは scope 外）。

代表的な使い方:
- ファイル サーバーへの `FS-FileServer` / `FS-DFS-Namespace` 追加
- ドメイン コントローラ昇格前の `AD-Domain-Services` + `DNS` インストール
- メンバー サーバー / 管理用ホストへの `RSAT-AD-Tools` / `GPMC` 投入
- IIS Web サーバー `Web-Server` 一式 + 管理ツール

姉妹モジュール: クライアント OS の機能（NetFx3 / SMB1Protocol / TelnetClient 等）は [fabriq__modules__windows_feature_config.md](fabriq__modules__windows_feature_config.md) を使用。

### クライアント OS では Skipped 返却（混在フリート対応）

`Win32_OperatingSystem.ProductType` で OS 種別を判定し、ProductType=1（Workstation / クライアント SKU）では **何もせず `Status=Skipped` を返す**（ServerManager モジュールが非搭載のため）。混在フリート向けの共通プロファイルに本モジュールを含めても、クライアントでは安全にスキップされる設計。

ProductType の意味:
- 1: Workstation（クライアント SKU）→ Skipped
- 2: Domain Controller → 実行
- 3: Server（メンバーサーバー）→ 実行

## 入力 (CSV)
`server_feature_list.csv` の主な列:
- `Enabled`: 1=実行 / 0=スキップ
- `Name`: ServerManager 上の機能名（**大文字小文字を区別**。`Get-WindowsFeature` で列挙可能）
- `IncludeAllSubFeature`: 1 で `-IncludeAllSubFeature` 付与（子役割サービス・子機能を一括インストール）
- `IncludeManagementTools`: 1 で `-IncludeManagementTools` 付与。**`Install-WindowsFeature` は default では管理ツールを導入しない**ため、運用後 GUI 管理ツールを使う役割では明示 1 を推奨
- `Source`: feature files が見つからない場合の媒体パス（任意、空欄 = ローカルストア → WU フォールバック）
- `Description`, `Segment`

### Source 列の形式（windows_feature_config と同一規約）

| 形式 | 解釈 | 例 |
|---|---|---|
| ドライブレター絶対 | そのまま使用 | `D:\sources\sxs` |
| UNC 絶対 | そのまま使用 | `\\fileserver\sources\sxs` |
| `/relative/...` | モジュール相対パス | `/payload/winsxs` |
| `\relative\...` | 同上（`\` 起点） | `\payload\winsxs` |
| `relative\...` | 同上（leading separator なし） | `payload\winsxs` |
| 空欄 | `-Source` を渡さない（ローカルストア → WU） | (空) |

通常は対応する Windows Server インストール メディアの `sources\sxs\` フォルダ、または UNC 共有上の同等パスを指す。クライアント OS の DISM とは異なり cab 単位の運用ではないため、本モジュールは **ディレクトリ存在チェックのみ**（ファイル列挙バリデーションは省略）。

## 主要ステップ（6 ステップ）

| Step | 内容 |
|---|---|
| 1 | `server_feature_list.csv` 読み込み（Enabled=1 のみ） |
| 2 | 前提チェック: 管理者権限 / ProductType ∈ {2, 3} / `Import-Module ServerManager` 成功 |
| 3 | ドライラン（`Get-WindowsFeature` で現 InstallState 取得 + Source パス事前解決） |
| 4 | 実行確認（AutoPilot は自動 Y） |
| 5 | `Install-WindowsFeature` 適用ループ（`-Restart` は付与しない） |
| 5.5 | **Post-Apply Verification**（InstallState を読み戻して PASS 判定） |
| 6 | `New-BatchResult -Verified -MessageSuffix "(restart required for N item(s))"` で集計返却 |

### ドライラン（Step 3）のマーカー

| マーカー | 意味 | 色 |
|---|---|---|
| `[INSTALL]` | 新規インストール予定 | Yellow |
| `[ALREADY-INSTALLED]` | 既に Installed、Step 5 で skip | Gray |
| `[PENDING-RESTART]` | 既に InstallPending（再起動待ち）、skip | DarkYellow |
| `[NOT FOUND]` | ServerManager が認識しない Name | Red |
| `[NEEDS SOURCE OR WU]` | Removed 状態 + Source 未指定（WU フォールバック予告） | Magenta |

### 適用ループ（Step 5）

- パラメータ組み立て:
  ```powershell
  Install-WindowsFeature -Name <Name> [-IncludeAllSubFeature] [-IncludeManagementTools] [-Source <path>]
  ```
- **`-Restart` は意図的に付与しない**（reboot は profile `__RESTART__` で orchestrate、AutoPilot フローを deterministic に保つため）
- cmdlet 戻り値 `.Success=$false` も failure 扱いで集計（Try/Catch 例外でなくても拾う）
- `RestartNeeded='Yes'` をログ + サマリで通知し、最終 `New-BatchResult` の MessageSuffix に件数を埋める
- 既に Installed / InstallPending なら冪等 Skip

エラー時の hint: cmdlet が `source files could not be found` / 0x800F081F / 0x800F0954（日本語版「ソース ファイルを見つけることができません」も含む）を返した場合、Source パスや Windows Server build の整合確認を促す Warning を表示。WSUS バイパス設定（`RepairContentServerSource=2`）は windows_feature_config の Guide と同じレジストリで対応可能。

## Post-Apply Verification（Step 5.5）

`Get-WindowsFeature` を再実行し、`InstallState` が `Installed` または `InstallPending` に到達しているかを 1 行ごとに検証。

- **pending 状態は「適用が受理されたが再起動待ち」として PASS 扱い**（windows_feature_config / hostname_config と同方針）
- `verifyFail=0` かつ `verifyPass>0` で `New-BatchResult -Verified $true`

## Studio での編集 UX（preset.csv）

`preset.csv` により Studio の `_list.csv` 編集 UI で以下がドロップダウン化:

- `Enabled`: 有効 / 無効
- `IncludeAllSubFeature`: サブ機能を含める / メイン機能のみ
- `IncludeManagementTools`: 管理ツールを含める / 含めない
- `Name`: **主要 32 機能の curated 一覧**（役割 16 + 機能 8 + RSAT 8）+ ラベル

curated 一覧に含まれない機能名（`Web-Mgmt-Console` / `Web-Asp-Net45` 等の IIS 子役割サービス、`Storage-Replica` 等のニッチ機能）は Studio 側で直接タイプ入力するか、`preset.csv` に追記する（encoding は UTF-8 BOM + CRLF）。正確な機能名一覧は対象 Server 上で次のコマンドで確認可能:

```powershell
Get-WindowsFeature | Where-Object Installed -eq $false | Select Name, DisplayName
```

## 注意点・運用メモ

- **管理者権限必須**、Windows Server SKU 必須（クライアントでは Skipped）
- **本モジュールは決して再起動しない**（profile `__RESTART__` で制御）
- `Name` は ServerManager の表記に**厳密一致**（大文字小文字含む）
- 複数行で同じ Name 指定は可能だが、Step 5.5 検証は最後に評価された state のみ参照するため結果が紛らわしい。**重複は避ける**
- `IncludeManagementTools` は default で 0。これは `Install-WindowsFeature` cmdlet 自体の仕様で、Server Manager GUI からの追加と異なり PowerShell 経由では管理ツールが default で入らない。GUI 管理ツールを使うなら 1 を明示
- Source 列のパスはキッティング環境ごとに UNC 到達性が変わるため、AutoPilot 運用では事前 staging 状態の確認を CSV 設計時点で確定する
- offline VHD はスコープ外（tonebender 等のイメージ事前準備側に委ねる）

## 検証
windows_feature_config と並んで Verification を **完全実装** している。`Get-WindowsFeature` で読み戻した `InstallState` を `Test-InstallStateApplied`（pending を PASS 扱い）で判定し、`-Verified` を `New-BatchResult` に渡すため実行履歴の Verified 列に PASS/FAIL が記録される。

冪等性: 各行の dry-run と実行ループの両方で、`Installed` / `InstallPending` は `[ALREADY-INSTALLED]` / `[PENDING-RESTART]` / `Show-Skip` 扱い。再実行しても `Install-WindowsFeature` 呼び出しは発生しない。
