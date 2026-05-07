# hostlist.csv スキーマリファレンス

> **対象**: fabriq_evidence_manager / reference
> **対象バージョン**: 3.8.1（取得元: `E:\fabriq_evidence_manager\FabriqEvidenceManager\FabriqEvidenceManager.csproj` `<Version>` / commit `45eae22`）
> **対応 fabriq_studio**: hostlist 編集画面の出力（`HostListExportDialog`）
> **ドキュメント更新日**: 2026-05-07

`hostlist.csv` は fabriq フレームワーク全体で **PC ごとの期待値（Expected）** を定義する中央マスター。fabriq 本体（kernel/csv/hostlist.csv）/ fabriq_studio（編集 UI + export）/ fabriq_evidence_manager（突合の期待値ソース）の 3 者が共有する。本ドキュメントは consumer 側（fabriq_evidence_manager）が読む列定義を実装ベースで明文化する。

---

## ファイル仕様

| 項目 | 仕様 |
|---|---|
| エンコーディング | **BOM 付き UTF-8**（`File.ReadAllLines(... Encoding.UTF8)`） |
| 改行 | CRLF（PowerShell 既定）/ LF どちらも `ReadAllLines` で吸収 |
| 区切り | `,` カンマ |
| クォート | ダブルクォート囲み + `""` エスケープ対応 |
| ヘッダ | **1 行目固定**、列順非依存（header-index lookup） |
| 突合キー | **`NewPCName` 列**（同名複数行は最後勝ちで `Dictionary` 上書き）|
| 空行 | スキップ |
| 必須列 | `NewPCName`（突合キー、空行は破棄）|

`HostlistService.Load(csvPath)` 実装：

```csharp
var headers = ParseCsvLine(lines[0]);
var headerIndex = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase);
for (int i = 0; i < headers.Length; i++)
    headerIndex[headers[i].Trim()] = i;
```

列名は **大文字小文字無視** で照合。未定義列は無視され、欠損値は空文字列として扱う。

---

## 列定義（全 43 列）

`HostlistEntry` モデルにマップされる列：

### 識別列（3 列）

| 列名 | 型 | 用途 | 突合 |
|---|---|---|---|
| `AdminID` | string | 管理者 ID | × |
| `OldPCName` | string | 旧 PC 名（resetting 前） | × |
| `NewPCName` | string | **新 PC 名（突合キー）** | ○（マッチング） |

`NewPCName` が空の行は `HostlistService` 内で破棄。`AdminID / OldPCName` は識別目的のみで突合対象外（fabriq 側のキッティングプロセスでは利用される）。

### 有線 LAN 設定（3 列）

| 列名 | 型 | バインド | 突合項目名 |
|---|---|---|---|
| `EthernetIP` | string | `EthernetIP` | `Ethernet IP` |
| `EthernetSubnet` | string | `EthernetSubnet` | `Ethernet サブネット` |
| `EthernetGateway` | string | `EthernetGateway` | `Ethernet ゲートウェイ` |

突合実測値は §06 `06_NetworkConfig.csv` の `IsEthernet=true` 行から抽出。

### 無線 LAN 設定（3 列）

| 列名 | 型 | バインド | 突合項目名 |
|---|---|---|---|
| `WifiIP` | string | `WifiIP` | `Wi-Fi IP` |
| `WifiSubnet` | string | `WifiSubnet` | `Wi-Fi サブネット` |
| `WifiGateway` | string | `WifiGateway` | `Wi-Fi ゲートウェイ` |

突合実測値は §06 の `IsWifi=true` 行から抽出。

### DNS 設定（4 列）

| 列名 | 型 | バインド |
|---|---|---|
| `DNS1` | string | `DNS1` |
| `DNS2` | string | `DNS2` |
| `DNS3` | string | `DNS3` |
| `DNS4` | string | `DNS4` |

`HostlistEntry.GetDnsList()` で **非空値を抽出 + ソート済みリスト化**：

```csharp
public List<string> GetDnsList() =>
    new[] { DNS1, DNS2, DNS3, DNS4 }
        .Where(d => !string.IsNullOrWhiteSpace(d))
        .Order()
        .ToList();
```

突合では Expected を `string.Join(", ", expected)` / Actual を `string.Join(", ", merged_sorted)` 形式で文字列比較。実測値は **Ethernet + Wi-Fi の DNS をマージ → IPv6 除外（`:` を含む値）→ Distinct + ソート**。

順序差異・重複・IPv6 ノイズに対して堅牢な比較となる。

### BitLocker（1 列）

| 列名 | 型 | バインド | 用途 |
|---|---|---|---|
| `Pin` | string | `Pin` | BitLocker PIN（fabriq 側の `bitlocker_config` モジュールが読む） |

**本アプリ（evidence_manager）は突合対象としない**。BitLocker 突合は §08 `08_BitLocker.txt` の C ドライブ `ProtectionStatus + ConversionStatus` のみで判定する（PIN 値そのものを比較対象にすると ENC: 暗号化値が混入したときに常に Mismatch になる懸念があるため）。

### プリンタ（最大 30 列）

`Printer1Name / Printer1Driver / Printer1Port` 〜 `Printer10Name / Printer10Driver / Printer10Port` の **3 列 × 10 台 = 最大 30 列**。

`HostlistService.MapToEntry` で展開：

```csharp
var printers = new List<PrinterExpectation>();
for (int i = 1; i <= 10; i++)
{
    var name = F($"Printer{i}Name");
    if (string.IsNullOrWhiteSpace(name)) continue;     // ← Name 空ならスキップ
    printers.Add(new PrinterExpectation
    {
        Name = name,
        Driver = F($"Printer{i}Driver"),
        Port = F($"Printer{i}Port"),
    });
}
```

`Printer{i}Name` が空の枠はスキップ（飛び番号許容、`Printer3` だけ埋まっていても OK）。

突合では `EvidenceVerificationService` が **プリンタ名（OrdinalIgnoreCase）でマッチング** + **ポート正規化比較**：

- 実測値が `IP_192.168.1.50` 形式の場合、`IP_` プレフィックスを除去して `192.168.1.50` と比較
- ポート不一致 = `Mismatch`、プリンタ自体が実機にない = `NoActual`

---

## 全列一覧

```csv
"AdminID","OldPCName","NewPCName","EthernetIP","EthernetSubnet","EthernetGateway","WifiIP","WifiSubnet","WifiGateway","DNS1","DNS2","DNS3","DNS4","Pin","Printer1Name","Printer1Driver","Printer1Port","Printer2Name","Printer2Driver","Printer2Port","Printer3Name","Printer3Driver","Printer3Port","Printer4Name","Printer4Driver","Printer4Port","Printer5Name","Printer5Driver","Printer5Port","Printer6Name","Printer6Driver","Printer6Port","Printer7Name","Printer7Driver","Printer7Port","Printer8Name","Printer8Driver","Printer8Port","Printer9Name","Printer9Driver","Printer9Port","Printer10Name","Printer10Driver","Printer10Port"
```

順番は規約上の参照順だが、**実装は header-index lookup なので並び替えても動く**。

### 列カウント

| 区分 | 列数 |
|---|---|
| 識別 | 3 |
| 有線 | 3 |
| 無線 | 3 |
| DNS | 4 |
| BitLocker | 1 |
| プリンタ | 3 × 10 = 30 |
| **合計** | **44**（Excel 上のカラム） |

> ※ fabriq_studio の `HostEntry` モデルでは「43 列」と表現されているが、本アプリの `HostlistEntry` は `AdminID + OldPCName + NewPCName + ネットワーク 6 + DNS 4 + Pin + プリンタ 30 = 44` 列を読み込む。差は識別目的の運用上の数え方（`AdminID` を含むかどうか）の違いで、CSV の実列数は 44。

---

## 暗号化フィールド（ENC: プレフィックス）の扱い

fabriq_studio の hostlist 編集画面は機密フィールド（特に `Pin`、必要に応じて `EthernetIP` 等）を **AES-256-CBC で暗号化** し、`ENC:` プレフィックス付き Base64 文字列として CSV に保持できる：

```csv
"AdminID","OldPCName","NewPCName","Pin"
"admin01","OLD-PC-01","NEW-PC-01","ENC:U2FsdGVkX1+abc..."
```

### consumer 側（fabriq_evidence_manager）の挙動

**復号機能を持たない**。パスフレーズ入力 UI 無し、`CryptoService` 等は非搭載（fabriq_studio 側の責任範囲）。

そのため：

- `Pin` 列に `ENC:` 値があっても本アプリは突合対象外（`HostlistEntry.Pin` にそのまま入るが、`EvidenceVerificationService` で参照しない）
- **突合対象列**（IP / Subnet / Gateway / DNS / Printer Port 等）に `ENC:` 値が混入していると **常に Mismatch** になる（実測値は復号後の値、期待値は暗号文字列のまま、文字列比較で不一致）

### 回避策

fabriq_studio の `HostListExportDialog` の **「Decrypt オプション」を ON にして export** することで、突合対象列の `ENC:` 値が復号後の平文で出力される：

```
<parent>/hostlist_export_yyyyMMdd_HHmmss/
├── hostlist.csv         ← 復号済み（突合対象列）+ ENC: 値（Pin 等）
└── README.txt
```

この export 済み CSV を本アプリで読み込むのが標準運用。`E:\fabriq\kernel\csv\hostlist.csv` を直接読ませるのは **業務中に kernel/csv/ 側を書き換えるリスクがある** ため非推奨。

---

## ヘッダ行の堅牢性

`HostlistService.MapToEntry` の `GetField` ヘルパは未定義列・短い行に対して空文字列を返す：

```csharp
private static string GetField(string[] fields, Dictionary<string, int> headerIndex, string columnName) =>
    headerIndex.TryGetValue(columnName, out var idx) && idx < fields.Length
        ? fields[idx].Trim()
        : string.Empty;
```

- **新規列の追加**: 古いバージョンの本アプリが新列付き CSV を読んでも、未定義列は黙って無視される
- **列の削除**: 古い CSV を新バージョンで読むと、削除列は空文字列扱い
- **列順入れ替え**: header-index lookup なので影響なし
- **行の途中切れ**（`fields.Length` 不足）: 末尾列は空文字列扱い

これにより fabriq_studio 側の hostlist 列追加（将来的な PIN2 / Printer11 等）に対しても **本アプリは silent 互換** で動作する。

---

## fabriq_studio との対応関係

fabriq_studio は **編集 UI + 暗号化 + export** を担い、本アプリは **読み取り + 突合実行** を担う。役割分担：

| 機能 | fabriq_studio | fabriq_evidence_manager |
|---|---|---|
| 列定義（fabriq_studio.HostEntry / fabriq_evidence_manager.HostlistEntry） | 43 ObservableProperty で展開 | 14 + Printer × 10 で読み込み |
| 行追加・削除・複製 | `HostListView` UI | × |
| バッチ暗号化／復号 | `CryptoHelper.ExcludedColumns` 以外を一括処理 | × |
| パスフレーズ照合 | `passphrase_verify.txt` の `surkitinisme` トークンを復号 | × |
| Export | `HostListExportDialog`（タイムスタンプフォルダ） | × |
| 読み込み | × | `HostlistService.Load` |
| 突合 | × | `EvidenceVerificationService.Verify` |

fabriq_studio 側のドキュメントは [fabriq_studio__apps__01_main_pages.md](fabriq_studio__apps__01_main_pages.md) §「2. 端末一覧（HostList → HostDetail）」を参照。

---

## サンプル

最小構成（プリンタなし、Wi-Fi なし）：

```csv
"AdminID","OldPCName","NewPCName","EthernetIP","EthernetSubnet","EthernetGateway","DNS1","DNS2","Pin"
"admin01","OLD-PC-01","NEW-PC-01","192.168.1.10","255.255.255.0","192.168.1.1","192.168.1.2","192.168.1.3","123456"
"admin01","OLD-PC-02","NEW-PC-02","192.168.1.11","255.255.255.0","192.168.1.1","192.168.1.2","192.168.1.3","234567"
```

プリンタ 2 台 + Wi-Fi 設定あり：

```csv
"AdminID","OldPCName","NewPCName","EthernetIP","EthernetSubnet","EthernetGateway","WifiIP","WifiSubnet","WifiGateway","DNS1","DNS2","Pin","Printer1Name","Printer1Driver","Printer1Port","Printer2Name","Printer2Driver","Printer2Port"
"admin01","OLD-PC-01","NEW-PC-01","192.168.1.10","255.255.255.0","192.168.1.1","192.168.2.10","255.255.255.0","192.168.2.1","192.168.1.2","192.168.1.3","123456","Office Printer","HP Universal Driver","192.168.1.50","Color Printer","Canon iR-ADV","192.168.1.51"
```

DNS 4 件（IPv6 ノイズあり）+ プリンタの IP_ プレフィックス：

```csv
"AdminID","OldPCName","NewPCName","DNS1","DNS2","DNS3","DNS4","Printer1Name","Printer1Port"
"admin","old","new","192.168.1.2","192.168.1.3","8.8.8.8","8.8.4.4","HQ-Printer","192.168.1.50"
```

実機の §06 出力で DNSServers が `"192.168.1.2,fec0:0:0:ffff::1,8.8.8.8,192.168.1.3,8.8.4.4"` のような場合、IPv6 (`fec0:...`) を除外 → ソート → `8.8.4.4, 8.8.8.8, 192.168.1.2, 192.168.1.3` になり、Expected の sorted 結果と一致。`Printer1Port = 192.168.1.50` と実機の `IP_192.168.1.50` が `IP_` 除去後一致。

---

## 関連ドキュメント

- hostlist 突合の使い方: [fabriq_evidence_manager__usage__02_hostlist_verification.md](fabriq_evidence_manager__usage__02_hostlist_verification.md)
- pc_information ファイル形式（突合実測値の出処）: [fabriq_evidence_manager__reference__file_format__pc_information.md](fabriq_evidence_manager__reference__file_format__pc_information.md)
- Models 索引（`HostlistEntry / PrinterExpectation / VerificationItem`）: [fabriq_evidence_manager__reference__model_catalog.md](fabriq_evidence_manager__reference__model_catalog.md)
- fabriq_studio 側の hostlist 編集画面: [fabriq_studio__apps__01_main_pages.md](fabriq_studio__apps__01_main_pages.md)
- 階層構造（HostlistService / EvidenceVerificationService 詳細）: [fabriq_evidence_manager__architecture__01_layers.md](fabriq_evidence_manager__architecture__01_layers.md)
