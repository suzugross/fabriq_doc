# hostlist.csv スキーマ（fabriq_studio 編集 UI 視点）

> **対象**: fabriq_studio / reference
> **対象バージョン**: commit `3897c6e`（取得元: `git -C E:\fabriq_studio rev-parse --short HEAD`、2026-05-06、csproj に `<Version>` 未設定）
> **ドキュメント更新日**: 2026-05-07

`kernel/csv/hostlist.csv` は fabriq シリーズで PC ごとの期待値（Expected）を定義する中央マスターで、fabriq_studio の `HostListView` / `HostDetailView` が編集 UI を担う。本ドキュメントは **編集側（producer）** の `HostEntry` モデル + 暗号化 + Export の挙動を扱う。

「読み取り側（consumer = fabriq_evidence_manager / fabriq 本体実行時）」の視点は [fabriq_evidence_manager__reference__hostlist_csv_schema.md](fabriq_evidence_manager__reference__hostlist_csv_schema.md) を参照。両 doc は **同じ CSV を別々の責務角度から扱う**ペアになる。

---

## 編集モデル: `HostEntry`

[Models/HostEntry.cs](file:///E:/fabriq_studio/FabriqStudio/Models/HostEntry.cs)。`ObservableObject` を継承し、**43 列すべてを `[ObservableProperty] private string` でフィールド宣言**する Source Generator パターン：

```csharp
public partial class HostEntry : ObservableObject
{
    [ObservableProperty] private string _adminID         = "";
    [ObservableProperty] private string _oldPCName       = "";
    [ObservableProperty] private string _newPCName       = "";
    // ...43 フィールドすべて string、初期値は ""
}
```

**型は全列 string**（数値・bool 列は無し）。空文字列を「未設定」と扱う。

CommunityToolkit.Mvvm の `ObservableProperty` Source Generator が public プロパティ（`AdminID` / `OldPCName` / ...）と `INotifyPropertyChanged` 通知を自動生成する。これにより：

- WPF `TextBox` の TwoWay バインドが直接動く（追加配線不要）
- per-field の Dirty 検知が `PropertyChanged` イベントで取れる

---

## 全 43 列定義

| 区分 | 列数 | 列名 | 暗号化可 | 突合キー | 備考 |
|---|---|---|---|---|---|
| **識別** | 3 | `AdminID` | × | × | `CryptoHelper.ExcludedColumns` に含まれる（管理 ID は機密扱い外） |
|  |  | `OldPCName` | ○ | × | 旧 PC 名（出荷時名） |
|  |  | `NewPCName` | ○ | **○** | 新 PC 名（**突合キー、auto-detect 対象**） |
| **有線 LAN** | 3 | `EthernetIP` | ○ | × | 静的 IP |
|  |  | `EthernetSubnet` | ○ | × |  |
|  |  | `EthernetGateway` | ○ | × |  |
| **無線 LAN** | 3 | `WifiIP` / `WifiSubnet` / `WifiGateway` | ○ | × |  |
| **DNS** | 4 | `DNS1` / `DNS2` / `DNS3` / `DNS4` | ○ | × | 4 件最大 |
| **BitLocker** | 1 | `Pin` | ○ | × | BitLocker PIN（機密） |
| **プリンタ** | 30 | `Printer1Name` / `Printer1Driver` / `Printer1Port` 〜 `Printer10*` | ○ | × | 3 列 × 10 台 |

**合計 44 列**（Excel 列カウント）= 識別 3 + Ethernet 3 + Wifi 3 + DNS 4 + Pin 1 + Printer 30 = 44。

> ※ `HostEntry` のクラスコメントは「**全 43 カラム**」と記載されているが、実フィールド数は `[ObservableProperty]` 44 個。識別 3 列を「2 列扱い」（`OldPCName + NewPCName`、`AdminID` は識別不要）等の運用上の数え方による表記揺れ。CSV 実列数は 44 で、consumer 側 `fabriq_evidence_manager.HostlistEntry` の読み込みも 44 列を前提とする。

### 暗号化対象外列（`CryptoHelper.ExcludedColumns`）

[Helpers/CryptoHelper.cs](file:///E:/fabriq_studio/FabriqStudio/Helpers/CryptoHelper.cs) で **fabriq_studio 全体共通**の暗号化 NG 列名を `HashSet<string>` で宣言（`StringComparer.OrdinalIgnoreCase`）：

```csharp
private static readonly HashSet<string> ExcludedColumns = new(StringComparer.OrdinalIgnoreCase)
{
    // ID / Primary Key
    "AdminID", "TaskID", "StepID", "ID", "No",
    // フラグ
    "Enabled",
    // 順序 / セグメント
    "Order", "Step", "Segment",
    // 分類 / メタ
    "Category", "Type", "Kind", "MenuName",
    // 数値制御
    "MaxRetry", "IntervalSec", "TimeoutSec",
    // スクリプト参照 / アクション
    "Script", "ScriptPath", "Action",
    // Condition
    "Condition",
}
```

`HostEntry` で実質的に該当するのは **`AdminID` のみ**（他列は hostlist.csv に存在しない）。残り 43 列は暗号化対象。

`CryptoHelper.IsEncryptableColumn(name)` で判定。バッチ暗号化系コマンド（`HostListView` の暗号化ボタン）はこのフィルタで対象を選別する。

### 突合キー

`NewPCName` が **唯一の突合キー**：

- fabriq 本体起動時の auto-detect: `$env:COMPUTERNAME` と `NewPCName` の照合（[fabriq__usage__02_session_setup.md](fabriq__usage__02_session_setup.md) §「Auto-detect」）
- fabriq_evidence_manager の hostlist 突合: `NewPCName` で行検索（[fabriq_evidence_manager__usage__02_hostlist_verification.md](fabriq_evidence_manager__usage__02_hostlist_verification.md)）

`NewPCName` が空の行は consumer 側で **読み捨て**になる（HostlistService.Load 内で `IsNullOrWhiteSpace` チェック）。

---

## Dirty 検知 + Clone / ContentEquals

`HostEntry` の特殊メソッド 2 つは **JSON シリアライズを使った generic な実装**：

```csharp
public HostEntry Clone() =>
    System.Text.Json.JsonSerializer.Deserialize<HostEntry>(
        System.Text.Json.JsonSerializer.Serialize(this))!;

public bool ContentEquals(HostEntry other) =>
    System.Text.Json.JsonSerializer.Serialize(this) ==
    System.Text.Json.JsonSerializer.Serialize(other);
```

意図：

- `[ObservableProperty]` Source Generator が **public プロパティを自動公開**するため、`System.Text.Json` がフィールド変更に追従して全列をシリアライズ可能
- 列が将来 45 列・46 列に増えても `Clone` / `ContentEquals` のコード修正が不要
- WPF の Dirty 検知は `HostDetailViewModel` が **load 時にスナップショットを `Clone` で保持** → 保存・キャンセル時に `ContentEquals` で比較

JSON ベースのため `null` と `""` は区別される（ContentEquals が一致しない）。`HostEntry` の初期値はすべて `""` のため、null 混入は通常起きない。

---

## 暗号化値の表現

機密フィールド（`Pin` / `EthernetIP` 等）は `HostEntry.Pin = "ENC:U2FsdGVkX1+abc..."` のような文字列形式で保持される。CSV に書き出す際もこの prefix 付き文字列がそのまま出力される。

復号は **fabriq 本体実行時に `Set-SelectedHostEnvironment` 内の `Resolve-HostValue` ヘルパが透過的に行う**（[fabriq__usage__02_session_setup.md](fabriq__usage__02_session_setup.md) §「ENC: 値の透過復号」）：

```powershell
function Resolve-HostValue {
    param([string]$Value)
    if ($Value.StartsWith('ENC:') -and -not [string]::IsNullOrWhiteSpace($global:FabriqMasterPassphrase)) {
        return (Unprotect-FabriqValue -EncryptedValue $Value -Passphrase $global:FabriqMasterPassphrase)
    }
    return $Value
}
```

fabriq_studio 側で復号して平文を保存することはしない（暗号化したまま hostlist.csv に置くことが前提）。

### 暗号化アルゴリズム

`ICryptoService` 実装 [Services/CryptoService.cs](file:///E:/fabriq_studio/FabriqStudio/Services/CryptoService.cs) が PowerShell `Unprotect-FabriqValue` と互換：

- AES-256-CBC + PBKDF2-HMAC-SHA256
- イテレーション 100,000 回、固定ソルト `fabriq-fixed-salt-2024`
- UTF-8 平文 → Base64 暗号文 → `ENC:` prefix

詳細は [fabriq_studio__contracts__crypto_interop.md](fabriq_studio__contracts__crypto_interop.md) を参照。

---

## バッチ暗号化 / 復号 UI

`HostListView` の上部ツールバーから：

- `[一括暗号化]` — 選択行（または全行）の **暗号化対象列**（`ExcludedColumns` に無い列）を `ICryptoService.Encrypt` で `ENC:` 付き暗号文に変換
- `[一括復号]` — 同様に `ENC:` 付き値を `Decrypt` で平文に戻す

実行前に **パスフレーズ照合**：

```csharp
public static string? ValidatePassphrase(ICryptoService crypto)
    => crypto.HasPassphrase
        ? null
        : "パスフレーズが設定されていません。\n左ペイン下部の「🔑 パスフレーズ」から設定してください。";
```

パスフレーズ未設定なら `MessageBox` で警告して中断。`crypto.HasPassphrase` は `IWorkspaceService.RootPath/kernel/txt/passphrase_verify.txt` の `surkitinisme` トークンを復号して照合した結果。

### 結果サマリ: `BatchCryptoResult`

```csharp
public record BatchCryptoResult(int Processed, int Skipped, IReadOnlyList<string> Errors)
{
    public bool HasErrors => Errors.Count > 0;
    public string ToSummary(bool isEncrypt) { /* "暗号化: N 件処理, M 件スキップ\n⚠ K 件エラー..." */ }
}
```

- `Processed`: 処理成功件数（実際に変換されたセル数）
- `Skipped`: 既に暗号化済み（暗号化時）/ 既に平文（復号時）/ 暗号化対象外列
- `Errors`: 復号失敗等のエラーメッセージ（先頭 10 件まで `ToSummary` に表示）

---

## hostlist 一括 Export（`HostListExportDialog`）

CSV のスナップショットを **タイムスタンプ付きフォルダ** に書き出す機能。fabriq 本体に流す前の確認・配布用、または fabriq_evidence_manager に渡すために `Decrypt` オプションで復号後の平文を出す用途。

### Request DTO: `HostListExportRequest`

[Models/HostListExportRequest.cs](file:///E:/fabriq_studio/FabriqStudio/Models/HostListExportRequest.cs)、record 型：

```csharp
public record HostListExportRequest(
    IReadOnlyList<HostEntry> Hosts,    // UI 側の現在状態
    string ParentFolder,                // ユーザー選択先
    string Memo,                        // README.txt 用ユーザーメモ（空可）
    bool Decrypt);                      // true = ENC: を復号して平文出力
```

### Result DTO: `HostListExportResult`

```csharp
public record HostListExportResult(
    string ExportFolderPath,         // 生成された絶対パス
    int HostCount,                   // 出力件数
    int DecryptedCells,              // 復号成功セル数（Decrypt=false なら 0）
    int RemainingEncCells,           // 出力 CSV に残った ENC: セル数
    IReadOnlyList<string> Errors)
{
    public bool HasErrors => Errors.Count > 0;
}
```

### 出力ディレクトリ構造

```
{ParentFolder}/hostlist_export_{yyyyMMdd_HHmmss}/
├── hostlist.csv     ← BOM 付き UTF-8 / fabriq 本体と同じ書式
└── README.txt       ← BOM 無し UTF-8 / テキストエディタ互換
```

- ディレクトリ命名: `hostlist_export_` + 秒精度タイムスタンプ。同日複数回出力でも衝突しない
- `hostlist.csv`: BOM 付き UTF-8（`UTF8Encoding(encoderShouldEmitUTF8Identifier: true)`）+ CsvHelper の `WriteRecords` で出力
- `README.txt`: BOM 無し UTF-8。エクスポート日時 / 件数 / Decrypt 設定 / 復号済セル数 / 残 ENC セル数 / ユーザーメモ を記録

### Decrypt オプションの挙動

`Decrypt=true` の場合、`HostEntry.Clone` でディープコピーした上で `EncryptableProps`（reflection で抽出した暗号化対象 string プロパティ群、Lazy 静的キャッシュ）に対して：

```csharp
foreach (var host in clones)
{
    foreach (var prop in EncryptableProps.Value)
    {
        var value = prop.GetValue(host)?.ToString() ?? "";
        if (!value.StartsWith("ENC:", StringComparison.Ordinal)) continue;
        try
        {
            prop.SetValue(host, _crypto.Decrypt(value, _crypto.MasterPassphrase!));
            decrypted++;
        }
        catch (Exception ex)
        {
            errors.Add($"AdminID={host.AdminID}/{prop.Name}: {ex.Message}");
        }
    }
}
```

エラーキーは `AdminID=<adminId>/<PropertyName>` 形式（どの host のどの列で失敗したかが特定可能）。**UI 側 `HostEntry` インスタンスは変更しない**（`Clone` 経由のため）。

`crypto.HasPassphrase = false` の場合は `Decrypt=true` でも復号スキップ + エラーリストに 1 件追加。

### 残 ENC セル数の意味

`RemainingEncCells` は **出力された CSV に最終的に残っている `ENC:` プレフィックス付きセル数**：

| `Decrypt` | 復号成功 | 復号失敗 | 結果の `ENC:` 残数 |
|---|---|---|---|
| false | - | - | 元の暗号化セル数すべて |
| true | 全成功 | なし | 0 |
| true | 一部失敗 | あり | 失敗分のみ残存 |

fabriq_evidence_manager で読み込ませる用途では **`Decrypt=true` + `RemainingEncCells == 0`** が望ましい状態。詳細は [fabriq_evidence_manager__usage__02_hostlist_verification.md](fabriq_evidence_manager__usage__02_hostlist_verification.md) §「暗号化フィールドの扱い」。

---

## consumer 側との対応関係

| 参照系 | 列の扱い | 暗号化対応 |
|---|---|---|
| **fabriq_studio**（producer = 編集 UI） | `HostEntry` 全 43 列を WPF TwoWay バインド + 暗号化 / 復号 / export | あり（編集 UI から `[一括暗号化]` / `[一括復号]`、export で `Decrypt` トグル） |
| **fabriq 本体**（実行時 consumer） | `Set-SelectedHostEnvironment` が hostlist.csv の選択行から `SELECTED_*` env vars を立てる | あり（`Resolve-HostValue` が `ENC:` プレフィックスを `$global:FabriqMasterPassphrase` で透過復号） |
| **fabriq_evidence_manager**（読取 consumer） | `HostlistEntry` で `NewPCName` をキーに突合 | **無し**（復号機能を持たない、`Decrypt=true` で export 済み CSV が必須） |

---

## 関連ドキュメント

- 編集画面の UI 詳細: [fabriq_studio__apps__01_main_pages.md](fabriq_studio__apps__01_main_pages.md) §「2. 端末一覧（HostList → HostDetail）」
- ワークスペース概念（hostlist.csv の置き場所）: [fabriq_studio__architecture__02_workspace.md](fabriq_studio__architecture__02_workspace.md)
- 暗号化アルゴリズムと PowerShell 互換性: [fabriq_studio__contracts__crypto_interop.md](fabriq_studio__contracts__crypto_interop.md)
- consumer 側 hostlist スキーマ（読み取り視点）: [fabriq_evidence_manager__reference__hostlist_csv_schema.md](fabriq_evidence_manager__reference__hostlist_csv_schema.md)
- fabriq 本体側のセッション開始時 env var 確定処理: [fabriq__usage__02_session_setup.md](fabriq__usage__02_session_setup.md)
- fabriq_evidence_manager での hostlist 突合: [fabriq_evidence_manager__usage__02_hostlist_verification.md](fabriq_evidence_manager__usage__02_hostlist_verification.md)
