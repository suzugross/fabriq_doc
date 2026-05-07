# Pianist Profile スキーマ + section marker DSL

> **対象**: fabriq_studio / reference
> **対象バージョン**: commit `3897c6e`（取得元: `git -C E:\fabriq_studio rev-parse --short HEAD`、2026-05-06）/ pianist v1.5.0 系
> **ドキュメント更新日**: 2026-05-07

Pianist Profile は fabriq の **拡張モジュール `extended/pianist`** が消費するプロファイル形式で、**1 Profile = 1 ディレクトリ + 複数ファイル** から成る。fabriq_studio 側 `PianistProfileEditorViewModel` が 5 タブ + 4 sub-tab エディタとしてこれを編集する。

本ドキュメントは **Pianist Profile のディレクトリ構造 + 各ファイルのスキーマ + section marker DSL + 16 関連 Models** を扱う。Editor 画面の UI 操作は [fabriq_studio__apps__02_pianist_profile_editor.md](fabriq_studio__apps__02_pianist_profile_editor.md)。

---

## ディレクトリ構造

```
{workspace}/modules/extended/pianist/profiles/<name>/
├── pianist.json                    ← メタデータ（schema=1）
├── procedure.csv                   ← 全 Phase の Step 列（フラット 8 列）
├── values.csv                      ← per-host 変数テーブル（wide format）
├── shortcuts.csv                   ← ファイル参照ショートカット
├── instructions/                   ← Phase 単位の手順書 + DSL
│   ├── <PhaseID1>.txt
│   ├── <PhaseID2>.txt
│   └── ...
└── screenshots/                    ← [Samples] が参照する画像
    ├── before.png
    ├── after.png
    └── ...
```

`<name>` は **半角英数 + アンダースコア**（フォルダ名 = プロファイル ID）。`pianist_list.csv` の `ProfileName` 列でこれを参照する。

`instructions/<PhaseID>.txt` の `<PhaseID>` は `procedure.csv` の `PhaseID` 列と 1 対 1 対応。**`procedure.csv` に存在しない PhaseID の `.txt`（孤児ファイル）も pianist.ps1 は無視するため、Studio 側も読み込んでおくだけで実害なし**（保存時に削除するわけでもない）。

---

## 1. `pianist.json` — メタデータ

[Models/PianistProfileMetadata.cs](file:///E:/fabriq_studio/FabriqStudio/Models/PianistProfileMetadata.cs)。`ObservableObject` + `JsonPropertyName` / `JsonPropertyOrder` 指定で **pianist.ps1 が読む snake_case 厳密書式** に合わせる。

### スキーマ

```json
{
  "schema": 1,
  "label": "Office 365 Setup",
  "description": "テナント参加 + Office アクティベーション",
  "target_app": "Office",
  "default_phase": "P01_signin",
  "version": "0.1.0"
}
```

| 列 | 型 | 順序 | 意味 |
|---|---|---|---|
| `schema` | int | 0 | スキーマバージョン（現状固定 `1`） |
| `label` | string | 1 | UI 表示名 |
| `description` | string | 2 | 説明文 |
| `target_app` | string | 3 | 対象アプリ名（例: `Office` / `kintone`）|
| `default_phase` | string | 4 | 起動時の初期 Phase ID |
| `version` | string | 5 | プロファイル版（既定 `"0.1.0"`） |

### 出力規約

- **UTF-8 BOM なし** + **LF** + **末尾改行 1 つ**
- `JsonPropertyOrder` でキー順を fixed（diff 安定）
- 不正 JSON でも Studio はフォールバック（`Label = フォルダ名`、`Description = "(invalid pianist.json)"`）してエディタを起動できる

### 不在時のフォールバック

`pianist.json` 不在 → Studio は **フォルダ名を Label に流用** した空 metadata で動作（pianist.ps1 L313-315 と同じセマンティクス）。

---

## 2. `procedure.csv` — Step 列（v1.5.0 フラット 8 列）

[Models/PianistStep.cs](file:///E:/fabriq_studio/FabriqStudio/Models/PianistStep.cs)。

### 列順

```csv
PhaseID,PhaseLabel,Color,StepNo,Action,Value,Wait,Note
P01,サインイン,#4CAF50,1,Open,https://www.office.com/login,2000,Office login page
P01,サインイン,#4CAF50,2,Wait,5000,,後続 SendKeys のため
P01,サインイン,#4CAF50,3,SendKeys,$AccountId{ENTER},500,
```

| 列 | 型 | 意味 |
|---|---|---|
| `PhaseID` | string | Phase 識別子（同 ID の行群が 1 Phase）|
| `PhaseLabel` | string | UI 表示用 Phase 名（先頭 Step の値が代表値、denormalize 保持）|
| `Color` | string | UI 用 Phase バッジ色（`#RRGGBB` or 名前）|
| `StepNo` | int | Phase 内の Step 順序 |
| `Action` | string | アクション英名（`Open` / `Wait` / `WaitWin` / `SendKeys` / `Paste` / `Click` / `Key` 等）|
| `Value` | string | アクションの引数（例: SendKeys なら送信文字列）|
| `Wait` | string | 後続待機 ms（Action により意味が変わる、後述）|
| `Note` | string | 自由記述メモ |

### v1.4.0 → v1.5.0 の差分

v1.4.0 までは末尾に `Screenshot` 列があった（一度も runtime で消費されなかった vestigial 列）。v1.5.0 で撤去され、見本画像参照は **`instructions/<PhaseID>.txt` の `[Samples]` section** に統一された（後述）。

レガシー 9 列 CSV 読み込みは CsvHelper の暗黙挙動（モデルに無いヘッダーは drop）で **透過対応**。Studio で開いて保存すると 8 列形式に正規化される。

### Action / Color の英名規約

- CSV には **英名で書く**（pianist.ps1 の switch が英名前提）
- Studio UI は **日本語ラベル**で提示し、保存時に英名へ書き戻す
- 変換は ViewModel 側の `PianistActionOption` を介して行う（後述 §13）

### `Wait` 列の意味は Action 依存

| Action | `Wait` の意味 |
|---|---|
| `WaitWin` | ウィンドウ待ちタイムアウト ms |
| `Wait` | 停止 ms（ただし `Value` 列が優先、後方互換用）|
| その他 | Step 完了後の待機 ms |

CSV スキーマは触らず Studio UI で吸収する（pianist 改修なし）。

### 寛容パース設定

`PianistProfileService.TolerantReadConfig`：

```csharp
new CsvConfiguration(CultureInfo.InvariantCulture)
{
    BadDataFound      = null,    // 未エスケープの " を含むセル許容
    MissingFieldFound = null,    // 行末コンマ不足 → 末尾列を空文字補完
}
```

PowerShell 側 `Import-Csv` は寛容だが、CsvHelper は RFC 4180 厳格。sample profile の自然な表記（kintone profile の Note `"Kintone"` 強調 / 行末コンマ不足）が落ちないよう **Pianist のみ** 寛容化。共通 `ICsvService` は触らない（他 CSV の厳格性を維持）。

---

## 3. `values.csv` — wide format / per-host 変数テーブル

[Models/PianistValueTable.cs](file:///E:/fabriq_studio/FabriqStudio/Models/PianistValueTable.cs) + [PianistValueRow.cs](file:///E:/fabriq_studio/FabriqStudio/Models/PianistValueRow.cs)。pianist v1.1.0 で wide format に移行（旧 long format は legacy 検出 + Phase 7 で自動変換予定）。

### スキーマ（wide format）

```csv
NewPCName,AccountId,Password,TenantUrl
*,user@example.com,ENC:U2FsdGVkX1+...,https://example.tenant.com
NEW-PC-01,user01@example.com,ENC:U2FsdGVkX1+...,
NEW-PC-02,user02@example.com,ENC:U2FsdGVkX1+...,
```

- **1 列目固定**: `NewPCName`
- **2 列目以降**: 変数列（`AccountId` / `Password` / `TenantUrl` ...、列数可変、ヘッダー順保持）
- **先頭固定 1 行**: `NewPCName=*` の **共通デフォルト行**（fall-through ソース）

### `*` 行の役割（fall-through）

個別ホスト行（`NEW-PC-01` 等）のセルが空のとき、**pianist.ps1 が自動的に `*` 行へフォールバック**して値を解決する。Studio の grid 表示も同じ意味論で `*` 行値を **dim italic** で継承表示する。

実装：`PianistValueTable.OnStarPropertyChanged` が `*` 行のセル変更を購読 → 各依存行（非 `*` 行）の `RaiseCellChanged(col)` を呼んで再評価通知を伝播 → grid の dim italic 表示が即座に更新される。

`PianistValueTable.EnsureStarRow()`：

- 既存 `*` 行があれば先頭（index 0）に移動
- 無ければ新規生成（全変数列に空セル）
- 各行の `Table` 逆参照を貼り直す（service ロード後に呼ばれた場合の保険）

### セル値の表現

セル値は **生の文字列**：

- 平文: `user@example.com`
- 暗号化: `ENC:U2FsdGVkX1+...`（[fabriq_studio__contracts__crypto_interop.md](fabriq_studio__contracts__crypto_interop.md) 準拠）

暗号化／復号は **HostDetail と同じ右クリック ContextMenu UX** で行い、保存時の自動変換はしない（保存はメモリ表現をそのまま書く）。

### 列名の変更通知

`PianistValueRow.this[string column]` インデクサが set 時に **二重通知** を発火：

```csharp
OnPropertyChanged($"Item[{column}]");   // 該当列の MultiBinding 個別通知
OnPropertyChanged("Item[]");            // 任意のインデクサ値変更（標準）
```

WPF の MultiBinding sub-binding が個別通知を取りこぼすケースに備えた防御策。HostDetail の単一 binding と異なり pianist の grid 表示は MultiBinding 経由のため信頼性を高める。

### Legacy long format 検出

旧 long format（pianist 1.0.0）の列：

```csv
Key,Value,Encrypted,Note
AccountId,user@example.com,0,
Password,xxx,1,
```

[Models/PianistLegacyValueEntry.cs](file:///E:/fabriq_studio/FabriqStudio/Models/PianistLegacyValueEntry.cs) でモデル化。検出ロジック：

- ヘッダーに `NewPCName` 列が **無い** → `WasLegacyFormat = true`
- `VariableColumns` / `Rows` は空のまま返る
- Studio は「新形式に変換しますか？」ダイアログを出す責務（Phase 7 で実装予定）

移行時の対応：`Key` → wide 列名、`Value` → `*` 行のセル値、`Encrypted="1"` → 値の頭に `ENC:` prefix を付与。

---

## 4. `shortcuts.csv` — ファイル参照ショートカット

[Models/PianistShortcut.cs](file:///E:/fabriq_studio/FabriqStudio/Models/PianistShortcut.cs)。

```csv
Label,Type,Path,Args,Note
公式マニュアル,URL,https://docs.example.com,,設定ガイド
社内 Wiki,File,\\fileserver\wiki\office365.html,,
```

| 列 | 意味 |
|---|---|
| `Label` | UI ラベル |
| `Type` | `URL` / `File` 等 |
| `Path` | URL or ローカルパス |
| `Args` | コマンドライン引数（任意）|
| `Note` | メモ |

`pianist.ps1 v1.x` は UI から呼んでおらず、**ファイル参照のみ**（ユーザーが手動でアクセス）。Studio v1 では単純な表 grid で編集できる程度に留める。

---

## 5. `pianist_list.csv` — モジュール側登録

[Models/PianistListEntry.cs](file:///E:/fabriq_studio/FabriqStudio/Models/PianistListEntry.cs)。本ファイルは `modules/extended/pianist/pianist_list.csv` に置かれ、**どの Profile を fabriq から呼べるようにするか** を宣言する：

```csv
Enabled,ProfileName,Group,Description,Segment
1,office_setup,Setup,Office 365 設定,
1,kintone_login,Daily,Kintone ログイン,daily
0,test_profile,Test,テスト用,
```

| 列 | 意味 |
|---|---|
| `Enabled` | `1`/`0` 文字列。Studio 内は `bool` |
| `ProfileName` | `profiles/<Name>/` の **フォルダ名と完全一致**（実在必須）|
| `Group` | グループ表示名 |
| `Description` | 説明 |
| `Segment` | profile 経由実行時の `$env:FABRIQ_SEGMENT` filter キー |

`pianist.ps1 Step 1` で `Import-ModuleCsv -FilterEnabled` で読まれ、`Enabled=1` 行が候補となる。`ProfileName` が `profiles/<Name>/` に実在しないと pianist.ps1 Step 2 で Error 終了。**Studio 側ではドロップダウン追加時に自動でフィルタ**され、タイポ事故を防ぐ。

---

## 6. `instructions/<PhaseID>.txt` — section marker DSL

[Models/PianistInstructionFile.cs](file:///E:/fabriq_studio/FabriqStudio/Models/PianistInstructionFile.cs) + [Helpers/PianistInstructionParser.cs](file:///E:/fabriq_studio/FabriqStudio/Helpers/PianistInstructionParser.cs)。

pianist v1.4.0 で導入された 4 section 構造の DSL。**pianist.ps1 の `Parse-PianistInstructionFile`（`E:\fabriq\modules\extended\pianist\pianist.ps1` L937 以降）とバイト単位で同一セマンティクス**。Studio で書いて pianist で読む round-trip と、pianist で書いて Studio で読む round-trip の両方を保証。

### 書式例

```
[RPA]
このフェーズで pianist が実行する SendKeys / Open は以下:
- Office login URL を Edge で開く
- メールアドレス入力 → ENTER
- パスワード入力 → ENTER

[Manual]
1. ブラウザで Office Portal が開いたら、通知ポップアップを閉じる
2. プロフィールアイコンに表示されるアカウント名が想定と一致するか確認
3. 違う場合はサインアウトしてやり直す

[Variables]
AccountId
Password
TenantUrl

[Samples]
office_login_screen.png  ログイン画面
office_dashboard.png     ダッシュボードに到達した状態
```

### 4 sections

| section | 用途 | UI sub-tab |
|---|---|---|
| `[RPA]` | pianist が自動実行する手順の概要（人間向け） | RPA |
| `[Manual]` | 人間が手作業で行う手順 | Manual |
| `[Variables]` | このフェーズで参照する変数の明示宣言 | Variables |
| `[Samples]` | 参考画像の参照（`screenshots/` 配下のファイル） | Samples |

### パース規則（5 件）

| 規則 | 詳細 |
|---|---|
| section header | 正規表現 `^\[([A-Za-z]+)\]$`（行 trim 後の完全一致）|
| 改行 | CRLF / LF / CR を吸収、内部表現は LF |
| Pre-section 行 | 最初の `[Section]` 直前までの行は `_Pre` バケツ。最終的に Manual の先頭へ折り込む（lenient parsing）|
| Legacy fallback | section が一つも無いファイルは **全文を Manual に流し込み**、`WasLegacyFormat = true` 立てる |
| section 内の改行 | LF → CRLF 正規化 + 末尾の `\r\n` / 空白をトリム |

### `[Variables]` のパース

- 区切り: `,`、空白、タブのいずれか（`[,\s]+`）
- 検証: `^[A-Za-z_][A-Za-z0-9_]*$`（不一致なら drop）
- `#` で始まる行はコメント
- 空行はスキップ

```
[Variables]
AccountId, Password
# 以下は別グループ
TenantUrl
ServerName
```

→ `["AccountId", "Password", "TenantUrl", "ServerName"]`

### `[Samples]` のパース

- 各行: `^(\S+)(\s+(.+))?$`（ファイル名 + 任意の caption）
- `#` で始まる行はコメント
- 空行はスキップ

```
[Samples]
before.png  ログイン前の画面
after.png   ログイン後のダッシュボード
no_caption.png
```

→ 3 件の `PianistSampleEntry`（[Models/PianistSampleEntry.cs](file:///E:/fabriq_studio/FabriqStudio/Models/PianistSampleEntry.cs)）：

```csharp
public partial class PianistSampleEntry : ObservableObject
{
    [ObservableProperty] private string _file = "";
    [ObservableProperty] private string _caption = "";
    [ObservableProperty] private bool _exists;        // 物理ファイル存在判定（外部 set）
    public bool IsMissing => !Exists;
    public string DisplayName => Exists ? File : $"(missing) {File}";
}
```

物理ファイルは `<profile>/screenshots/<File>`。欠損時 `"(missing) <name>"` のプレースホルダ表示（pianist runtime も同様）。

### シリアライズ規則（3 件）

| 規則 | 詳細 |
|---|---|
| 出力順 | `[RPA]` → `[Manual]` → `[Variables]` → `[Samples]` 固定 |
| 空 section の省略 | 中身が空の section は出さない |
| 改行 | CRLF + 末尾改行 1 つ |

`[Variables]` は **1 行 1 変数で書く**（カンマ区切りも合法だが diff の見やすさ優先）。

---

## 7. Pianist 16 Models 索引

| # | Model | 役割 |
|---|---|---|
| 1 | `PianistProfileEntry` | プロファイルフォルダ 1 件のエントリ（Name + FolderPath）|
| 2 | `PianistProfileMetadata` | `pianist.json` の 6 フィールド（schema/label/description/target_app/default_phase/version）|
| 3 | `PianistProfileData` | 1 プロファイルの全データ集約（Entry/Metadata/Steps/Values/Shortcuts/Instructions） |
| 4 | `PianistStep` | `procedure.csv` の 1 行（8 列）|
| 5 | `PianistShortcut` | `shortcuts.csv` の 1 行（5 列）|
| 6 | `PianistValueTable` | `values.csv` 全体（VariableColumns + Rows + Star + WasLegacyFormat）|
| 7 | `PianistValueRow` | `values.csv` の 1 行（NewPCName + Cells dictionary）|
| 8 | `PianistInstructionFile` | `instructions/<PhaseID>.txt` パース結果（RPA/Manual/Variables/Samples + WasLegacyFormat）|
| 9 | `PianistSampleEntry` | `[Samples]` 1 行（File/Caption/Exists 物理判定）|
| 10 | `PianistPhaseSummary` | Phase 一覧表示用集約（PhaseID/PhaseLabel/Color/StepCount）|
| 11 | `PianistKeyPreset` | SendKeys プリセット（Code + Description、ComboBox `TextSearch.TextPath="Code"` バインド）|
| 12 | `PianistActionOption` | Action 列の選択肢（Code 英名 + Label 日本語、`SelectedValuePath=Code` / `DisplayMemberPath=Label`）|
| 13 | `PianistListEntry` | `pianist_list.csv` の 1 行（5 列）|
| 14 | `PianistVariableSelection` | Variables sub-tab の編集状態（Name + IsIncluded + IsAutoDiscovered + SampleValue + IsOrphan）|
| 15 | `PianistValidationIssue` | 整合性チェック結果（Severity Error/Warning/Info + Category + Message + Source）|
| 16 | `PianistLegacyValueEntry` | 旧 long format values.csv の 1 行（Key/Value/Encrypted/Note、移行用）|

---

## 8. Variables sub-tab の auto-discover

`PianistVariableSelection`（# 14）は **3 つの情報源** を統合する：

```
1. values.csv の VariableColumns       ← 絶対参照源（列定義そのもの）
2. instructions/<PhaseID>.txt の [Variables]  ← 明示宣言
3. procedure.csv の Step.Value / Step.Note 内 $<name> 参照  ← auto-discover
```

| Selection 状態 | 意味 |
|---|---|
| `IsIncluded=true` | `[Variables]` section に書き出す（チェックボックス相当） |
| `IsAutoDiscovered=true` | 該当 Phase の Step が `$Name` として参照している（UI に「🔍 auto」バッジ表示）|
| `IsOrphan=true` | `[Variables]` にあるが values.csv に列が無い孤児（赤背景で分離表示 + 削除ボタン）|

pianist.ps1 は **auto-union する**ため、`[Variables]` に書かなくても auto-discovered 変数は Copy Values に出る。Studio が `IsIncluded` チェックボックスを公開するのは **「これは Step が使っているので含めるのが自然」というユーザの明示的な意思を尊重して round-trip を破壊しない** ため。

---

## 9. ファイル I/O の規約

`PianistProfileService.LoadProfileAsync` の処理順：

```
1. LoadMetadataAsync     ← pianist.json
2. LoadStepsAsync        ← procedure.csv
3. LoadValuesAsync       ← values.csv（wide format check + EnsureStarRow）
4. LoadShortcutsAsync    ← shortcuts.csv
5. LoadInstructions      ← instructions/*.txt（PianistInstructionParser.Parse）
```

### エンコーディング

| ファイル | 読み込み | 書き込み |
|---|---|---|
| `pianist.json` | UTF-8 BOM 自動吸収（`File.OpenRead` + `JsonSerializer`） | UTF-8 BOM **無し** + LF + 末尾改行 1 つ（§10 規約）|
| `procedure.csv` / `values.csv` / `shortcuts.csv` | UTF-8（`StreamReader Encoding.UTF8`、BOM 有無は CsvHelper が吸収）| BOM 付き UTF-8（fabriq 本体と一致）|
| `instructions/*.txt` | UTF-8（BOM / 改行形式吸収、内部 LF 正規化）| CRLF + 末尾改行 1 つ |

### 寛容パース

`PianistProfileService.TolerantReadConfig`（前述）が `procedure.csv` / `values.csv` / `shortcuts.csv` の読み込みに適用される。`pianist.json` 不正時はフォールバック metadata で動作。

---

## 10. Pianist テスト実行（PianistTestRunService）

Studio から fabriq エンジンの `pianist.ps1` を **子プロセス起動** してテスト実行できる機能（kernel 3.x、studio v1.4 で追加）。

### 子プロセス起動

`Start-Process powershell.exe -ArgumentList "-File pianist.ps1 -ProfileName <name> -TestRun"` の形式（exact args は `IPianistTestRunService.cs` 参照）。Studio は Console output を読み取らず、**子プロセスの完了を待ってログファイルを参照**する。

### Test Run の制限

- Studio から実行した場合は **AutoLogon / 再起動跨ぎ機能は無効**（fabriq 本体のメインダッシュボード経由でのみ有効）
- evidence base path は studio セッション内の一時フォルダ

詳細は [fabriq_studio__apps__02_pianist_profile_editor.md](fabriq_studio__apps__02_pianist_profile_editor.md) §「Test Run」を参照。

---

## 11. Validation（PianistValidationIssue）

`PianistValidationIssue`（Severity = `Error` / `Warning` / `Info`）は Studio 内の整合性チェックで使う：

| Severity | 例 |
|---|---|
| `Error` | `pianist.json` 不在 / `default_phase` が procedure.csv に存在しない / values.csv の `*` 行欠損 |
| `Warning` | `[Variables]` の orphan / `screenshots/` の参照ファイル欠損 / Action 英名の typo |
| `Info` | `[Manual]` 空 / レガシー long format 検出 |

`Source` フィールドに具体的な発生箇所（`PhaseID, Step 番号, 列名` 等）を記録。Editor 上部に Issue 一覧を表示してジャンプできる。

---

## 12. Hot 領域: pianist v1.5.0+ の動向

直近 2 週間（〜 2026-05-07）の commit:

| 日付 | 内容 |
|---|---|
| 2026-05-04 | `pianist_v1.5_phaseD_follow_up`、`pianist_profile_delete_add` |
| 2026-05-03 | `pianist_test_run_add`、`pianist_list_csv_editor_add`、`pianist_step_drag_drop_reorder_add`、`pianist_window_picker_add`、`pianist_key_presets_extend / repeat_examples_add` |
| 2026-05-02 | `pianist_profile_editor_add`（プロファイル削除機能の最初） |

短期間に集中的な機能追加。本ドキュメントが対象とする v1.5.0 系の安定後、v1.6.0 では Run 中の Stop / Pause / Speed ボタン追加（fabriq 本体 CHANGELOG `[Unreleased]`）が予告されている。

---

## 13. Action 英名と日本語ラベルの対応（暫定）

`PianistActionOption` の Code / Label ペアは ViewModel 側でハードコードされている（`PianistProfileEditorViewModel`）：

| Code（CSV 英名） | Label（日本語）| 用途 |
|---|---|---|
| `Open` | 開く | URL / ファイルを既定アプリで起動 |
| `Wait` | 待つ | 指定 ms 停止 |
| `WaitWin` | ウィンドウ待ち | タイトル一致のウィンドウが出るまで polling |
| `SendKeys` | キー送信 | SendKeys で文字列 / 特殊キー送信 |
| `Paste` | ペースト | クリップボード経由で貼り付け |
| `Click` | クリック | 座標 / 要素クリック |
| `Key` | キー単発 | 単一キー送信 |
| ... | ... | ... |

完全な一覧は `PianistProfileEditorViewModel.ActionOptions` を参照（時期によって追加が入る）。

---

## 関連ドキュメント

- Pianist Profile Editor の UI 詳細（5 タブ + 4 sub-tab エディタ）: [fabriq_studio__apps__02_pianist_profile_editor.md](fabriq_studio__apps__02_pianist_profile_editor.md)
- 全体像と他編集機能: [fabriq_studio__apps__01_main_pages.md](fabriq_studio__apps__01_main_pages.md)
- 暗号化アルゴリズム: [fabriq_studio__contracts__crypto_interop.md](fabriq_studio__contracts__crypto_interop.md)
- ワークスペース構造（profiles ディレクトリの位置）: [fabriq_studio__architecture__02_workspace.md](fabriq_studio__architecture__02_workspace.md)
- fabriq 本体側 pianist モジュール: [fabriq__modules__pianist.md](fabriq__modules__pianist.md)
- fabriq_studio Models 索引（37 Models 全体）: [fabriq_studio__reference__models_catalog.md](fabriq_studio__reference__models_catalog.md)
- fabriq_studio Services 索引: [fabriq_studio__reference__services_catalog.md](fabriq_studio__reference__services_catalog.md)
