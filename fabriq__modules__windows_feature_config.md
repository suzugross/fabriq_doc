# windows_feature_config (Standard)

> **対象**: fabriq / modules/standard/windows_feature_config
> **対象バージョン**: モジュール 0.1.0 / kernel 3.2.5（取得元: `E:\fabriq\modules\standard\windows_feature_config\VERSION` / `E:\fabriq\kernel\KERNEL_VERSION`、commit `fed181a`、2026-05-10）
> **ドキュメント更新日**: 2026-05-10

**カテゴリ**: System
**メニュー名**: Windows Feature Config
**VERSION**: 0.1.0  / **REQUIRES_KERNEL**: 2.0.0
**Post-Apply Verification**: 実装あり（`Get-WindowsOptionalFeature` で State を読み戻し、目標 state または pending state なら PASS）
**サブスクリプト**: なし

## 目的
Windows のオプション機能（Optional Features）を CSV マニフェストに従って **オンライン** で有効化／無効化するモジュール。`Enable-WindowsOptionalFeature` / `Disable-WindowsOptionalFeature` の薄いラッパで、**稼働中 OS のみ対応**（オフライン WIM への注入は scope 外で、tonebender 等のイメージ事前準備側に委ねる）。

代表的な使い方:
- `.NET Framework 3.5`（NetFx3、payload removed）の有効化（モジュール同梱 sxs / マウント済み Windows ISO / WU 経由の 3 系統サポート）
- 官公庁ハードニング向け `SMB1Protocol` / `WindowsMediaPlayer` / 旧 IE の無効化
- 運用機能 `TelnetClient` / `TFTP` / `Hyper-V` / `IIS` 系の有効化

姉妹モジュール: Windows Server の役割・機能は [fabriq__modules__server_feature_config.md](fabriq__modules__server_feature_config.md) へ分離。

## 入力 (CSV)
`windows_feature_list.csv` の主な列:
- `Enabled`: 1=実行 / 0=スキップ
- `Action`: `Enable` / `Disable`（それ以外は `[INVALID]` でスキップ）
- `FeatureName`: DISM 上の機能名（**大文字小文字を区別**。`Get-WindowsOptionalFeature -Online` で列挙可能）
- `IncludeAllSubFeatures`: 1 で `-All` 付与（Enable 時のみ意味あり、子機能を一括有効化）
- `Source`: payload removed 機能（NetFx3 等）向けの sxs パス（5 形式受容、後述）
- `LimitAccess`: Source 設定時のみ意味あり。1=WU フォールバック禁止（閉域安全側）/ 0=許可。**空欄時は 1（default）**
- `Description`, `Segment`

### Source 列の 5 形式

| 形式 | 解釈 | 例 |
|---|---|---|
| ドライブレター絶対 | そのまま使用 | `D:\sources\sxs` |
| UNC 絶対 | そのまま使用 | `\\nas\share\sxs` |
| `/payload/...` | モジュール内 `payload/` 配下 | `/payload/dotnetfx35` |
| `\payload\...` | 同上（Windows 流の `\` 起点） | `\payload\dotnetfx35` |
| `payload\...` | 同上（leading separator なし） | `payload\dotnetfx35` |
| 空欄 | `-Source` を渡さない（WU 経由） | (空) |

### `payload/` ディレクトリ
モジュール直下に `payload/dotnetfx35/` 等の空テンプレディレクトリが含まれる。site 側で sxs 配下の `*.cab` を staging する場所。**robocopy `/E` でのオーバーレイ更新が site に展開済みの cab を上書き削除しない**ため、site staging は更新後も保持される設計。

NetFx3 用 sxs の取得は「キッティング対象と同 build の Windows ISO の `<mounted>\sources\sxs\` を `payload\dotnetfx35\` にコピー」する手順が Guide.txt に記載（メジャーまたぎは確実 NG、同メジャー内 build 違いは原則可だが LCU 段差で稀に fail）。

## 主要ステップ（7 ステップ）

| Step | 内容 |
|---|---|
| 1 | `windows_feature_list.csv` 読み込み（Enabled=1 のみ） |
| 2 | 管理者権限チェック（`Test-AdminPrivilege`） |
| 3 | ドライラン（`Get-WindowsOptionalFeature` で現 State 取得 + Source パス事前検証） |
| 4 | 実行確認（AutoPilot は自動 Y） |
| 5 | DISM cmdlet 適用ループ（`Enable-WindowsOptionalFeature` / `Disable-WindowsOptionalFeature`、常に `-NoRestart`） |
| 5.5 | **Post-Apply Verification**（State を読み戻して目標値到達を確認） |
| 6 | `New-BatchResult -Verified` で集計返却 |

### ドライラン（Step 3）のマーカー

| マーカー | 意味 | 色 |
|---|---|---|
| `[APPLY]` | 変更予定 | Yellow |
| `[Current]` | 既に目標 state、Step 5 で skip | Gray |
| `[NOT FOUND]` | DISM が認識しない FeatureName | Red |
| `[NEEDS SOURCE OR WU]` | payload removed 状態 + Source 未指定（WU フォールバック予告） | Magenta |
| `[BAD SOURCE]` | Source パスが存在しない or `*.cab` 不在 | Red |
| `[INVALID]` | Action が Enable/Disable 以外 | Red |

Source パスは `Test-FeatureSourcePath` で「ディレクトリ存在 + `*.cab` 1 件以上」を事前検査する。

### 適用ループ（Step 5）

- `Enable`: `Enable-WindowsOptionalFeature -Online -FeatureName <name> -NoRestart [-All] [-Source <path>] [-LimitAccess]`
- `Disable`: `Disable-WindowsOptionalFeature -Online -FeatureName <name> -NoRestart`
- `RestartNeeded=True` 戻り値はログのみ（**再起動は profile 側 `__RESTART__` の責務**、本モジュールは決して再起動しない）
- 既に目標 state なら冪等 Skip
- `Enable` で payload removed + Source 未指定なら `Show-Warning` で WU フォールバック予告

エラー時の hint: cmdlet が `source files could not be found` / 0x800F081F / 0x800F0954（日本語版「ソース ファイルを見つけることができません」も含む）を返した場合、Source パスの再確認・build の整合・WSUS バイパス（後述）の判断を促す追加 Warning を表示する。

## Post-Apply Verification（Step 5.5）

`Get-WindowsOptionalFeature` を再実行し、State が目標値（`Enable` → `Enabled`/`EnablePending` / `Disable` → `Disabled`/`DisablePending`/`DisabledWithPayloadRemoved`）に到達しているかを 1 行ごとに検証。

- **pending 状態は「適用が受理されたが再起動待ち」として PASS 扱い**（`hostname_config` と同方針）
- `verifyFail=0` かつ `verifyPass>0` で `New-BatchResult -Verified $true`
- 全行 PASS でない場合は `-Verified $false`

`Action` が Enable/Disable 以外（`[INVALID]`）の行は検証対象から除外。

## 注意点・運用メモ

- **管理者権限必須**
- 常に `-NoRestart` 付与で**自身で再起動はしない**。NetFx3 は通常 `RestartNeeded=False`、IIS / Hyper-V は True を返すことが多い
- `FeatureName` は DISM の表記に**厳密一致**（大文字小文字含む）
- 複数行で同じ FeatureName を指定可能だが、Step 5.5 検証は最後に評価された state のみ参照するため結果が紛らわしい。**重複は避ける**
- `LimitAccess` の default は 1（閉域安全側）。開放網で WU フォールバックを許容する場合は明示的に 0
- Source 列はモジュール相対形式（`/payload/...`）が**強く推奨**。USB/NAS のドライブレターはキッティング環境ごとに変わるため、移植性のため

### WSUS 環境での WU 経路許可（補助情報）

WSUS が GPO 構成されている環境では、Source 空欄での WU 経由 FoD 取得が **0x800F0954** でブロックされる。バイパスにはレジストリ設定:

```
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\Servicing
  RepairContentServerSource = 2 (DWORD)
```

**本モジュールはこのレジストリを触らない**。必要時は `reg_hklm_config` に行追加して事前適用（関心の分離）。

## 検証
本モジュールは Verification を **完全実装** している数少ないモジュールの 1 つ。`Get-WindowsOptionalFeature` で読み戻した State を `Test-FeatureStateApplied` で目標値判定し、pending を含め PASS 集計。`-Verified` を `New-BatchResult` に渡すため、実行履歴の Verified 列に PASS/FAIL が記録される。

冪等性: 各行の dry-run と実行ループの両方で、現 State が既に目標値である場合は `[Current]` / `Show-Skip`。再実行しても DISM 呼び出しは発生しない。
