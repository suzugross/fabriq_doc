# 暗号化相互運用契約（PowerShell との互換性）

> **対象**: fabriq_studio / contracts / 暗号化
> **対象バージョン**: commit `3897c6e`
> **ドキュメント更新日**: 2026-05-06

fabriq_studio の `CryptoService` は fabriq 本体の PowerShell 関数 `Unprotect-FabriqValue`（`E:\fabriq\kernel\common.ps1`）と **完全互換** で実装されている。両者は同じ平文 / 同じパスフレーズに対して **必ず同じ暗号文** を生成し、互いの暗号文を復号できる。

このドキュメントは仕様の真のソースとして、両側の実装が将来分岐した場合に参照すべき契約を明文化する。

---

## アルゴリズム仕様

| 項目 | 値 |
|---|---|
| 鍵導出 | PBKDF2-HMAC-SHA256 |
| イテレーション数 | **100,000** |
| ソルト | **固定** `"fabriq-fixed-salt-2024"`（UTF-8 バイト列） |
| 暗号化 | AES-256-CBC |
| パディング | PKCS7 |
| エンコード（平文） | UTF-8 |
| エンコード（暗号文） | Base64 |
| 出力プレフィックス | `ENC:` |
| KEY サイズ | 32 バイト（AES-256） |
| IV サイズ | 16 バイト（AES ブロックサイズ 128 bit） |

---

## KEY と IV の導出順序（最重要）

PBKDF2 の出力ストリームから **KEY を先に 32 バイト、続いて IV を 16 バイト** 取り出す。`Rfc2898DeriveBytes` の `GetBytes()` は呼び出し回数で内部位置が進むため、**順序を入れ替えると別の鍵対が得られる**。

[Services/CryptoService.cs](file:///E:/fabriq_studio/FabriqStudio/Services/CryptoService.cs) の実装:

```csharp
private static (byte[] Key, byte[] IV) DeriveKeyIv(string passphrase)
{
    using var kdf = new Rfc2898DeriveBytes(
        passphrase, FixedSalt, Iterations, HashAlgorithmName.SHA256);
    var key = kdf.GetBytes(KeySize);   // 先に 32 バイト
    var iv  = kdf.GetBytes(IvSize);    // 続いて 16 バイト
    return (key, iv);
}
```

PowerShell 側（`Unprotect-FabriqValue`）も同じ順序で実装されている。**この順序は破壊的変更に該当する** ので、片側を変更する場合はもう片側も同コミットで合わせる必要がある。

---

## 出力フォーマット

```
平文        : "secret"
暗号文      : "ENC:<base64-encoded-ciphertext>"
```

- `ENC:` プレフィックスは **暗号化されていることのマーカー**
- 復号時はプレフィックス有無を許容（[CryptoService.cs](file:///E:/fabriq_studio/FabriqStudio/Services/CryptoService.cs) の `Decrypt` は `StartsWith("ENC:")` で剥がしてから Base64 デコードする）
- CSV / JSON ファイル中のセル値は基本的にプレフィックス込みで保存される
- 平文として保存したい値はプレフィックス無しで書く（fabriq 側もプレフィックス無しは平文として透過）

---

## パスフレーズ管理

### 状態保持

`ICryptoService.MasterPassphrase`（nullable string）にプロセスメモリ内で保持する。永続化はしない（セキュリティ上、設定ファイルに書かない設計）。

`ICryptoService.HasPassphrase` プロパティで「パスフレーズ設定済みか」を判定。

### パスフレーズ未設定時のガード

`Helpers/CryptoHelper.cs`:

```csharp
public static string? ValidatePassphrase(ICryptoService crypto)
    => crypto.HasPassphrase
        ? null
        : "パスフレーズが設定されていません。\n左ペイン下部の「🔑 パスフレーズ」から設定してください。";
```

バッチ暗号化系の操作（hostlist 一括暗号化など）はこのチェックを必ず通す。

### 検証トークン（passphrase_verify.txt）

ワークスペースに `kernel/txt/passphrase_verify.txt` が存在する場合、そこには平文 `"surkitinisme"` を **現在のパスフレーズで暗号化したもの** が書き込まれている。MainViewModel が `SetPassphrase` 操作の際に以下の照合を行う:

1. ユーザが入力した新パスフレーズで `passphrase_verify.txt` の内容を復号
2. 復号結果が `"surkitinisme"` と一致 → パスフレーズ正しい
3. 不一致または例外 → 「パスフレーズが正しくありません」エラー表示してブロック

これにより、複数の Studio 起動間 / 異なるユーザ間でパスフレーズが食い違う事故を防ぐ。

新規ワークスペースでまだトークンが無い場合は、初回設定時に `Encrypt("surkitinisme", passphrase)` で生成して書き出す（`kernel/txt/` フォルダが無ければ作成）。

`MasterPassphrase = null` でクリアした場合、検証トークンファイルは消さない（次回設定時に再照合できるように）。

---

## 暗号化対象列 allowlist

CSV のセル全てを暗号化するわけではなく、ID 列・制御列・スクリプトパスなどは平文で残す必要がある。`Helpers/CryptoHelper.cs` の `ExcludedColumns`:

```
AdminID, TaskID, StepID, ID, No,
Enabled,
Order, Step, Segment,
Category, Type, Kind, MenuName,
MaxRetry, IntervalSec, TimeoutSec,
Script, ScriptPath, Action,
Condition,
```

`IsEncryptableColumn(columnName)` は大文字小文字を区別せずこのセットに含まれない列名を「暗号化可能」と判定する。バッチ暗号化系の処理（hostlist など）はこの allowlist を尊重し、対象外列はスキップする。

新しい列を追加した場合、暗号化対象にすべきでないなら `ExcludedColumns` に追加する。`CryptoHelper` の編集は片箇所で全機能に効く。

---

## バッチ操作の戻り値

`Helpers/CryptoHelper.cs` の `BatchCryptoResult` レコード:

```csharp
public record BatchCryptoResult(int Processed, int Skipped, IReadOnlyList<string> Errors)
{
    public bool   HasErrors { get; }
    public string ToSummary(bool isEncrypt);
}
```

- `Processed` — 暗号化／復号した件数
- `Skipped` — `ExcludedColumns` または既に対応する状態（ENC: 付なのに再暗号要求など）でスキップした件数
- `Errors` — 個別行で失敗したエラーメッセージ（最大 10 件まで `ToSummary` で表示）

---

## Pianist Test Run での動作

`IPianistTestRunService` は子プロセス起動時にラッパスクリプト内で:

```powershell
$global:FabriqMasterPassphrase = '<Studio が保持する MasterPassphrase>'
```

を注入する。pianist.ps1 / kernel/common.ps1 はこのグローバル変数があれば `Unprotect-FabriqValue` を自動適用するため、values.csv の `ENC:` セルを実行時に正しく復号できる。

注入されたパスフレーズは子プロセス終了で消える。Studio 本体のメモリにはそのまま残る。

---

## 互換性に関わる変更管理

以下のいずれかを変更する場合、PowerShell 側 (`Unprotect-FabriqValue`) と **同コミットで両側を更新** する必要がある:

- イテレーション数（100,000）
- 固定ソルト文字列（`"fabriq-fixed-salt-2024"`）
- KEY/IV の導出順序
- KEY / IV のサイズ
- AES モード（CBC）
- パディングモード（PKCS7）
- 平文・暗号文のエンコード（UTF-8 / Base64）
- `ENC:` プレフィックス文字列

これらは fabriq KERNEL_API.md の公開契約として扱う（実装の真のソース確認は両側の差分テストでのみ可能）。

`ExcludedColumns` の追加は片側のみ（Studio）で完結するが、新しい列を CSV に追加した際にどちらかのみ「暗号化対象として扱う／扱わない」が分かれると Studio 上の表示と pianist 実行時の挙動が食い違うため、列を追加した時点で `ExcludedColumns` の見直しを行うこと。
