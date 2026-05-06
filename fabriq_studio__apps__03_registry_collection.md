# レジストリ辞書（Registry Collection）

> **対象**: fabriq_studio / apps / Registry Collection
> **対象バージョン**: commit `3897c6e`
> **ドキュメント更新日**: 2026-05-06

Windows レジストリ設定のテンプレートを 100 件以上カタログ化し、ユーザーが選んだエントリをワークスペースの fabriq モジュール CSV へ転記する機能。

---

## 位置付け

fabriq の標準モジュール `reg_hklm_config` / `reg_hkcu_config` は CSV ファイル `reg_hklm_list.csv` / `reg_hkcu_list.csv` に登録されたレジストリ設定を一括適用する。これらの CSV を毎回手書きする運用は事故が起きやすい（ハイブ名・型名のタイポ、KeyPath のエスケープ崩壊など）。

レジストリ辞書は **検証済みのプリセット集** をカタログ化し、ユーザーは GUI から選んでワークスペースに転記するだけで済むようにする。RDP 有効化、UAC 弱体化、SMBv1 無効化などの代表的な設定が登録されている（README より 100 件超）。

ワークスペース非依存の機能（カタログ自体はパッケージに同梱されておりどのワークスペースを開いていなくても閲覧・編集可能）。エクスポート時のみワークスペースが必要。

---

## カタログのファイル配置

```
<exe ディレクトリ>/registry_collection/
└── catalog.json    ── 全エントリの永続化先（順序保持の JSON 配列）
```

ビルド時 / Publish 時に `FabriqStudio.csproj` の `AfterTargets="Build"` / `AfterTargets="Publish"` ターゲットで `<MSBuildProjectDirectory>/registry_collection/` から出力ディレクトリにコピーされる。

`AppDomain.CurrentDomain.BaseDirectory` 起点なのでポータブル運用対応。複数ワークスペース間でカタログは共有される。

---

## エントリのスキーマ

[Models/RegistryTemplateEntry.cs](file:///E:/fabriq_studio/FabriqStudio/Models/RegistryTemplateEntry.cs):

| フィールド | 型 | 内容 |
|---|---|---|
| `id` | string | 8 文字 hex の一意 ID（Guid からの切り出し）。重複行の特定用 |
| `category` | string | カテゴリ分類（"セキュリティ" "ネットワーク" 等） |
| `title` | string | エントリ表示名 |
| `hive` | string | レジストリハイブ。`"HKLM"` / `"HKCU"` |
| `keyPath` | string | フルパス形式（例: `HKEY_LOCAL_MACHINE\SYSTEM\...`） |
| `keyName` | string | 値名 |
| `type` | string | 値の型（`REG_DWORD` / `REG_SZ` / `REG_BINARY` 等、デフォルト `REG_DWORD`） |
| `value` | string | 値そのもの |
| `description` | string | 設定の説明（旧 `docs/*.txt` を JSON 内にインライン化） |
| `tags` | string | セミコロン区切りタグ（例: `"RDP;リモートデスクトップ;接続"`）。検索・フィルタリング用 |

---

## サービス API（IRegistryCollectionService）

[Services/IRegistryCollectionService.cs](file:///E:/fabriq_studio/FabriqStudio/Services/IRegistryCollectionService.cs):

| メソッド | 役割 |
|---|---|
| `Entries` | 現在ロード済み全エントリ（順序保持） |
| `EnsureInitializedAsync()` | 起動時の初期化。catalog.json があればロード、無ければ空状態で待機。`App.OnStartup` で **VM 構築前** に呼ぶ |
| `ReloadAsync()` | catalog.json を再ロード |
| `AddAsync(entry)` | 末尾に追加して保存 |
| `UpdateAsync(entry)` | `entry.Id` で対象行を特定して差し替えて保存。Id が見つからなければ no-op |
| `RemoveAsync(id)` | 指定 Id を削除して保存 |
| `ExportToWorkspaceAsync(entry, workspaceRootPath)` | 1 件をワークスペースの reg_hklm/hkcu_list.csv に追記 |

`ExportResult` の戻り値は `Added` / `Skipped` / `Error` を返す（OK = 1 / 重複でスキップ = 1 / エラーは Error 文字列）。

---

## エクスポート規則

`ExportToWorkspaceAsync` の挙動:

1. `entry.Hive` を見て `HKLM` なら `modules/standard/reg_hklm_config/reg_hklm_list.csv`、`HKCU` なら `modules/standard/reg_hkcu_config/reg_hkcu_list.csv` を対象とする
2. 既存 CSV を読み込み（無ければ新規作成）
3. **`KeyPath + KeyName` の組み合わせ** で重複チェック（既存行と一致 → Skip）
4. 新規行を追記して保存

重複検知のキーが `Id` ではなく `KeyPath + KeyName` なのは、**意味的に同じレジストリ値** を二重登録させないため（カタログ側で複数の Title 表記があっても、最終的に CSV に書く中身が同じならスキップする）。

---

## UI 構成（RegistryCollectionView）

| 領域 | 内容 |
|---|---|
| 左ペイン | カテゴリ・タグでのフィルタ |
| 中央 | エントリ一覧 DataGrid（Title / Category / Hive / KeyPath / KeyName / Value など） |
| 右上 | 検索ボックス（Title / Description / Tags 横断） |
| 右下 | 詳細プレビュー（選択中エントリの Description 全文） |

### 操作

- **追加 / 編集 / 削除**: カタログ自体の編集
- **エクスポート**: 選択 1 件を現ワークスペースに転記。ワークスペース未開時はエラーメッセージで促す
- **複数選択** + 一括エクスポート（Picker ダイアログ経由）

ViewModel: `RegistryCollectionViewModel`（一覧）+ `RegistryPickerViewModel`（エクスポート対象選択ダイアログ）。

---

## カタログ運用の注意

- catalog.json は人間が直接編集することも可能（JSON フォーマット）。`Id` フィールドは衝突しない hex で振られているため、手編集時は既存行の Id を消さないように注意
- ビルド時に毎回コピーされるため、配布版で catalog を更新するにはソース側 `<MSBuildProjectDirectory>/registry_collection/catalog.json` を編集してリビルドする
- ワークスペース側の `reg_hklm_list.csv` / `reg_hkcu_list.csv` は他の編集ツール（fabriq の Module Detail 画面など）からも触られるため、エクスポートは「追記のみ」設計（既存行の上書きはしない）
