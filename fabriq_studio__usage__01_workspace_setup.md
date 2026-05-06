# ワークスペースセットアップ手順

> **対象**: fabriq_studio / usage
> **対象バージョン**: commit `3897c6e`
> **ドキュメント更新日**: 2026-05-06

fabriq_studio をインストールしてから実際に編集を始めるまでの手順をまとめる。

---

## 前提条件

- **Windows 10 / 11**
- **.NET 8.0 ランタイム**（self-contained 配布版を使う場合は不要）
- 開発時は **.NET 8.0 SDK** + **Visual Studio 2022** 推奨

---

## 入手方法

### A. ビルド済み配布版（self-contained）を使う

`E:\publish_fabriq_studio\` などに配置された `FabriqStudio.exe` を起動するだけで動作する。

- `FabriqStudio.exe` 本体
- `registry_collection/catalog.json` — レジストリ辞書
- `template/template_fabriq/looper_template/` — Script Looper テンプレート
- `template/template_fabriq/fabriq/` — fabriq 本体テンプレート（**配布元で配置済みかを確認**）
- `config/workspace.json` — 起動後に自動生成されるワークスペース永続化ファイル

### B. ソースから自分でビルドする

```powershell
# クローン
git clone <repo-url>
cd fabriq_studio

# ビルド
dotnet build FabriqStudio.sln

# デバッグ実行
dotnet run --project FabriqStudio
```

ビルド時に `FabriqStudio.csproj` の `AfterTargets="Build"` ターゲットで `registry_collection/` と `template/` が出力ディレクトリにコピーされる。

#### 配布用パブリッシュ

```powershell
dotnet publish FabriqStudio/FabriqStudio.csproj `
    -c Release `
    -o "E:/publish_fabriq_studio" `
    --self-contained true `
    -r win-x64
```

注意点:

- **必ず `.csproj` を直接指定する**。`.sln` を対象にすると `-o` が無視され（`NETSDK1194`）、出力先が `<project>/bin/.../publish/` 既定に流れる
- **出力パスはスラッシュ表記**（`"E:/publish_fabriq_studio"`）で書く。bash 経由で実行するとバックスラッシュが食われて相対パス化し、`E:\fabriq_studio\publish_fabriq_studio` に誤出力される。PowerShell 直接実行なら `E:\...` でも可だが、シェル非依存のためスラッシュで統一する
- `AfterTargets="Publish"` で `registry_collection/` と `template/` が PublishDir にコピーされる

### C. fabriq 本体テンプレートの配置

「テンプレートから新規ワークスペースを作成」機能を使うには、fabriq フレームワーク本体を以下に配置する必要がある:

```
<exe ディレクトリ>/template/template_fabriq/fabriq/
├── kernel/
├── modules/
├── profiles/
├── commands/
├── evidence/
├── Fabriq.bat
└── Deploy.bat
```

このディレクトリは `.gitignore` 対象のため Git から除外されている。fabriq 本体リポジトリ（`E:\fabriq`）から手動でコピーするか、CI/CD で配布パッケージ化する際に同梱する。

未配置のまま「新規作成」を実行すると以下のエラーで中断する:

```
テンプレートフォルダが見つかりません。
アプリケーションを再インストールしてください。
（<expected path>）
```

---

## 初回起動

`FabriqStudio.exe` を起動すると **WelcomeView** が表示される（ワークスペース未設定の状態）。左ペイン上部の編集メニューはすべて非活性。

選択肢は 2 つ:

1. **既存の fabriq ワークスペースを開く** — 既存の `E:\fabriq` 等を選択
2. **テンプレートから新規作成** — Studio 同梱テンプレートから `<新フォルダ>` を作る

---

## 既存の fabriq を開く

1. 左ペインに表示されている「ワークスペースを開く」操作（または WelcomeView 経由）でフォルダ選択ダイアログを開く
2. fabriq ルートディレクトリ（`kernel/` と `modules/` を含むフォルダ）を選択
3. 検証 OK なら自動的に `BasicParamsView` に遷移
4. 検証 NG（kernel/ または modules/ が無い）の場合はエラーメッセージを表示してブロック

選択したパスは `<exe>\config\workspace.json` に永続化される。次回起動時は自動的に同じワークスペースで起動する。

---

## テンプレートから新規ワークスペースを作成

1. 左ペイン上部の「📋 テンプレートから新規作成」ボタンを押す（ワークスペース未開時のみ表示）
2. 親フォルダを選択するダイアログ（「新規ワークスペースの作成先フォルダを選択してください」）
3. フォルダ名入力ダイアログ（`NewWorkspaceDialog`）
4. `<親>/<新フォルダ名>` が既存の場合はエラーで中断
5. テンプレートを再帰コピー（`.git` `.claude` `dev` を除外）
6. 成功すると自動的に新ワークスペースを開く（`BasicParamsView` に遷移）

テンプレート未配置時のエラー対応は前述の C. を参照。

---

## ワークスペースを切替・再ロード・閉じる

### 切替

WelcomeView または別フォルダ選択で `IWorkspaceService.Open(path)` を呼ぶ。`WorkspaceChanged` イベントが発火し、すべての ViewModel が新ワークスペースのデータを再ロードする。

未保存の変更がある画面を開いていた場合、`MainViewModel.ConfirmDiscardIfDirty()` が破棄確認ダイアログを出す。キャンセルすれば切替は取消。

### 再ロード（最新の情報に更新）

左ペイン上部「ワークスペース」セクションの「更新」ボタンで `Reload()` を呼ぶ。確認ダイアログ:

```
最新の情報に更新しますか？
未保存の編集内容はすべて破棄されます。
```

OK なら `WorkspaceChanged(NewPath==OldPath)` が発火し、各 VM がデータを再ロード。

外部ツール（fabriq の他の編集スクリプトなど）でワークスペースの CSV を更新した場合、Studio 側で再ロードしないと古いデータを編集してしまうため、外部編集後は必ず再ロードすること。

### 閉じる

「閉じる」ボタンで `IWorkspaceService.Close()` を呼ぶ。`<exe>\config\workspace.json` も削除されるため、次回起動時は WelcomeView から始まる。

---

## パスフレーズの初期設定

### 設定手順

1. 左ペイン下部の「🔑 パスフレーズ」ボタンを押す
2. `PassphraseDialog` でパスフレーズを入力
3. **既存の検証トークン（`<workspace>/kernel/txt/passphrase_verify.txt`）が存在する場合**
   - そのトークンを入力パスフレーズで復号して `surkitinisme` と照合
   - 一致 → 設定成功、Studio のメモリに保持
   - 不一致 → 「パスフレーズが正しくありません」エラー
4. **検証トークンが存在しない場合（新規ワークスペース）**
   - 入力パスフレーズで `Encrypt("surkitinisme", passphrase)` を生成して `passphrase_verify.txt` に書き出し
   - 設定成功

設定後はパスフレーズステータスが「設定済み」（緑色）に変わる。

### クリア

ダイアログで空文字を入力 → `OK` で `MasterPassphrase = null`。検証トークンファイルは残す（次回設定時に再照合できるように）。

### 注意

- パスフレーズは **プロセスメモリのみに保持**。Studio を終了すると消える
- ワークスペース内の hostlist などに `ENC:` プレフィックス付きのセルがある場合、パスフレーズ未設定だと表示・編集はできるが復号値の確認はできない
- **同じワークスペースで異なるパスフレーズを使い回さない**。検証トークンが上書きされてしまうと既存の暗号化セルが復号できなくなる

---

## ワークスペース別のディレクトリ確認

開いた直後に以下のフォルダがあることを確認する（fabriq 本体の標準構成）:

| ディレクトリ | 必須 | 内容 |
|---|---|---|
| `kernel/` | ✓ | fabriq コア関数群 |
| `modules/` | ✓ | standard / extended のモジュール |
| `profiles/` | (任意) | プロファイル CSV 群 |
| `commands/` | (任意) | コマンド定義 |
| `evidence/` | (任意) | エビデンス収集設定 |
| `kernel/csv/` | (任意) | hostlist.csv / workers.csv 等 |
| `kernel/txt/` | (任意) | passphrase_verify.txt 配置場所 |

`kernel/csv/` `kernel/txt/` は Studio が初回書き込み時に自動作成する（パスフレーズ初期設定など）。

---

## トラブルシューティング

### 起動したが WelcomeView から先に進めない

- 永続化ファイル（`<exe>\config\workspace.json`）に書かれたパスが消えているか壊れている可能性
- 該当ファイルを削除してから再起動 → 通常の WelcomeView から再選択

### 「fabriq のルートフォルダではありません」エラー

- 選択したフォルダ直下に `kernel/` と `modules/` の **両方** があるか確認
- 親フォルダではなく fabriq ルートそのものを選んでいるか確認（例: `E:\fabriq` であって `E:\` ではない）

### Pianist 系ダイアログが起動できない

- 該当ワークスペースに `modules/extended/pianist/` が存在するか確認
- `pianist.ps1` の有無は実行時にのみ要求される（編集は pianist.ps1 不在でも可能）

### Pianist Test Run が即座に失敗する

- パスフレーズ未設定で `ENC:` セルが解決できない可能性 → パスフレーズ設定後に再実行
- workspace の `kernel/common.ps1` が存在しないと dot-source 失敗 → workspace を再選択

### fabriq オーバーレイ更新で「Fabriq.exe が実行中」エラー

- 別ウィンドウで Fabriq.exe を開いていないか確認 → タスクマネージャーで `Fabriq.exe` を終了してから再実行

### バックアップで失敗する

- 出力先親フォルダの書込権限を確認
- 既存の `fabriq_backup_yyyyMMdd_HHmmss/` と同名フォルダが偶然衝突していないか確認（同一秒内の連続バックアップ）
