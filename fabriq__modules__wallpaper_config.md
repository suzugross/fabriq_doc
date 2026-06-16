# wallpaper_config (Standard)

> **対象**: fabriq / modules / wallpaper_config
> **対象バージョン**: kernel 3.6.0（取得元: `E:\fabriq\kernel\KERNEL_VERSION` / commit 0fca159）/ モジュール 1.2.0（取得元: `E:\fabriq\modules\standard\wallpaper_config\VERSION`、REQUIRES_KERNEL 3.5.0）
> **ドキュメント更新日**: 2026-06-16

**カテゴリ**: Desktop
**メニュー名**: Wallpaper Config
**VERSION**: 1.2.0  / **REQUIRES_KERNEL**: 3.5.0
**Post-Apply Verification**: あり（レジストリ読み返しで検証し `New-BatchResult -Verified` を渡す）
**サブスクリプト**: なし（`SystemParametersInfo` を `Add-Type` で同居）

## 目的
デスクトップ壁紙を画像または単色 (SolidColor) で設定するモジュール。
キッティング向けに堅牢化されており、画像 (Type=Image) は適用前に必ずローカルの固定フォルダ
`C:\Windows\Web\Wallpaper\fabriq` へ SHA256 冪等コピーし、レジストリ / `SystemParametersInfo` /
Active Setup にはコピー先のローカル絶対パスのみを書き込む。これにより USB メモリから実行しても
USB 取り外し・再起動・ドライブレター変更で壁紙が消えない。
Windows API (`SystemParametersInfo SPI_SETDESKWALLPAPER`) で現在ユーザー分は即時反映するため再起動不要。
画像壁紙では Style（Fill / Fit / Stretch / Tile / Center / Span）を指定でき、
SolidColor では RGB 値で単色背景を直接適用する。`Resolve-HkcuRoot` 経由で SYSTEM 起動時にも
ログオンユーザーの HKCU ハイブにリダイレクトして書き込む昇格対応。
キッティング後に作成される新規ユーザーには Active Setup（初回ログオン1回）で反映する。

## 入力 (CSV)
`wallpaper_list.csv`
- `Enabled`: 1=実行 / 0=スキップ
- `Type`: `Image` / `SolidColor`（省略時 Image）
- `FileName`: 画像ファイル名（`wallpaper/` 配下相対パス）または絶対パス（Type=Image 時）
- `Style`: `Fill` / `Fit` / `Stretch` / `Tile` / `Center` / `Span`（省略時 Fill）
- `Color`: RGB 値スペース区切り（例 `0 99 177`、Type=SolidColor 時）
- `Description`: 説明
- `Segment`: Segment フィルタ（任意）

サンプル: green.jpg (Image, Fill) / Navy Blue (SolidColor, RGB 0 99 177) いずれも Enabled=0。
画像対応形式: jpg, jpeg, png, bmp, gif, tif, tiff。

## 主要ステップ
1. `wallpaper_list.csv` 読み込み（`Import-ModuleCsv -FilterEnabled`）
2. 画像ファイル存在 + 形式確認、または RGB 値妥当性確認 → 一覧表示
   （同名画像が複数あれば「後勝ち」警告）
3. 実行確認（AutoPilot 自動 Y）
4. **Desktop Spotlight 無効化**（壁紙適用の前）: 現在/次回ログオンユーザー HKCU と、
   既定プロファイル `C:\Users\Default\ntuser.dat` を一時 reg load した hive の両方に
   `...\DesktopSpotlight\Settings\EnabledState`(REG_DWORD)=0 を書き込み、書込後に読み返して検証 → hive unload
5. Type 別適用（各項目）:
   - **Image**: `Resolve-WallpaperLocalDest`（`Test-FabriqSafePathComponent` でリーフ名ガード）→
     `C:\Windows\Web\Wallpaper\fabriq` へ SHA256 冪等コピー →
     `Control Panel\Desktop\WallpaperStyle` / `TileWallpaper` レジストリ書き込み →
     非リダイレクト時は `SystemParametersInfo(SPI_SETDESKWALLPAPER, ..., destPath, SPIF_UPDATEINIFILE | SPIF_SENDCHANGE)`
     でローカルコピー先パスを即時反映（リダイレクト時は `WallPaper` 値をステージのみ）
   - **SolidColor**: `WallPaper` をクリア + `Control Panel\Colors\Background` に
     `R G B` 文字列を書き込み + 非リダイレクト時は `SystemParametersInfo` トリガー
   - 各項目の適用後に **Post-Apply Verification**（レジストリ読み返し、後述「検証」節）
6. **新規ユーザー反映（Active Setup）**: 最後に適用された最終状態 (`$finalState`) を初回ログオンで
   再適用するスクリプト `wallpaper_default_apply.ps1` を `Register-FabriqActiveSetup` で配備・HKLM 登録
7. `New-BatchResult -Verified $verified` 返却（検証結果を Verified 列に反映）

## 注意点・運用メモ
- 管理者権限必須（`Test-AdminPrivilege` で失敗時 fail-closed）。理由は `C:\Windows` 配下への書き込み、
  HKLM の Active Setup 登録、既定プロファイルハイブの `reg load` のため。`Resolve-HkcuRoot` で
  SYSTEM/別ユーザー起動時はログオンユーザーの HKCU ハイブへリダイレクト書き込み（次回ログオンで反映）。
- `wallpaper/` ディレクトリは module ルート配下で、画像をここに置けば相対パス指定可能。
  絶対パスでも OK。いずれも適用前に `C:\Windows\Web\Wallpaper\fabriq` へコピーしてから参照する。
- ローカルコピーは SHA256 冪等（コピー先に同名・同内容のファイルがあればスキップ）。同名で内容が異なる
  画像を複数指定すると後勝ちになるため、異なる画像には異なるファイル名を付ける（事前に警告も出る）。
- Style 省略時のデフォルトは Fill
- SolidColor 時は FileName / Style は無視（SolidColor はローカルコピー対象外）
- 新規ユーザー（キッティング後に作成されるユーザー）には Active Setup で反映する。既定プロファイル
  hive の `Control Panel\Desktop\WallPaper` を書いても Windows 10/11 ではテーマエンジンが初回ログオン時に
  上書きするため、壁紙パスそのものは Default hive ではなく Active Setup（テーマ適用後に走行）で反映する。
  なお Desktop Spotlight の `EnabledState` はテーマ管理外の通常設定のため、こちらは Default hive 書き込みで
  新規ユーザーへ継承される。

## 検証
Post-Apply Verification を実装している。レジストリ読み返しヘルパー `Test-RegStringEquals`（REG_SZ）/
`Test-RegDwordEquals`（REG_DWORD）で期待値との一致を確認し、その結果を集計して
`New-BatchResult -Verified` に渡す。検証対象は以下のとおり。

- **Desktop Spotlight**: 現在/次回ログオンユーザー HKCU と既定 hive それぞれで
  `...\DesktopSpotlight\Settings\EnabledState` = 0 を読み返し確認。いずれか失敗で `$spotlightDegraded`。
- **壁紙（各項目）**:
  - Image: ローカルコピー先 `destPath` の存在 + `Control Panel\Desktop\WallPaper` が `destPath` と一致。
  - SolidColor: `Control Panel\Desktop\WallPaper` が空 + `Control Panel\Colors\Background` が指定 RGB と一致。
  - 失敗があれば `$verifyFail` を計上。
- **新規ユーザー Active Setup**: HKLM の Active Setup キーと配備スクリプト
  `C:\ProgramData\fabriq\wallpaper_default_apply.ps1` の存在を確認。失敗で `$newUserDegraded`。

最終的な `-Verified` は
`-not $spotlightDegraded -and -not $hiveUnloadFailed -and $verifyFail -eq 0 -and -not $newUserDegraded`
で決まる。既定 hive のアンロードに失敗した場合（`$hiveUnloadFailed`）は ntuser.dat がロックされたまま
残り後続の sysprep / 新規プロファイル作成に影響しうるため、Fail を 1 加算したうえで `-Verified $false`
を明示的に返す。SolidColor / Image いずれの適用ループでも、戻り値判定（`SystemParametersInfo` の
戻り値や例外）に加えてレジストリ読み返しによる検証を行う。
