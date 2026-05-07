# 管理者メモ（manager_memo.json）

> **対象**: fabriq_evidence_manager / usage
> **対象バージョン**: 3.8.1（取得元: `E:\fabriq_evidence_manager\FabriqEvidenceManager\FabriqEvidenceManager.csproj` `<Version>` / commit `45eae22`）
> **ドキュメント更新日**: 2026-05-07

PC 単位の自由記述メモ機能。fabriq 側の evidence サブツリーには触らず、**PC ルートディレクトリ直下の `manager_memo.json` ファイル** に永続化される。本アプリが evidence ツリー外に書き込む唯一の成果物。

---

## 何のためのメモか

評価対象 PC の検収中に「あとで担当者と確認」「キッティング作業者からの口頭情報」「Caution 黄色の許容判断根拠」等を **PC に紐付けて自由記述** で残す。Excel 台帳の備考欄では PC 行ごとに 1 セルしか取れず multi-line 記述が崩れるため、JSON 単独ファイルとして外出ししている。

主な用途：

- ベースライン差異の許容理由（"Office なしは案件特性で OK"）
- hostlist 不一致の業務判断（"DNS 4 件目は社内 DNS 移行中で許容"）
- 物理機側のメモ（"BIOS 設定変更履歴"）
- 引き継ぎ事項

---

## ファイル仕様

### 配置場所

```
{evidenceRoot}/
└── 2026_03_12_143025_NEW-PC-01_ABC123_evidence/    ← PC ルート（log_uploader 親フォルダ）
    ├── evidence/                                    ← fabriq 側出力（read-only、本アプリは触らない）
    │   ├── pc_information/
    │   ├── auto_capture/
    │   ├── bitlocker/
    │   ├── checklist/
    │   └── export_history/
    └── manager_memo.json                            ← 本アプリが書き込む唯一のファイル
```

`evidence/` の **外側** に置くのは fabriq 側の整合性を破らないため（evidence/ を再収集で上書きされてもメモは残る、CLAUDE.md R1 ソース不変）。

### スキーマ

```json
{
  "text": "管理者メモ本文（multi-line 可）\n2 行目\n3 行目",
  "lastUpdatedAt": "2026-05-07T14:32:11.234+09:00",
  "lastUpdatedBy": "suzuki"
}
```

- **PropertyNamingPolicy**: `JsonNamingPolicy.CamelCase`（C# の PascalCase プロパティを camelCase に変換）
- **WriteIndented**: true（人間可読）
- **Encoding**: UTF-8（BOM 無し、`File.WriteAllText(path, json, Encoding.UTF8)`）
- **ファイル名**: `PcMemoService.MemoFileName = "manager_memo.json"`（fabriq 側の他ツールと衝突しない名前を選択）

### Model 型

`Models/PcMemo.cs`：

```csharp
public sealed class PcMemo
{
    public string Text { get; init; } = string.Empty;
    public DateTimeOffset LastUpdatedAt { get; init; }
    public string LastUpdatedBy { get; init; } = string.Empty;
}
```

`init` プロパティのみで作成後変更不可。`Save` 時は新規 `PcMemo` を作って書き出す。

---

## 操作手順

### 1. メモを入力する

1. メイン画面 DataGrid で対象 PC をダブルクリック → `PcDetailWindow` 起動
2. ウィンドウ最上部のヘッダ直下、`◆ 管理者メモ` セクション
3. テキストボックス（`MinHeight=60 MaxHeight=120`、Wrap、Tab 入力なし）に記述

複数行記述可（`AcceptsReturn=True`）。Tab はフォーカス移動のみ（`AcceptsTab=False`）。

### 2. 保存する

`💾 保存` ボタン（`SaveMemoCommand`）：

- `Pc.PcRootDirectoryPath is null` の場合は `CanExecute=false` でボタン無効化（旧形式 evidence で PC ルートが特定できない場合）
- 保存成功で `Pc.Memo` プロパティを更新 → `MemoLastUpdatedDisplay` が `"最終更新: 2026/05/07 14:32 (suzuki)"` に変化
- メイン画面の DataGrid で **計画 PC 名列の末尾に `📝` アイコンが即時表示** される（`Pc.HasMemo` の通知経由）

`LastUpdatedBy` は `Environment.UserName`（Windows ログオンユーザー）を自動で入れる。手動指定 UI はない。

### 3. 既存メモを読む

`PcDetailWindow` 起動時、`PcDetailViewModel` のコンストラクタで `MemoText = pc.Memo?.Text ?? string.Empty` を設定。**Discovery 時に `NestedEvidenceDiscoveryService` が `manager_memo.json` を自動読み込み済み** のため、ウィンドウを開いた瞬間に既存メモが見える。

`MemoLastUpdatedDisplay`：

- 既存メモあり → `"最終更新: 2026/05/07 14:32 (suzuki)"`
- 未保存 → `"(未保存)"`

### 4. 削除する

UI 上の `削除` ボタンは v3.8.0 では未実装。**メモを空文字列にして保存** することで実質クリアできる（ファイルは残るが `Text=""` になり `HasMemo=false` で `📝` アイコンが消える）。

ファイル自体を完全に削除したい場合は `IPcMemoService.Delete(pcRootDirectory)` API が用意されているが、UI からの呼び出し経路はない（直接ファイルを `del manager_memo.json` で消すのが現状の運用）。

---

## DataGrid での視認

`PcEvidence.HasMemo`：

```csharp
public bool HasMemo => Memo is not null && !string.IsNullOrEmpty(Memo.Text);
```

`MainWindow.xaml` の計画 PC 名列：

```xml
<TextBlock Text=" 📝"
           FontSize="11"
           Foreground="#1565C0"
           ToolTip="管理者メモあり"
           Visibility="{Binding HasMemo, Converter={StaticResource BoolToVis}}"/>
```

メモを保存した PC は **アイコン付きで一覧化**、ToolTip ホバーで「管理者メモあり」が出る。フリート全体を眺めて「メモがある PC」が一目で分かる。

---

## ライフサイクル

| イベント | 動作 |
|---|---|
| アプリ起動 → evidence ロード | `NestedEvidenceDiscoveryService.ProcessFolder` 内で `pc.Memo = _memoService.Load(pcRootDir)` を実行。読み込み失敗（パース不能 / ファイル破損）は silent に null セット |
| 自動更新（30 秒） | 既存 PC のメモは **再読み込みしない**（インスタンス上の `Memo` を保持し続ける）。新規 PC のみメモが Discovery 時に読まれる |
| メモ保存 | `PcMemoService.Save(pcRootDir, memo)` → `JsonSerializer.Serialize` → `File.WriteAllText`。失敗時は例外を呼び出し側に伝播（UI 側でのエラーダイアログは未実装、将来課題） |
| evidence 再収集（fabriq 側） | manifest.json や pc_information/ が新しく書き換わるが、**`manager_memo.json` は触られない**（fabriq 側の出力対象外）。再収集後にメモがそのまま残る |
| アプリ終了 | メモリ上の Pc.Memo は破棄。次回起動時にファイルから再ロード |

---

## ファイル例

```
E:\evidence_root\2026_03_12_143025_NEW-PC-01_ABC123_evidence\manager_memo.json
```

```json
{
  "text": "Office 認証失敗の件:\n2026-05-06 顧客IT部門と協議し、本ロットでは Office 配備外で確定。\n本機の \"Office未インストール\" Warning は許容。\n\n参考: 案件管理票 #ACME-2026-Q2-PCS-049",
  "lastUpdatedAt": "2026-05-07T11:42:18.567+09:00",
  "lastUpdatedBy": "suzuki"
}
```

---

## トラブル対応

### `💾 保存` ボタンが押せない

`Pc.PcRootDirectoryPath` が null：

- evidence フォルダ命名規則が新形式 `_evidence` サフィックス無 → fabriq kernel を 2.2.2+ にして再収集する必要あり
- フォルダ階層が不正（log_uploader が中断した状態 等） → fabriq 側を修復

### 保存したのに次回起動時に消えている

- `manager_memo.json` がディスク書き込み失敗（権限不足 / 容量不足） → ファイルが実在するか確認
- `Pc.PcRootDirectoryPath` の指す先が変わっている（フォルダリネーム、ネットワークパス変更）→ 旧パス側に取り残されている可能性

### メモが文字化けする

- 手動でファイルを編集する場合は **UTF-8 で保存**（BOM 付きでも可）
- Notepad で開くと自動判別で BOM 無しが Shift-JIS 扱いされる場合あり → VS Code / Notepad++ 推奨

### メモを納品物に含めるか

- **`EvidenceCollectorService.CollectAsync` は `manager_memo.json` をコピーしない**：`pc_information / auto_capture / bitlocker / checklist / export_history` のみが対象
- メモはあくまで **検収作業者の作業ノート** で、顧客には見せない前提
- 「納品物に含めたい」場合は将来別途オプションが必要（現状はファイルを手動で納品ディレクトリにコピーする）

---

## 設計上の判断

### なぜ JSON 1 ファイル / PC か

- 1 PC 1 メモで履歴・複数メモ・コメント追加形式は将来課題
- SQLite 等のストレージは「納品時に PC ディレクトリ単位でフォルダ運搬する」用途と相性が悪い（DB ファイルを横断しない）
- JSON 単一ファイルなら **ファイルサーバ上で直接 grep / ls で確認可能**

### なぜ evidence/ の外側に置くか

- fabriq 側の `evidence_config` モジュールが `evidence/` 配下のファイルを再生成する可能性がある（再収集で上書き）
- evidence/ 内に置くと fabriq 側のクリーンアップ処理（`Clean-Evidence` 等）で消される懸念
- 「ファイル所有 = ツール所有」を明確化（`evidence/` = fabriq、`manager_memo.json` = evidence_manager）

### なぜ自動更新で再読み込みしないか

- 自動更新中に「メモ編集中」だと上書きしてしまう懸念
- メモは PC 単位で扱う性質上、複数ユーザーが並列編集する想定がない（同一 PC を 2 人が同時に検収するワークフローはない）
- 「ファイルを直接編集 → アプリは気付かない」を許容する方針

---

## 関連ドキュメント

- 取り込み手順（メモ自動読込のタイミング）: [fabriq_evidence_manager__usage__01_import.md](fabriq_evidence_manager__usage__01_import.md)
- 納品出力（メモは納品対象外の理由）: [fabriq_evidence_manager__usage__04_export_delivery.md](fabriq_evidence_manager__usage__04_export_delivery.md)
- PC 詳細ウィンドウ（メモ UI 詳細）: [fabriq_evidence_manager__apps__02_pc_detail_window.md](fabriq_evidence_manager__apps__02_pc_detail_window.md)
- 階層構造（PcMemoService 詳細）: [fabriq_evidence_manager__architecture__01_layers.md](fabriq_evidence_manager__architecture__01_layers.md)
