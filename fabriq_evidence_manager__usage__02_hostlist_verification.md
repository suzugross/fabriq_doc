# hostlist 突合の使い方

> **対象**: fabriq_evidence_manager / usage
> **対象バージョン**: 3.8.1（取得元: `E:\fabriq_evidence_manager\FabriqEvidenceManager\FabriqEvidenceManager.csproj` `<Version>` / commit `45eae22`）
> **ドキュメント更新日**: 2026-05-07

`hostlist.csv` を「期待値」、各 PC のエビデンスを「実測値」として **PC ごとに突き合わせる** 機能。fabriq_studio で編集した端末マスター CSV が手元にある状態で、フリート全 PC が hostlist 通りにキッティングされたかを機械的に検収する。

---

## 前提条件

### hostlist.csv の準備

通常は **fabriq_studio で編集 + export した hostlist.csv** をそのまま使う：

- fabriq_studio の `HostListView` → `HostListExportDialog` で **タイムスタンプ付きフォルダ** に出力されたもの
- 出力例: `<parent>/hostlist_export_20260512_143025/hostlist.csv`
- export 時に **暗号化フィールドが復号されているか確認**（後述）

`E:\fabriq\kernel\csv\hostlist.csv` を直接読ませるのも可能だが、業務中に kernel/csv/ を書き換えるリスクがあるため、export 済みを推奨。

### 暗号化フィールドの扱い

fabriq_studio が hostlist.csv 内の機密フィールド（BitLocker PIN 等）を `ENC:` プレフィックス付きの AES-256-CBC 暗号文として保持するケースがあるが、本アプリは **復号機能を持たない**（パスフレーズ入力 UI なし、`CryptoService` も非搭載）。突合に使う列で `ENC:` 値が入っていると Expected がそのまま `ENC:abc...` になり全 PC で Mismatch になる。

回避策：

- fabriq_studio の `HostListExportDialog` で **「Decrypt オプション」を ON にして export** する
- もしくは突合不要な暗号列（例: PIN）はそのまま `ENC:` で運用し、本アプリは無視する（突合対象 7 項目に PIN 列は含まれない）

---

## hostlist.csv の必須列

`HostlistService.MapToEntry` が読む列は次の通り。**列順非依存**（ヘッダ名で lookup）。未定義列は無視され、未指定列は空文字列扱い。

| 列名 | 用途 | 突合 |
|---|---|---|
| `AdminID` | 管理者ID | （識別のみ、突合対象外） |
| `OldPCName` | 旧 PC 名 | （識別のみ） |
| `NewPCName` | **新 PC 名（突合キー）** | 突合キー |
| `EthernetIP` / `EthernetSubnet` / `EthernetGateway` | 有線 LAN 設定 | 突合対象 |
| `WifiIP` / `WifiSubnet` / `WifiGateway` | 無線 LAN 設定 | 突合対象 |
| `DNS1` / `DNS2` / `DNS3` / `DNS4` | DNS（最大 4 件） | ソート済みリスト比較 |
| `Pin` | BitLocker PIN | （突合対象外） |
| `Printer1Name` / `Printer1Driver` / `Printer1Port` 〜 `Printer10Name`〜 | プリンタ最大 10 台 | 突合対象 |

`NewPCName` が空の行は無視される（`HostlistService.Load` 内）。同名 `NewPCName` が複数行ある場合は **最後勝ち**（`Dictionary` が後で書き換える）。

---

## 操作手順

### 1. hostlist.csv を読み込む

メイン画面 Row 1 の `Hostlist:` 行：

1. `CSV 読込...` ボタン → `OpenFileDialog`（`*.csv` フィルタ）
2. fabriq_studio が export した `hostlist.csv` を選択
3. `OK` 押下

ステータスバー: `hostlist.csv を読み込みました (N 件)。`、`HostlistStatusText` が `"3件読込済"` 等に更新。読み込み成功で **全 PC の Caution 判定が再実行** され、不一致がある PC が黄色行に変わる。

### 2. 個別 PC の突合詳細を見る

DataGrid 行をダブルクリック → `PcDetailWindow` を起動 → セクション `HOSTLIST VERIFICATION` を確認。各項目に `Expected` / `Actual` / `Status` の 3 列が並ぶ。

### 3. フリート全体の突合結果を Excel に出力する

納品データ出力時、`MainWindowViewModel.BuildExcelExportOptions` が **全 PC の `VerificationReport` を Dictionary** にして `IExcelExportService.ExportAsync` に渡す。Excel main book のシート「PC情報一覧」末尾に各 PC の突合 OK 件数 / NG 件数列が、別シート「突合詳細」に PC ごとの全項目展開が出る。

詳細は [fabriq_evidence_manager__usage__04_export_delivery.md](fabriq_evidence_manager__usage__04_export_delivery.md) を参照。

---

## 突合項目（7 グループ + α）

`EvidenceVerificationService.Verify(pc, options)` が `HostlistEntry` を期待値として組み立てる項目：

| 項目名 | Expected の出処 | Actual の出処 |
|---|---|---|
| ホスト名 | `entry.NewPCName` | `01_SystemInfo.txt` の `Hostname:` 行 |
| Ethernet IP | `entry.EthernetIP` | `06_NetworkConfig.csv` の `IsEthernet=true` 行の `IPv4Address` |
| Ethernet サブネット | `entry.EthernetSubnet` | 同 `SubnetMask` |
| Ethernet ゲートウェイ | `entry.EthernetGateway` | 同 `DefaultGateway` |
| Wi-Fi IP | `entry.WifiIP` | `06_NetworkConfig.csv` の `IsWifi=true` 行の `IPv4Address` |
| Wi-Fi サブネット | `entry.WifiSubnet` | 同 `SubnetMask` |
| Wi-Fi ゲートウェイ | `entry.WifiGateway` | 同 `DefaultGateway` |
| DNS | `entry.GetDnsList()`（DNS1〜4 をソート済み） | Ethernet + Wi-Fi の DNS をマージ → IPv6 除外 → `Distinct + Order` |
| プリンタ: `{Name}` | `entry.Printers[i].Port` | `07_Printers.csv` の同名行の `Port`（`IP_xxx.xxx.xxx.xxx` プレフィックス除去で正規化） |
| 余剰プリンタ: `{Name}` | `(hostlistに未登録)` | `07_Printers.csv` の hostlist 未登録行（OS デフォルト除く） |
| BitLocker (C:) | `On (FullyEncrypted) or EncryptionInProgress` | `08_BitLocker.txt` の C ドライブ `ProtectionStatus + ConversionStatus` |

### DNS 比較の正規化

DNS は **ソート済みカンマ区切り文字列** どうしの比較。Expected も Actual も内部で `Order()` 適用済みなので、PC 側で DNS 順序が hostlist と異なっていても `Match` 扱い。Wi-Fi のみ / Ethernet のみ片方が DNS を持つケースは両 NIC の DNS がマージされる。

IPv6 アドレス（fec0:... 等）は Actual 側で除外する（DNS では `fec0:0:0:ffff::1` 等の予約アドレスがリンクローカル DNS として登録されることがあり、hostlist との比較ノイズになるため）。

### プリンタポート正規化

実測値が `IP_192.168.1.50` 形式の場合、`IP_` プレフィックスを除去して hostlist の `192.168.1.50` 表記と比較する。これは fabriq 側のプリンタドライバ命名規則（`Add-Printer -PortName "IP_${ip}"` 出力）への対応。

### BitLocker C: の許容判定

`IsBitLockerAcceptable(volume)` が True を返す条件：

- `ProtectionStatus = On`（暗号化保護済み）
- または `ConversionStatus = EncryptionInProgress`（暗号化進行中、進捗があれば許容）

C ドライブ未検出 → `NoActual`、C ドライブはあるが暗号化未完了かつ非進行中 → `Mismatch`。

---

## 余剰プリンタ検出（オプション）

設定ダイアログ §「hostlist 突合の調整」の `余剰プリンタ検出` チェックボックス（既定 OFF）。

ON 時の挙動：

- hostlist に `Printer1Name〜Printer10Name` で登録されていない実機プリンタを検出
- ただし **OS デフォルトプリンタ** は除外（部分一致で判定）：
  - `Microsoft Print to PDF`
  - `Microsoft XPS Document Writer`
  - `OneNote` / `Send to OneNote`
  - `Fax`
- 残った余剰プリンタを `項目名 = "余剰プリンタ: {Name}"` / `Status = Mismatch` で計上

OFF が既定の理由：fabriq 側の通常運用では PC が hostlist 外のプリンタ（メーカ配布ドライバ等）を持つケースが多く、ON にすると Caution 件数が増えすぎる。「余剰検出が必要な特定案件」のときだけ手動で ON にする。

---

## VerificationStatus の意味

| Status | 色 | 意味 |
|---|---|---|
| `Match` | 緑 | 期待値 = 実測値 |
| `Mismatch` | 赤 | 期待値 ≠ 実測値（両方とも非空） |
| `NoExpected` | グレー | hostlist 側が空（未設定）。実測値があっても警告にならない |
| `NoActual` | オレンジ | hostlist 側はあるが実測値が空（取得失敗 or 設定漏れ） |

`PcEvidence.HasCaution = true` 判定で集計される `MismatchCount` は **`Mismatch + NoActual`** の件数（`NoExpected` は除外、設定不備の責任を hostlist 側に問わない方針）。

---

## DataGrid 行の Caution 表示

hostlist 読み込み済みかつ `MismatchCount > 0` の PC は黄色行 + `CautionMessage = "hostlist不一致(N件)"` で表示される。何が不一致かは PC 詳細ウィンドウの `HOSTLIST VERIFICATION` セクションで確認する。

---

## トラブル対応

### hostlist 読み込みが「読込エラー」になる

`MessageBox.Show("hostlist.csv の読み込みに失敗しました。")` + 例外メッセージ：

| 原因 | 対処 |
|---|---|
| ファイルが他プロセスで開かれている（Excel 等） | Excel で開いていれば閉じてから再試行 |
| 文字コードが Shift-JIS | UTF-8 (BOM 付き) で保存し直す |
| ヘッダ行のカラム名が違う | `NewPCName` 等の必須列名がスペル通りか確認 |

### 全 PC で Mismatch が出る

- hostlist の `NewPCName` と evidence のフォルダ名 / `manifest.selectedNewPcName` が一致していない可能性
- `HostlistService.FindByPcName` は `OrdinalIgnoreCase` で一致検索するが、空白混入や全角混入には敏感

### IP は合っているのに「不一致」になる

DNS 列の比較順序ずれ → 内部で sort して比較しているはずだが、hostlist 側に **空文字列 + ピリオドを含むダミー行**（`...` 等）が混じっていると差分判定される。`DNS1〜DNS4` を実値のみにする。

### プリンタが NoActual になる

実機にはプリンタがあるが evidence の `07_Printers.csv` に出力されていないケース：

- fabriq 側 `evidence_config` モジュールがプリンタ列挙に失敗（権限不足等） → fabriq 側を確認
- `07_Printers.csv` 自体が存在しない → manifest.sections[id="07"] の `status` を確認

### 余剰プリンタを警告したくない

設定 → `余剰プリンタ検出` を OFF（既定）。

---

## 関連ドキュメント

- 取り込み手順（前段）: [fabriq_evidence_manager__usage__01_import.md](fabriq_evidence_manager__usage__01_import.md)
- ベースライン突合（hostlist と直交する別軸）: [fabriq_evidence_manager__usage__03_baseline.md](fabriq_evidence_manager__usage__03_baseline.md)
- 設定ダイアログ: [fabriq_evidence_manager__apps__03_settings_window.md](fabriq_evidence_manager__apps__03_settings_window.md)
- PC 詳細ウィンドウ: [fabriq_evidence_manager__apps__02_pc_detail_window.md](fabriq_evidence_manager__apps__02_pc_detail_window.md)
- 階層構造（HostlistService / EvidenceVerificationService 詳細）: [fabriq_evidence_manager__architecture__01_layers.md](fabriq_evidence_manager__architecture__01_layers.md)
- fabriq_studio 側の hostlist 編集: [fabriq_studio__apps__01_main_pages.md](fabriq_studio__apps__01_main_pages.md)
