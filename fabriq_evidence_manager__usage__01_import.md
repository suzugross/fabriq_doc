# evidence 取り込み手順

> **対象**: fabriq_evidence_manager / usage
> **対象バージョン**: 3.8.0（取得元: `E:\fabriq_evidence_manager\FabriqEvidenceManager\FabriqEvidenceManager.csproj` `<Version>`）
> **ドキュメント更新日**: 2026-05-07

fabriq 側 `log_uploader` が集約した evidence ディレクトリを、本アプリで取り込んでフリート一覧化するまでの手順。

---

## 前提条件

### 入力 evidence の要件

- fabriq kernel **2.2.2 以上** + evidence_config モジュール **1.3.0 以上** で収集された evidence であること（`pc_information/manifest.json` `schemaVersion=1` を必須とする）
- 各 PC ディレクトリは `{yyyy_MM_dd_HHmmss}_{PCName}_{SerialNumber}_evidence/evidence/...` の階層を持つこと
- fabriq 側で `log_uploader` モジュールが正常完了し、上記の親フォルダがネットワーク共有 / ローカルディスクに揃っていること

旧形式 evidence（manifest.json 不在 / schemaVersion≠1）はリストには載るが各セクションの構造化パースは行われない。詳細は [fabriq_evidence_manager__contracts__manifest_schema.md](fabriq_evidence_manager__contracts__manifest_schema.md) §「旧 evidence の扱い」を参照。

### アプリ側の要件

- **Windows 10 / 11**
- self-contained single-file 配布版を使う場合: `E:\publish_fabriq_evidence_manager\FabriqEvidenceManager.exe` 単体で動作（.NET 8 ランタイム同梱）
- 開発時: .NET 8 SDK + Visual Studio 2022 / `dotnet build`

---

## 入手方法

### A. self-contained 配布版を使う

```
E:\publish_fabriq_evidence_manager\FabriqEvidenceManager.exe
```

ダブルクリックで起動。設定ファイルなし、初回起動でも何も自動生成しない（設定は全てメモリ保持・プロセス終了で破棄）。

### B. ソースからビルドする

```powershell
# ビルド
dotnet build E:\fabriq_evidence_manager\fabriq_evidence_manager.sln

# デバッグ実行
dotnet run --project E:\fabriq_evidence_manager\FabriqEvidenceManager\FabriqEvidenceManager.csproj
```

### C. パブリッシュ（self-contained, single-file）

```powershell
dotnet publish E:\fabriq_evidence_manager\FabriqEvidenceManager\FabriqEvidenceManager.csproj `
    -c Release -r win-x64 --self-contained true `
    -p:PublishSingleFile=true -p:IncludeNativeLibrariesForSelfExtract=true `
    -o E:\publish_fabriq_evidence_manager
```

出力先は固定で `E:\publish_fabriq_evidence_manager\`（CLAUDE.md `Build & Publish` 欄の規約）。

---

## 取り込み手順

### 1. evidence ルートディレクトリを選ぶ

メイン画面 Row 1 の `Evidence フォルダ` グループボックス：

1. `参照...` ボタンクリック → `OpenFolderDialog`
2. **`{timestamp}_{PCName}_{Serial}_evidence` 形式の親フォルダ群が並んでいる親階層** を選択
3. ダイアログを `OK` で閉じる

ここで選ぶのは log_uploader が積み上げた集約サーバ上のフォルダ（社内ファイルサーバの収集ディレクトリ等）の **親** になる。例：

```
\\fileserver\fabriq\projects\acme\evidence_root\        ← ここを選ぶ
├── 2026_03_12_143025_NEW-PC-01_ABC123_evidence\
├── 2026_03_12_143112_NEW-PC-02_DEF456_evidence\
├── 2026_03_12_143258_NEW-PC-03_GHI789_evidence\
└── ...
```

### 2. 自動で読み込みが走る

`参照...` でフォルダ選択した直後、`SelectEvidenceFolderCommand` が **そのまま `LoadEvidenceCommand` を自動チェイン実行** する。手動で `読み込み` ボタンを押す必要はない。

- ステータスバー: `evidence を読み込み中...` + LINK/ACT LED が緑点滅
- 完了: `読み込み完了: N 台のPCを検出しました。`
- DataGrid に PC 一覧（PC 名でアルファベット順ソート）

### 3. 後から再読み込みする

設定ダイアログでチェック項目を変えた、または fabriq 側で再収集が走った場合は `読み込み` ボタンで明示的に再ロード：

- **`TargetPCs.Clear()` してから再構築** する（差分マージではなく完全再生成）
- `SelectedPc` / `SearchText` / スクロール位置はリセットされる

### 4. 自動更新（差分マージ）を有効化する

`自動更新 (30秒)` チェックボックス ON で `DispatcherTimer` 起動。30 秒ごとに `RefreshEvidenceAsync` が背景で走る：

- 既存 PC は **インスタンスを保持したままプロパティ更新**（`SelectedPc` / `SearchText` / スクロール位置維持）
- 新規 PC は `TargetPCs.Add(...)` 末尾追加
- 検出時のみ `自動更新: N 台の新規PCを検出しました。` をステータスバー表示
- 例外は silent（`MessageBox` を出さない）

差分マージのキーは `{PcName}\t{SerialNumber}\t{CollectionDate}` のタブ連結文字列。同じ PC で再収集しても CollectionDate が変わるため別エントリとして取り込まれる。

---

## 取り込み結果の見方

### DataGrid の 11 列

| 列 | 意味 |
|---|---|
| 計画 PC 名 | manifest.selectedNewPcName（hostlist で割り当てた名前）。`📝` アイコンは管理者メモあり |
| 実 PC 名 | manifest.computerName（OS 上の現在名）。**赤背景 = 計画名と不一致**（rename 失敗の可能性）|
| シリアル | 採用ソース併記。**オレンジ背景 = フォールバック取得**（BIOS 以外から SN 取得、SMBIOS 不整合疑い）|
| 収集日 | `yyyy_MM_dd_HHmmss` |
| MAC (イーサネット) / MAC (Wi-Fi) | 接続名で識別 |
| BitLocker (回復キー) | ドライブ別 1 行 = `{drive}: {recoveryPassword}` |
| Win 認証 | `認証済` / `未認証` / `(未取得)` |
| Office 認証 | `認証済` / `サインイン待ち` / `認証失敗` / `未インストール` / `(未取得)` |
| 検出エビデンス | `PC情報 / キャプチャ / BitLocker / チェックリスト / 履歴` の存在組合せ |
| ステータス | `WarningMessage`（赤 / 欠損）+ `CautionMessage`（オレンジ / 要確認） |

### 行ハイライト 2 軸

| 色 | フラグ | 意味 |
|---|---|---|
| 赤 (`#FFCDD2`) | `HasWarning` | エビデンス欠損（取得失敗系）。Caution より優先 |
| 黄 (`#FFF9C4`) | `HasCaution` | 要確認（突合差異・rename 失敗・SN フォールバック等） |
| 透明 | (両方 false) | 正常 |

判定軸の責務分離は [fabriq_evidence_manager__apps__03_settings_window.md](fabriq_evidence_manager__apps__03_settings_window.md) §「取得チェック項目」を参照。

### フィルタ / 検索

`FILTER` テキストボックスは **PC 名 / シリアル / 警告メッセージ** の OR 部分一致（`OrdinalIgnoreCase`）。ヒット件数は右上の `DisplayCountText` に `"3 / 10 件"` 形式で出る。

### 行ダブルクリック → PC 詳細ウィンドウ

フリート画面で気になる PC を見つけたら DataGrid 行をダブルクリック → `PcDetailWindow` が **modeless** で開く。複数 PC を順に開けば並列に並べて比較できる（[fabriq_evidence_manager__apps__02_pc_detail_window.md](fabriq_evidence_manager__apps__02_pc_detail_window.md) §「スナップショット保持と並列比較」）。

---

## manifest エラーの解釈

`Discovery` は manifest 不良 PC でも load を中断せず、`PcEvidence.ManifestError` に理由を入れてリストに載せる。エラーメッセージは PC 詳細ウィンドウのヘッダに表示され、警告メッセージとも併用される。

| メッセージ | 原因 | 対処 |
|---|---|---|
| `manifest.json 不在: {path}` | `pc_information/manifest.json` が物理ファイルとして存在しない | fabriq kernel 2.2.1 以前 / evidence_config 1.2.0 以前で収集された旧 evidence。fabriq 側を更新して再収集 |
| `未対応の manifest schemaVersion: N` | manifest の `schemaVersion` が 1 以外（将来 schemaVersion=2 が公開されたケース） | 本アプリのバージョンアップが必要。silent fallback はしない |
| `manifest 解析失敗: {detail}` | JSON 不正 / `manifestType` 不一致 / 必須フィールド欠損 | manifest.json を直接開いて構文を確認、`manifestType="fabriq-evidence-manifest"` か確認 |

---

## トラブル対応

### 「PC情報が見つかりませんでした」が出る

ダイアログで `OK` を押した直後にメッセージボックスが出る場合、選択フォルダ直下に `_evidence` サフィックス付きの親フォルダが見つからないケース：

- **`参照...` で 1 階層深いフォルダを選んでしまっている**（`{timestamp}_{PCName}_{Serial}_evidence` を直接選んでしまっている）→ 1 階層上を選び直す
- **fabriq 側で log_uploader が完了していない** → fabriq 側のログを確認
- **旧形式（`_evidence` サフィックス無し）のフォルダしか無い** → fabriq kernel を 2.2.2+ に更新

### 警告（赤行）が想定外に多い

設定ダイアログで取得チェック項目を OFF にする：

- Office 不要案件 → `Office インストール` / `Office ライセンス認証` を OFF
- フリート Wi-Fi 未配備 → `MACアドレス (Wi-Fi)` を OFF
- 等

詳細は [fabriq_evidence_manager__apps__03_settings_window.md](fabriq_evidence_manager__apps__03_settings_window.md) §「セクション 1: エビデンス取得チェック」を参照。

### 自動更新が動かない

`自動更新 (30秒)` を ON にしても新規 PC が反映されない場合：

- 手動 `読み込み` 中（`IsLoading=true`）はスキップされる仕様 → ボタン無効化が解けるのを待つ
- 例外発生は silent。Visual Studio デバッガで `[Refresh Error]` の `Debug.WriteLine` を確認

### MessageBox が連発する

`OperationCanceledException` 以外の例外は `MessageBox` で都度通知される設計。連発する場合は最初の 1 件のメッセージを記録 → `EvidenceRootPath` をクリアして再選択。

---

## 関連ドキュメント

- フリート画面の操作詳細: [fabriq_evidence_manager__apps__01_main_window.md](fabriq_evidence_manager__apps__01_main_window.md)
- PC 詳細ウィンドウ: [fabriq_evidence_manager__apps__02_pc_detail_window.md](fabriq_evidence_manager__apps__02_pc_detail_window.md)
- hostlist 突合の使い方: [fabriq_evidence_manager__usage__02_hostlist_verification.md](fabriq_evidence_manager__usage__02_hostlist_verification.md)
- ベースライン突合の使い方: [fabriq_evidence_manager__usage__03_baseline.md](fabriq_evidence_manager__usage__03_baseline.md)
- 入力 evidence 構造: [fabriq_evidence_manager__architecture__02_evidence_input.md](fabriq_evidence_manager__architecture__02_evidence_input.md)
- consumer 側 manifest 消費契約: [fabriq_evidence_manager__contracts__manifest_schema.md](fabriq_evidence_manager__contracts__manifest_schema.md)
