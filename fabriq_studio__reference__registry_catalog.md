# Registry catalog.json + テンプレートスキーマ

> **対象**: fabriq_studio / reference
> **対象バージョン**: commit `3897c6e`（取得元: `git -C E:\fabriq_studio rev-parse --short HEAD`、2026-05-06）
> **ドキュメント更新日**: 2026-05-07

`registry_collection/catalog.json` は fabriq_studio に同梱される **レジストリ設定テンプレートカタログ**（100 件超）。`RegistryCollectionView` がこのカタログを表示し、`[ワークスペースへエクスポート]` で対象 fabriq の `reg_hklm_list.csv` / `reg_hkcu_list.csv` に転記する。

本ドキュメントは **catalog.json のスキーマ + RegistryTemplateEntry モデル + ワークスペースへの export 動作** を扱う。Editor 画面の UI 操作は [fabriq_studio__apps__03_registry_collection.md](fabriq_studio__apps__03_registry_collection.md)。

---

## 1. ファイル配置

### catalog.json の永続化先

```
{exe と同じフォルダ}/registry_collection/catalog.json
```

`AppDomain.CurrentDomain.BaseDirectory` 起点で、**ポータブル運用に対応**：

- 開発時: `bin/Debug/net8.0-windows/registry_collection/catalog.json`（ビルド時に csproj `AfterTargets="Build"` でコピー）
- 配布時: `<publish 先>/registry_collection/catalog.json`（`AfterTargets="Publish"` でコピー）
- ソース管理: `FabriqStudio/registry_collection/catalog.json`（実体）

[Services/RegistryCollectionService.cs](file:///E:/fabriq_studio/FabriqStudio/Services/RegistryCollectionService.cs)：

```csharp
private static readonly string DataDir = Path.Combine(
    AppDomain.CurrentDomain.BaseDirectory,
    "registry_collection");

private static readonly string CatalogPath = Path.Combine(DataDir, "catalog.json");
```

### 起動時のロード

`RegistryCollectionService.EnsureInitializedAsync` が App.OnStartup（VM 構築前）で：

1. `DataDir` を `Directory.CreateDirectory` で確保
2. `catalog.json` 存在時のみ `ReloadAsync` でデシリアライズ
3. 不在 / 破損時は **空リストで graceful degradation**（例外なく起動継続）

破損時のフォールバックは `try/catch` で空リストに falling back する設計。

---

## 2. catalog.json スキーマ

### ルート構造

```json
{
  "version": 1,
  "entries": [
    { /* RegistryTemplateEntry × N */ }
  ]
}
```

| キー | 型 | 意味 |
|---|---|---|
| `version` | int | スキーマバージョン（現状固定 `1`、`CatalogData.Version`）|
| `entries` | array | `RegistryTemplateEntry` の配列 |

### JSON シリアライズ設定

```csharp
private static readonly JsonSerializerOptions JsonOptions = new()
{
    WriteIndented = true,
    PropertyNameCaseInsensitive = true,
};
```

- インデント付き（diff 安定 + 人間可読）
- プロパティ名は大文字小文字無視で読む（手書き編集の保険）

---

## 3. `RegistryTemplateEntry` 9 フィールド

[Models/RegistryTemplateEntry.cs](file:///E:/fabriq_studio/FabriqStudio/Models/RegistryTemplateEntry.cs)。`sealed class` + `JsonPropertyName` 指定（camelCase 出力）。

| 列 | 型 | 既定値 | 意味 |
|---|---|---|---|
| `id` | string | `Guid.NewGuid().ToString("N")[..8]` | エントリ一意識別子（**8 文字 hex**）|
| `category` | string | `""` | 分類（`Security` / `Network` / `Display` 等、UI でグルーピング表示） |
| `title` | string | `""` | UI 表示用タイトル（日本語、例: `"リモートデスクトップを有効化"`） |
| `hive` | string | `"HKLM"` | レジストリハイブ（`HKLM` / `HKCU`） |
| `keyPath` | string | `""` | フルパス（`HKEY_LOCAL_MACHINE\SYSTEM\...`、ハイブ名展開済み） |
| `keyName` | string | `""` | 値の名前（例: `fDenyTSConnections`） |
| `type` | string | `"REG_DWORD"` | 値の型（`REG_DWORD` / `REG_SZ` / `REG_BINARY` 等） |
| `value` | string | `""` | 値（型に関わらず文字列で保持、PowerShell 側でパース） |
| `description` | string | `""` | 説明文（**多段落マークダウン風、後述**） |
| `tags` | string | `""` | セミコロン区切りのタグ（例: `"RDP;リモートデスクトップ;接続"`） |

### `id` の 8 文字 hex

GUID の先頭 8 文字を採用（`Guid.NewGuid().ToString("N")[..8]`）。**衝突可能性は 16^8 ≒ 43 億分の 1** で、catalog 規模（100 件超）であれば実用上一意。`AddAsync` 時に既存 ID との衝突チェックは行っていない（衝突時は `RemoveAsync` / `UpdateAsync` の `FindIndex` がどちらかをヒットする挙動になる）。

開発時の手動メンテナンスでは `00000001` 〜 `00000010` のような **連番風 hex** を採用しているケースが多い（既定の同梱 catalog がそう）。

### `description` の書式

長文の多段落テキスト。慣例的に次の構造：

```
【<タイトル再掲>】

<冒頭の説明文>

■ <小見出し>
  <インデント本文>
  ...

■ 確認方法
  ...

■ 注意事項
  ...

■ 既定値
  ...
```

例（catalog.json `id=00000001`）：

```
【リモートデスクトップを有効化】

fDenyTSConnections = 0 に設定することで、リモートデスクトップ接続を許可します。

■ 注意事項
  - この設定だけでは接続できない場合があります。
  - Windowsファイアウォールのリモートデスクトップ規則（TCP 3389）も
    合わせて許可してください（firewall_config モジュールで設定可能）。
  - ユーザーアカウントに「リモートデスクトップユーザー」グループへの
    追加も必要な場合があります。

■ 確認方法
  設定 → システム → リモートデスクトップ → 「リモートデスクトップを有効にする」がON

■ 既定値
  fDenyTSConnections = 1（無効）
```

書式は **規約として固定されておらず**、現在の同梱 catalog がこのスタイルというだけ。Studio UI は `description` をそのまま表示する（マークダウンレンダリングはしない、改行と等幅フォント程度）。

歴史的経緯：以前は `docs/<id>.txt` として別ファイルだった説明文を、JSON 内にインライン化した（`RegistryTemplateEntry` クラスコメント参照）。

### `tags` のセミコロン区切り

```
"RDP;リモートデスクトップ;接続"
```

→ Studio 内で `';'` 分割してフィルタチップ表示。タグ名の正規化（前後空白 trim 等）は **UI 側 ViewModel** で行い、JSON には正規化前の生文字列を保持。

検索ボックスでタグ部分一致検索が走る（`title` / `description` / `tags` を OR で対象、`OrdinalIgnoreCase`）。

### `keyPath` の正規化

`HKEY_LOCAL_MACHINE\...` のフルパス形式で保持する。`hive` 列との **冗長表現**（`hive="HKLM"` かつ `keyPath="HKEY_LOCAL_MACHINE\..."`）：

- `hive` は **export 先 CSV の決定**（reg_hklm_list.csv vs reg_hkcu_list.csv）に使う
- `keyPath` は **CSV にそのまま書く値**（PowerShell 側 `reg add` がフルパス前提）

両者の整合性は Studio Editor 側で保つ（`hive` 切替時に `keyPath` の prefix を自動置換するのが理想だが、現状は手動）。

---

## 4. CRUD 操作

### Reload

```csharp
public async Task ReloadAsync()
{
    if (!File.Exists(CatalogPath)) { _entries = []; return; }
    try
    {
        var json = await File.ReadAllTextAsync(CatalogPath);
        var data = JsonSerializer.Deserialize<CatalogData>(json, JsonOptions);
        _entries = data?.Entries ?? [];
    }
    catch
    {
        _entries = [];   // 破損時 graceful degradation
    }
}
```

- ファイル不在 → 空リスト（fresh 状態）
- 破損 / 不正 JSON → 空リスト（catch silent）
- 正常 → `_entries` 差し替え

### Add / Update / Remove

```csharp
public async Task AddAsync(RegistryTemplateEntry entry)
{
    _entries.Add(entry);
    await SaveAsync();
}

public async Task UpdateAsync(RegistryTemplateEntry entry)
{
    var index = _entries.FindIndex(e => e.Id == entry.Id);
    if (index < 0) return;       // ID 不一致なら no-op
    _entries[index] = entry;
    await SaveAsync();
}

public async Task RemoveAsync(string id)
{
    var removed = _entries.RemoveAll(e => e.Id == id);
    if (removed > 0) await SaveAsync();   // ヒット時のみ保存
}
```

すべて非同期で書き込み。`SaveAsync` が失敗しても in-memory `_entries` は更新済みなので、次回起動時に「保存されていなかった」状態で立ち上がる（次回再試行可）。

### Save の冪等性

```csharp
private async Task SaveAsync()
{
    try
    {
        Directory.CreateDirectory(DataDir);
        var data = new CatalogData { Entries = _entries };
        var json = JsonSerializer.Serialize(data, JsonOptions);
        await File.WriteAllTextAsync(CatalogPath, json);
    }
    catch
    {
        // 保存失敗はランタイムに影響させない
    }
}
```

書き込み失敗（権限なし / ディスクフル / etc.）は `catch` で握りつぶす（コメント: `保存失敗はランタイムに影響させない`）。次回起動時の `ReloadAsync` で前回保存分を読み戻すか、in-memory が破棄される。

---

## 5. ワークスペースへの Export

`ExportToWorkspaceAsync` がカタログ 1 エントリを fabriq ワークスペースの reg_config CSV に書き込む。

### 出力先 CSV の決定

```csharp
var relPath = entry.Hive.Equals("HKCU", StringComparison.OrdinalIgnoreCase)
    ? HkcuRelPath
    : HklmRelPath;

private const string HklmRelPath = @"modules\standard\reg_hklm_config\reg_hklm_list.csv";
private const string HkcuRelPath = @"modules\standard\reg_hkcu_config\reg_hkcu_list.csv";
```

- `hive == "HKCU"` → `reg_hkcu_list.csv`
- それ以外（`HKLM` 含む）→ `reg_hklm_list.csv`

### Read-Modify-Write 戦略

`ExportSingle` は **append ではなく read-modify-write**：

```
1. 既存 CSV を LoadRegConfigRows で読み込み（不在なら空リスト）
2. 重複チェック: KeyPath + KeyName の OrdinalIgnoreCase 完全一致
   ヒット → ExportResult { Skipped = 1 } で即 return
3. AdminID を既存最大値 + 1 で採番
4. 全件を BOM 付き UTF-8 で書き戻す
```

append を採用しなかった理由（コメント）：「**append ではなく read-modify-write にすることで改行問題を回避する**」。BOM の重複・末尾改行不整合・CRLF / LF 混在等の事故を予防。

### `RegConfigRow` の 7 列

```csv
Enabled,AdminID,SettingTitle,KeyPath,KeyName,Type,Value
1,1,リモートデスクトップを有効化,HKEY_LOCAL_MACHINE\...\Terminal Server,fDenyTSConnections,REG_DWORD,0
1,2,UACをサイレントモードに設定,HKEY_LOCAL_MACHINE\...\Policies\System,ConsentPromptBehaviorAdmin,REG_DWORD,0
```

[CsvHelper.Configuration.Attributes.Name] で各列名を明示：

```csharp
private sealed class RegConfigRow
{
    [Name("Enabled")]      public string Enabled      { get; set; } = "1";
    [Name("AdminID")]      public string AdminId      { get; set; } = "1";
    [Name("SettingTitle")] public string SettingTitle { get; set; } = "";
    [Name("KeyPath")]      public string KeyPath      { get; set; } = "";
    [Name("KeyName")]      public string KeyName      { get; set; } = "";
    [Name("Type")]         public string Type         { get; set; } = "";
    [Name("Value")]        public string Value        { get; set; } = "";
}
```

### Catalog の `RegistryTemplateEntry` → reg_config CSV の `RegConfigRow` 写像

| catalog | csv | 備考 |
|---|---|---|
| - | `Enabled` | `"1"` 固定（追加時に有効） |
| - | `AdminID` | 既存最大値 + 1 で採番（同 CSV 内一意） |
| `title` | `SettingTitle` | 直写し |
| `keyPath` | `KeyPath` | 直写し |
| `keyName` | `KeyName` | 直写し |
| `type` | `Type` | 直写し |
| `value` | `Value` | 直写し |
| `category` / `description` / `tags` / `id` | （写さない） | catalog 側のメタデータのみ、CSV には載せない |

### 重複判定

`KeyPath + KeyName` の `StringComparer.OrdinalIgnoreCase` 完全一致：

```csharp
if (existing.Any(r =>
        string.Equals(r.KeyPath, entry.KeyPath, StringComparison.OrdinalIgnoreCase) &&
        string.Equals(r.KeyName, entry.KeyName, StringComparison.OrdinalIgnoreCase)))
{
    return new ExportResult { Skipped = 1 };
}
```

`Value` の差は無視（既に同じレジストリキーが追加されていれば、別の値で再追加は許さない）。

### `ExportResult`

```csharp
public class ExportResult
{
    public int Added   { get; set; }    // 追加成功件数（0 or 1）
    public int Skipped { get; set; }    // 重複スキップ（0 or 1）
    public string? Error { get; set; }  // エラー時のメッセージ（null = 成功）
}
```

UI は `Added > 0` で成功ダイアログ、`Skipped > 0` で「既に追加済み」通知、`Error != null` でエラーダイアログを出す。

### エラーハンドリング

```csharp
catch (UnauthorizedAccessException ex) → Error = "アクセスが拒否されました。\n{message}"
catch (IOException ex)                 → Error = "ファイル操作中にエラーが発生しました。\n{message}"
catch (Exception ex)                   → Error = "予期しないエラーが発生しました。\n{message}"
```

UnauthorizedAccessException / IOException は明示的にユーザー向けのメッセージを生成。それ以外は generic message。

### ディレクトリ自動生成

```csharp
Directory.CreateDirectory(Path.GetDirectoryName(csvPath)!);
```

書き込み前に親ディレクトリを確保。**ワークスペースが空（モジュール構造が未作成）のときも export が動く**ようにする保険。

### CSV の文字コード

```csharp
using var writer = new StreamWriter(csvPath, append: false,
    encoding: new UTF8Encoding(encoderShouldEmitUTF8Identifier: true));
```

**BOM 付き UTF-8** で書き戻す（PowerShell 5.1 の `Import-Csv` と互換性維持のため）。読み込みは `detectEncodingFromByteOrderMarks: true` で BOM 有無どちらも対応。

---

## 6. 起動時の表示構築（RegistryCollectionViewModel）

`RegistryCollectionViewModel` が `_service.Entries` を使って WPF DataGrid に表示する：

| UI 要素 | 動作 |
|---|---|
| Category ComboBox | distinct な `category` 値 + "All" |
| Search TextBox | `title` / `description` / `tags` を OR + OrdinalIgnoreCase で部分一致 |
| Hive フィルタ | `HKLM` / `HKCU` / `All` |
| エントリリスト | RegistryTemplateEntry × N、各行に `[Edit]` / `[Delete]` / `[Export to Workspace]` |

詳細 UI は [fabriq_studio__apps__03_registry_collection.md](fabriq_studio__apps__03_registry_collection.md) を参照。

---

## 7. 同梱 catalog.json の規模感

執筆時点（commit `3897c6e`）の同梱 catalog は **100 エントリ超**、カテゴリ別の代表例：

| Category | 主な内容 |
|---|---|
| Security | RDP / UAC / SMBv1 / DontDisplayLastUserName / WindowsDefender 例外 等 |
| Network | DNS suffix / IPv6 priority / NetBIOS / 共有フォルダ設定 |
| Display | DPI scale / DesktopWindowComposition / Theme |
| Privacy | Telemetry / 広告 ID / Cortana 無効化 |
| Performance | スワップファイル / 視覚効果 / 高速スタートアップ |
| ... | ... |

実際のエントリ詳細は **`registry_collection/catalog.json` を直接参照**するのが正確（本ドキュメント執筆後に追加・更新される可能性あり）。

---

## 8. 同梱 catalog の更新ワークフロー

開発ブランチで catalog を編集して fabriq_studio 配布版に同梱する流れ：

1. fabriq_studio リポジトリ内の `FabriqStudio/registry_collection/catalog.json` を **手動編集** または **Studio から CRUD 操作**（後者の場合は出力先が `bin/Debug/...` 側になるので、リポジトリ側にコピーバックが必要）
2. JSON フォーマットチェック（`WriteIndented = true` で出力されているなら整っている）
3. ビルド `dotnet build FabriqStudio.sln` → `bin/Debug/.../registry_collection/catalog.json` にコピー（`AfterTargets="Build"`）
4. 配布パブリッシュ `dotnet publish ... -o E:/publish_fabriq_studio` → `<publish>/registry_collection/catalog.json` にコピー（`AfterTargets="Publish"`）

エンドユーザー側は `RegistryCollectionView` で更に追加・編集・削除して **個人 catalog として育てる** ことができる（永続化先が exe ディレクトリの `registry_collection/catalog.json` のため）。配布版の更新と個人カスタマイズの **マージ機構は無い**（後発版の配布で上書きされる、個人編集は失われる）。

---

## 関連ドキュメント

- Registry Collection View の UI 詳細: [fabriq_studio__apps__03_registry_collection.md](fabriq_studio__apps__03_registry_collection.md)
- ワークスペース構造（reg_*_list.csv の位置）: [fabriq_studio__architecture__02_workspace.md](fabriq_studio__architecture__02_workspace.md)
- fabriq 本体の reg_hklm_config モジュール: [fabriq__modules__reg_hklm_config.md](fabriq__modules__reg_hklm_config.md)
- fabriq 本体の reg_hkcu_config モジュール: [fabriq__modules__reg_hkcu_config.md](fabriq__modules__reg_hkcu_config.md)
- Services 索引: [fabriq_studio__reference__services_catalog.md](fabriq_studio__reference__services_catalog.md)
- Models 索引: [fabriq_studio__reference__models_catalog.md](fabriq_studio__reference__models_catalog.md)
