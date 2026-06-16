# インストールと初回起動

> **対象**: fabriq / usage
> **対象バージョン**: kernel 3.6.0（取得元: `E:\fabriq\kernel\KERNEL_VERSION`）+ commit `0fca159`（取得元: `git -C E:\fabriq rev-parse --short HEAD`、2026-06-16）
> **ドキュメント更新日**: 2026-06-16

USB 媒体を運搬して対象 PC に fabriq を展開し、初回起動を成功させるまでの手順。前提となる **`fabriq_studio` でのマスターパスフレーズ設定** がここで完了していることが起動条件となる。

---

## 前提条件

| 項目 | 要件 |
|---|---|
| OS | **Windows 11** |
| PowerShell | **5.1 以降**（既定で同梱） |
| 権限 | **管理者権限**（`Fabriq.exe` が UAC で自動昇格するため、ログオンユーザーが昇格可能であれば足りる） |
| パスフレーズトークン | **`kernel/txt/passphrase_verify.txt`** が存在すること（`fabriq_studio` で生成） |
| ライセンス系 | 7-Zip 25.01 同梱（`modules/standard/printer_driver_config/tools/`）— 別途インストール不要 |

`passphrase_verify.txt` 不在時は **fabriq は起動できない**（後述）。`fabriq_studio` ワークスペースで対象の fabriq フォルダを開き、マスターパスフレーズを 1 度設定するとこのトークンが生成される。

---

## 1. fabriq フォルダの配備

### 媒体配布フロー

```
配布元 PC で fabriq_studio から build / export
        │
        ▼
USB メモリ ルートに fabriq/ フォルダを配置（kernel/ modules/ profiles/ apps/ 等を含む）
        │
        ▼
USB メモリを対象 PC に挿す
        │
        ▼
USB 上の fabriq/ フォルダを対象 PC の配備先（既定 `C:\Windows\work\fabriq`）へコピー
```

> **`Deploy.bat` は kernel 3.6.0（TM t-0042）で廃止・削除済み**（取得元: `E:\fabriq\CHANGELOG.md` `### Removed` L122-128）。USB→対象 PC 自動デプロイツールだったが、運用で一度も使用しておらず不要として削除された。ソースに `E:\fabriq\Deploy.bat` は存在しない（Glob 確認）。配備は **`fabriq/` フォルダを配備先へ手動コピー**（エクスプローラのコピー / `robocopy` 等）して行う。以下に旧 `Deploy.bat` が担っていた挙動を参考として残すが、いずれも現行フレームワークの構成要素ではない。

### 旧 Deploy.bat の動作（廃止済み・参考）

> 以下は 3.6.0 以前に存在した `Deploy.bat` の手順。**現行版には存在しない**。

1. **管理者権限チェック** — `net session` で確認、未昇格なら PowerShell で UAC 昇格してから自身を再実行
2. **ソースドライブの Volume Serial Number 取得** — `vol` コマンド出力から hex serial を抽出（fallback として WMI `Win32_LogicalDisk.VolumeSerialNumber`）
3. **`source_media.id` 保存** — `kernel/source_media.id` に Volume Serial を書き込む（後述、セッション情報の trail に使う）
4. **配備先ディレクトリ選択** — 対話的プロンプト：
   - `[1]` `C:\Windows\work\fabriq`（既定）
   - `[2]` カスタムパス（手入力）
5. **配備サマリ表示と確認** — Source / Destination / Media ID を提示、`Y/N` 確認
6. **robocopy** — `/MIR`（mirror）モードで配備先にコピー、`/R:2 /W:1`（リトライ 2 回・待機 1 秒）
7. **完了後の選択** — `続けて Fabriq を実行しますか？ (Y/N)` で `Y` なら配備先の `Fabriq.exe` を起動

### 配備先ディレクトリの推奨

`C:\Windows\work\fabriq`（既定）。配備先パスは fabriq 自体の動作に影響しないが、次の理由で `C:\Windows\work\` 系列を推奨：

- ユーザープロファイル配下を避ける（`profile_delete` モジュール対象の領域から外れる）
- ドライブ直下を汚さない
- Defender スキャン除外を一括設定しやすい

USB から直接 `Fabriq.exe` を起動することも理論上可能だが、媒体抜去で fabriq が落ちる・パフォーマンスが劣る・配備中ファイル書き込みで USB 寿命を消費する等の理由で **配備先で起動するのが標準運用**。

### `source_media.id` の役割

`Initialize-Session` は **メディアシリアルをセッション情報に記録** する。`MediaSerial` の決定は次の優先順位で行われる（取得元: `E:\fabriq\kernel\common.ps1` L3384-3400）：

```
Priority 1: 既存 session.json に記録された MediaSerial を採用（再起動跨ぎ復元）
Priority 2: kernel/source_media.id を読む（外部プロビジョニング marker。fabriq 自身は生成しない）
Priority 3: 現ドライブの Volume Serial を Get-VolumeSerial で取得
```

これは **「どの媒体から配備された fabriq か」を実行履歴 CSV に残すため**。複数の USB を順に展開しているとき、どの USB が原因の不具合かを後から追跡できる。

> **`source_media.id`（Priority 2）の生成元だった `Deploy.bat` は kernel 3.6.0（TM t-0042）で廃止・削除済み**（取得元: `E:\fabriq\CHANGELOG.md` `### Removed` L122-128）。`source_media.id` は fabriq 自身が作成するファイルではなく、削除後の **通常運用では Priority 2 は不在となり Priority 3（`Get-VolumeSerial` フォールバック）が使われる**ため、MediaSerial の記録自体には影響しない（取得元: `common.ps1` L3387-3399 のコメント・`main.ps1` L1599-1602）。外部のプロビジョニング工程が任意に `kernel/source_media.id` を置けば従来どおり Priority 2 として読まれる。

---

## 2. パスフレーズ前提の確立

`Fabriq.exe` 初回起動時に最初に検査されるのが `kernel/txt/passphrase_verify.txt`：

```
{kernel}/txt/passphrase_verify.txt   ← 平文 "surkitinisme" を ENC: AES-256-CBC で暗号化したトークン
```

**動作**: `Test-MasterPassphrase`（`common.ps1` L508）が起動時に呼ばれ、ユーザー入力のパスフレーズで `ENC:` 値を復号、結果が `"surkitinisme"` と一致するかを確認する。

**生成方法**: 配備元 PC で `fabriq_studio` を起動 → 対象の fabriq フォルダをワークスペースとして開く → メニューの「マスターパスフレーズ設定」を選んで任意のパスフレーズを入力 → `passphrase_verify.txt` が生成される。

詳細は [fabriq_studio__usage__01_workspace_setup.md](fabriq_studio__usage__01_workspace_setup.md) を参照。

**配備時の注意点**: `passphrase_verify.txt` を含めて配備先へコピーすること（旧 `Deploy.bat` での自動展開は kernel 3.6.0 で廃止済み。フォルダ全体を手動コピーする）。パスフレーズが必要な現場へは fabriq_studio の出力をそのまま運搬する設計のため、**fabriq 単体には passphrase 設定機能がない**（CLI / GUI どちらも持たない）。

---

## 3. 初回起動

`{配備先}\Fabriq.exe` をダブルクリックする。

### 起動シーケンス

```
Fabriq.exe（C# ランチャ、UAC 自動昇格）
        │
        ▼
管理者権限の PowerShell 5.1 コンソール起動（ConsoleSize 75x35、QuickEdit OFF、Sleep 抑止）
        │
        ▼
kernel/main.ps1 を dot-source
        │
        ▼
common.ps1 / manifesto.ps1 / fabriq_operator.ps1 を順次 dot-source
        │
        ▼
Start-Transcript（ログ書き込み開始: logs/{ts}_{uid}_{hn}.log）
        │
        ▼
Resume Detection（後述）
        │  ├── wu_state.json あり → Windows Update Loop に分岐（最優先）
        │  ├── resume_state.json あり → Profile resume へ
        │  └── どちらも無し → Fresh Start
        │
        ▼
Fresh Start: passphrase_verify.txt 存在確認 → なければ exit 1
        │
        ▼
hostlist.csv 読み込み + Show-SessionSetupForm 表示
        │
        ▼
Worker 選択 + Host 選択 + Master Passphrase 入力
        │
        ▼
Set-SelectedHostEnvironment（SELECTED_* env vars 設定）
Initialize-EvidenceBasePath（evidence/{ts}_{pc}_{sn}_evidence/ 作成）
Initialize-Session（workers / mediaSerial / session.json）
Initialize-ExecutionHistory（logs/history/execution_history.csv）
Initialize-ModuleSystem（modules/{standard,extended}/*/module.csv 検出）
Load-Profiles（profiles/*.csv 一覧化）
Show-ExecutionToolbar（in-process STA Runspace 起動、main.ps1 L1702）
        │
        ▼
Show-OperatorDashboard（メインダッシュボード）
```

### Resume Detection の優先順位

`main.ps1` 起動時、Fresh Start に入る前に過去セッションの再開判定を行う：

| 優先度 | 検出ファイル | 動作 |
|---|---|---|
| **1（最優先）** | `modules/standard/windows_update/wu_state.json` | `Invoke-WindowsUpdateLoop` を起動。WU 中の再起動跨ぎ復元 |
| **2** | `kernel/json/resume_state.json` (`schemaVersion=2 + ExecutionMode=Flex`) | FlexProfile resume ルート |
| **3** | `kernel/json/resume_state.json` (`schemaVersion=1` または `ExecutionMode=Linear`) | Linear Profile resume：AutoPilot 中 → `Wait-SystemReady` + `Invoke-AutoResumeCountdown` 60s で自動継続 / それ以外 → `Confirm-Execution "Resume profile execution?"` |
| 4（fall-through） | （なし） | Fresh Start： Show-SessionSetupForm 表示 |

resume 中のパスフレーズ復元は DPAPI（Windows データ保護 API）で session 跨ぎに保持。失敗時は手動再入力（最大 3 回）にフォールバック。

### Fresh Start の検査順

Fresh Start ルートで実行される検査は次の順：

1. `passphrase_verify.txt` 存在 → 不在なら `Show-Error` 表示後 `exit 1`
2. `hostlist.csv` 存在（`kernel/csv/hostlist.csv`）→ 不在なら `Show-Error` 表示後 `exit 1`
3. ConsoleSize 75x35 セット
4. ロゴ表示（manifesto テーマ）
5. session form 表示（Worker / Host / Passphrase の 3 入力）

セッション開始フォームの操作詳細は [fabriq__usage__02_session_setup.md](fabriq__usage__02_session_setup.md) を参照。

---

## 4. トラブル対応

### 「passphrase_verify.txt が見つかりません」

`Show-Error "Passphrase verification token not found: kernel/txt/passphrase_verify.txt"` が出て `exit 1`。

**原因**: fabriq_studio でのマスターパスフレーズ設定が未完了 / 配布元の fabriq フォルダにトークンが含まれていない。

**対処**:
1. 配布元 PC で fabriq_studio を起動
2. 該当の fabriq フォルダをワークスペースとして開く
3. マスターパスフレーズを設定 → トークン生成
4. fabriq フォルダを USB に取り直し、配備先へコピーして再配備（旧 `Deploy.bat` は kernel 3.6.0 で廃止済み）

USB 上で直接 `passphrase_verify.txt` を作る方法は無い（暗号化ロジックが fabriq_studio 側に閉じている）。

### 「hostlist.csv が見つかりません」

`Show-Error "hostlist.csv not found: kernel/csv/hostlist.csv"` が出て fresh start を中断。

**原因**: 配布元の fabriq フォルダから `kernel/csv/hostlist.csv` が抜けている、またはコピー時に除外されている（旧 `Deploy.bat` の robocopy 除外設定は kernel 3.6.0 の Deploy.bat 廃止に伴い該当しない）。

**対処**: 配布元の `kernel/csv/hostlist.csv` を確認、必要なら fabriq_studio で 1 行以上のエントリを編集してから配備し直す。空 CSV でも起動はできるが、対象 PC の自動選択（`$env:COMPUTERNAME` 一致）は機能しない。

### パスフレーズ入力で何度も拒否される

`Test-MasterPassphrase` が `false` を返し続ける状態。

**原因**:
- `passphrase_verify.txt` が異なるパスフレーズで生成されている（別の fabriq_studio セッションで違う値を設定した）
- ファイルが破損している（先頭が `ENC:` で始まらない）

**対処**: fabriq_studio 側で再設定する。`passphrase_verify.txt` を直接編集することは不可（trustless な検証フローのため、暗号文を構築するには正しい passphrase が必要）。

### Resume が誤発火する（前回セッションが残っている）

`resume_state.json` または `wu_state.json` が前回中断時に残ったまま fresh start したい場合。

**対処**: 各ファイルを直接削除：

```powershell
Remove-Item kernel/json/resume_state.json -ErrorAction SilentlyContinue
Remove-Item modules/standard/windows_update/wu_state.json -ErrorAction SilentlyContinue
```

または、ダッシュボードに到達できる状態であれば `Refabriq` ボタンで session 系を一括リセット（[fabriq__usage__05_evidence_and_quick_actions.md](fabriq__usage__05_evidence_and_quick_actions.md) §「Refabriq」）。

### 起動直後にコンソールが消える

UAC 昇格に失敗したか、別の致命的エラーで `exit 1` を踏んだ。

**対処**: `Fabriq.exe` の代わりに **PowerShell コンソールから直接 `kernel/main.ps1` を実行**してエラー内容を確認する：

```powershell
cd C:\Windows\work\fabriq
Start-Process powershell -Verb RunAs -ArgumentList "-NoExit", "-File", ".\kernel\main.ps1"
```

`-NoExit` で `exit 1` 後もコンソールが残るため、`Show-Error` のメッセージが読める。

---

## 5. 起動成功後の次の手順

`Fabriq.exe` 起動 → Resume なし → SessionSetupForm 表示までが「初回起動」の範囲。続いて：

1. **セッション開始フォームでの 3 入力**（Worker / Host / Passphrase）→ [fabriq__usage__02_session_setup.md](fabriq__usage__02_session_setup.md)
2. **Profile を選んで Linear 一括実行** → [fabriq__usage__03_profile_execution_linear.md](fabriq__usage__03_profile_execution_linear.md)
3. **FlexProfile で部分実行** → [fabriq__usage__04_flexprofile_dashboard.md](fabriq__usage__04_flexprofile_dashboard.md)
4. **Evidence 確認 / Quick Actions** → [fabriq__usage__05_evidence_and_quick_actions.md](fabriq__usage__05_evidence_and_quick_actions.md)

---

## 関連ドキュメント

- 全体像と読書順: [fabriq__overview__readme.md](fabriq__overview__readme.md)
- セッション開始フォームの操作: [fabriq__usage__02_session_setup.md](fabriq__usage__02_session_setup.md)
- メインダッシュボード仕様（UI 詳細）: [fabriq__apps__01_fabriq_operator_dashboard.md](fabriq__apps__01_fabriq_operator_dashboard.md)
- カーネル全体像: [fabriq__kernel__01_overview.md](fabriq__kernel__01_overview.md)
- オーケストレーション内部: [fabriq__kernel__03_orchestration.md](fabriq__kernel__03_orchestration.md)
- 再起動跨ぎの実装: [fabriq__kernel__05_resume_restart.md](fabriq__kernel__05_resume_restart.md)
- fabriq_studio でのワークスペース・パスフレーズ設定: [fabriq_studio__usage__01_workspace_setup.md](fabriq_studio__usage__01_workspace_setup.md)
