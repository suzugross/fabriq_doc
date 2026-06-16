# credential_config (Standard)

> **対象**: fabriq / modules/standard/credential_config
> **対象バージョン**: モジュール 0.1.0 / kernel 3.6.0（取得元: `E:\fabriq\modules\standard\credential_config\VERSION` / `E:\fabriq\kernel\KERNEL_VERSION`、commit `0fca159`、2026-06-16）
> **ドキュメント更新日**: 2026-06-16

**カテゴリ**: Security
**メニュー名**: Credential Manager Config
**VERSION**: 0.1.0  / **REQUIRES_KERNEL**: 2.0.0
**Post-Apply Verification**: 意図的に非対応（cmdkey はパスワードを読み返せず、`cmdkey /list` は presence ベース検証で false PASS を生むため。`-Verified` は `$null`、検証除外リストに登録）
**サブスクリプト**: なし（単一スクリプト `credential_config.ps1`）

## 目的
`credential_list.csv` に書いた内容を、ネイティブの `cmdkey.exe` 経由で Windows 資格情報マネージャ（vault）へ一括登録するモジュール。`Generic` / `Domain` / `RDP` の 3 種別に対応する（`credential_config.ps1:4-6`）。

cmdkey は **このモジュールを実行しているアカウント（キッティング作業用アカウント）の vault にのみ書き込む**。各 vault はユーザーごとに DPAPI で暗号化されるため、後から OOBE 後にログオンする end-user の vault には届かない。本モジュールはキッティング作業中に作業者自身が使う資格情報を current アカウントへ登録する **kitting-session staging** が用途であり、納品先 end-user の vault をプロビジョニングするものではない（`credential_config.ps1:8-13`、`Guide.txt:12-26`）。

## 入力 (CSV)
`credential_list.csv`（列順: `Enabled,CredType,TargetName,UserName,Password,Description,Segment`、`credential_list.csv:1`）:

- `Enabled`: `1`=登録、それ以外=無視（`credential_config.ps1:126`、`Guide.txt:74`）
- `CredType`: `Generic` / `Domain` / `RDP`（大文字小文字不問。`ToUpperInvariant` で正規化、`credential_config.ps1:60,62`）
- `TargetName`: 資格情報の target。RDP は素のホスト/FQDN/IP を指定し `TERMSRV/` が自動付与される（既に `TERMSRV/` で始まる場合は二重付与しない、`credential_config.ps1:87`、`Guide.txt:76-77`）
- `UserName`: `cmdkey /user:` に渡す（`credential_config.ps1:231`、`Guide.txt:78`）
- `Password`: 平文パスワード。ログ上はマスク表示（`credential_config.ps1:193`、`Guide.txt:79`）
- `Description`: 表示用の説明（任意。空なら `TargetName` を表示、`credential_config.ps1:59`、`Guide.txt:80`）
- `Segment`: セグメント名（任意。`Guide.txt:81-82`）

`RequiredColumns` は `Enabled,CredType,TargetName,UserName,Password` の 5 列（`credential_config.ps1:120`）。`Import-ModuleCsv` の `ENC:` プレフィックス + マスターパスフレーズ暗号化にそのまま対応する（任意、`Guide.txt:46-47`）。出荷時の `credential_list.csv` は全行 `Enabled=0`（誤登録防止。実 CSV も 3 行すべて先頭 `0`、`credential_list.csv:2-4`、`Guide.txt:87-88`）。

### CredType の選び方
- `Generic`: 汎用（アプリ / Web / サービス全般）。SMB リダイレクタは Generic を参照しないため共有アクセス用途には選ばない（`Guide.txt:8,93-95`）
- `Domain`: ドメイン パスワード。Windows 統合認証（SMB / ファイル共有 / ホスト認証）用。名前に反して AD ドメインは不要で WORKGROUP 環境のファイルサーバにも使う（`Guide.txt:9,96-98`）
- `RDP`: リモートデスクトップ。`Generic` 種別で `TERMSRV/` を target に自動付与する（type label は "RDP (Generic, TERMSRV)"、`credential_config.ps1:86-90`、`Guide.txt:10,99`）

cmdkey の verb は種別ごとに決まる: `Generic` → `/generic:<target>`、`Domain` → `/add:<target>`、`RDP` → `/generic:TERMSRV/<target>`（`credential_config.ps1:75-91`）。

## 主要ステップ
1. **CSV 読み込み**: `-FilterEnabled` を使わず全行ロードし、`Enabled -eq "1"` で手動フィルタ（`autologon_config` パターン）。`-FilterEnabled` は有効行ゼロのとき `@()` を返し呼び出し元で `$null` に展開され、真のロード失敗と区別できなくなるため（`credential_config.ps1:113-126`）
   - ロード失敗 → `Error`（`credential_config.ps1:122-124`）
   - 有効行ゼロ → `Skipped`（`credential_config.ps1:127-131`）
2. **行検証（副作用前に全体停止）**: 各行を `Resolve-CredentialPlan` で正規化。`CredType` 不正 / `TargetName` 空 / `UserName` 空 / `Password` 空、および正規化後の effective target が空の行を不正としてマーク（`credential_config.ps1:62-95`）。不正行が 1 件でもあれば、副作用を起こす前に全体を `Error` 停止（`credential_config.ps1:152-159`、`Guide.txt:84-85`）。cmdkey は不正/空 target でも exit 0 を返して何もしないため、検証は raw 入力ではなく計算済み effective target に対して行う（`credential_config.ps1:136-139`）
3. **vault スコープガード**: 現在の identity を取得し、SID が `S-1-5-18`（LocalSystem / SYSTEM）なら `Error` 停止（SYSTEM は対話ユーザーの vault を持たず書き込んでも無意味なため、dead vault を作らない、`credential_config.ps1:167-175`、`Guide.txt:28-30`）
4. **ドライラン表示**: 書込み先 vault（current user 名）を明示し、`[REGISTER]` 行で Type / Target（effective）/ User / マスク済み Pass を一覧表示（`credential_config.ps1:178-199`）。マスクは先頭1文字 + `*` + 末尾1文字、2文字以下は `**`（`credential_config.ps1:40-47`）
5. **実行確認**: `Confirm-ModuleExecution`（AutoPilot は自動 Y、`credential_config.ps1:204-205`）
6. **適用ループ**: 各 plan につき `cmdkey.exe @cmdkeyArgs` を実行。引数はリストで組み立て、パスワードを単一の argv 要素として渡す（コマンドライン文字列補間しない）。cmdkey の出力は `Out-Null` で破棄し、本モジュールが手組みのマスク済みメッセージのみ表示（`credential_config.ps1:223-255`）
   - 成功判定オラクル = cmdkey 終了コード 0（Step 2 で非空引数は保証済み、`credential_config.ps1:237`）
   - 例外時は `$_` を意図的に出力しない（ネイティブコマンドのエラーレコードが引数リスト＝パスワードをエコーし得るため、display name / type / effective target のみ表示、`credential_config.ps1:246-252`）
7. **集計**: `New-BatchResult -Success/-Skip/-Fail`（`credential_config.ps1:269-270`）

## 注意点・運用メモ
- 管理者権限は不要（自アカウントの vault を書くため、`credential_config.ps1:16`）
- cmdkey `/add`・`/generic` は既存の同一 target を無言で上書き（last-write-wins）するため登録は冪等。スキップ最適化はパスワード一致を確認できず古いパスワードを温存する危険があるため採用せず、存在しても常に再登録する設計（`credential_config.ps1:210-216`、`Guide.txt:64-69`）
- **セキュリティ注意（監査フリート向け）**: cmdkey はパスワードを `/pass:` 引数としてコマンドラインに渡す仕様のため、パスワードがプロセス生成監査（Windows イベント 4688 / Sysmon EID 1）に平文で記録され得る。これは registry 書き込みの autologon_config には無い露出面。コマンドラインログを取得しているフリートでは、平文がセキュリティイベントログ（および evidence バンドル）に残り得ることを理解した上で使用する（`credential_config.ps1:18-22`、`Guide.txt:33-40`）
- `credential_list.csv` の Password 列は平文。CSV は機微情報として ACL を絞り、git に含めず、キッティング完了後に削除する（`Guide.txt:43-45`）
- **sysprep との順序**: `sysprep /generalize` はプロファイルを再生成するため、generalize 前に登録した資格情報は OOBE 後の end-user プロファイルには引き継がれない。end-user 向けの資格情報を generalize 前に登録しないこと（`Guide.txt:122-126`）
- **後始末（推奨）**: キッティング用アカウントの vault に書いた資格情報は、そのアカウントでログオンできる者が復元可能。納品前に各 target を `cmdkey /delete:<target>` で削除するか、キッティング用プロファイルごと削除する（`Guide.txt:129-134`）
- 既知の制限: パスワードに二重引用符 `"` を含む場合、Windows PowerShell 5.1 のネイティブ引数引き渡しで正しく渡らないことがある（`Guide.txt:138-140`）

## 検証
意図的に Post-Apply Verification 非対応（検証除外リストに登録、`credential_config.ps1:257-264`、`Guide.txt:50-60`）:
cmdkey は登録済みパスワードを読み返せず（`cmdkey /list` は target と user は表示するがパスワード値は一切表示しない）、`/list` は存在しない target でも exit 0 を返し問い合わせた target 名を常にエコーする。このため presence ベースの検証は「古い / 誤ったパスワードが残っていても PASS」という false PASS を生む。よって `-Verified` は渡さず `$null` を返し、登録の成否は cmdkey の終了コード（0=成功）のみで判定する。

完了確認は手動で `cmdkey /list` により target / user の存在を確認する（パスワードの正否は確認不可）。
