# evidence_config (Standard)

> **対象**: fabriq / modules/standard/evidence_config
> **対象バージョン**: モジュール 1.8.1 / kernel 3.6.0（取得元: `E:\fabriq\modules\standard\evidence_config\VERSION` / `E:\fabriq\kernel\KERNEL_VERSION`、commit `0fca159`、2026-06-16）
> **ドキュメント更新日**: 2026-06-16

**カテゴリ**: Evidence
**メニュー名**: Collect Evidence
**VERSION**: 1.8.1  / **REQUIRES_KERNEL**: 2.0.0
**Post-Apply Verification**: 概念的に不適用（読み出し系のためシステム状態を変更しない）
**サブスクリプト**: なし

## 目的
PC のシステム情報を **§01〜§33 + サブ "8b" = 計 34 セクション** にわたって収集し、テキスト／CSV／HTML のエビデンスファイル一式を `evidence/pc_information/<日付>_<UID>_<PC名>/` に書き出す、fabriq の納品エビデンス採取の中核モジュールです。受入検査・官公庁監査・トラブル時の証跡として機能し、収集完了時に **`manifest.json`** を生成して各セクションの `id / title / files / status / reason / elapsedMs` を機械可読サマリ化します。これにより外部ツール（fabriq_evidence_manager 等）が manifest を起点にエビデンスを統合できる契約構造になっています。シリアル番号は 4 ソース横断 + canonical 選定 + OEM 無効値（"Default string" 等）の拒否までやる多重ソース戦略。v1.6.0 では §27〜§31 の inventory 拡張（2026-04-30）、**v1.7.0 でセクション別 enable/disable 切替（後述）**、v1.7.1 で §04 のメンバー判定 bug 修正（後述）、**v1.8.0 で §32 Credential Manager / §33 Outlook Mail Accounts 追加**、v1.8.1 で §29 Memory Slots の null 安全化を追加。

## 入力 (CSV)
**v1.6.x までは設定 CSV なし**（全セクション固定実行）でしたが、**v1.7.0 から `evidence_list.csv` でセクション別の enable/disable** が可能になりました。

### `evidence_list.csv`（v1.7.0 以降）

列: `Id` / `Title` / `Enabled`。各セクション（§01〜§33 + §8b）の収集を個別に有効/無効化できます。

**default-on policy**: CSV 不在 / 該当 `Id` 行が無い / `Enabled` が空または不正値 のいずれの場合も **有効として扱う**。これにより、CSV を全く編集せずに使う限り従来通り全 34 セクション（§01〜§33 + §8b）が収集されます（CSV を編集しない限り従来挙動を維持）。

無効化されたセクションの記録方法:
- `manifest.json` 上で `status = "Skipped"`、`reason = "Disabled by configuration (evidence_list.csv)"` として記録される
- master log に `"[Section XX] Title : Skipped (disabled by configuration)"` がグレーで 1 行
- これにより外部 evidence consumer は「configuration によるスキップ」と「intrinsic skip（Server-only / バッテリ非搭載 等）」を区別できる

Studio 側では `preset.csv` により `Enabled` 列がドロップダウン編集 UI になります。

## 収集セクション（v1.8.0 時点で §01〜§33 + サブ "8b" = 計 34 セクション）
- §01 システム基本情報 / §02 ローカルユーザー (CSV) / §03 ローカルグループ (CSV) / §04 ローカルグループメンバー (CSV)
- §05 Domain / Azure AD Status + User Profiles（`05_DomainStatus.txt` に加え、PC 上のユーザープロファイル一覧 `05_UserProfiles.csv` を出力。サブステップ 5d で追加） / §06 ネットワーク (CSV) / §07 プリンター/ポート (CSV)
- §08 BitLocker / **§8b ディスク・パーティション情報 (CSV、サブセクション)** / §09 MAC アドレス (CSV)
- §10 PC シリアル番号（多重ソース + canonical 選定 + OEM 無効値拒否）
- §11 インストール済みソフトウェア (CSV) / §12 ファイアウォール状態 (CSV) / §13 Windows Optional Features (CSV)
- §14 Server Roles & Features（Server OS のみ、Client は Skipped）
- §15 電源設定 / §16 WiFi プロファイル / §17 Restore Points (CSV) / §18 Defender 状態
- §19 Windows Update 履歴 (CSV) / §20 System TEMP テキストログバックアップ（セーフティネット）
- §21 Windows ライセンス（SoftwareLicensingProduct + slmgr /dlv）
- §22 Office ライセンス（OSPP + vNext per-user スキャン + 自動解釈 verdict）
- §23 Security Baseline（TPM / Secure Boot / VBS / HVCI / Credential Guard / LSA Protection / BIOS）
- §24 Group Policy Report（`gpresult /h` HTML + サマリ TXT）
- §25 Certificates CSV（4 ストア統合、HasPrivateKey フラグのみ）
- §26 Battery Report（`powercfg /batteryreport` HTML、受入検査での容量契約エビデンス）
- §27 Environment Variables (CSV、2026-04-30 追加)
- §28 Startup Items (CSV、2026-04-30 追加)
- §29 Memory Slots (CSV、2026-04-30 追加)
- §30 PnP Devices (CSV、2026-04-30 追加)
- §31 Hardware Identifiers（2026-04-30 追加）
- §32 Credential Manager（`32_Credentials.csv`、v1.8.0 追加。Win32 `CredEnumerateW` で実行ユーザーの資格情報コンテナをメタデータのみ列挙。列: TargetName / Type / UserName / Persist / Comment / LastWritten / IsSystemNoise / SourceUser。パスワード本体 `CredentialBlob` は読み取りも記録もせず、復号 API（`CryptUnprotectData` 等）も不使用。DPAPI 制約により対象は構造的に実行ユーザーのみで `SourceUser` 列に明記。OS 自動管理エントリは除外せず `IsSystemNoise=True` でフラグ付け。空 vault はヘッダーのみ CSV で Success）
- §33 Outlook Mail Accounts（`33_OutlookAccounts.csv` + `33_OutlookDataFiles.csv`、v1.8.0 追加。Office {16.0, 15.0}\Outlook\Profiles レジストリをメタデータのみ走査、COM 不使用。管理者として別アカウント実行時は `Resolve-HkcuRoot` でログオン中ユーザーのハイブへリダイレクトし、読んだハイブを `SourceUser` 列に自己記述。POP3/IMAP/SMTP サーバ・ポート・SSL と配信先 PST の 3 段解決〔EntryID バイナリスキャン → email とのファイル名一致 → 単一候補〕。Password 系レジストリ値（DPAPI blob 含む）は存在確認すら行わない。Profiles キー不在は intrinsic Skipped、アカウント単位の読取失敗は Partial 降格で CSV は書き切る）

## 主要ステップ
1. 出力先パス決定（`$global:FabriqEvidenceBasePath` 有無で統一パスモード／レガシーモード分岐）
2. 実行確認（AutoPilot 時は自動 Y）
3. 各セクションを順次収集
4. 統合ファイル `_ALL_<PC名>_Log.txt` + 個別ファイル + CSV/HTML 出力
5. `manifest.json` 生成（`schemaVersion=1`、再実行時は `manifest.json.bak` に rotate）

## 注意点・運用メモ
- **管理者権限必須**（BitLocker / Firewall ルール / Defender 状態など admin 限定情報を含む）
- §22d の自動解釈は M365 sub の OSPP「飾りキー」問題（OSPP は常に Grace 表示になる）を内部で吸収し、Manifest の `Partial` / `Failed` 区別を出力
- §23 の各 probe は inner try/catch で個別退避（1 つ失敗してもセクションは Success）
- §24 の `gpresult` ユーザー側 RSoP は実行ユーザー（kitting admin01 等）視点であり、最終エンドユーザー視点ではない点に注意
- §26 はバッテリ非搭載で Skipped 完結
- 環境変数: `$env:SELECTED_NEW_PCNAME` / `$env:COMPUTERNAME`、グローバル `$global:FabriqUniqueId` / `$global:FabriqEvidenceBasePath`
- §32 / §33 は v1.8.0 で fabriq_backuper の実装を evidence 向けに移植したもの。manifest の `schemaVersion` は **1 のまま据え置き**（セクション ID 追加は `kernel/EVIDENCE_MANIFEST.md` §4.1 前方互換ルールの範囲内で、`fabriq_evidence_manager` は未知セクションを raw シートへ自動出力する設計）
- v1.7.1 (PATCH): §04（ローカルグループメンバー）で `net localgroup` 出力末尾の「コマンドは正常に完了しました。」を検知できず偽メンバーが evidence に混入していた bug を、日本語版 Windows 対応の UTF-8 BOM 付与で修正
- v1.8.1 (PATCH): §29（Memory Slots）で `Win32_PhysicalMemory` の `PartNumber` / `SerialNumber` が null（はんだ付け LPDDR ノート PC・一部 Hyper-V/OEM SMBIOS）のとき `.Trim()` 例外でセクション失陥し `29_MemorySlots.csv` 欠落 + モジュールが Partial/Fail 記録になっていた問題を、null 安全な文字列キャスト `"$($m.PartNumber)".Trim()` で修正

## 検証
Post-Apply Verification は **概念的に不適用**。本モジュールは「読み出し系」であり、システム状態を変更しないため「適用後の読み返し検証」が論理的に存在しません。`-Verified` は未渡しで Verified 列は空欄。代わりに `manifest.json` のセクション単位 `status` フィールド（Success / Skipped / Failed / Partial の 4 値）が品質情報を担います。`kernel/EVIDENCE_MANIFEST.md` で公式契約として `schemaVersion=1` が定義されており、外部 evidence consumer はこの manifest を起点にエビデンスをパースする設計です。
