# ワークスペースモデルと未保存変更ガード

> **対象**: fabriq_studio / architecture / workspace
> **対象バージョン**: commit `3897c6e`
> **ドキュメント更新日**: 2026-05-06

## ワークスペースとは

fabriq_studio における「ワークスペース」は **1 つの fabriq フレームワークインスタンス** を指す。具体的には次の必須要素を持つディレクトリ:

```
<workspace_root>/
├── kernel/      ※必須（fabriq コア関数群）
├── modules/     ※必須（standard / extended のモジュール）
├── profiles/    （プロファイル CSV）
├── commands/    （コマンド定義）
└── evidence/    （エビデンス収集設定）
```

`IWorkspaceService.Validate(string)` は最低限 `kernel/` と `modules/` の存在のみ検証する。それ以外（`profiles/`, `commands/`, `evidence/`）は欠損可（実行時に必要に応じて生成される）。

## 状態モデル

```
┌──────────────────────────────────────────────────────────┐
│                  IWorkspaceService                       │
├──────────────────────────────────────────────────────────┤
│  RootPath : string?  （null = 未設定 / 非null = 開いている）│
│  IsOpen   : bool                                          │
│  WorkspaceChanged : event                                 │
└──┬───────┬─────────┬──────────────┬───────────────────────┘
   │       │         │              │
 Open  ┌──Close──┐ Reload    TryRestorePersisted
   │  │         │   │              │
   ▼  ▼         ▼   ▼              ▼
[NewPath, OldPath]  [NewPath==OldPath]   (silent)
```

### Open(path)

1. `Validate(path)` で kernel/ + modules/ の存在を検証（NG なら `ArgumentException`）
2. `RootPath` を更新（末尾の `\` `/` は trim）
3. `<exe>\config\workspace.json` に永続化
4. `WorkspaceChanged(NewPath=新パス, OldPath=旧 or null)` を発火

### Close()

1. `RootPath = null`
2. 永続化ファイルを削除
3. `WorkspaceChanged(NewPath=null, OldPath=旧)` を発火

### Reload()

1. `RootPath` 不変
2. `WorkspaceChanged(NewPath=現パス, OldPath=現パス)` を発火（ガード条件 `NewPath == OldPath` で各 VM がデータ再ロードのみ実行する手がかり）
3. ワークスペースが閉じている場合は no-op

### TryRestorePersisted()

- アプリ起動時、`App.OnStartup` が VM 構築前に呼ぶ
- `<exe>\config\workspace.json` を読んで前回パスを取得
- `Validate` が通ったときのみ `_rootPath` を直接代入（**`WorkspaceChanged` は発火しない**）
- VM コンストラクタは `IsOpen` を見て初期データを直接ロードする

## 永続化ファイル

```
<exe ディレクトリ>/config/workspace.json
```

`AppDomain.CurrentDomain.BaseDirectory` 起点なので、ポータブル運用（USB に置いて持ち運ぶ）にも対応する。スキーマ:

```json
{
  "rootPath": "E:\\some\\fabriq\\workspace"
}
```

書き込み・読み込み失敗は無視する設計（永続化失敗で動作を止めない）。

## テンプレートから新規ワークスペース作成

`IWorkspaceService.CreateFromTemplateAsync(targetPath)` の流れ:

1. テンプレートパス: `<exe>\template\template_fabriq\fabriq\`
2. 存在確認 → 無ければ「再インストールしてください」エラー
3. `Task.Run(() => CopyDirectoryRecursive(...))` で再帰コピー
4. 除外ディレクトリ: `.git`, `.claude`, `dev`（大文字小文字無視）
5. 既存ファイルがあれば `File.Copy(..., overwrite: false)` でエラー（重複防止）

呼び出し側（`MainViewModel.CreateNewWorkspaceAsync`）の責務:

1. `OpenFolderDialog` で親フォルダ選択
2. `NewWorkspaceDialog` でフォルダ名入力
3. `targetPath = Path.Combine(parent, folderName)` の存在チェック（あれば中断）
4. `CreateFromTemplateAsync` 実行
5. 成功なら `Open(targetPath)` で自動的に開く

`template/template_fabriq/fabriq/` は `.gitignore` 対象のため、配布前に fabriq 本体を別途配置する必要がある（[fabriq_studio__usage__01_workspace_setup.md](fabriq_studio__usage__01_workspace_setup.md) 参照）。

## 未保存変更ガード（IDirtyAwareViewModel）

ワークスペース切替・画面遷移・アプリ終了でデータを失わないよう、編集系画面に `IDirtyAwareViewModel` を実装させ、遷移直前に確認ダイアログを出す仕組み。

### インターフェース

```csharp
public interface IDirtyAwareViewModel
{
    bool   HasUnsavedChanges { get; }
    string DirtyDescription   { get; }
    void   DiscardChanges();
}
```

- **`HasUnsavedChanges`**: 遷移時に都度評価される。複数の Dirty フラグを持つ画面は OR で集約する。`PropertyChanged` 通知は不要
- **`DirtyDescription`**: ダイアログ本文に表示する画面・対象識別子（例: `"端末詳細 (PC-001)"`、`"プロファイル: foo"`）
- **`DiscardChanges()`**: in-memory 状態のロールバック。共有エンティティを編集している画面（`HostDetail` など）はこれを呼ばないと「破棄」後も親リストに編集中の値が残って見える

### ガード呼び出し（DirtyConfirmHelper）

`MainViewModel.ConfirmDiscardIfDirty()` がすべての遷移系操作の前に呼ぶ:

```csharp
public bool ConfirmDiscardIfDirty()
    => DirtyConfirmHelper.ConfirmDiscard(CurrentPage as IDirtyAwareViewModel);
```

`DirtyConfirmHelper` がダイアログ文言と `DiscardChanges()` 呼び出しを集約。OK なら `true`、キャンセルなら `false` を返し、呼び出し元（`Navigate` / `CloseWorkspace` / `ReloadWorkspace` / `CreateNewWorkspace` / `NavigateBackMessage` 受信）はそれを見て遷移を中断できる。

### 実装している ViewModel

ガードの効く編集画面（参考、コードベース上の検索による把握）:

- `HostDetailViewModel`
- `ModuleDetailViewModel`
- `AppConfigViewModel`
- `ProfileDetailViewModel`
- `BasicParamsViewModel`
- `LooperEditorViewModel`
- `PianistProfileEditorViewModel`

一覧画面（`HostListViewModel`, `ModuleEditViewModel`, `RegistryCollectionViewModel` など）は通常 Dirty 状態を持たないため未実装。

## ワークスペース変更ハンドラ（MainViewModel）

`MainViewModel` のコンストラクタで `WorkspaceChanged` を購読し、3 ケースに分岐する:

| ケース | 条件 | 処理 |
|---|---|---|
| Close | `e.NewPath is null` | `IsWorkspaceOpen = false`, `WorkspaceName = ""`, `CurrentPage = _welcomeVm` |
| Reload | `e.NewPath == e.OldPath` | `CurrentPage` 維持（各 VM が個別に `WorkspaceChanged` を受信して再ロードする） |
| Open | それ以外 | `IsWorkspaceOpen = true`, `WorkspaceName = フォルダ名`, `CurrentPage = _basicParamsVm`（メイン画面に遷移） |

各 ViewModel は **必要に応じて** `WorkspaceChanged` を独自に購読する。`IWorkspaceService` 自体は集中ハブなので、購読数が増えても疎結合のまま拡張できる。

## ワークスペースに関わる代表的なパス

`IWorkspaceService.RootPath` を起点に Studio が触るファイル群:

| 用途 | パス |
|---|---|
| 端末一覧 | `kernel/csv/hostlist.csv` |
| 作業者一覧 | `kernel/csv/workers.csv` |
| カテゴリ定義 | `kernel/csv/categories.csv` |
| ログ送信先 | `kernel/csv/log_destinations.csv` |
| パスフレーズ検証トークン | `kernel/txt/passphrase_verify.txt` |
| モジュール定義 | `modules/{standard,extended}/<name>/module.csv` |
| モジュール選択肢 | `modules/{standard,extended}/<name>/preset.csv` |
| プロファイル | `profiles/<name>.csv` |
| Pianist プロファイル | `modules/extended/pianist/profiles/<name>/` |
| Pianist カタログ | `modules/extended/pianist/pianist_list.csv` |
| プリンタドライバ一覧 | `modules/standard/printer_driver_config/printer_driver_list.csv` |
| レジストリ一覧 (export 先) | `modules/standard/reg_hklm_config/reg_hklm_list.csv` / `reg_hkcu_config/reg_hkcu_list.csv` |
| Looper 設定 | `modules/extended/<name>/looper_list.csv`（export 先） |

これら CSV のスキーマは fabriq 本体（`E:\fabriq`）の契約に従う。Studio は契約準拠のまま I/O を行うことに徹する。
