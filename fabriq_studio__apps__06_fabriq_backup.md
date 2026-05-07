# fabriq バックアップ（FabriqBackup）

> **対象**: fabriq_studio / apps / Fabriq Backup Dialog
> **対象バージョン**: commit `3897c6e`（取得元: `git -C E:\fabriq_studio rev-parse --short HEAD`）
> **ドキュメント更新日**: 2026-05-07

現在開いているワークスペース全体を、PS1 や CHANGELOG 等を除外した **設定ファイルのみのミラーコピー**として別フォルダに複製するダイアログツール。

主な目的:

- 顧客現場へ持ち出す際の **設定値のみのスナップショット** を作る
- 検証環境にコピーする際に、**ロジック（PS1）と設定（CSV/JSON）を切り離す**
- 「壊れたら戻れる退避コピー」を取りつつ、配布物として軽量化する（実行ロジックは fabriq 本体に存在するため除外）

---

## 概要

| 項目 | 内容 |
|---|---|
| ダイアログ | [Views/FabriqBackupDialog.xaml](file:///E:/fabriq_studio/FabriqStudio/Views/FabriqBackupDialog.xaml) |
| Service | `IFabriqBackupService` / `FabriqBackupService` |
| Model | `FabriqBackupRequest` / `FabriqBackupResult`（[Models/FabriqBackupRequest.cs](file:///E:/fabriq_studio/FabriqStudio/Models/FabriqBackupRequest.cs)） |
| 実行モード | モーダルダイアログ（コードビハインドで Service 直呼び） |
| 入力 | ワークスペースルート + 出力先親フォルダ + メモ |
| 出力 | タイムスタンプ付きフォルダにミラー + メタファイル 2 種 |

---

## DTO（Request / Result）

### `FabriqBackupRequest`

```csharp
public record FabriqBackupRequest(
    string SourceRoot,    // コピー元の fabriq ルート（ワークスペース絶対パス）
    string ParentFolder,  // ユーザー選択の出力先親フォルダ
    string Memo);         // USER_MEMO.txt 用、空可
```

immutable record。Service が並列呼び出しされても安全。

### `FabriqBackupResult`

```csharp
public record FabriqBackupResult(
    string BackupFolderPath,           // 生成されたバックアップフォルダ絶対パス
    int    CopiedFileCount,            // コピーされたファイル総数（素材フォルダ含む）
    int    ExcludedFileCount,          // ポリシーで除外された数（素材フォルダ内は除く）
    long   TotalBytes,                 // コピー合計バイト数
    IReadOnlyList<string> Errors)      // コピー失敗等のエラー
{
    public bool HasErrors => Errors.Count > 0;
}
```

---

## 出力構造

```
<ParentFolder>/fabriq_backup_yyyyMMdd_HHmmss/
├── kernel/
│   ├── csv/                      ── 設定 CSV のミラー
│   ├── json/                     ── 設定 JSON のミラー（ランタイム状態は除外）
│   └── txt/
│       └── ...
├── modules/
│   ├── standard/<module>/
│   │   ├── *.csv                 ── 設定 CSV
│   │   ├── INF/                  ── 素材フォルダ（除外を外して丸ごと保存）
│   │   ├── assets/               ── 素材フォルダ
│   │   └── ...
│   └── extended/<module>/
│       └── ...
├── profiles/                     ── プロファイル一式
├── ...（fabriq の通常構成をミラー）
├── USER_MEMO.txt                 ── ダイアログでユーザが入力したメモ
└── BACKUP_INFO.txt               ── 自動生成（タイムスタンプ・件数・除外ルール一覧）
```

タイムスタンプ形式: `fabriq_backup_yyyyMMdd_HHmmss`（例: `fabriq_backup_20260507_143012`）。

---

## 除外ポリシー（[FabriqBackupService.cs](file:///E:/fabriq_studio/FabriqStudio/Services/FabriqBackupService.cs)）

除外は 4 階層で適用される。**素材フォルダ例外**（後述）が最後の上書きルールとして優先される。

### 1. トップレベル除外ディレクトリ

```csharp
private static readonly HashSet<string> ExcludedDirs = new(StringComparer.OrdinalIgnoreCase)
{
    ".git", ".claude", "dev", "logs", "apps", "commands", "evidence",
    "kernel/ps1",
};
```

fabriq ルート直下にあるこれらのディレクトリは **丸ごとスキップ**。`/` 区切りで `kernel/ps1` のように 2 階層も指定可能（`Replace('\\', '/')` で正規化される）。

| ディレクトリ | 除外理由 |
|---|---|
| `.git` / `.claude` | バージョン管理 / AI 設定 — 設定ではない |
| `dev` | 開発専用ファイル一式 |
| `logs` | 過去ログ — バックアップに含めない |
| `apps` | 同梱バイナリ群（fabriq 本体由来） |
| `commands` | コマンドスクリプト一式 |
| `evidence` | 過去収集エビデンス — 別チャネルで管理 |
| `kernel/ps1` | カーネル PS1 一式 — fabriq 本体側 |

### 2. グローバル拡張子除外

```csharp
private static readonly HashSet<string> ExcludedExtensions = new(StringComparer.OrdinalIgnoreCase)
{
    ".ps1", ".bat", ".bak", ".tmp", ".log",
};
```

どこに配置されていても常に除外。**設定値のみ保存** する方針の中核。

### 3. グローバルファイル名除外

```csharp
private static readonly HashSet<string> ExcludedFilenames = new(StringComparer.OrdinalIgnoreCase)
{
    ".gitignore", ".gitkeep",
    "CLAUDE.md", "CHANGELOG.md",
    "KERNEL_VERSION", "KERNEL_API.md",
    "VERSION", "REQUIRES_KERNEL",
    "Guide.txt", "preset.csv",
    "Fabriq.exe",
};
```

| カテゴリ | ファイル | 理由 |
|---|---|---|
| Git メタ | `.gitignore`/`.gitkeep` | 配布先で不要 |
| ドキュメント | `CLAUDE.md`/`CHANGELOG.md` | fabriq 本体由来、設定ではない |
| 版管理 | `KERNEL_VERSION`/`VERSION`/`REQUIRES_KERNEL` | バックアップ後に上書きインストールする際の混乱を防ぐ |
| API 仕様 | `KERNEL_API.md` | fabriq 本体由来 |
| ガイド | `Guide.txt` | モジュール手順書（fabriq 本体由来） |
| プリセット | `preset.csv` | プリセット候補値定義（設計時のもの） |
| 実行ファイル | `Fabriq.exe` | バイナリ |

### 4. 相対パス固有除外（ランタイム状態）

```csharp
private static readonly HashSet<string> ExcludedRelativePaths = new(StringComparer.OrdinalIgnoreCase)
{
    "kernel/json/skip_request.flag",
    "kernel/json/art_pulse.txt",
    "kernel/json/resume_state.json",
    "kernel/json/status.json",
    "kernel/json/session.json",
    "kernel/txt/art_sentences.txt",
    "kernel/txt/silence.flag",
};
```

ランタイム実行で生成される一時状態ファイル群。**復元時に過去のセッション状態を持ち越さない** ためのフィルタ。

### 5. 素材フォルダ例外（asset folder rule）

```csharp
private static bool IsModuleAssetFolder(string relativeDir)
{
    var segs = relativeDir.Split(...);
    return segs.Length >= 4
        && segs[0].Equals("modules", StringComparison.OrdinalIgnoreCase);
}
```

判定式: **`modules/<kind>/<module>/<subdir>/...`** の形（**4 セグメント以上**で先頭が `modules`）。

該当する場合、`CopyDirectoryRaw` が呼ばれ、上記 1〜4 のすべての除外ルールを **無視して丸ごと再帰コピー** する。これにより:

- `modules/standard/printer_driver_config/INF/` （実 INF ファイル群）
- `modules/standard/wallpaper_config/assets/` （壁紙画像）
- `modules/extended/pianist/profiles/<name>/screenshots/` （Sample 用画像）
- `modules/standard/cert_config/source/` （証明書ファイル群）

などの **モジュール固有の素材ファイル**（PS1 で .ps1 拡張子を持っていても、画像・INF・証明書も含めて）を完全保持する。

> 例: `modules/standard/printer_driver_config/INF/EPSON/x.exe` は通常 `.exe` のため除外対象だが、4 セグメント以上のため素材フォルダと判定 → 丸ごとコピーされる。これがないと **ドライバインストール用の SFX が保存されず、復元しても動かない**。

---

## ダイアログ UI（[FabriqBackupDialog.xaml](file:///E:/fabriq_studio/FabriqStudio/Views/FabriqBackupDialog.xaml)）

```
┌─ fabriq バックアップ ──────────────────────────────┐
│                                                      │
│ サマリー: ワークスペース <パス> をバックアップします │
│                                                      │
│ 出力先の親フォルダ                                   │
│   [_______________________________] [参照...]        │
│                                                      │
│ メモ（USER_MEMO.txt）                                │
│   ┌──────────────────────────────────────────────┐  │
│   │ (自由記述、AcceptsReturn=True、最大 180px)   │  │
│   └──────────────────────────────────────────────┘  │
│                                                      │
│ 除外ルール（概要）                                   │
│   ディレクトリ: .git / .claude / dev / logs ...     │
│   拡張子: .ps1 / .bat / .bak / .tmp / .log          │
│   ファイル: VERSION / Guide.txt / Fabriq.exe ...    │
│   例外: modules 配下の素材フォルダは丸ごと保存     │
│                                                      │
│            [キャンセル]  [バックアップ開始]         │
└─────────────────────────────────────────────────────┘
```

特徴:

- **ダークテーマ固定**（`Background="#2D2D30"`）— Studio の他ダイアログと統一
- 出力先親フォルダは `IsReadOnly="True"` で **手入力不可**（ファイル選択のみ。タイポによる誤フォルダ消失を防ぐ）
- メモは `MaxHeight="180"` で長文も入力可（VerticalScrollBarVisibility="Auto"）
- **除外ルール概要パネル**を常時表示し、「何が保存されないのか」を実行前に明示
- `IsEnabled="False"` で初期化された OK ボタンは、出力先親フォルダ選択後に有効化される（コードビハインド）
- `IsCancel="True"` / `IsDefault="True"` でキーボード操作（Esc/Enter）に対応

---

## バックアップ実行フロー

[FabriqBackupService.BackupSync](file:///E:/fabriq_studio/FabriqStudio/Services/FabriqBackupService.cs):

1. `Directory.Exists(SourceRoot)` 検証（無ければ `DirectoryNotFoundException`）
2. `now = DateTime.Now`、`folderName = "fabriq_backup_<yyyyMMdd_HHmmss>"` を組み立て
3. `Directory.CreateDirectory(backupFolder)`
4. `state = new CopyState()`（カウンタとエラーリスト保持）
5. `CopyDirectoryFiltered(SourceRoot, backupFolder, "", state)` で再帰コピー開始
   - 各ディレクトリで `IsDirExcluded` 判定 → 除外なら丸ごとスキップ
   - 各ファイルで `IsFileExcluded` 判定 → 除外なら `state.ExcludedFiles++`
   - `IsModuleAssetFolder` の場合は `CopyDirectoryRaw` に切り替え（除外ルール無視）
6. `WriteUserMemo(backupFolder, Memo)` で `USER_MEMO.txt` 配置（**除外ルール対象外**で必ず作成）
7. `WriteBackupInfo(backupFolder, now, state, request)` で `BACKUP_INFO.txt` 配置
8. `FabriqBackupResult` を返却（`BackupFolderPath`, `CopiedFiles`, `ExcludedFiles`, `TotalBytes`, `Errors`）

### TryCopyFile のエラー方針

```csharp
private static void TryCopyFile(string src, string dst, CopyState state)
{
    try
    {
        File.Copy(src, dst, overwrite: false);
        state.CopiedFiles++;
        state.TotalBytes += new FileInfo(dst).Length;
    }
    catch (Exception ex)
    {
        state.Errors.Add($"{src}: {ex.Message}");
    }
}
```

- **`overwrite: false` 固定**: タイムスタンプフォルダなので衝突しない設計だが、念のため上書きしない
- 失敗時は `state.Errors` に追加して **継続**（1 ファイル失敗でバックアップ全体を止めない）
- `state.Errors` は `BACKUP_INFO.txt` の末尾と `FabriqBackupResult.Errors` に最大 50 件まで反映

---

## メタファイル

### `USER_MEMO.txt`

ユーザーが入力したメモを **BOM 無し UTF-8** で保存。空メモの場合は `"(no memo)"` を書く。末尾は `TrimEnd() + Environment.NewLine` で正規化。

### `BACKUP_INFO.txt`

```
fabriq studio — fabriq backup
================================
Created at         : 2026-05-07 14:30:12
Source root        : E:\fabriq
Copied files       : 1247
Excluded files     : 89
Total bytes        : 35,432,891

Errors (0)
----------------

Excluded top-level dirs: .git, .claude, dev, logs, apps, commands, evidence, kernel/ps1
Excluded extensions    : .ps1, .bat, .bak, .tmp, .log
Excluded filenames     : .gitignore, .gitkeep, CLAUDE.md, CHANGELOG.md, KERNEL_VERSION, ...
Asset folder rule      : modules/<kind>/<name>/<subdir>/ は除外を無視して丸ごと保存
```

**自己記述的**（self-describing）な内容: 復元する人が「なぜこのファイルがあって、なぜこのファイルが無いのか」を見るだけで判定できる。

---

## 制約と注意点

- **ワークスペースの読み取り中ロック**: バックアップ中は他の Studio 機能から workspace を編集してはいけない（`File.Copy` がロック競合する可能性）。実装側で排他制御はしていないため運用で注意する。
- **シンボリックリンクは展開される**: `File.Copy` のデフォルト挙動でリンク先が複製される。ジャンクション・ハードリンクのある fabriq workspace では非互換になりうる（現状は影響なし）。
- **権限エラー**: 一部の `kernel/json/*.json` がランタイムロック状態だと `IOException` が `state.Errors` に記録される。バックアップ自体は完走する。
- **タイムスタンプの衝突**: 同一秒に 2 回バックアップを走らせると 2 回目はフォルダ作成失敗。1 秒以上の間隔を空ける。

---

## 関連ドキュメント

| ドキュメント | 関係 |
|---|---|
| [fabriq_studio__reference__models_catalog.md](fabriq_studio__reference__models_catalog.md) | `FabriqBackupRequest` / `FabriqBackupResult` 仕様 |
| [fabriq_studio__reference__services_catalog.md](fabriq_studio__reference__services_catalog.md) | `IFabriqBackupService` シグネチャ |
| [fabriq_studio__apps__07_fabriq_update.md](fabriq_studio__apps__07_fabriq_update.md) | バックアップを **更新前に自動的に取る** Update Dialog |
| [fabriq__kernel__11_directory_layout.md](fabriq__kernel__11_directory_layout.md) | バックアップが対象とする fabriq の通常ディレクトリ構成 |

---

## 変更履歴

- 2026-05-07 初版作成（`apps__04_other_tools.md` の FabriqBackup セクションから個別化、除外ポリシー 5 階層と素材フォルダ例外を網羅）
