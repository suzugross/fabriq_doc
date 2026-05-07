# 納品データ出力の使い方

> **対象**: fabriq_evidence_manager / usage
> **対象バージョン**: 3.8.1（取得元: `E:\fabriq_evidence_manager\FabriqEvidenceManager\FabriqEvidenceManager.csproj` `<Version>` / commit `45eae22`）
> **ドキュメント更新日**: 2026-05-07

検収済みの evidence フリートを **顧客納品用にディレクトリ整形 + 改ざん検出ハッシュ付き + Excel 台帳付き** で出力する機能。元データには一切触らず、すべてコピーで完結する。

---

## いつ使うか

- フリート全 PC の検収（hostlist 突合・ベースライン突合・チェックリスト確認）が完了し、案件として「これで納品」と決まった段階
- 顧客への成果物として「PC 単位のエビデンスフォルダ群 + 一覧 Excel + ハッシュマニフェスト」をパッケージ化したい
- 案件の中間レビュー時、現時点までのエビデンスを提出したい

---

## 操作手順

### 1. 出力先親ディレクトリを選ぶ

メイン画面 Row 3 の `納品データ出力` ボタン → `OpenFolderDialog`：

1. 出力したい親ディレクトリを選ぶ（顧客フォルダ / プロジェクト保管庫 / Desktop 等）
2. `OK` 押下

選択したフォルダ直下に `{YYYY_MM_DD_HHmmss}_fabriq_evi/` という **タイムスタンプ付きフォルダが新規作成** される（`EvidenceCollectorService.CreateDeliveryDirectory`）。同日複数回出力しても秒精度で衝突しない。

### 2. コピー処理を待つ

ステータスバー: `フォルダ整理中... 5/10 台完了 (50%)` のように進捗が逐次更新（`IProgress<int>` で 0〜100）、LINK/ACT LED が緑点滅。

各 PC ごとに：

- `pc_information/` ディレクトリを再帰コピー
- `auto_capture/` ディレクトリを再帰コピー
- `bitlocker/` ディレクトリを再帰コピー
- `checklist.{ext}` 単独コピー（元の拡張子保持）
- `export_history.{ext}` 単独コピー（元の拡張子保持）
- `manifest_sha256.txt` を新規生成（PC ディレクトリ内全ファイルの SHA-256 ハッシュ列挙）

### 3. Excel 台帳の生成を待つ

全 PC のコピーが完了したら：

`Excel 台帳を出力中...`

→ `{YYYY_MM_DD_HHmmss}_PC情報一覧表.xlsx` を 納品ディレクトリ直下に生成 + 詳細データを持つ PC ごとに `pc_details/` サブディレクトリ内に **個別 .xlsx を分割出力**。

### 4. 完了ダイアログを確認

`MessageBox`：

```
納品データを出力しました。

出力先: E:\delivery\2026_05_07_143025_fabriq_evi
PC台数: 10 台
Excel:  2026_05_07_143025_PC情報一覧表.xlsx
```

コピー失敗ファイルがあった場合は警告ダイアログ：

```
以下のファイルのコピーに失敗しました（スキップ済み）:

E:\source\...\auto_capture\003.png: ファイルがロックされています
E:\source\...\bitlocker\PC02_C.txt: アクセス拒否
... 他 3 件
```

---

## 出力ディレクトリ構造

```
{outputRoot}/
└── {YYYY_MM_DD_HHmmss}_fabriq_evi/             ← 1 回の出力 = 1 ディレクトリ
    ├── 2026_05_07_143025_PC情報一覧表.xlsx     ← main 台帳
    ├── pc_details/                               ← PC 個別詳細（ある PC のみ）
    │   ├── NEW-PC-01_ABC123XYZ.xlsx
    │   ├── NEW-PC-02_DEF456UVW.xlsx
    │   └── ...
    ├── NEW-PC-01_ABC123XYZ_2026_03_12/          ← 1 PC = 1 サブフォルダ
    │   ├── pc_information/                       ── そのままコピー
    │   ├── auto_capture/                         ── そのままコピー
    │   ├── bitlocker/                            ── そのままコピー
    │   ├── checklist.html                        ── 単独コピー（拡張子保持）
    │   ├── export_history.csv                    ── 単独コピー（拡張子保持）
    │   └── manifest_sha256.txt                   ── 新規生成
    ├── NEW-PC-02_DEF456UVW_2026_03_12/
    ├── ...
    └── (PC 台数分のサブディレクトリ)
```

### PC サブフォルダ命名

```
{PcName}_{SerialNumber}_{CollectionDate}
```

優先順位:

1. SerialNumber が空なら `{PcName}` のみ
2. CollectionDate が空なら `{PcName}_{SerialNumber}`
3. すべて揃えば `{PcName}_{SerialNumber}_{CollectionDate}`

同名衝突時は **コピー側で `overwrite: true`** で上書き（同 PC の再出力に対応）。

### checklist / export_history の拡張子保持

元ファイル `checklist_20260312_143025.html` を `checklist.html` にリネームしてコピー。`.csv` も同様に `export_history.csv` にリネーム。**fabriq 側の細かい命名（タイムスタンプ等）は捨てる**ため、納品時に「中身が違う複数バージョン」を渡す事故が起きない。

### manifest_sha256.txt のフォーマット

```
3a7bd3e2360a3d29eea436fcfb7e44c735d117c42d1c1835420b6b9942dd4f1b  pc_information\01_SystemInfo.txt
9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08  pc_information\02_LocalUsers.csv
...
```

- 各行 = `{ハッシュ 64 桁小文字}  {納品ディレクトリ相対パス}`（hash と path の間にスペース 2 個、Linux `sha256sum` 形式に揃える）
- **manifest_sha256.txt 自身は除外**（自己参照を避ける）
- I/O 失敗ファイルは `[ERROR: {message}]  {relativePath}` 行で記録 + `errors` リストに追加

これにより納品先（顧客 / 第三者検収機関）でも `sha256sum -c manifest_sha256.txt`（Linux）/ `Get-FileHash -Algorithm SHA256` 突合（Windows）でファイル改ざん検出が可能。

---

## main Excel 台帳（30+ シート）

`ExcelExportService.ExportAsync` が ClosedXML で生成。シート構成は次の 3 グループ：

### グループ 1: フリート総覧（必ず出る）

- **PC情報一覧** — 1 行 / PC のフリート総覧（基本情報 + NIC + BitLocker + ライセンス + ベースライン突合判定 + 突合 OK/NG + 管理者メモ）。先頭 6 行にオプションで案件メタデータブロック（後述）。PC 名セルから `pc_details/` 個別 .xlsx へハイパーリンク
- **突合詳細** — hostlist 突合（VerificationReport）の項目展開
- **チェックリスト** — チェックリスト HTML パース結果
- **実行履歴** — export_history.csv の内容

### グループ 2: ベースライン突合（baseline 設定済みのとき）

- **ベースライン_実行サマリ** / **_SystemInfo** / **_チェックリスト** / **_アプリ_Desktop** / **_アプリ_Store** / **_ドメイン** / **_ライセンス**（v3.8.1 で `_アプリ` が `_Desktop` / `_Store` に分離）

### グループ 3: 各 PC のセクション横断（持っている PC があるとき条件付きで生成）

- **ドメイン状態** / **電源設定** / **Windows Defender** / **Windowsライセンス** / **Officeライセンス**
- **ユーザー・グループ** / **FWプロファイル** / **FWルール** / **オプション機能**
- **ユーザープロファイル** / **ディスク・パーティション** / **WiFiプロファイル** / **復元ポイント**
- **Windows Update** / **メモリ** / **HW識別子** / **環境変数** / **スタートアップ** / **PnPデバイス**
- **グループポリシー** / **セキュリティベースライン** / **証明書**
- **§<id> {Title}** — 未知 manifest section が見つかったら動的に専用シート生成（前方互換）

各シートに色分けされた 3 段ヘッダ（青/緑/紫/オレンジ等、用途別、`ExcelExportService` 冒頭で `XLColor.FromHtml` 定義）。32,767 文字超のセル値は `[TRUNCATED]` マーカーで切り詰め。

---

## PC 個別詳細 Excel（`pc_details/` 配下）

main 台帳の各 PC 行から **PC 名セルがハイパーリンク** で個別 .xlsx に飛ぶ。個別 .xlsx は：

- そのPCに関する全セクションのデータを 1 ブックに集約
- 1 PC あたり 5〜30 シート規模（取得済みセクション数による）
- ファイル名: `{PcName}_{Serial}.xlsx`、衝突時は `{PcName}_{Serial}_2.xlsx` 等の suffix 付与

詳細データを持たない PC（ライセンス系も DomainStatus も Memory も空 等）は `pc_details/` サブブックを生成しない（`HasAnyDetailData(pc, options)` 判定）。main 台帳のリンクも張られない。

---

## 案件メタデータブロック（オプション）

main 台帳の Sheet 1 冒頭 6 行に「案件メタデータブロック」を出力する `ExcelDeliveryMetadata`：

| フィールド | 用途 |
|---|---|
| `ProjectName` | 案件名 |
| `CustomerName` | 顧客名 |
| `WorkerName` | 作業者名 |
| `WorkDate` | 作業日 |

`fabriqKernelVersion / evidenceConfigVersion / PC 台数 / 突合 OK・NG 件数` は pcEvidences と manifest から **自動算出**（メタデータには持たない）。

**v3.8.1 でも UI からの入力経路は未実装**。コードから `ExcelExportOptions.DeliveryMetadata = new ExcelDeliveryMetadata { ... }` を渡すパスは生きているため、将来案件メタデータダイアログを追加するときにそのまま接続できる。

`DeliveryMetadata is null` のときは従来通り 1 行目から列ヘッダで開始（メタデータ行を出さない）。

---

## コピーエラーの挙動

`File.Copy` / `Directory.EnumerateFiles` で発生する `IOException` / `UnauthorizedAccessException` は **個別ファイル単位でスキップ + リスト返却**。`OperationCanceledException` 以外は throw しない設計：

```csharp
try
{
    File.Copy(filePath, Path.Combine(destDir, fileName), overwrite: true);
}
catch (Exception ex) when (ex is IOException or UnauthorizedAccessException)
{
    errors.Add($"{filePath}: {ex.Message}");
}
```

これにより 1 ファイルの破損 / ロックで出力全体が止まらない。完了時の `MessageBox` で先頭 10 件 + 残り件数のサマリを警告表示する。

manifest_sha256.txt にも `[ERROR: ...]` 行が残るため、**納品先で「コピー失敗ファイル」を欠損として検出可能**（ハッシュ計算が走らなかったファイルが行として記録される）。

### キャンセル

ボタン押下後にキャンセルする UI は v3.8.1 でも未実装だが、`CollectAsync` / `ExportAsync` の `CancellationToken` 経路は実装済み。将来「キャンセル」ボタンを追加するときに `IsCancellationRequested` チェックポイントが既に各ループに入っているため、変更は最小で済む。

---

## ハッシュマニフェストの活用

```powershell
# Windows 側で検証
Get-ChildItem -Recurse -File E:\delivery\2026_05_07_143025_fabriq_evi\NEW-PC-01_ABC123XYZ_2026_03_12 |
    ForEach-Object {
        $rel = Resolve-Path -Relative $_.FullName
        $hash = (Get-FileHash $_ -Algorithm SHA256).Hash.ToLower()
        "$hash  $rel"
    }
```

```bash
# Linux / WSL 側で検証
cd /mnt/e/delivery/2026_05_07_143025_fabriq_evi/NEW-PC-01_ABC123XYZ_2026_03_12
sha256sum -c manifest_sha256.txt
```

監査時の改ざん検出 / 媒体運搬時の破損検出 / 顧客先での再展開後の整合性確認に使える。

---

## トラブル対応

### 「コピー警告」が大量に出る

権限不足が最頻値：

- 出力先がネットワークドライブで権限が無い → ローカル `E:\delivery\` 等に切り替え
- evidence ソース側のファイルが Excel 等で開かれている → 閉じてから再出力

### Excel 出力が固まる / 失敗する

- フリート 100 台超 + 各 PC 30+ シート × ベースライン 6 シート × 個別 .xlsx 100 ファイル = **メモリと I/O が重い**。フリート 50 台未満を推奨
- `OutOfMemoryException` が出る場合は ClosedXML が一度に保持するセル数を超えている。フリートを分割して 2 回出力するなど
- `[TRUNCATED]` マーカーが多発する場合は §27 環境変数 / §24 GP の raw text が長すぎる。fabriq 側で出力を絞る

### 「pc_details/ サブブックがリンクから開かない」

- main 台帳と pc_details/ サブディレクトリが同フォルダ内にあることを確認
- main 台帳だけ別フォルダに移動するとハイパーリンクが切れる（**フォルダ単位で運搬する**）

### ファイル名衝突

`GetPcDetailFileName` が `usedNames` HashSet で衝突を検知し `_2 / _3 ...` の suffix 付与で回避する。同 PC が異なる収集日で複数回出ているフリートでもファイル衝突しない。

---

## 関連ドキュメント

- 取り込み手順（前段）: [fabriq_evidence_manager__usage__01_import.md](fabriq_evidence_manager__usage__01_import.md)
- hostlist 突合: [fabriq_evidence_manager__usage__02_hostlist_verification.md](fabriq_evidence_manager__usage__02_hostlist_verification.md)
- ベースライン突合: [fabriq_evidence_manager__usage__03_baseline.md](fabriq_evidence_manager__usage__03_baseline.md)
- 階層構造（CollectorService / ExcelExportService 詳細）: [fabriq_evidence_manager__architecture__01_layers.md](fabriq_evidence_manager__architecture__01_layers.md)
- フリート画面（出力ボタンの位置）: [fabriq_evidence_manager__apps__01_main_window.md](fabriq_evidence_manager__apps__01_main_window.md)
