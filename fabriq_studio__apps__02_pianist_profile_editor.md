# Pianist Profile Editor

> **対象**: fabriq_studio / apps / Pianist Profile Editor
> **対象バージョン**: commit `3897c6e`（procedure.csv は pianist v1.5.0 / 8 列スキーマ）
> **ドキュメント更新日**: 2026-05-06

fabriq の `modules/extended/pianist/` で動作する **簡易 RPA + 手順書ハイブリッドモジュール「Pianist」** のプロファイルを編集するエディタ。Studio の中で最も複雑な機能であり、現在も活発に拡張が続いている領域（直近の git log 上位 15 コミットのうち 8 件が Pianist 関連）。

---

## Pianist の位置付け

Pianist は fabriq の拡張モジュールの 1 つで、`modules/extended/pianist/profiles/<name>/` フォルダ単位でプロファイルを保持する。1 プロファイル = 1 つの「手順」を表し、以下を組み合わせて記述する:

- **RPA 動作**（自動化）: SendKeys / WaitWin / Open など、PC 操作の自動化
- **Manual 動作**（人手）: 手順書として人間に表示するメッセージ
- **Variables**: 端末別に切り替えるパラメータ
- **Samples**: 操作前後の見本画像参照

実行は `pianist.ps1`（`E:\fabriq\modules\extended\pianist\pianist.ps1`）が担当し、Studio はそのプロファイルファイルを編集する I/O ツールに徹する。

---

## プロファイルのファイル構成

```
modules/extended/pianist/profiles/<profile_name>/
├── pianist.json                  ── メタ情報（schema / label / description / target_app / default_phase / version）
├── procedure.csv                 ── Phase × Step の主表（8 列、v1.5.0 スキーマ）
├── values.csv                    ── 変数テーブル（wide format、`*` 行 + ホスト別行）
├── shortcuts.csv                 ── 手動起動用ショートカット（v1.x は参照のみ）
├── instructions/
│   ├── P01.txt                   ── PhaseID ごとの section marker DSL
│   ├── P02.txt
│   └── ...
└── screenshots/
    └── *.png                     ── Samples で参照する画像
```

### ファイル別エンコーディング規約（§10）

- `pianist.json`: BOM 無し UTF-8 + LF + 末尾改行 1 つ
- `*.csv`: BOM 付き UTF-8 + CRLF（Excel 互換性のため）
- `instructions/*.txt`: BOM 無し UTF-8 + LF + 末尾改行 1 つ

Studio の `IPianistProfileService.SaveProfileAsync` はこの規約を厳守して書き出す（pianist.ps1 とのバイト一致 round-trip を保証するため）。

---

## エディタの 5 タブ

`PianistProfileEditorView.xaml` は次の 5 タブで構成される:

### タブ 1: メタ（Metadata）

`pianist.json` のフィールドを直接編集:

| フィールド | 型 | 用途 |
|---|---|---|
| `schema` | int | スキーマバージョン（現行 1） |
| `label` | string | UI 表示用の人間可読名 |
| `description` | string | 概要説明 |
| `target_app` | string | 対象アプリ識別子（任意） |
| `default_phase` | string | 起動直後のデフォルト Phase ID |
| `version` | string | プロファイル個別版（SemVer） |

Model: [Models/PianistProfileMetadata.cs](file:///E:/fabriq_studio/FabriqStudio/Models/PianistProfileMetadata.cs)。`[JsonPropertyOrder]` で出力時のキー順を固定（schema → label → description → target_app → default_phase → version）。

### タブ 2: Phase 一覧（procedure.csv の表示・編集）

`procedure.csv` の全行を Phase でグルーピングして表示。

#### procedure.csv の 8 列スキーマ

```
PhaseID, PhaseLabel, Color, StepNo, Action, Value, Wait, Note
```

- `PhaseID` — Phase の一意 ID（例: `P01`）
- `PhaseLabel` — Phase 表示名
- `Color` — Phase の表示色（CSV 上は英名 `Red`/`Blue`/...、UI では日本語ラベル変換）
- `StepNo` — Phase 内の Step 番号
- `Action` — `SendKeys`, `WaitWin`, `Open`, `Wait`, `Manual` 等。CSV 上は英名（pianist.ps1 の switch 前提）
- `Value` — Action のパラメータ
- `Wait` — Action 種別で意味が変わる（§4.4）:
  - `WaitWin`: タイムアウト ms
  - `Wait`: 停止 ms（ただし Value 列が優先）
  - その他: 後続待機 ms
- `Note` — メモ

> **v1.4.0 → v1.5.0 の変更**: 末尾に `Screenshot` 列があったが撤去。一度も runtime で消費されなかった vestigial 列。見本画像参照は `instructions/<PhaseID>.txt` の `[Samples]` section に統一。Studio はレガシー 9 列 CSV を読み込んだ際、CsvHelper の暗黙挙動で `Screenshot` を drop し、保存時に 8 列に正規化する。

#### Step の編集機能

- Step 行の **Drag & Drop 並び替え**（pianist_step_drag_drop_reorder_add コミット）
- Step 追加・削除
- Action 種別による入力ヒント（Key Preset 例の繰り返し: pianist_key_presets_repeat_examples_add）
- Phase の編集ダイアログ: `PianistPhaseEditDialog`、`PianistPhaseDeleteDialog`
- Phase 追加: `PianistColumnNameDialog`

Model: [Models/PianistStep.cs](file:///E:/fabriq_studio/FabriqStudio/Models/PianistStep.cs)。

### タブ 3: 変数（values.csv の wide format）

ホスト別の変数値テーブル。

#### values.csv の wide format（v1.x 系）

```
NewPCName, var_a, var_b, var_c, ...
*       ,  1   ,  X  ,     ,
PC-001  ,      ,     ,  Y  ,
PC-002  ,  2   ,     ,     ,
```

- 1 行目はヘッダー: `NewPCName` 固定 + 変数列名
- `*` 行 = **共通デフォルト**。Studio UI で先頭に固定表示（[Models/PianistValueTable.cs](file:///E:/fabriq_studio/FabriqStudio/Models/PianistValueTable.cs) の `EnsureStarRow`）
- 通常行は `NewPCName` でホスト指定。空セルは `*` 行から継承
- セル値は暗号化（`ENC:` プレフィックス付）または平文。両者を混在させて保存できる

#### `*` 行の継承表示

`*` 行のセル値が変更されると、依存行（非 `*`）の同列が空であれば即座に再描画される（`PianistValueTable.SubscribeStarChanges` で `Item[col]` PropertyChanged を中継し、各行の `RaiseCellChanged(col)` を発火する仕組み）。

Studio 側 DataGrid は dim italic で「継承中」を表現する（`PianistValueTemplateSelector` + `PianistCellInheritedConverter` + `PianistCellDisplayConverter`）。

#### 暗号化セルのハンドリング

- セルを右クリック → ContextMenu で「暗号化」「復号」「平文表示の切替」を明示操作
- パスフレーズ未設定時は `CryptoHelper.ValidatePassphrase` でブロック
- `IPianistProfileService.SaveProfileAsync` は **メモリ上の表現（暗号文 or 平文）をそのまま** values.csv に書き出す（自動暗号化／復号は行わない）

#### Legacy long format（v1.x 以前）の検出

旧形式（`Key`, `Value`, `Encrypted`, `Note` の 4 列）を検出した場合、`PianistValueTable.WasLegacyFormat = true` でフラグ立てする。Studio は読み込み時に「新形式に変換しますか？」ダイアログを出す責務を持つ（Phase 7 で実装予定）。読み込み専用 API として `IPianistProfileService.LoadLegacyValuesAsync` を持つ。

### タブ 4: ショートカット（shortcuts.csv）

`shortcuts.csv`（`Label, Type, Path, Args, Note`）を表で編集。pianist.ps1 v1.x は UI から呼んでいないファイル参照のみのため、Studio v1 では単純な表 grid で編集する程度。Model: [Models/PianistShortcut.cs](file:///E:/fabriq_studio/FabriqStudio/Models/PianistShortcut.cs)。

### タブ 5: プレビュー

実行時の手順表示をシミュレートする読み取り専用ビュー。Phase 別に instruction の 4 セクションがどう見えるかをユーザに見せる。

---

## Phase ごとの instruction エディタ（4 sub-tab）

タブ 2 で Phase を選択すると、その Phase の `instructions/<PhaseID>.txt` を **section marker DSL** として 4 sub-tab で編集する。

### DSL の構造

```
[RPA]
（自動化に関する補足。pianist UI で表示）

[Manual]
（人手で行う手順書本文。pianist UI で表示）

[Variables]
var_a, var_b
var_c

[Samples]
before.png  キャプション
after.png
```

### パース規則（[Helpers/PianistInstructionParser.cs](file:///E:/fabriq_studio/FabriqStudio/Helpers/PianistInstructionParser.cs)）

Studio のパーサは pianist.ps1 の `Parse-PianistInstructionFile`（`E:\fabriq\modules\extended\pianist\pianist.ps1` L937 以降）と **バイト単位で一致** する仕様:

- section header は `^\[([A-Za-z]+)\]$`（行 trim 後の完全一致）
- section の前にある行は `_Pre` バケツに入り、最後に Manual 先頭へ折り込む
- section が一つも無いファイルは全文を Manual に流し込む（legacy 互換）
- `Variables` は `[,\s]+` で分割、`^[A-Za-z_][A-Za-z0-9_]*$` のみ採用、`#` コメント可
- `Samples` は `^(\S+)(\s+(.+))?$`、`#` コメント可、空行スキップ
- 改行は CRLF / LF / CR を吸収、内部表現は LF

### シリアライズ規則

出力は `[RPA]` → `[Manual]` → `[Variables]` → `[Samples]` の順。中身が空の section は省略。改行は CRLF、末尾改行 1 つ。

> **重要**: パーサの実装は Studio で書いて pianist で読む round-trip と、pianist で書いて Studio で読む round-trip の **両方** が成立しなければならない。テストは存在しないが、規則を厳守すれば成立する設計。

Model: [Models/PianistInstructionFile.cs](file:///E:/fabriq_studio/FabriqStudio/Models/PianistInstructionFile.cs)（`RPA` / `Manual` / `Variables` / `Samples` + `WasLegacyFormat`）。

---

## 補助ダイアログ（7 種）

| ダイアログ | 用途 |
|---|---|
| `PianistNewProfileDialog` | 新規プロファイル名入力（`IPianistProfileService.ValidateNewProfileName` で半角英数+`_` 制限） |
| `PianistColumnNameDialog` | values.csv の変数列追加・リネーム |
| `PianistPhaseEditDialog` | Phase ID / Label / Color の編集 |
| `PianistPhaseDeleteDialog` | Phase 削除前の確認 |
| `PianistWindowPickerDialog` | デスクトップ上のウィンドウタイトルを選択（`Helpers/WindowEnumerator` 経由）。`WaitWin` Action のパラメータ入力支援 |
| `PianistListEditDialog` | `pianist_list.csv` の編集（カタログ） |
| `PianistTestRunDialog` | プロファイルのテスト実行 + ログ表示 |

---

## カタログ — pianist_list.csv

`modules/extended/pianist/pianist_list.csv` は Pianist で実行可能なプロファイルのカタログ。Model: [Models/PianistListEntry.cs](file:///E:/fabriq_studio/FabriqStudio/Models/PianistListEntry.cs)。

CSV 列:

- `Enabled`（"1"/"0"）
- `ProfileName` — `profiles/<name>/` のフォルダ名と一致必須
- `Group`
- `Description`
- `Segment`

pianist.ps1 Step 1 は `Import-ModuleCsv -FilterEnabled` でこのファイルを読み、Enabled=1 行を実行候補とする。Studio 側ではドロップダウン追加時に「実在する `<ProfileName>/` フォルダ」のみフィルタするため、タイポによる pianist Step 2 Validate Error を未然に防ぐ。

エディタは `PianistListEditDialog` 経由で開く。`IPianistProfileService.LoadPianistListAsync` / `SavePianistListAsync` で I/O。

### プロファイル削除と catalog の関係

`IPianistProfileService.DeleteProfileAsync` は **`<name>/` フォルダ全体を物理削除** するが、**`pianist_list.csv` には触らない**。catalog に該当行が残った状態だと pianist.ps1 Step 2 Validate で Error 終了するため、Studio はその挙動をユーザに伝え、別操作で catalog を更新させる方針。

---

## Test Run — IPianistTestRunService

Studio から fabriq エンジン（pianist.ps1）に編集中のプロファイルを渡してテスト実行する機能。

### 実装上のキーポイント

[Services/IPianistTestRunService.cs](file:///E:/fabriq_studio/FabriqStudio/Services/IPianistTestRunService.cs) より:

- `powershell.exe -STA -EncodedCommand` で子プロセスを起動
- `kernel/common.ps1` を dot-source した上で `pianist.ps1` を呼ぶ（pianist.ps1 は kernel 関数群に強く依存し、単体実行不可）
- profile picker は `Import-ModuleCsv` を **モック上書き** することで合成 1 行を返し auto-skip（pianist.ps1:891 の `Items.Count -eq 1` shortcut を利用） — workspace 側 `pianist_list.csv` を改変しない
- `ICryptoService.MasterPassphrase` をラッパスクリプト内で `$global:FabriqMasterPassphrase` に注入し ENC: セルの復号を有効化
- GUI 操作待ちのため既定タイムアウトなし。`CancellationToken` 経由でユーザーがキャンセルしたときだけ `Process.Kill(entireProcessTree: true)`

### 結果の取り出し

pianist.ps1 が末尾に出力する Sentinel `===PIANIST_TEST_RESULT===` の直後の JSON を parse して `ModuleResultStatus` / `ModuleResultMessage` / `ModuleResultVerified` を抽出する。プロセスがクラッシュ / Kill された場合は `null`。

`PianistTestRunResult` レコード:

```csharp
public record PianistTestRunResult(
    string  Log,
    int     ExitCode,
    string? ModuleResultStatus,
    string? ModuleResultMessage,
    bool?   ModuleResultVerified,
    bool    WasCancelled);
```

UI 側は `PianistTestRunDialog` でログをリアルタイム表示し、結果ペインで Sentinel parse 結果を構造化表示する。

---

## I/O サービス（IPianistProfileService）

[Services/IPianistProfileService.cs](file:///E:/fabriq_studio/FabriqStudio/Services/IPianistProfileService.cs) で定義された API:

| メソッド | 役割 |
|---|---|
| `GetProfilesAsync()` | `profiles/` 直下のサブディレクトリ列挙 |
| `LoadProfileAsync(entry)` | json + 3 CSV + instructions/*.txt を全部読み込んで `PianistProfileData` で返す。欠損ファイルは空オブジェクトで埋める寛容処理（§7.1 と同じ哲学） |
| `SaveProfileAsync(data, crypto)` | §10 規約で書き出し。values.csv のセル値は暗号化／復号せずメモリ表現のまま |
| `LoadLegacyValuesAsync(entry)` | 旧 long format values.csv の読み出し |
| `ValidateNewProfileName(name)` | 半角英数+`_` のみ、未使用名 |
| `CreateNewProfileAsync(name)` | 5 ファイルを最小限 placeholder で生成（P01 に Wait 1000ms の Step 1 件 + `*` 行のみの空 values.csv） |
| `DeleteProfileAsync(entry)` | `<name>/` フォルダ物理削除（取り消し不可、pianist_list.csv には触らない） |
| `LoadPianistListAsync()` / `SavePianistListAsync()` | カタログ I/O |

---

## ViewModel との関係

`PianistProfileEditorViewModel`（Singleton）は:

1. `IPianistProfileService.GetProfilesAsync()` で一覧取得
2. ユーザーが選択したら `LoadProfileAsync(entry)` → `PianistProfileData` をプロパティに保持
3. 5 タブ + 4 sub-tab 全てがこの 1 つの `PianistProfileData` を共有
4. `IDirtyAwareViewModel` を実装（プロパティ群の変更を OR で集約）
5. 保存ボタンで `SaveProfileAsync` → 成否に応じてダイアログ表示

ワークスペース変更時は `IWorkspaceService.WorkspaceChanged` を購読してプロファイル一覧を再ロードする。
