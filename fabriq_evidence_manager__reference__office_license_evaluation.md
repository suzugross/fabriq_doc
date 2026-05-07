# Office ライセンス評価ロジック（§22）

> **対象**: fabriq_evidence_manager / reference
> **対象バージョン**: 3.8.1（取得元: `E:\fabriq_evidence_manager\FabriqEvidenceManager\FabriqEvidenceManager.csproj` `<Version>` / commit `45eae22`）
> **ドキュメント更新日**: 2026-05-07

`§22 Office License` は本アプリで最も判定が複雑なセクション。fabriq 側 evidence_config の **v1.5.0 で manifest §22 verdict + INTERPRETATION ブロックが追加** され、本アプリは**新旧両 evidence を扱うため二系統の判定経路を持つ**。本ドキュメントは判定の権威ソース選択・5 状態の `OfficeLicenseEvaluation` 列挙・v1.5+ と v1.4 の違いを実装ベースで明文化する。

---

## なぜ複雑か

Office のライセンス状態は単一の OS フィールドからは決まらない：

- **Click-to-Run（C2R）レジストリ**: インストール製品 ID + Update Channel
- **OSPP.vbs /dstatus**: Volume / Retail のライセンス状態（per-product）
- **vNext per-user license file**: M365 サブスクリプションのユーザー別 provision 状態

これら 3 つの情報を「OS が認証されているか」に単純に集約できないため、fabriq 側 v1.5.0+ で **PowerShell 側で突合済みの verdict（`Success / Partial / Failed`）を manifest §22 status に書き出す** 仕様が公開化された。consumer 側は manifest verdict を信じればよく、再判定は不要。

ただし v1.4 以前の evidence では INTERPRETATION ブロックが無く verdict も「単に section 収集が完了した」の意味しかなかった。本アプリは **両方の evidence をサポート** するため、INTERPRETATION の有無で経路を切り分ける。

---

## v1.4 と v1.5+ の差分

| 項目 | v1.4 以前 | v1.5.0+ |
|---|---|---|
| Click-to-Run `DetectedAsSubscription` 行 | 無 | **あり**（subscription / 買切り判定の補助）|
| INTERPRETATION ブロック | 無 | **あり** |
| `vNext Provisioned: N` 行 | 無 | あり |
| `OSPP shows Grace/Notify: <bool>` 行 | 無 | あり |
| `CONCLUSION: ...` 行 | 無 | あり |
| `22_OfficeVnextLicenses.csv` | 無 | **あり**（CSV、16 列） |
| manifest §22 status の意味 | "セクション収集完了" のみ（VL+OSPP_Grace でも Success）| **真の verdict** （`Success/Partial/Failed`）|

判定切替えのキーは `OfficeLicenseData.InterpretationConclusion is not null` の 1 行：

```csharp
public OfficeLicenseEvaluation OfficeLicenseEvaluation
{
    get
    {
        var lic = OfficeLicense;
        if (lic is null) return OfficeLicenseEvaluation.NotEvaluated;
        if (!lic.IsOfficeInstalled) return OfficeLicenseEvaluation.NotInstalled;

        // v1.5.0+ INTERPRETATION ブロックがある場合は manifest verdict を信頼
        if (lic.InterpretationConclusion is not null)
        {
            var verdict = Manifest?.Sections.FirstOrDefault(s => s.Id == "22")?.Status;
            return verdict switch
            {
                SectionStatus.Success => OfficeLicenseEvaluation.Licensed,
                SectionStatus.Partial => OfficeLicenseEvaluation.SignInPending,
                SectionStatus.Failed  => OfficeLicenseEvaluation.AuthFailed,
                _ => OfficeLicenseEvaluation.AuthFailed,
            };
        }

        // v1.4 以前: OSPP product / vNext で個別判定
        if (lic.VnextEntries.Any(e => e.IsProvisioned)) return OfficeLicenseEvaluation.Licensed;
        if (lic.Products.Any(p => p.IsLicensed)) return OfficeLicenseEvaluation.Licensed;
        return OfficeLicenseEvaluation.AuthFailed;
    }
}
```

`PcEvidence.OfficeLicenseEvaluation` 算出プロパティが評価結果を返し、Warning / Caution / Excel / DataGrid 各レイヤがこれを参照する。

---

## OfficeLicenseEvaluation 列挙（5 状態）

| 値 | 意味 | 流れる軸 | DataGrid 表示 |
|---|---|---|---|
| `NotEvaluated` | OfficeLicense 未取得（manifest §22 不在 / Failed） | Warning（`Office情報未取得`）| `(未取得)` |
| `NotInstalled` | C2R / OSPP どちらも未検出 | Warning（`Office未インストール`）| `未インストール` |
| `Licensed` | manifest §22 verdict=Success / 旧 evidence の OSPP IsLicensed=true | （正常、警告なし） | `認証済` |
| `SignInPending` | manifest §22 verdict=Partial（M365 サブスクリプションだが vNext 未 Provisioned）| **Caution**（`Officeサインイン待ち`）| `サインイン待ち` |
| `AuthFailed` | manifest §22 verdict=Failed / 旧 evidence の OSPP IsLicensed=false | Warning（`Officeライセンス認証失敗`）| `認証失敗` |

5 状態のうち **`SignInPending` だけが Caution（黄）に流れる** のが本アプリの設計上の重要点（[fabriq_evidence_manager__architecture__03_warning_caution_model.md](fabriq_evidence_manager__architecture__03_warning_caution_model.md) §「Office License の振り分け」参照）。

### `SignInPending` を Caution 側に分離した理由

M365 サブスクリプション機で「OS イメージ展開直後でユーザーがまだサインインしていない」状態は **真の認証失敗ではない**：

- 24h 以内にユーザーがサインインすれば自動 provision される（vNext のしくみ）
- `OfficeLicense.VnextEntries.Any(e => e.IsProvisioned)` が現時点で false でも、Office 自体は通常起動できる（trial mode）
- 検収作業者は黄色を見て「明日もう一度確認」「ユーザー側のサインイン手順案内」等の運用判断を打つ

一方 `Failed` は「VL ライセンス切れ / KMS 不通 / プロダクトキー消失」等の **物理対処が必要な事象** であり赤色（Warning）に分けている。

---

## v1.5+ 経路: manifest §22 verdict を権威化

### 入力

`22_OfficeLicense.txt` の 4 ブロック構成：

```
---- Office Click-to-Run Configuration (registry) ----
ProductReleaseIds: O365ProPlusRetail
VersionToReport: 16.0.17932.20742
Platform: x64
CDNBaseUrl: ...
UpdateChannel: ...
AudienceData: Production::LTSC2024
ClientCulture: ja-jp
DetectedAsSubscription: True

---- OSPP.vbs path ----
C:\Program Files\Microsoft Office\Office16\OSPP.VBS

---- cscript OSPP.vbs /dstatus (raw) ----
（Product ブロックが "---" で区切られた人間向け raw text）

---- vNext Per-User License Files ----
（人間向けサマリ。consumer は raw を捨てて CSV を真値とする）

---- INTERPRETATION ----
OSPP shows Grace/Notify:    True
vNext Provisioned:          2
  CONCLUSION: LICENSED (M365 subscription, 2 Provisioned vNext license(s)).
```

加えて `22_OfficeVnextLicenses.csv`（16 列）。

### 判定フロー

```
1. OfficeLicense is null
   → NotEvaluated（manifest §22 不在 / Failed で Sections に entry がないケース）

2. !IsOfficeInstalled
   → NotInstalled
   ※ IsOfficeInstalled = (ClickToRun is not null) || (OsppPath is not null)

3. InterpretationConclusion is not null（v1.5.0+）
   → manifest.Sections.FirstOrDefault(s => s.Id == "22")?.Status を見る:
        Success → Licensed
        Partial → SignInPending     ← Caution
        Failed  → AuthFailed
        その他/null → AuthFailed   （防御的）

4. (上記すべて該当しない、つまり InterpretationConclusion is null = v1.4 以前)
   → 旧経路へ（次節）
```

### v1.5+ の verdict セマンティクス

producer 側 `evidence_config` v1.5.0 が manifest §22 status に書く 3 値：

| Status | 意味 | 典型ケース |
|---|---|---|
| `Success` | LICENSED 確定 | M365 サブスクリプション + vNext provision 済み / VL + OSPP LICENSED |
| `Partial` | サインイン待ち | M365 サブスクリプション + vNext 未 Provisioned（24h 以内にサインイン期待） |
| `Failed` | 認証失敗 | VL + OSPP NOT_LICENSED / Retail + OOB_GRACE 切れ等 |

producer 側が C2R + OSPP + vNext を突合済みなので、consumer 側はこれを丸呑みする方針。

---

## v1.4 旧経路: OSPP / vNext で個別判定

INTERPRETATION ブロック不在（`InterpretationConclusion is null`）の場合、本アプリは **OSPP IsLicensed と vNext IsProvisioned を C# 側で個別判定** する：

```csharp
if (lic.VnextEntries.Any(e => e.IsProvisioned)) return OfficeLicenseEvaluation.Licensed;
if (lic.Products.Any(p => p.IsLicensed))         return OfficeLicenseEvaluation.Licensed;
return OfficeLicenseEvaluation.AuthFailed;
```

優先順位：

1. **vNext provision 済みエントリが 1 件でもある** → `Licensed`
   - subscription 機が混じっていれば vNext あり = Licensed と認める
2. **OSPP の LICENSE STATUS が `LICENSED` のプロダクトが 1 件でもある** → `Licensed`
   - VL / Retail なら OSPP の LICENSED 判定のみ受理
3. それ以外 → `AuthFailed`

### v1.4 旧経路の判定限界

manifest §22 status は v1.4 以前では「単に collection が完了した」の意味しかなく、**実態を保証しない**：

- VL + OSPP_GRACE でも status=Success
- M365 + vNext 0 件 でも status=Success

そのため v1.4 旧経路は本アプリ側で OSPP/vNext を見て再判定する。**manifest verdict は信頼しない**。

ただし v1.4 旧経路では `SignInPending` 状態を本アプリが検知できない（INTERPRETATION ブロック無し → vNext 未 Provisioned が「すべての OSPP も未 LICENSED」と区別できない）。これは v1.4 以前の evidence の限界として受け入れる。

---

## IsOfficeInstalled の判定

`OfficeLicenseData.IsOfficeInstalled` は parser 側で計算される flag：

```csharp
return new OfficeLicenseData
{
    IsOfficeInstalled = clickToRun is not null || osppPath is not null,
    ...
};
```

| ClickToRun | OsppPath | IsOfficeInstalled |
|---|---|---|
| あり | あり | **true** |
| あり | null（OSPP not found） | **true**（C2R Subscription 機で OSPP が無いケース） |
| null | あり（MSI-based Office） | **true** |
| null | null | **false** |

`ClickToRun` 判定キー: parse 結果の Dictionary に `ProductReleaseIds` キーが含まれているか。`OsppPath` 判定キー: `---- OSPP.vbs path ----` ヘッダ + 次行に絶対パス文字列があれば設定（`---- OSPP.vbs ----` 単独 → not found → null）。

---

## SubscriptionType 表示の決定（Excel `Officeライセンス` シート）

`ExcelExportService.ResolveSubscriptionType(lic)` の優先順位：

```csharp
private static string ResolveSubscriptionType(OfficeLicenseData lic)
{
    if (!lic.IsOfficeInstalled) return "Not installed";
    if (lic.IsSubscriptionDetected is null) return "Unknown";          // v1.4 以前
    if (lic.IsSubscriptionDetected.Value) return "M365 Subscription";

    var ids = lic.ClickToRun?.ProductReleaseIds ?? string.Empty;
    return ids.Contains("Volume", StringComparison.OrdinalIgnoreCase)
        ? "Volume"
        : "Buy-once";
}
```

| 条件 | 表示 |
|---|---|
| `!IsOfficeInstalled` | `Not installed` |
| v1.4 以前（`IsSubscriptionDetected is null`） | `Unknown` |
| v1.5+ かつ `IsSubscriptionDetected = true` | `M365 Subscription` |
| v1.5+ かつ `IsSubscriptionDetected = false` かつ ProductReleaseIds に "Volume" 含む | `Volume` |
| v1.5+ かつ `IsSubscriptionDetected = false` かつ "Volume" 含まない | `Buy-once` |

### `IsSubscriptionDetected` の出処

v1.5+ Click-to-Run ブロックの `DetectedAsSubscription:` 行を bool パース。`Test-OfficeSubscriptionSku` の結果を fabriq 側が記録。`null` は v1.4 以前 / C2R 未検出のケース。

---

## ApplyOfficeLicenseRowColor（Excel 着色ロジック）

`ExcelExportService.ApplyOfficeLicenseRowColor` は **行全体に着色** する：

```csharp
if (verdict is null)
{
    // 旧 evidence: manifest §22 不在
    if (!lic.IsOfficeInstalled)
        range.Style.Fill.BackgroundColor = InfoBg;             // 薄グレー
    else if (product is null || !product.IsLicensed)
    {
        range.Style.Fill.BackgroundColor = MismatchBg;          // 薄赤
        range.Style.Font.FontColor = MismatchFont;
    }
    return;
}

switch (verdict.Value)
{
    case SectionStatus.Failed:
        range.Style.Fill.BackgroundColor = MismatchBg;          // 薄赤
        range.Style.Font.FontColor = MismatchFont;
        break;
    case SectionStatus.Partial:
        range.Style.Fill.BackgroundColor = WarningBg;           // 薄黄
        break;
    case SectionStatus.Success:
        if (!lic.IsOfficeInstalled)
            range.Style.Fill.BackgroundColor = InfoBg;          // 情報行
        else
            range.Style.Fill.BackgroundColor = MatchBg;         // 薄緑
        break;
    case SectionStatus.Skipped:
        range.Style.Font.FontColor = XLColor.Gray;
        break;
}
```

要約：

| verdict | IsOfficeInstalled | 行着色 |
|---|---|---|
| null（旧 evidence）| false | 薄グレー（情報行）|
| null | true + IsLicensed | 着色なし |
| null | true + NOT IsLicensed | **薄赤 + 赤文字** |
| Success | false | 薄グレー |
| Success | true | **薄緑** |
| Partial | * | **薄黄**（M11 監査用） |
| Failed | * | **薄赤 + 赤文字** |
| Skipped | * | Gray 文字 |

---

## 各レイヤでの利用

### DataGrid（MainWindow）

「Office 認証」列の `OfficeLicenseStatusDisplay`：

```csharp
public string OfficeLicenseStatusDisplay => OfficeLicenseEvaluation switch
{
    OfficeLicenseEvaluation.Licensed       => "認証済",
    OfficeLicenseEvaluation.SignInPending  => "サインイン待ち",
    OfficeLicenseEvaluation.AuthFailed     => "認証失敗",
    OfficeLicenseEvaluation.NotInstalled   => "未インストール",
    _                                      => "(未取得)",
};
```

`LicenseStatusToBackgroundConverter` でセル背景色を 4 段階で変える（薄緑 / 薄黄 / 薄赤 / 薄グレー）。

### Warning（Excel 着色とは別系統）

`ValidateEvidence` 内で：

```csharp
if (options?.CheckOfficeInstalled != false)
{
    if (pc.OfficeLicense is null) warnings.Add("Office情報未取得");
    else if (!pc.OfficeLicense.IsOfficeInstalled) warnings.Add("Office未インストール");
}
if (options?.CheckOfficeLicense != false
    && pc.OfficeLicense is { IsOfficeInstalled: true }
    && pc.OfficeLicenseEvaluation == OfficeLicenseEvaluation.AuthFailed)
{
    warnings.Add("Officeライセンス認証失敗");
}
```

3 つの警告条件：

- `CheckOfficeInstalled=ON` + OfficeLicense null → `Office情報未取得`
- `CheckOfficeInstalled=ON` + IsOfficeInstalled=false → `Office未インストール`
- `CheckOfficeLicense=ON` + IsOfficeInstalled=true + Evaluation=AuthFailed → `Officeライセンス認証失敗`

### Caution

`EvaluateCaution` 内で：

```csharp
if (CheckOptions.CheckOfficeLicense
    && pc.OfficeLicenseEvaluation == OfficeLicenseEvaluation.SignInPending)
{
    cautions.Add("Officeサインイン待ち");
}
```

`CheckOfficeLicense=ON` + Evaluation=SignInPending のみ追加。

### Baseline 突合（License カテゴリ）

`LicenseComparator` は `OfficeLicenseEvaluation` ではなく **typed model のフィールド直接比較**：

- `IsOfficeInstalled`（インストール状態）
- `ClickToRun.ProductReleaseIds / UpdateChannel`
- `Products[0].LicenseStatusText`

Evaluation の 5 状態判定はあくまで「Warning / Caution / DataGrid 表示用」、ベースライン突合は別軸。詳細は [fabriq_evidence_manager__usage__03_baseline.md](fabriq_evidence_manager__usage__03_baseline.md) §「ライセンス（License）」を参照。

---

## evidence バージョン互換マトリクス

| evidence_config 版 | INTERPRETATION ブロック | vNext CSV | DetectedAsSubscription | 本アプリの経路 | SignInPending 検知 |
|---|---|---|---|---|---|
| 1.3.0 〜 1.4.x | 無 | 無 | 無 | **v1.4 旧経路** | 不可 |
| 1.5.0+ | あり | あり | あり | **v1.5+ 経路** | 可 |
| 1.6.0+ | あり | あり | あり | v1.5+ 経路 | 可 |

producer 側の今後のバージョンアップで manifest schemaVersion=2 が出るときに `OfficeLicenseEvaluation` の枠組みも見直す可能性はあるが、現行 schemaVersion=1 内での §22 拡張は本実装で吸収できる設計（INTERPRETATION の有無 + manifest verdict のフォールバックチェーンで対応）。

---

## 関連ドキュメント

- §22 ファイル形式（生のテキスト構造）: [fabriq_evidence_manager__reference__file_format__pc_information.md](fabriq_evidence_manager__reference__file_format__pc_information.md)
- Models 索引（`OfficeLicenseData / OfficeClickToRunConfig / OfficeProductLicense / OfficeVnextLicenseEntry / OfficeLicenseEvaluation`）: [fabriq_evidence_manager__reference__model_catalog.md](fabriq_evidence_manager__reference__model_catalog.md)
- Warning / Caution 2 軸判定（Office Partial = Caution の理由）: [fabriq_evidence_manager__architecture__03_warning_caution_model.md](fabriq_evidence_manager__architecture__03_warning_caution_model.md)
- Excel 出力（`Officeライセンス` シート 23 列 + 着色）: [fabriq_evidence_manager__reference__excel_layout.md](fabriq_evidence_manager__reference__excel_layout.md)
- セクション ID dispatch（§22 の専用ロジック呼出）: [fabriq_evidence_manager__contracts__section_dispatch.md](fabriq_evidence_manager__contracts__section_dispatch.md)
- 設定ダイアログ（`CheckOfficeInstalled / CheckOfficeLicense`）: [fabriq_evidence_manager__apps__03_settings_window.md](fabriq_evidence_manager__apps__03_settings_window.md)
