# SerialNumber 採用ロジック（§10）

> **対象**: fabriq_evidence_manager / reference
> **対象バージョン**: 3.8.0（取得元: `E:\fabriq_evidence_manager\FabriqEvidenceManager\FabriqEvidenceManager.csproj` `<Version>`）
> **ドキュメント更新日**: 2026-05-07

`10_SerialNumber.txt` のパース処理は本アプリで最も特殊なロジックを持つセクション。「PC 1 台 = SN 1 つ」が実は **5 つの候補ソースから採用 1 つを選ぶ多段判定** であり、フォールバック取得時は監査警告対象になる。本ドキュメントは判定フロー全体を実装ベースで明文化する。

---

## なぜ複雑か

PC のシリアルナンバーは Windows / WMI / レジストリの **5 つの異なる場所から取得可能** だが、それぞれが返す値は同じではない（OEM の SMBIOS 実装次第で食い違う）。fabriq 側 `evidence_config` モジュール §10 は **5 候補すべてを並べて出力** し、内部の優先順位ポリシーで「採用値」を決める。

consumer 側（本アプリ）は：

1. 採用値が何だったか
2. 採用ソースが Primary canonical（`Win32_BIOS.SerialNumber`）だったか
3. それ以外（フォールバック）だったか
4. 全候補の試行結果と各 Validity タグ

を記録し、フォールバック取得時はオレンジ色で audit 警告する。

---

## 入力ファイル形式

`10_SerialNumber.txt` は 4 ブロック構成：

```
==== PC Serial Number ====

---- Canonical Serial Number ----
ABC123XYZ
(Source: Win32_BIOS.SerialNumber)

---- All Sources ----
Win32_BIOS.SerialNumber                          : ABC123XYZ                [VALID, MATCH]
Win32_ComputerSystemProduct.IdentifyingNumber    : ABC123XYZ                [VALID, MATCH]
Win32_SystemEnclosure.SerialNumber               : (empty)                  [INVALID: empty]
Win32_BaseBoard.SerialNumber                     : MB-987654                [VALID]
Registry SystemSerialNumber                      : ABC123XYZ                [VALID, MATCH]

---- Reference ID ----
Win32_ComputerSystemProduct.UUID                 : AAAA-BBBB-CCCC-DDDD-EEEE [VALID]

---- Selection Policy ----
（fabriq 側が採用ソースをどのポリシーで決めたかの human-readable な説明）
```

各ブロックを state machine `SerialBlockState` でパース：

```
None → Canonical → AllSources → ReferenceId → None (Selection Policy は捨てる)
```

`---- ` で始まり ` ----` で終わる行をブロック切替トリガとして使う。

---

## ブロック 1: Canonical Serial Number

採用値とそのソースを記録するブロック。

```
ABC123XYZ                  ← 採用値（1 行目）
(Source: Win32_BIOS.SerialNumber)   ← 採用ソースラベル
```

または取得不能ケース：

```
(Unretrievable)            ← canonicalValue は空文字列のまま
(Source: none)             ← canonicalSourceLabel は空文字列
```

| 行パターン | 抽出 |
|---|---|
| `(Unretrievable)` | `CanonicalValue = ""`（特別扱い、後続の値行を読まない） |
| `(Source: ...)` | `CanonicalSourceLabel`（`none` で始まれば空文字列） |
| その他（最初の非空行） | `CanonicalValue`（既に値が入っていれば上書きしない） |

---

## ブロック 2: All Sources（5 候補の試行結果）

形式: **`{Label}    :    {Value}    [{Tag}]`**

正規表現: `^(?<label>.+?)\s+:\s+(?<value>.*?)\s+\[(?<tag>.+)\]\s*$`

### 5 候補ソース

| Label | 出処 | IsCanonicalCandidate |
|---|---|---|
| `Win32_BIOS.SerialNumber` | WMI Win32_BIOS | **true（Primary canonical）** |
| `Win32_ComputerSystemProduct.IdentifyingNumber` | WMI Win32_ComputerSystemProduct | true |
| `Win32_SystemEnclosure.SerialNumber` | WMI Win32_SystemEnclosure | true |
| `Win32_BaseBoard.SerialNumber` | WMI Win32_BaseBoard | **false（マザーボード SN、record-only）** |
| `Registry SystemSerialNumber` | HKLM レジストリ | true |

`Win32_BaseBoard.SerialNumber` だけ false なのは「マザーボード SN は PC SN ではない」という規約による（修理交換でマザーボードを差し替えると変わってしまう値であり、PC 識別子としては不適切）。

`IsCanonicalCandidateLabel` メソッドは Label 名を switch 式で判定（ハードコード、PowerShell 側 `IsCanonicalCandidate` と整合）。

### Validity タグ解釈

`[Tag]` 部分から `(Validity, Reason, MatchesCanonical)` の 3 値を導出：

| タグ | Validity | Reason | MatchesCanonical |
|---|---|---|---|
| `[VALID, MATCH]` | `Valid` | null | **true** |
| `[VALID, DIFFERENT]` | `Valid` | null | false |
| `[VALID]`（修飾子なし） | `Valid` | null | false |
| `[INVALID: ...]` | `Invalid` | コロン後の本文 | false |
| `[INVALID]` | `Invalid` | null | false |
| `[QUERY FAILED: ...]` | `QueryFailed` | コロン後の本文 | false |
| `[QUERY FAILED]` | `QueryFailed` | null | false |
| **その他（未知タグ）** | `Invalid` | tag 全体 | false |

### Value の特殊扱い

| Value | 解釈 |
|---|---|
| `(empty)` | 空文字列に変換 |
| その他 | trim してそのまま |

### 出力

各行 → `SerialSourceEntry` 1 件：

```csharp
new SerialSourceEntry
{
    Label = label,
    Value = value,
    Validity = validity,           // Valid / Invalid / QueryFailed
    ValidityReason = reason,       // null / 詳細文字列
    MatchesCanonical = matches,
    IsCanonicalCandidate = IsCanonicalCandidateLabel(label),
    IsSelectedCanonical = (label == canonicalSourceLabel),  // 採用された 1 行のみ true
}
```

---

## ブロック 3: Reference ID

UUID は SN ではないが audit 用に保持する。

```
Win32_ComputerSystemProduct.UUID : AAAA-BBBB-CCCC-DDDD-EEEE [VALID]
```

正規表現マッチで Value 抽出 → `(empty)` でなければ `ReferenceUuid` に格納。

`Win32_ComputerSystemProduct.UUID` 自体は **canonical 候補ではない**（UUID は識別子だが SN ではない）。本アプリは表示の audit 補助情報として PcDetailWindow / Excel に出すだけ。

---

## ブロック 4: Selection Policy（捨てる）

fabriq 側 `evidence_config` モジュールが採用判定の根拠を人間向け text として書き出すブロック。本アプリはパースせず、state を `None` に戻して以降の行を捨てる。

```
---- Selection Policy ----
1. Win32_BIOS.SerialNumber: Primary canonical (preferred).
2. ...
```

ここに書かれている内容は **PowerShell 側のロジックの説明** であり、consumer 側がフォールバック判定を再実行する必要はない。Canonical Source Label を信じれば足りる。

---

## state machine 遷移図

```
                    ┌──────────────────────────┐
                    │ State: None (初期)        │
                    │                            │
                    │ "==== PC Serial Number ===="
                    │   は何もしない (state 不変)│
                    └──────────┬─────────────────┘
                               │
                  "---- Canonical Serial Number ----"
                               ▼
        ┌─────────────────────────────────────────┐
        │ State: Canonical                         │
        │                                          │
        │ - "(Unretrievable)" → canonicalValue 空  │
        │ - "(Source: X)" → canonicalSourceLabel  │
        │ - その他 (1 行のみ) → canonicalValue    │
        └─────────────┬────────────────────────────┘
                      │
        "---- All Sources ----"
                      ▼
        ┌─────────────────────────────────────────┐
        │ State: AllSources                        │
        │                                          │
        │ - "{Label} : {Value} [{Tag}]" を         │
        │   SerialSourceEntry に分解して List 追加│
        └─────────────┬────────────────────────────┘
                      │
        "---- Reference ID ----"
                      ▼
        ┌─────────────────────────────────────────┐
        │ State: ReferenceId                       │
        │                                          │
        │ - "{Label} : {Value} [{Tag}]" の         │
        │   Value を ReferenceUuid に格納         │
        └─────────────┬────────────────────────────┘
                      │
        "---- Selection Policy ----" (or 他の "---- ----")
                      ▼
                 State: None
                      │
                  ファイル終端
                      ▼
        ┌─────────────────────────────────────────┐
        │ Build SerialNumberDetail                 │
        │                                          │
        │ - CanonicalValue / CanonicalSourceLabel │
        │ - Sources + IsSelectedCanonical 付与    │
        │ - ReferenceUuid                          │
        └──────────────────────────────────────────┘
```

state は `---- {名前} ----` 行で遷移するが、未知の `----` 区切りはすべて `None` に倒す（防御的）。

---

## consumer 側出力（PcEvidence へのマップ）

`ApplySerialNumberDetail(pc, dir, section)` が `ParseSerialNumberDetailed` の戻り値を 4 プロパティに分解：

| `SerialNumberDetail` | `PcEvidence` |
|---|---|
| `CanonicalValue` | `SerialNumber` |
| `CanonicalSourceLabel`（空なら null）| `SerialNumberSource` |
| `Sources : List<SerialSourceEntry>` | `SerialSourceTrail.AddRange(...)` |
| `ReferenceUuid` | `SerialReferenceUuid` |

これらに加えて算出プロパティが派生：

```csharp
public bool IsSerialFromPrimarySource =>
    SerialNumberSource == SerialSourceEntry.PrimaryCanonicalLabel;
//                       == "Win32_BIOS.SerialNumber"

public bool HasSerialFallbackWarning =>
    !string.IsNullOrEmpty(SerialNumber) && !IsSerialFromPrimarySource;

public string SerialNumberDisplay =>
    string.IsNullOrEmpty(SerialNumber)
        ? "(取得不能)"
        : IsSerialFromPrimarySource
            ? SerialNumber
            : $"{SerialNumber} ⚠ ({SerialSourceEntry.ShortLabel(SerialNumberSource)})";
```

`HasSerialFallbackWarning` の発動条件：

- **SN は取れた**（`SerialNumber` 非空）
- **採用ソースが BIOS 以外**（`SerialNumberSource != "Win32_BIOS.SerialNumber"`）

両条件を満たす ⇒ OEM SMBIOS 不整合の可能性を audit 警告。

---

## ShortLabel ヘルパ

`SerialSourceEntry.ShortLabel(string?)` は UI / Excel の狭幅表示用：

| Label | ShortLabel |
|---|---|
| `Win32_BIOS.SerialNumber` | `BIOS` |
| `Win32_ComputerSystemProduct.IdentifyingNumber` | `CSProduct` |
| `Win32_SystemEnclosure.SerialNumber` | `Enclosure` |
| `Win32_BaseBoard.SerialNumber` | `BaseBoard` |
| `Registry SystemSerialNumber` | `Registry` |
| null / "" | `?` |
| その他（未知）| ラベルそのまま返す |

例: `SerialNumberDisplay = "ABC123XYZ ⚠ (CSProduct)"`

---

## DataGrid / Excel への反映

### MainWindow DataGrid

「シリアル」列 (`SerialNumberDisplay` バインド)：

| 状態 | 表示 | スタイル |
|---|---|---|
| 取得不能 | `(取得不能)` | デフォルト色 |
| Primary canonical | `ABC123XYZ` | デフォルト色 |
| フォールバック | `ABC123XYZ ⚠ (CSProduct)` | **オレンジ背景 + オレンジ文字 + SemiBold** |

```xml
<DataGridTextColumn Header="シリアル" Binding="{Binding SerialNumberDisplay}">
    <DataGridTextColumn.ElementStyle>
        <Style TargetType="TextBlock">
            <Style.Triggers>
                <DataTrigger Binding="{Binding HasSerialFallbackWarning}" Value="True">
                    <Setter Property="Foreground" Value="#E65100"/>
                    <Setter Property="Background" Value="#FFF3E0"/>
                    <Setter Property="FontWeight" Value="SemiBold"/>
                </DataTrigger>
            </Style.Triggers>
        </Style>
    </DataGridTextColumn.ElementStyle>
</DataGridTextColumn>
```

### Excel main 台帳「PC情報一覧」シート

「シリアル番号」「SN ソース」の 2 列に展開：

| シリアル番号 | SN ソース | 着色（SN ソース列） |
|---|---|---|
| 値 | `(取得不能)`（SN 空時） | Gray 文字 |
| 値 | `BIOS` | デフォルト |
| 値 | `CSProduct` 等 | **`CautionBg` 背景 + `CautionFont` 文字 + 太字** |

```csharp
sourceCell.Value = SerialSourceEntry.ShortLabel(pc.SerialNumberSource);
if (pc.HasSerialFallbackWarning)
{
    sourceCell.Style.Fill.BackgroundColor = CautionBg;     // #FFF3E0
    sourceCell.Style.Font.FontColor = CautionFont;          // #E65100
    sourceCell.Style.Font.Bold = true;
}
```

### PcDetailWindow

ヘッダ直下にそのまま `SerialNumberDisplay` を表示。フォールバック時はオレンジ強調 + SemiBold。`SerialSourceTrail` の全候補は **PcDetailWindow には現状表示されない**（将来的な詳細パネルとして余地あり、Excel にも未出力）。

---

## Caution（黄）への流入

`MainWindowViewModel.EvaluateCaution(pc)` 内：

```csharp
if (pc.HasSerialFallbackWarning)
    cautions.Add($"SNフォールバック(source={SerialSourceEntry.ShortLabel(pc.SerialNumberSource)})");
```

→ Caution メッセージ例: `SNフォールバック(source=CSProduct)` が CautionMessage に並ぶ。DataGrid 行が黄色になる（赤と同時時は赤勝ち）。

詳細は [fabriq_evidence_manager__architecture__03_warning_caution_model.md](fabriq_evidence_manager__architecture__03_warning_caution_model.md) §「Caution（黄）の判定軸」を参照。

---

## SN 取得不能ケース（Warning 側）

採用候補 5 つすべてが Invalid / QueryFailed → `Canonical Serial Number` ブロック値は `(Unretrievable)` → `CanonicalValue = ""`。

このとき：

- `SerialNumberDisplay = "(取得不能)"`
- `HasSerialFallbackWarning = false`（SN 自体が空のため、フォールバック判定外）
- **Warning 側で `"SN取得不能"` メッセージが立つ**（[architecture__03](fabriq_evidence_manager__architecture__03_warning_caution_model.md) §「Warning に立つ条件」）

`SerialNumber` 取得不能は CheckOption に紐づかない常時警告のため、設定ダイアログで OFF にできない。

---

## Audit シナリオ別フローチャート

### シナリオ A: 完全一致（理想）

```
全 5 候補 = "ABC123XYZ" (or 4 候補一致 + BaseBoard だけ別値)
↓
Canonical: ABC123XYZ (Source: Win32_BIOS.SerialNumber)
↓
SerialNumber = ABC123XYZ
SerialNumberSource = "Win32_BIOS.SerialNumber"
IsSerialFromPrimarySource = true
HasSerialFallbackWarning = false
↓
DataGrid: "ABC123XYZ" (デフォルト色)
Caution: 立たない
```

### シナリオ B: BIOS が空、CSP が値持ち（フォールバック）

```
Win32_BIOS.SerialNumber : (empty) [INVALID: empty]
Win32_ComputerSystemProduct.IdentifyingNumber : ABC123XYZ [VALID]
他 : ABC123XYZ [VALID, MATCH] や (empty)
↓
fabriq 側ポリシー: BIOS 失敗時 → CSP にフォールバック
Canonical: ABC123XYZ (Source: Win32_ComputerSystemProduct.IdentifyingNumber)
↓
SerialNumber = ABC123XYZ
SerialNumberSource = "Win32_ComputerSystemProduct.IdentifyingNumber"
IsSerialFromPrimarySource = false
HasSerialFallbackWarning = true        ← 警告対象
↓
DataGrid: "ABC123XYZ ⚠ (CSProduct)" (オレンジ背景)
Caution: "SNフォールバック(source=CSProduct)" 追加
Excel: SN ソース列がオレンジ背景
```

### シナリオ C: 全候補失敗

```
Win32_BIOS.SerialNumber : (empty) [INVALID]
Win32_ComputerSystemProduct.IdentifyingNumber : (empty) [INVALID]
Win32_SystemEnclosure.SerialNumber : QueryFailed
Win32_BaseBoard.SerialNumber : MB-987654 [VALID]   ← record-only、採用候補外
Registry SystemSerialNumber : QueryFailed
↓
Canonical: (Unretrievable)
↓
SerialNumber = ""
SerialNumberSource = null
IsSerialFromPrimarySource = false
HasSerialFallbackWarning = false   ← SN 自体が空なので「フォールバック」ではない
↓
DataGrid: "(取得不能)" (デフォルト色)
Warning: "SN取得不能" 追加（赤行）
Caution: SN 関連は立たない
```

### シナリオ D: 値はあるが MATCH しない（候補間で値が違う）

```
Win32_BIOS.SerialNumber : ABC123 [VALID, DIFFERENT]
Win32_ComputerSystemProduct.IdentifyingNumber : XYZ789 [VALID, DIFFERENT]
他 : 値 or (empty)
↓
fabriq 側ポリシー: BIOS が VALID なら BIOS を採用 (DIFFERENT でも)
Canonical: ABC123 (Source: Win32_BIOS.SerialNumber)
↓
SerialNumber = ABC123
SerialNumberSource = "Win32_BIOS.SerialNumber"
IsSerialFromPrimarySource = true
HasSerialFallbackWarning = false   ← BIOS なので警告は立たない
↓
DataGrid: "ABC123" (デフォルト色)
Excel: 着色なし
```

ただし **SerialSourceTrail** には全 5 行が記録されており、後で SerialSourceTrail を SystemEnclosure / CSProduct と突き合わせれば「全候補値の食い違い」を検出可能（v3.8.0 では UI 出力していない）。

---

## まとめ: フォールバック警告の意味

| 状態 | SN 値 | 採用ソース | フォールバック警告 | 監査対応 |
|---|---|---|---|---|
| 完全取得 | あり | BIOS | 出ない | OK |
| BIOS 失敗 + 他で取得 | あり | CSP / Enclosure / Registry | **発動** | OEM SMBIOS 確認、必要なら本体 SN 物理確認 |
| 全候補失敗 | 空 | null | 出ない（代わりに Warning 側で `SN取得不能`）| 物理 SN シール確認、再収集 |
| BaseBoard だけ取得（記録のみ）| 空 | null | 出ない | 同上（BaseBoard は採用候補外）|

「フォールバック警告 = OEM SMBIOS の BIOS フィールド書き込みに不備」を示す現場運用上の早期検知シグナルとして機能する。

---

## 関連ドキュメント

- §10 ファイル形式（生のテキスト構造）: [fabriq_evidence_manager__reference__file_format__pc_information.md](fabriq_evidence_manager__reference__file_format__pc_information.md)
- Models 索引（`SerialSourceEntry / SerialNumberDetail / SerialSourceValidity`）: [fabriq_evidence_manager__reference__model_catalog.md](fabriq_evidence_manager__reference__model_catalog.md)
- Warning / Caution 2 軸判定: [fabriq_evidence_manager__architecture__03_warning_caution_model.md](fabriq_evidence_manager__architecture__03_warning_caution_model.md)
- Excel 出力（SN ソース列の着色規則）: [fabriq_evidence_manager__reference__excel_layout.md](fabriq_evidence_manager__reference__excel_layout.md)
- セクション ID dispatch（§10 の専用ロジック呼出）: [fabriq_evidence_manager__contracts__section_dispatch.md](fabriq_evidence_manager__contracts__section_dispatch.md)
