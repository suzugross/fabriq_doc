# local_user_config (Standard)

> **対象**: fabriq / modules/standard/local_user_config
> **対象バージョン**: モジュール 1.1.0 / kernel 3.2.5（取得元: `E:\fabriq\modules\standard\local_user_config\VERSION` / `E:\fabriq\kernel\KERNEL_VERSION`、commit `fed181a`、2026-05-10）
> **ドキュメント更新日**: 2026-05-10

**カテゴリ**: User Management
**メニュー名**: Create Local Users / Delete Local Users
**VERSION**: 1.1.0  / **REQUIRES_KERNEL**: 2.0.0
**Post-Apply Verification**: Create のみ実装あり（ユーザー存在 + グループ所属検証）／Delete はなし
**サブスクリプト**: `local_user_config.ps1`（作成）, `local_user_delete.ps1`（削除）

## 目的
ローカルユーザーアカウントの作成・削除を行います。
全 PC 共通の `local_user_list.csv` と PC 固有の `local_user_host_list.csv` を
union 適用するハイブリッド構造で、共通管理者アカウントは前者で一元管理しつつ、
特定機種だけにローカル運用ユーザーを足す等の柔軟な運用が可能です。
グループ所属はセミコロン区切り複数指定（例: `Users;Remote Desktop Users`）。

## v1.1.0 の変更（v1.0.x からの差分）

`local_user_list.csv` 側が **Segment フィルタで全行 0 件**になっても、`local_user_host_list.csv` 側にマッチする行があれば処理を続行する **PC 固有単独モード** をサポート（v1.0.x では false Error を出していた）。

| シナリオ | v1.0.x | v1.1.0 |
|---|---|---|
| 共通 CSV ファイルが見つからない | Error 終了 | Error 終了（変更なし） |
| 共通 CSV ヘッダのみ / Segment フィルタで全行除外 + host 側 0 件 | Error（false Error） | **Skipped（正常終了）** |
| 共通 CSV 0 件 + host 側にマッチ行あり | Error（false Error） | **共通分は 0 行、host 行のみで処理続行** |

`Import-ModuleCsv` が "Segment 0 match" で `@()` を返したのを PowerShell が auto-unwrap して `$null` にしてしまう挙動を踏まえ、ファイル不在は Error / それ以外は空配列扱いに分岐する修正。Profile で create / delete などの Segment 別運用を行うときに、共通ユーザーを置かず PC 個別ユーザーのみを host_list.csv 側で Segment 区別する運用がサポートされる。

## 入力 (CSV)
**`local_user_list.csv`（全 PC 共通）**:
- `Enabled`, `UserName`, `Password`
- `PasswordNeverExpires`: 1=無期限, 0=有効期限あり
- `UserMayNotChangePassword`: 1=変更禁止, 0=変更可
- `Group`: 所属グループ（セミコロン区切り複数可）
- `Description`, `Segment`

**`local_user_host_list.csv`（任意・PC 固有）**:
- 上記列に加え `NewPCName`（hostlist の NewPCName と完全一致、SELECTED_NEW_PCNAME で突合）

## 主要ステップ（Create）
1. `local_user_list.csv` 読み込み
2. `local_user_host_list.csv` 存在時、SELECTED_NEW_PCNAME 一致行を append
3. ユーザー一覧表示（無効行は [DISABLED] と表示）
4. 実行確認（AutoPilot は自動 Y）
5. 作成ループ: `Get-LocalUser` で既存検出時 Skip、`New-LocalUser` で作成
   （`AccountNeverExpires=$true` 固定、PasswordNeverExpires / UserMayNotChangePassword は CSV 値反映）
6. グループ追加: `Add-LocalGroupMember` を Group 列各値ごとに実行
7. Step 5.5: Verification → `New-BatchResult -Verified`

## 主要ステップ（Delete）
1. CSV 読み込み（Enabled フィルタなし＝全行が削除候補）
2. host_list union
3. 一覧表示 + 確認
4. `Remove-LocalUser`、不在は Skip
5. `New-BatchResult` で集計（Verified なし）

## 注意点・運用メモ
- 管理者権限必須（`New-LocalUser` / `Remove-LocalUser`）
- パスワードは平文 CSV 保存。crypto レイヤ（`ENC:` プレフィクス + passphrase）対応は
  framework 共通機能で別途提供
- ユーザー作成失敗時もグループ追加は実行されないが、グループ追加失敗だけでは
  `successCount` には影響せず、警告のみ
- Delete スクリプトは Enabled フィルタを使わない（全行対象）。
  CSV を「削除対象リスト」として再利用する思想

## 検証
Create 側のみ Verification 実装。`Get-LocalUser` でユーザー存在を確認し、
`Get-LocalGroupMember` で `Group` 列指定の各グループに対し正しく所属しているかを
正規表現マッチで検証。1 件でも欠落があれば `-Verified $false`。
Delete 側は Verification 未実装で `-Verified` 引数を渡さない。
