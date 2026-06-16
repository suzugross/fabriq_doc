# 特殊マーカー（Special Markers）契約

> **対象**: fabriq / 契約（特殊マーカー）
> **対象バージョン**: kernel 3.6.0（取得元: `E:\fabriq\kernel\KERNEL_VERSION`）+ commit `0fca159`（取得元: `git -C E:\fabriq rev-parse --short HEAD`、2026-06-16）
> **ドキュメント更新日**: 2026-06-16

profile CSV の `ScriptPath` 列に書ける特殊識別子。`Resolve-ProfileModules` がこれらを解釈してプロファイル全体の挙動を制御する。

---

## 現行マーカー（6 種、kernel 3.6.x）

### 1. `__AUTOPILOT__`（kernel 2.0.0〜）

```csv
Order,ScriptPath,Enabled,Description,Segment,ErrorMode,Group
10,__AUTOPILOT__,1,WaitSec=3,,,
```

**動作**:
- `Enabled=1` のとき `$global:AutoPilotMode = $true` を立てる
- `Description` 列に `WaitSec=N` 形式があれば `$global:AutoPilotWaitSec` を `N` に設定（regex `WaitSec=(\d+)`）
- 効果は **以降のすべてのモジュール**に及ぶ（プロファイル末尾まで）
- ValidModules には**入らない**（メタデータ抽出のみ）

**AutoPilot の意味**:
- `Confirm-Execution` / `Confirm-ModuleExecution` が自動 Y を返す（Y/N プロンプトをスキップ）
- `Wait-KeyPress` が即 return（Press-Enter 待機をスキップ）
- 各モジュール開始前に `AutoPilotWaitSec` 秒の inter-module wait
- Error/Partial 時に `_ErrorMode` 列で分岐（`skip` / `retry` / `ask`）
- `Show-AutoPilotErrorDialog` が `ask` モードで Retry/Skip ダイアログ表示

**注意**: AutoPilot は「**確認スキップ + auto-resume**」であり「完全無人」ではない。operator が脇で見ていて状況に応じて Esc できる前提（feedback memory `feedback_autopilot_wording`）。

### 2. `__ASYNC__`（kernel 2.1.0〜 / 3.3.0 で意味論拡張）

```csv
20,__ASYNC__,1,以降を Runspace 化,,,
```

**動作**:
- `Enabled=1` かつ `async_config.json` の `Enabled=true` のときのみ効果発動（kill switch）
- 以降のすべての通常モジュールに `_IsAsync=$true` を attach（sticky）
- `Invoke-BatchExecution` が `_IsAsync` を見て `Invoke-SafeCommandAsync`（Runspace + monitoring）を選ぶ
- ValidModules には入らない
- 効果範囲: プロファイル末尾まで（途中で sync に戻すマーカーは無い）

**kernel 3.3.0 での意味論拡張（後方互換）**:

`async_config.json` に新フィールド `DefaultAsync` が追加され、**shipped default は `true`**。`DefaultAsync=true` のとき profile 1 行目から `_IsAsync=$true` が全モジュールに attach されるため、`__ASYNC__` マーカーは **idempotent な ON-only no-op** として動作する（既に async なので flip しない）。マーカー自体は廃止されておらず、以下の用途で引き続き有効：

- `DefaultAsync=false` 環境への portable な profile を作る場合（明示的に async opt-in したいケース）
- profile を読んだ運用者に「ここから async が効く」と視覚的に示したい場合（self-documenting）

**優先順位**（高 → 低）: `Enabled=false`（kill switch、全モジュールを同期に降格） > `DefaultAsync=true`（全モジュール async） > `__ASYNC__` マーカー（マーカー以降を async）

**意味**: モジュール内ハングや長時間処理に対して、Status Monitor の Skip ボタン or `DefaultTimeoutSec` で強制中断できるようにする。kernel 3.3.0 以降は profile 側の書き忘れによる安全網の無効化を防ぐため、全モジュール async がデフォルトになっている。詳細は [fabriq__kernel__08_async_execution.md](fabriq__kernel__08_async_execution.md)。

### 3. `__RESTART__`（kernel 2.0.0〜）

```csv
40,__RESTART__,1,再起動,,,
```

**動作**:
- ValidModules に `_IsRestart=$true` の専用 PSCustomObject として入る（MenuName=`[RESTART]`, Category=`System`）
- `Invoke-BatchExecution` の loop が検出すると：
  1. `Save-ResumeState` で `kernel/json/resume_state.json` に状態保存
  2. `Register-FabriqRunOnce` で次回起動を予約
  3. `Add-ExecutionResult [RESTART] Success` で履歴記録
  4. `Invoke-CountdownRestart -Seconds 5` で再起動（loop から return）
- 再起動後の resume で「Order > restart 行の Order」のモジュールから続行

**前提**: `autologon_config` モジュールが事前に AutoLogon を組んでいないと、再起動後に手動ログオンが必要になる（無人運用が成立しない）。

### 4. `__REEXPLORER__`（kernel 2.0.0〜）

```csv
50,__REEXPLORER__,1,Explorer 再起動,,,
```

**動作**:
- ValidModules に `_IsReexplorer=$true` で入る
- `Invoke-BatchExecution` 検出時:
  ```powershell
  Stop-Process -Name explorer -Force
  最大 15 秒、1 秒間隔で Get-Process explorer の復活を待つ
  自動復活しなければ Start-Process explorer.exe を強制発火
  Add-ExecutionResult [REEXPLORER] Success/Error
  ```
- 結果は `$completedResults` に正しい Status で記録される（kernel 3.x で改善、それ以前は hardcoded "Success" だった）

**用途**:
- HKCU レジストリ変更（reg_hkcu_config）を即時反映する
- タスクバー設定 / Start Layout / Explorer 動作系の変更後に必須
- Active Setup / Startup Batch 系のリフレッシュ

### 5. `__AUTO_to_<User>__`（kernel 2.0.0〜）

```csv
70,__AUTO_to_admin01__,1,管理者で AutoLogon 設定,,,
80,__AUTO_to_user01__,1,ユーザーで AutoLogon 設定,,,
```

**動作**:
- 正規表現 `^__AUTO_to_(.+)__$` でマッチ、`$Matches[1]` を `$autoLogonUser` として抽出
- `autologon_config` モジュールを参照し、PSCustomObject を copy
- `_AutoLogonUser` 属性を attach
- MenuName を `[AUTO:User] AutoLogon Configuration` に書き換え
- `Invoke-BatchExecution` 実行直前に `$env:FABRIQ_AUTOLOGON_USER = $module._AutoLogonUser`、終了後に `$null` でクリア
- `autologon_config` 内で `$env:FABRIQ_AUTOLOGON_USER` を読んで `autologon_list.csv` から該当行を選択

**用途**:
- 同じプロファイル内で複数の AutoLogon 切替が必要なシナリオ（admin で WU → user で BitLocker など）
- profile に `autologon_config` を直書きすると user 引数が固定されるが、`__AUTO_to_<User>__` で動的選択可能

### 6. `__GATE__`（kernel 3.6.0〜）— 前進バリア（forward barrier）

```csv
30,__GATE__,1,検証ゲート,,,
```

**動作（マーカー登録）**:
- `Resolve-ProfileModules` の `specialMarkers` に `'__GATE__' = @{ MenuName = "[GATE]"; Flag = "_IsGate" }` で登録（`E:\fabriq\kernel\common.ps1` L3678）
- `__GATE__` 行は `_IsGate=$true` の checkpoint として ValidModules に入るが、**何も実行しない**。`Invoke-BatchExecution` の loop が `_IsGate` を検出すると `[GATE] checkpoint at Order N` を表示して `continue`（`E:\fabriq\kernel\main.ps1` L458-462）

**前進バリアの意味（admission control）**:
- 各 `__GATE__` は「**直前のゲート（または profile 開始）〜自身**」の窓を守る。窓内のモジュールに **Status=Error / Partial**、または **Post-Apply Verification 失敗**（`VerifiedMap[Order] -eq $false`）が1つでもあれば、その gate は **unsatisfied**
- `Get-FabriqGateBarrier`（純関数・内部ヘルパ、`E:\fabriq\kernel\common.ps1` L3747-3811）が Rows を Order 昇順に走査し、**最初の unsatisfied gate の Order** を barrier として返す（無ければ `$null`）
- `Invoke-BatchExecution` が各モジュール実行直前に `Get-FabriqGateBarrier` を評価し、`barrier != $null` かつ `module.Order >= barrier` なら `[GATE] Blocked: Order N (name) is beyond the unsatisfied gate at Order M. Resolve the failure(s) above and re-run.` を表示し、当該モジュールは **Pending 据え置き**（新ステータス記録なし）で `continue`（`E:\fabriq\kernel\main.ps1` L464-476）
- 完走時、ブロックがあれば `[GATE] N module(s) blocked by an unsatisfied gate (Orders: ...)` をサマリ表示（`E:\fabriq\kernel\main.ps1` L709）

**ブロックする / しない条件**:
- **ブロックする**: Status=`Error` / `Partial`（`common.ps1` L3802）、または Verified=`$false`（L3807）
- **ブロックしない**: `Success` / `Skipped` / `Cancelled` / `Pending`（未実行・StatusMap に不在）。Verified が `$null`（未検証モジュール）/ `$true` も非ブロック（L3766-3768, L3804-3807）。ゲートは「失敗」を守るものであり「未実行（省略）」は守らない

**動的評価**:
- 判定に渡す StatusMap は **判定時点の状態**（当該 run の蓄積 + session history）。`__RESTART__` 跨ぎも history 経由で保持されるため、フォワード実行中に発生した失敗もゲートで止まる（`E:\fabriq\kernel\main.ps1` L464-470 のコメント、L3758-3762）

**決定的ソート**:
- Order 昇順、Order 同値では **非ゲート(0) → ゲート(1)** の二次キーを使い、同 Order のモジュールはそのゲートの窓の**内側**に数える（保守的。PS 5.1 の `Sort-Object` は非安定のため明示二次キー / `common.ps1` L3778-3785）

**FlexProfile グレーアウト（UI 反映）**:
- `flex_dashboard.ps1` が同じ `Get-FabriqGateBarrier` で barrier を計算（`E:\fabriq\apps\fabriq_operator\lib\flex_dashboard.ps1` L148）
- gate 行: 青タイント `ARGB(214,224,240)` + `Checked` セル ReadOnly + tooltip（L453-456）
- barrier 以降の blocked 行: `Checked=false` + ReadOnly、MenuName / Order を灰色 `ARGB(150,150,150)`（L463-468）、`[Select All]` からも除外（`row.Tag.IsGate` / `row.Tag.Blocked` を bulk-select から外す / L704-708）
- ただし **Status / Verified バッジ色と `[Log]` ボタンは有効のまま**（失敗を可視化し続ける / L460-462）。enforcement の権威は kernel 側にあり、UI は反映のみ（L140-141）

**後方互換**:
- `ModuleResult` 契約は不変。`__GATE__` を含まない既存 profile は **挙動完全不変**。`KERNEL_API.md` §4.2 マーカ表に since 3.6.0 で記載済み

**用途**:
- 検証ステージの区切りに置き、上流のいずれかが Error/Partial または Post-Apply Verification 失敗のまま下流（Order >= gate）を走らせないための前進バリア
- 例: ネットワーク系を窓に括って `__GATE__` を置き、ドメイン参加の前提が満たされない限り後続を Pending 留めにする

---

## 削除済みマーカー（kernel 3.0.0 で MAJOR 破壊削除）

### `__SHUTDOWN__` / `__PAUSE__` / `__STOPLOG__` / `__STARTLOG__`

| マーカー | 旧動作 | 削除理由 |
|---|---|---|
| `__SHUTDOWN__` | プロファイル末尾でシャットダウン | 唯一の使用箇所（profile 末尾）が廃止済み、`Invoke-CountdownShutdown` 内部関数も削除 |
| `__PAUSE__` | profile 中で `Wait-KeyPress` を強制発火 | 実運用での参照ゼロ |
| `__STOPLOG__` / `__STARTLOG__` | Transcript の一時停止/再開 | 実運用での参照ゼロ |

### graceful degradation

旧プロファイルがこれらを含んでいても fabriq はクラッシュせず、`Resolve-ProfileModules` の `$invalidPaths` 経由で「module not found」warning として降格、他モジュールの実行は継続する。fabriq_studio のマーカーパレットでも既に除外されているため、新規 profile 作成時には誤って書けない。

---

## 同 Group 内 / Group 跨ぎでのマーカー挙動（kernel 3.2.0+）

FlexProfile Groups バーの `[Run: <Group>]` ボタン経由で実行する場合：

- **同 Group 内**: マーカー（`__RESTART__` / `__REEXPLORER__` / `__AUTOPILOT__` / `__ASYNC__` / `__AUTO_to_<User>__`）はすべて **そのまま実行**
- **Group 跨ぎ間のマーカー**: 当該 Group 実行時には **skip**（**literal interpretation**）

operator が RESTART を含めたい場合は明示的に Group 値を打つ：

```csv
10,standard/hostname_config/...,1,,,,Network
20,standard/ipaddress_config/...,1,,,,Network
30,__RESTART__,1,,,,Network          ← Network グループで RESTART を含める
40,standard/domain_join/...,1,,,,Network
```

`[Run: Network]` クリックで 10 → 20 → 30 (RESTART) → 再起動 → 40 まで一気通貫。

Linear `[Execute Profile]` は Group 列を無視して全マーカーを順序通り実行する（旧来挙動維持）。

---

## マーカーと `IncludeDisabled` スイッチ

`Resolve-ProfileModules -IncludeDisabled` モード（FlexProfile 専用）でも：

- `__AUTOPILOT__` / `__ASYNC__` の効果発動には `Enabled=1` が必須（disabled 行は無視）
- `__RESTART__` / `__REEXPLORER__` / `__AUTO_to_<User>__` は ValidModules に入るが `_IsCheckedDefault=$false` がつく（dashboard で初期 unchecked）

「**disabled marker 行が global state を勝手に flip しない**」安全弁。FlexProfile でプロファイル全体を可視化しても、Enabled=0 のマーカー行が誤動作することはない。

**3.3.0 以降の補足**: `DefaultAsync=true` 環境では `__ASYNC__` 行が `Enabled=0` であっても、`_IsAsync=$true` は `async_config.json` 側から全モジュールに attach される。「`Enabled=0` のマーカー行が global state を flip しない」原則と、「config が global state を決める」原則が交わる点で、結果として `Enabled=0` の `__ASYNC__` でも全モジュール async になる（マーカーが no-op だから当然）。これは矛盾ではなく、マーカー単独で sync 化を取り消す手段が無いことを示している（取り消したいなら `async_config.json.DefaultAsync=false` か `Enabled=false`）。

---

## マーカー選択の運用知見

| 場面 | 推奨マーカー組み合わせ |
|---|---|
| 全自動キッティング（再起動含む） | `__AUTOPILOT__` + 各モジュール + `__RESTART__` 数箇所 + `autologon_config` 事前配置 |
| 短時間モジュールが連続するセクション | 単純な順序付けのみ（マーカー不要） |
| 長時間で hang リスクある winget / WU 系 | `__ASYNC__` を冒頭に置いて以降を runspace 化 |
| Operator 立ち会い・確認しながら実行 | マーカーなし（CLI モード廃止後は Linear `[Execute Profile]` で個別 Y/N） |
| Site-specific 段階的実行 | FlexProfile + Group 列で論理グループ化 |
| HKCU レジストリ変更後の即時反映 | reg_hkcu_config の直後に `__REEXPLORER__` |

---

## マーカー追加・削除のバージョン影響

- **新マーカー追加**: 公開 API への後方互換な追加 → kernel **MINOR** 昇格（例: 2.0.0 → 2.1.0 で `__ASYNC__` 追加）
- **既存マーカー削除**: 破壊的変更 → kernel **MAJOR** 昇格（例: 2.x → 3.0.0 で 4 マーカー削除）
- **マーカーの動作変更**: 後方互換でない場合は MAJOR / 互換なら MINOR

`KERNEL_API.md §4` がマーカー仕様の真のソース。Studio のマーカーパレットは `KERNEL_API.md` を参照して動的構築すべき設計。
