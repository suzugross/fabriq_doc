# robocopy_config (Standard)

> **対象**: fabriq / modules/standard/robocopy_config
> **対象バージョン**: モジュール 1.0.1 / kernel 3.2.5（取得元: `E:\fabriq\modules\standard\robocopy_config\VERSION` / `E:\fabriq\kernel\KERNEL_VERSION`、commit `fed181a`、2026-05-10）
> **ドキュメント更新日**: 2026-05-10

**カテゴリ**: Maintenance
**メニュー名**: Robocopy
**VERSION**: 1.0.1  / **REQUIRES_KERNEL**: 2.0.0
**Post-Apply Verification**: なし（ExitCode 判定のみ。実ファイル一致検証なし、`-Verified` 未渡し）
**サブスクリプト**: なし

## 目的
CSV (`robocopy_list.csv`) に列挙したジョブを `robocopy.exe` で順次実行するモジュール。
主要オプション (Recursive / Mirror / CopyACL / SkipOlder) を 0/1 フラグ列で表現してタイポ事故を防ぎ、
`/MT` `/Z` `/XD` などの追加引数は CustomOptions 列で自由記述する設計。
UNC 認証（`net use`）を内蔵しており、SMB 共有を含むコピーも 1 ジョブで完結させられる。
ENC: プレフィックスによる暗号化パスワードに対応。

## v1.0.1 の変更（セキュリティ強化、v1.0.0 からの差分）

UNC 認証パスワード（`AuthPass`）を `net use` の **コマンドライン引数経由** から **stdin パイプ経由** に変更（A3-T1-B robocopy）。本モジュールに残っていた最後の transcript / OS-layer leak vector を遮断する変更。

| Leak Vector | v1.0.0 の挙動 | v1.0.1 の挙動 |
|---|---|---|
| **`Win32_Process.CommandLine`** | net.exe lifetime 中の live snapshot で password が見えていた（同 logon session の他プロセス、AV / EDR / sysmon / `Get-CimInstance Win32_Process` 経由で取得可能） | **遮断**（コマンドライン引数から AuthPass が消えるため） |
| **Windows Security Log Event ID 4688** | `Audit Process Creation` + `IncludeCommandLine` 有効環境（B2G 標準 audit 設定）で永続的に password が記録され SIEM 転送先まで到達 | **遮断** |
| **PowerShell Module Logging** | 有効環境で native command 引数が間接的に記録 | **遮断** |
| **PS Transcript** | 変数参照 `$($item.AuthPass)` のまま記録（展開後の値は記録されない、v1.0.0 から不変） | 同上 |

**実装変更**: `& net use $share "$pass" /user:U` → `"$pass" | & net use $share /user:U`。net.exe は `/user:` 指定 + password 引数なしで prompt mode に入り、PowerShell の native command pipe からの 1 行を password として読む（標準 net.exe 挙動、Windows 10/11 で安定）。SMB session 確立後 robocopy は OS の SMB credential cache 経由で UNC パスへ透過アクセス。`& net use $share /delete /y` cleanup は password 不要のため変更なし。

**Guide.txt 記載の正確化**: v1.0.0 の Guide には「パスワードはコンソール / ログに一切出力されません」と記載されていたが、これは Win32_Process.CommandLine 経由で実際には漏洩していた虚偽記載だった。v1.0.1 では実装と整合する正確な表現（PS Transcript / Win32_Process.CommandLine / 4688 / Module Logging の各経路を明示）に書き換えた。

検討済却下案:
- PSDrive `-Persist` 化: drive letter mapping を新規導入する副作用が現状仕様（drive letter 不使用）と乖離するため見送り
- `WNetAddConnection2W` P/Invoke: 最も architectural だが Add-Type C# block の複雑性大、stdin pipe で同等のセキュリティ効果が得られるため不採用

公開 API・CSV スキーマ・ModuleResult 契約・operator 視点の動作フローはいずれも不変。`REQUIRES_KERNEL` 据え置き 2.0.0。

## 入力 (CSV)
`robocopy_list.csv`
- `Enabled`: 1=有効 / 0=無効
- `ID`: ジョブ識別子
- `Source`, `Destination`: コピー元・先パス（ローカル / UNC）
- `Recursive`: 1=`/E`
- `Mirror`: 1=`/MIR`（Recursive と両立時は Mirror 優先）
- `CopyACL`: 1=`/COPYALL /DCOPY:DAT` / 0=`/COPY:DAT /DCOPY:DAT`
- `SkipOlder`: 1=`/XO`
- `FileFilter`: ファイル名 / ワイルドカード（空欄=全ファイル、スペース区切り複数指定可）
- `CustomOptions`: 追加オプション自由記述
- `AuthUser`, `AuthPass`: UNC 認証情報（任意、ENC: 対応）
- `Description`, `Segment`

## 主要ステップ
1. `robocopy_list.csv` を `Import-ModuleCsv -FilterEnabled` で読み込み
2. 各ジョブのドライラン表示（オプション展開、AuthUser はマスク）
3. 実行確認（AutoPilot 自動 Y）
4. ジョブループ:
   - 4-1. AuthUser/AuthPass 両方ありなら `net use \\Server\Share /user:...` で接続
   - 4-2. `robocopy Source Destination [FileFilter] /R:3 /W:5 /NP <flags>` 実行（baseline オプション強制付与）
   - 4-3. ExitCode 評価 (0=変更なし / 1=コピー成功 / 2-3=余剰検出 / 4-7=ミスマッチ Warning / 8+=Error)
   - 4-4. finally で `net use /delete` 切断
5. 集計して `New-BatchResult` 返却（`-Verified` 未渡し）

## 注意点・運用メモ
- 管理者権限必須（CopyACL=1 で SeSecurityPrivilege/SeBackupPrivilege が必要）
- baseline オプション `/R:3 /W:5 /NP` は CSV から除外不可（ハングアップ防止のセーフガード）
- v1.0.1 以降、パスワードは `net use` コマンドライン引数として展開されない（stdin pipe 経由）。Win32_Process.CommandLine / Event ID 4688 / Module Logging いずれにも記録されない（PS Transcript には変数参照のまま残る、展開後の値は不記載）
- Mirror=1 は Source 不在ファイルを Destination から削除するため、誤 Source で意図せぬ削除事故の
  リスクあり。初回テストは Recursive=1 + SkipOlder=1 を推奨
- Mirror=1 は厳密な冪等ではない（Source 変化に追従するため）
- `FileFilter` は Source をディレクトリ指定のままにしてファイル名はこの列で渡す形式
- net use 失敗時はジョブ Skip して次へ進行（ジョブ間隔離）
- robocopy ExitCode 仕様の Warning レンジ (4-7) は Success 扱い

## 検証
robocopy の ExitCode は「ジョブが成功したか」を返すだけで、実ファイル単位の一致は保証しない。
本モジュールは実ファイル比較や ACL 比較の Step 5.5 を持たず、`New-BatchResult` に `-Verified` を
渡していないため履歴の Verified 列は空欄。手動検証としては `robocopy Source Destination /L /E /NP`
（`/L` でドライラン）や `icacls` での ACL 確認が Guide で案内されている。
将来的に DFS-R や Hash 比較を組み込むには copyfile_config 系の検証除外パターン
（`project_verification_exclusions.md` 参照）と整合を取る必要がある。
