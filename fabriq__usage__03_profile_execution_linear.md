# Profile Linear 実行と個別モジュール実行

> **対象**: fabriq / usage
> **対象バージョン**: kernel 3.2.2（取得元: `E:\fabriq\kernel\KERNEL_VERSION`）+ commit `e513cf1`（取得元: `git -C E:\fabriq rev-parse --short HEAD`、2026-05-06）
> **ドキュメント更新日**: 2026-05-07

ダッシュボード `Profiles` タブの **`[Execute Profile]`（Linear）** と `Modules` タブの **`[Execute]`（個別実行）** の使い分けと動作詳細。state-aware 部分実行は別 doc（[fabriq__usage__04_flexprofile_dashboard.md](fabriq__usage__04_flexprofile_dashboard.md)）。

---

## 前提

- セッション確立済み（[fabriq__usage__02_session_setup.md](fabriq__usage__02_session_setup.md) 完了）
- Profile Linear を使う場合は `profiles/*.csv` に対象 profile が存在
- 個別実行を使う場合は `modules/{standard,extended}/*/module.csv` が `Initialize-ModuleSystem` で読み込まれていること

---

## Linear と Flex の選択基準

| 観点 | Linear `[Execute Profile]` | Flex `[Execute (Flex)]` |
|---|---|---|
| 実行モデル | 先頭から末尾まで一気通貫 | 行を任意にチェックして部分実行 |
| AutoPilot トグル | あり（チェックボックスで ON/OFF） | 常に AutoPilot 挙動（トグル無し） |
| `[Complete]` finalize | 自然完了で自動発火 | 操作員が `[Complete]` 押下まで保留 |
| Group 列利用 | `Group` 列は **無視**（Linear は参照しない、kernel 3.2.0+） | `[Run: <Group>]` ボタンとして利用 |
| Resume の扱い | `__RESTART__` 跨ぎで自動再開 | 状態管理 dashboard で再表示 |
| 想定用途 | 最初から最後まで自動で流したい / シンプル運用 | 失敗箇所だけ再実行 / 段階的進行 / 人間判断挟む |

「最初から最後まで Linear」「途中で止まったら Flex で続き」という併用も可能（kernel 3.1.x 以降は **Linear が安定したのち Flex に置き換え予定**だが、現状は両経路並走）。

本ドキュメントは Linear を扱う。Flex の操作詳細は [fabriq__usage__04_flexprofile_dashboard.md](fabriq__usage__04_flexprofile_dashboard.md)。

---

## 1. Profiles タブの操作

ダッシュボード Tab 1。

### Profile DataGridView

`profiles/*.csv` を `Load-Profiles` が走査して充填。列：

| 列 | 内容 |
|---|---|
| `ProfileName` | ファイル名（拡張子なし） |
| `Modules` | 内含モジュール件数（マーカー除く） |
| `Total` | 全行数（マーカー含む） |
| `Path`（hidden） | フル CSV パス |

`profiles/` に CSV が無い場合、`Load-Profiles` が `Basic Setup` / `Full Setup` の 2 件をデフォルトで生成する。

### Profile Detail RichTextBox

選択行の中身を `Order Description` 形式で表示（リアルタイム）。これで実行前にフローを目視確認できる：

```
10  WaitSec=3
20  ホスト名設定
30  IP アドレス設定
40  再起動
50  レジストリ設定
```

`Description` 列が空の行は `Order` のみ表示される。

### `[AutoPilot]` チェックボックス

既定 ON。**Linear `[Execute Profile]` のみ**この値を参照する（Flex は無視）。

ON のとき：

- `Confirm-Execution` / `Wait-KeyPress` 等の Y/N プロンプト・キー入力待ちが **自動 Y / 自動 Press-Enter** になる
- モジュール間に `AutoPilotWaitSec`（既定 3 秒）のスリープが入る
- Error/Partial 発生時に `ErrorMode` 列の指定（`skip` / `retry` / 空＝ダイアログ）に従って自動分岐

OFF のとき：通常の対話モード。各モジュールが Y/N 確認を要求し、`Wait-KeyPress` でキー入力待ち。AutoPilot Wait は無効。

### ボタン

| ボタン | 動作 |
|---|---|
| `[View Details]` | 選択 Profile の CSV を Explorer で `select` 表示（CSV 編集なら `[Open CSV Editor]` 経由） |
| `[Execute (Flex)]` | 緑ボタン。`Action="FlexProfile"` を返し、別ダッシュボード（FlexProfile）に遷移。AutoPilot チェックボックス値は伝播しない（Flex 側で独立管理） |
| `[Execute Profile]` | 青ボタン（Linear path）。`Action="ExecuteProfile"` を返し、AutoPilot チェックボックス値を伝播 |

---

## 2. Linear 実行フロー

`[Execute Profile]` 押下時の `main.ps1` 側 dispatch（L1638）：

```
1. Resolve-ProfileModules -ProfileCsvPath $ProfilePath -AllModules $allModules
   ─ profile CSV 全行をパース、ScriptPath を modules/ 解決
   ─ 特殊マーカー（__RESTART__ 等）は _IsRestart / _IsAsync 等のフラグ立て
   ─ ValidModules / InvalidModules / Markers を返す

2. ValidModules.Count == 0 なら "No executable modules found in profile" → 中断

3. Show-Info "Executing profile: <ProfileName>"

4. Invoke-BatchExecution -SelectedModules ValidModules
                        -AutoPilot:$AutoPilot
                        -AutoPilotWaitSec <int>
                        -ProfilePath <CSV>
                        -ProfileName <Name>
                        -FullProfileModules ValidModules
   ─ ExecutionMode = Linear（既定）
   ─ FinalizeOnComplete = true（既定）

5. ResultSummary 構築（OK / NG / Skip 件数）

6. errCount > 0:
     Yellow "Profile Completed with Errors" + Warning
   else:
     Green "Profile Execution Completed"

7. Wait-KeyPress（任意キーで dashboard 帰還）
```

### Resolve-ProfileModules の動作

profile CSV から実行可能なモジュールリストを構築する関数。各行に対して：

- 通常モジュール（`standard/<name>/<script>.ps1` 形式の `ScriptPath`）→ `modules/standard/<name>/module.csv` を引いて MenuName / Category / Order を解決
- 特殊マーカー（`__RESTART__` 等）→ `_IsRestart` 等の内部フラグを立てた擬似モジュール
- 該当 module.csv が無い場合 → `InvalidModules` に入り「module not found」警告（Linear 実行時はスキップ、historic な互換維持のため）

### Invoke-BatchExecution の foreach ループ（モジュール 1 件あたり）

`common.ps1` L223〜の `Invoke-BatchExecution`。各 module で：

```
1. Show-BatchProgress -Current N -Total M -ItemName <MenuName>

2. 特殊マーカー分岐:
   ├── _IsRestart → Save-ResumeState + Register-FabriqRunOnce
   │                + Invoke-CountdownRestart → return（main.ps1 終了、再起動後 resume）
   ├── _IsReexplorer → Stop-Process -Name explorer + 復帰待ち（最大 15s）
   └── （通常モジュール）→ 下に続く

3. AutoPilot 中 + current > 1 なら：
     [AUTOPILOT] Next module in <WaitSec>s...
     Start-Sleep <WaitSec>

4. retry ループ:
   do {
     ├── _AutoLogonUser があれば $env:FABRIQ_AUTOLOGON_USER に渡す
     ├── _Segment があれば $env:FABRIQ_SEGMENT に渡す
     ├── _IsAsync かつ AsyncConfig.Enabled なら Invoke-SafeCommandAsync（Runspace）
     │   else Invoke-SafeCommand（同期）
     ├── env vars クリーンアップ
     └── result.Status が Error/Partial なら:
           AutoPilot 中 + ErrorMode によって分岐:
             "skip"  → そのまま記録して続行
             "retry" → autoRetryCount++、最大 5 回までリトライ
             ""      → Show-AutoPilotErrorDialog でダイアログ
   } while ($retryModule)

5. 結果記録:
   - Add-ExecutionResult（in-memory）
   - Write-ExecutionHistory（execution_history.csv 追記）
   - Capture-ScreenEvidence（スクリーンショット保存）

6. completedResults に hashtable 追加（後続の resume_state.json 用）
```

完走したら：

```
7. Linear かつ FinalizeOnComplete=true（既定）なら：
   Complete-ProfileExecution
     ─ HTML チェックリスト生成
     ─ extended/log_uploader モジュールがあれば自動発火
     ─ "Log Upload (cl)" を実行履歴に追加
```

ExecutionMode='Linear' の場合は `FinalizeOnComplete=true` で完了 = 即 finalize。Flex は `FinalizeOnComplete=false` で `[Complete]` 押下まで保留。

---

## 3. 特殊マーカーの operator 視点動作

profile CSV の `ScriptPath` 列に書く 5 種のマーカー。詳細仕様は [fabriq__contracts__special_markers.md](fabriq__contracts__special_markers.md)、ここでは「使うとどう見えるか」に絞る。

### `__AUTOPILOT__`

```csv
10,__AUTOPILOT__,1,WaitSec=3,,,
```

**作用**: 以降のモジュールを AutoPilot 化（`Confirm-Execution` / `Wait-KeyPress` を自動 Y / Enter）。`Description` 列の `WaitSec=N` でモジュール間ウェイト秒指定（既定 3 秒）。

**operator が見る変化**:

- 通常なら出る `Continue? (Y/N)` プロンプトが消え、`[AUTOPILOT] Next module in 3s...` というマゼンタ表示に置き換わる
- `Wait-KeyPress` の「任意キーを押してください」プロンプトもスキップ
- ただし **モジュール内の `Show-Error` や `Show-Warning` は通常通り表示**される（停止しない）

ダッシュボードの `[AutoPilot]` チェックボックスを使えば profile 内に `__AUTOPILOT__` を書かなくても同等動作になる。両者は同じ `$global:AutoPilotMode = $true` を立てる。

### `__RESTART__`

```csv
40,__RESTART__,1,再起動,,,
```

**作用**: 該当行に到達したら Windows 再起動 → RunOnce 経由で fabriq を自動再開。

**operator が見る変化**:

```
========================================
  Profile Restart Phase
========================================

  Progress: 3 modules completed
  Remaining: 2 modules after restart

System will restart in 10 seconds...
[ESC] to cancel
```

カウントダウン後再起動。再起動後：

```
1. Windows ログオン
2. RunOnce で Fabriq.exe が自動起動
3. main.ps1 の Resume Detection が resume_state.json を検出
4. AutoPilot 中なら Wait-SystemReady（タスクスケジューラ / ネット復帰待ち）
5. Invoke-AutoResumeCountdown 60s（操作員が止めたいなら ESC）
6. Restore-HostEnvironment（SELECTED_* 環境変数を json から復元）
7. パスフレーズ復元（DPAPI 暗号化された値を復号、失敗時は手動再入力）
8. Invoke-BatchExecution 再開（中断された Order の次から）
```

`__RESTART__` を使うために特別なセットアップは不要。Fabriq.exe の RunOnce 登録（`Register-FabriqRunOnce`）と DPAPI passphrase の永続化は kernel が透過的に処理する。

resume が終わると HTML チェックリストが完成し、`evidence/{ts}_{pc}_{sn}_evidence/` 配下に Phase 1（再起動前）+ Phase 2（再起動後）のスクリーンショット連番がそのまま並ぶ。

### `__ASYNC__`

```csv
50,__ASYNC__,1,,,,
60,standard/winget_install/winget_install.ps1,1,winget,,,
```

**作用**: 以降のモジュールを **監視付き Runspace**（別スレッド）で実行する（kernel 2.1.0 以降）。

**operator が見る変化**:

```
[ASYNC] Running in monitored runspace (Skip available via Status Monitor)
```

DarkCyan 色のメッセージ。Status Monitor 別ウィンドウで `[Skip]` ボタンが押せるようになり、ハングしているモジュールを強制中断できる。

`async_config.json` の `DefaultTimeoutSec` で自動タイムアウトも可能。`Enabled: false` で全体無効化（同期実行に戻す）。

### `__REEXPLORER__`

```csv
70,__REEXPLORER__,1,Explorer 再起動,,,
```

**作用**: Explorer プロセスを `Stop-Process -Name explorer -Force` で殺し、復帰を最大 15 秒待つ（Windows が自動再起動してくれる）。15 秒で復帰しなければ `Start-Process explorer.exe` で強制起動。

**operator が見る変化**: 一瞬タスクバーが消えて再描画される。レジストリ変更の即時反映（壁紙 / アイコン配置 / 等）に使う。

### `__AUTO_to_<User>__`

```csv
80,__AUTO_to_admin01__,1,admin01 で AutoLogon,,,
```

**作用**: `autologon_config` モジュールを呼び出し、その内部で `autologon_list.csv` の `User=admin01` 行のエントリ（パスフレーズ等）を使って自動ログオン設定する。

**operator が見る変化**: 通常の autologon_config 実行と同じだが、`$env:FABRIQ_AUTOLOGON_USER=admin01` が事前に立つことで対象ユーザーが固定される。複数ユーザーを切り替える profile で使う。

---

## 4. ErrorMode の挙動（AutoPilot 中のみ）

profile CSV の `ErrorMode` 列に書く値。AutoPilot 中に Error/Partial が発生したときの分岐：

| 値 | 挙動 |
|---|---|
| `skip` | `[AUTOPILOT] ErrorMode=Skip -> recording <Status> and continuing` で次モジュールへ。失敗 status は履歴に残る |
| `retry` | `[AUTOPILOT] ErrorMode=Retry -> auto-retry (N/5)` で最大 5 回まで自動リトライ。5 回到達で `max retry reached` 警告 + 失敗 status 記録 |
| 空 / `ask` / 未指定 | `Show-AutoPilotErrorDialog` で operator に Retry / Skip 確認ダイアログ |

**AutoPilot OFF の場合は ErrorMode 列は無視される**（通常通り Y/N ダイアログが出る）。

`autoRetryCount` は **モジュール単位でリセット**される（次のモジュールで再度 0 から）。同じモジュールを 5 回連続失敗すると諦めて次へ。

---

## 5. Linear 終了時の挙動

profile が完走（最後のモジュールまで到達）すると：

### 自動 finalize（`FinalizeOnComplete=true`）

Linear は既定で `FinalizeOnComplete=true` のため、`Invoke-BatchExecution` が `Complete-ProfileExecution` を自動発火する：

1. **HTML チェックリスト生成** — `evidence/{ts}_{pc}_{sn}_evidence/` 配下に `checklist_<ts>.html` を生成。verify-table（hostlist との突合）+ ModuleItems（実行結果テーブル、kernel 2.x で `Verified` 列追加）含む
2. **`extended/log_uploader` 自動発火** — `kernel/csv/log_destinations.csv` に登録された宛先（UNC 共有 / ローカル / SFTP 等）へ Transcript・実行履歴・evidence を一括アップロード
3. **`"Log Upload (cl)"` を実行履歴に追記** — 自動 finalize 経路の trail として記録

### Status Bar Summary

dashboard 復帰時に下部 status bar に表示：

```
Last: <ProfileName> - <X> OK  <Y> NG  <Z> Skip
```

これは `$script:lastGuiResultSummary` に格納され、次の profile/module 実行で更新される。

### 結果サマリ表示

`Wait-KeyPress` 前に：

- `errCount > 0` なら黄色枠で `Profile Completed with Errors` + `Some modules had errors.` 警告
- `errCount == 0` なら緑枠で `Profile Execution Completed`

任意キーでダッシュボードに戻る。

---

## 6. 個別モジュール実行（Modules タブ）

### Modules タブの構成

| 要素 | 役割 |
|---|---|
| `Category ComboBox` | "All" + categories.csv の name |
| `Search TextBox` | MenuName 部分一致 |
| Module DataGridView | # / Module / Category 列（Script / Order 列は hidden） |

`Initialize-ModuleSystem` が `module.csv` を全件走査し、`Build-CategoryMenu` が `categories.csv` の順序でグルーピングしたものを表示する。

### `[Execute]`（青ボタン）

選択行 1 件を即時実行する `Action="ExecuteModules"` を返す。

`main.ps1` 側 dispatch（L1680）：

```
Invoke-BatchExecution -SelectedModules @($script:allModuleData[$idx])
```

**Profile 系のパラメータ（ProfilePath / ProfileName / FullProfileModules）は渡されない**。これは：

- profile 復帰のための resume_state.json 書き込みをしない
- HTML チェックリストの自動 finalize をしない（`FinalizeOnComplete` のデフォは true だが、`ProfilePath` 空なので Complete-ProfileExecution が走らない）
- 単純な「1 モジュールだけ実行して履歴に追加」用途

```
Last batch: <X> OK  <Y> NG
```

がステータスバーに記録される。

### 行ダブルクリック

`[Execute]` 押下と同等動作（ショートカット）。

### Segment 切替

`Segment` 列を持つモジュール（hostname_config / app_config 等）は profile CSV からだけでなく、個別実行時に Segment を渡す方法はダッシュボードに無い。**Segment を使い分けたいなら profile 経由が必須**（個別実行は Segment=空のすべての行を対象とする `Import-ModuleCsv` 動作）。

---

## トラブル対応

### "No executable modules found in profile"

`Resolve-ProfileModules` が ValidModules を 0 件返した。考えられる原因：

- profile CSV のすべての行が `Enabled=0` か特殊マーカーのみ
- すべての `ScriptPath` が無効（typo / 移動済み）

**対処**: `[View Details]` で Detail RichTextBox を確認。`Open CSV Editor` で profile 編集。

### `__RESTART__` 後に再開しない

```
- RunOnce が登録できなかった（管理者権限不足）
- DPAPI passphrase の復号に失敗（ユーザーセッション変更等）
- パスフレーズ手動再入力で 3 回失敗
```

**対処**:

- `Register-FabriqRunOnce` のログを `Show-Error` 側で確認
- `kernel/json/resume_state.json` 残存ならダッシュボードに戻ってから再実行（自動 detect される）
- パスフレーズ手動入力フォールバックを使う

### AutoPilot Wait が長すぎる/短すぎる

- profile に `__AUTOPILOT__` を書いて `WaitSec=N` を Description で指定
- もしくはダッシュボード `[AutoPilot]` 横の Wait NumericUpDown（apps__01 §「Profiles タブ」）

### Error 後にダイアログが出てこない（AutoPilot 中）

`ErrorMode` 列に `skip` または `retry` が指定されていればダイアログは出ない（仕様）。**AutoPilot OFF にする** か `ErrorMode` を空にして発火させる。

### 個別実行（Modules タブ）が profile より速いはずなのに遅い

`__ASYNC__` は profile 経由でしか効かない。Modules タブは常に `Invoke-SafeCommand` 同期実行。長時間モジュールは profile に組み込んで `__ASYNC__` で囲む方が中断可能性が高い。

### 完走したのに HTML チェックリストが生成されない

Linear path の `Invoke-BatchExecution` 呼び出しは `ProfilePath` を渡す必要がある。Profiles タブから `[Execute Profile]` で実行すれば自動的に渡る。Modules タブの個別実行では `Complete-ProfileExecution` が発火しないため HTML 出ない。

→ 手動で `[Regenerate Checklist]`（[fabriq__usage__05_evidence_and_quick_actions.md](fabriq__usage__05_evidence_and_quick_actions.md) §「HTML チェックリスト」）を呼べる。

---

## 関連ドキュメント

- セッション開始: [fabriq__usage__02_session_setup.md](fabriq__usage__02_session_setup.md)
- FlexProfile（部分実行）: [fabriq__usage__04_flexprofile_dashboard.md](fabriq__usage__04_flexprofile_dashboard.md)
- Evidence と Quick Actions: [fabriq__usage__05_evidence_and_quick_actions.md](fabriq__usage__05_evidence_and_quick_actions.md)
- ダッシュボード UI 仕様: [fabriq__apps__01_fabriq_operator_dashboard.md](fabriq__apps__01_fabriq_operator_dashboard.md)
- 特殊マーカーの公開契約: [fabriq__contracts__special_markers.md](fabriq__contracts__special_markers.md)
- Profile CSV スキーマ契約: [fabriq__contracts__profile_csv_schema.md](fabriq__contracts__profile_csv_schema.md)
- ModuleResult 契約: [fabriq__contracts__module_result.md](fabriq__contracts__module_result.md)
- カーネルオーケストレーション内部: [fabriq__kernel__03_orchestration.md](fabriq__kernel__03_orchestration.md)
- 再起動跨ぎの実装: [fabriq__kernel__05_resume_restart.md](fabriq__kernel__05_resume_restart.md)
