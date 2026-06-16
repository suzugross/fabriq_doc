# fabriq_operator ダッシュボード仕様

> **対象**: fabriq / apps/fabriq_operator
> **対象バージョン**: kernel 3.6.0（取得元: `E:\fabriq\kernel\KERNEL_VERSION`）+ commit `0fca159`（取得元: `git -C E:\fabriq rev-parse --short HEAD`、2026-06-16）
> **ドキュメント更新日**: 2026-06-16

`apps/fabriq_operator/` は fabriq の操作員向けメインダッシュボードです。`fabriq_operator.ps1` は WinForms アセンブリを読み込み、`lib/` 以下の 8 ファイルを dot-source するだけのブートストラップで、機能本体はすべて `lib/*.ps1` 側に分散しています。

## ファイル構成

| ファイル | 役割 |
|---|---|
| `fabriq_operator.ps1` | WinForms 初期化 + lib/*.ps1 のロード |
| `lib/theme.ps1` | カラーパレット / フォント / `New-Styled*` ヘルパー群 |
| `lib/session_form.ps1` | セッション開始フォーム (`Show-SessionSetupForm`) |
| `lib/dashboard_form.ps1` | メインダッシュボード (`Show-OperatorDashboard`) |
| `lib/flex_dashboard.ps1` | FlexProfile ダッシュボード (`Show-FlexDashboard`) |
| `lib/quickactions_dialog.ps1` | "And More..." サブダイアログ (`Show-AndMoreDialog`) |
| `lib/apps_dialog.ps1` | FabriqApps ランチャー (`Show-AppsDialog`) |
| `lib/log_viewer.ps1` | エントリ別テレメトリ ログビューワ (`Show-ModuleLogViewer` / `Get-ModuleTelemetryLog`、kernel 3.6.0+) |
| `lib/execution_toolbar.ps1` | 実行ツールバー (`Show-ExecutionToolbar` / `Hide-ExecutionToolbar` / `Update-ExecutionToolbar`、旧 status_monitor 後継、3.4.0+) |

## セッション開始フォーム (`Show-SessionSetupForm`)

ダッシュボードに入る前に必ず表示される。返り値は `@{ WorkerID; WorkerName; SelectedHost; MasterPassphrase; Cancelled }`。

入力項目:

- **Worker** — DataGridView (ID / Name 列、ヘッダクリックで sort 可) で `kernel/csv/workers.csv` から選択。一覧外の作業者向けにマニュアル入力テキストボックスも併設し、両者は排他 (片方を入力すると他方をクリア)。
- **Target Host** — DataGridView (AdminID / OldPC / NewPC / IP / Pin)。ライブ検索 (AdminID と NewPCName のみが検索対象。OldPC/IP/Pin は意図的に検索から除外) と件数ラベル "X / Y"。`$env:COMPUTERNAME` と一致する NewPC があれば自動選択し、緑色の「* Auto-detected」ヒントを表示。Row.Tag に host 元オブジェクトを格納することで、sort や filter で行が並び替わっても選択を保持する。
- **Master Passphrase** — `UseSystemPasswordChar` のテキストボックス。`Test-MasterPassphrase` で検証し、失敗時は SelectAll + Focus で再入力を促す。

キーボードショートカット (この form だけは検索効率のためマウスオンリーから外れている):

- 検索 BOX で Esc → 検索クリア
- 検索 BOX で Enter → passphrase BOX へフォーカス移動
- passphrase BOX で Enter → [Start Session] 押下

## メインダッシュボード (`Show-OperatorDashboard`)

`fabriq operator` ウィンドウ (700×560) は **Header + TabControl + StatusBar** の 3 段構成。Header には CentreCOM 風青/黄/赤の 3 色アクセントストライプが入り、右側に `HostName | W: WorkerName` が表示される。

返り値の Action 値: `Quit / ExecuteProfile / FlexProfile / ExecuteModules / NewSession / OpenCsvEditor / OpenEvidence / WindowsUpdate / Restart / Refabriq / HistoryExport / RegenerateChecklist / Manifesto / SystemLauncher / AppsMode / LaunchApp`。これらは `kernel/main.ps1` の switch でディスパッチされる（`OpenEvidence` は Settings タブの Open Folder 経路、`LaunchApp` は FabriqApps ダイアログから個別アプリを起動する経路）。

### Tab 1: Profiles

| 要素 | 役割 |
|---|---|
| Profile DataGridView | ProfileName / Modules / Total / Path (hidden) の 4 列。`Load-Profiles` で `profiles/*.csv` を走査して充填 |
| Profile Detail RichTextBox | 選択行の中身を `Order Description` 形式で表示 (リアルタイム) |
| `[AutoPilot]` チェックボックス | デフォルト ON。Linear 実行時にのみ参照される |
| `[View Details]` | 選択 Profile の CSV を Explorer で `select` 表示 |
| `[Execute (Flex)]` | 緑ボタン。Action=`FlexProfile` を返す。AutoPilot は伝播しない (Flex 側で独立管理) |
| `[Execute Profile]` | 青ボタン (Linear path)。Action=`ExecuteProfile`。AutoPilot チェックボックス値を伝播 |

### Tab 2: Modules

| 要素 | 役割 |
|---|---|
| Category ComboBox | "All" + `$GroupedModules` の name を列挙 |
| Search TextBox | MenuName 部分一致 |
| Module DataGridView | # / Module / Category 列 + Script / Order 列 (hidden)。`Update-ModuleGrid` で再構築 |
| `[Execute]` | 単独行を選択して Action=`ExecuteModules`、`SelectedModules=@($script:allModuleData[$idx])` を返す |
| 行ダブルクリック | 同上 (1 クリック相当) |

### Tab 3: Settings

縦に並んだセクション構成:

- **Evidence Output Path** — `$global:FabriqEvidenceBasePath` を表示。`[Open Folder]` で Explorer 起動 (path 未生成時は親ディレクトリへフォールバック)。
- **Quick Actions (フロント行 5 + And More)**:
  - `[Open CSV Editor]` → Action=`OpenCsvEditor`
  - `[Windows Update]` → Action=`WindowsUpdate`
  - `[Refabriq]` → Action=`Refabriq`
  - `[System Launcher]` → Action=`SystemLauncher`
  - `[And More...]` → `Show-AndMoreDialog` をモーダル表示。返り値の Action (Restart / HistoryExport / RegenerateChecklist / AppsMode) を $result に転記
- **Session** — `Worker / Host / KanriNo` 表示 + `[New Session]` (Action=`NewSession`)
- **Manifesto** — `[Manifeste du Surkitinisme]` (Action=`Manifesto`)

### Status Bar

`$LastResultSummary` を表示するだけの最下段ストリップ。空のときは "Ready"。

## FlexProfile ダッシュボード (`Show-FlexDashboard`, kernel 3.1.5+)

Linear 経路の単方向実行に対し、**1 モジュール単位で再実行可能な状態管理 GUI**。`Resolve-ProfileModules ... -IncludeDisabled` で Profile 全行を取り、history.csv と最後の `$LastBatchResults` を上書きで反映した stateMap (Order → Status/Verified/Message) を作る。

返り値の Action: `Close / RunSingle / RunBatch / RunGroup / Complete / RestartNow / ResetState`。

### グリッド構成 (左→右)

| 列 | 役割 |
|---|---|
| Checked | 選択チェックボックス。デフォルトすべて未チェック (Flex 哲学: 白紙からピック) |
| # | Order |
| Group | 3.2.0+ の Group バー連動 (空欄ありのケースは `_Group` 列が無い旧 CSV) |
| Module | MenuName |
| Status | Pending / Success / Partial / Error / Skipped / Cancelled。CellFormatting で badge 描画 (Success=緑, Partial=黄, Error=赤, Skipped/Cancelled=グレー, Pending=薄グレー) |
| Verified | -/PASS/FAIL。PASS=緑バッジ, FAIL=赤バッジ |
| Log | `DataGridViewButtonColumn 'LogBtn'`（Text="Log", Width=52）。Run 列の左に追加。クリックで `Show-ModuleLogViewer -Order -ModuleName` をモーダル起動 (kernel 3.6.0+)。RunBtn と異なり **非破壊** で、`$result.Action` を触らず form も閉じない (grayout された blocked 行でも有効) |
| Run | `DataGridViewButtonColumn 'RunBtn'`（Text="Run", Width=56）。クリックで RunSingle dispatch (3.1.8 で footer の "Run This: M" を置き換え) |

`CellContentClick` は列名で厳密に分岐し、`LogBtn` クリックは `Show-ModuleLogViewer` を呼んで return、`RunBtn` のみ RunSingle へ進む。

右クリックメニュー: 「Mark as Pending (reset state)」が 1 項目。Action=`ResetState` で発火。

### `__GATE__` 前進バリアの UI 反映 (kernel 3.6.0+)

`__GATE__` 行のグレーアウトと barrier 以降の blocked 行 dimming は、すべて kernel と同じ純関数 `Get-FabriqGateBarrier`（`common.ps1`）でグリッド構築前に barrier（最初の unsatisfied gate の Order、無ければ `$null`）を算出して反映する。**enforcement の権威は kernel 側にあり、UI はその反映に徹する**（admission control は `Invoke-BatchExecution` が module 実行直前に同関数を再評価して行う）。

- **`__GATE__` 行**（`_IsGate=true`）: 行背景を青タイント ARGB(214,224,240) に塗り、`Checked` セルを `ReadOnly=true` にして実行・選択不可にする。Status セルの tooltip にゲートの役割を表示。`__GATE__` 行は実行されない checkpoint であり、`RunBtn` クリックは `$tag.IsGate` 判定で即 return する。
- **blocked 行**（barrier!=null かつ Order>=barrier）: `Checked` を false 固定 + `ReadOnly=true` にし、`MenuName` / `Order` セルの文字色を灰色 ARGB(150,150,150) に落とす。tooltip に「Blocked by unsatisfied gate at Order N」を表示。`RunBtn` クリックは確認ダイアログ（"This module is blocked by an unsatisfied gate."）を出して中断する。
- **Status / Verified バッジ色と `[Log]` ボタンは blocked 行でも有効のまま** — ブロックの原因となった失敗が見えるよう、CellFormatting のバッジ描画は維持され、テレメトリ閲覧も妨げない。
- **`[Select All]` からの除外**: gate 行・blocked 行は `IsGate` / `Blocked` タグで判定し、bulk-check の対象外（常に未チェック）。同様に `Run Selected` 集計でも gate / blocked 行は `continue` でスキップされる。

### Groups バー (3.2.0+)

Profile CSV に `Group` 列があり値が入っている場合、Header 直下に `[Run: <GroupName>]` ボタンを CSV 出現順で並べる。クリックで全 Group メンバーの Order を集めて `Action=RunGroup`、AutoPilot=true 強制で発火。横幅は 130px × N、最大 ~6 個まで panel に収まる (auto-wrap 無し、YAGNI 原則)。

### Footer

| 行 | 要素 |
|---|---|
| 上段 | `[Select All]` (Enabled=1 行のみ ✓) / `[Clear All]` / WaitSec NumericUpDown / `[Run Selected (N)]` 大ボタン / `[Restart Now]` |
| 下段 | `[Complete]` 大ボタン / `[Back]` |

`[Complete]` の文言は status に応じて動的変化:

- Issue (Error / Partial / 未実行で ✓ された Pending) があれば → 黄色 + "Complete with N issues"
- 何も実行していない (全 Pending) → 黄色 + "Complete (nothing executed)"
- それ以外 → 緑 + "Complete"

`[Complete]` 押下時、issue があれば Yes/No 確認ダイアログを出し、Yes ならフィナライズ進行。

### PendingFinalize 警告

`-PendingFinalize $true` で開かれた Flex は、Header の "Last finalized:" の代わりに **赤バッジ "PENDING FINALIZE"** を表示。X/Back で閉じようとすると「checklist が生成されない / evidence が upload されない」警告ダイアログを出して操作員に Complete を促す。

## "And More..." ダイアログ (`Show-AndMoreDialog`)

420×320 のサブダイアログ。Restart PC / Export History / Regenerate Checklist / FabriqApps の 4 項目を DataGridView で表示し、選択 → `[Launch]` で Action 文字列を main.ps1 と同じ語彙で返す。アクションは Cancel / Restart / HistoryExport / RegenerateChecklist / AppsMode。

## FabriqApps ダイアログ (`Show-AppsDialog`)

`apps/<name>/<name>.ps1` 規約で `apps/` 配下の各サブディレクトリを走査し、`fabriq_operator` と `fabriq_ios` を除外した一覧を提示する。`[Launch]` で Action=`Launch`, AppName, AppPath を返し、main.ps1 が `& $appPath` で起動する。

## モジュール ログビューワ (`log_viewer.ps1`, kernel 3.6.0+)

FlexProfile グリッドの各行 `[Log]` ボタンから呼ばれる **presentation-only** なビューワ。別ログ stream は作らず、Show-* 系が既に記録した **エントリ別テレメトリ JSONL**（`logs/telemetry/<SessionID>/modules/<seq>_<name>.jsonl`）を read-only で読み、`type=show.<level>` の `tag` / `msg` を level 別に色分けして modal RichTextBox に描画する。提供関数は 2 つ:

- `Get-ModuleTelemetryLog -Order [-ModulesDir]` — 指定 Profile Order の **最新 run** の `@{ Level; Tag; Message; Ts }` 配列を返す純データ取得関数。`-ModulesDir` はテスト用に注入可能で、省略時は現セッションの telemetry modules フォルダを既定とする。
- `Show-ModuleLogViewer -Order [-ModuleName]` — 上記を読んで modal フォーム（820×580, Sizable）に描画する。**非破壊**で、FlexProfile の状態を変えず呼び出し元はダッシュボードに留まる。

### 堅牢性契約 (t-0074 設計)

- **現セッションのみ参照**: `$script:SessionID` フォルダのみを読む。resume 時に作られるオーファンフォルダや他 Profile の run は混入しない。
- **エントリ識別は `envelope.start` の `profileOrder`**（Profile Order）を使う。filename の seq は `__RESTART__` 跨ぎでリセットして衝突するため不使用。`profileOrder` が 0 でない場合のみ採用し、無ければ `order` フィールドをフォールバックとして読む。
- **同一 Order の複数 run は最新優先**: `LastWriteTimeUtc` 最大のファイルを選択（実時計なので `__RESTART__` 耐性がある）。
- **壊れた JSONL 行はスキップ**（部分書き込み）で致命化しない。`ConvertFrom-Json` 失敗行は continue。
- **描画は 5000 行上限**（`$script:LogViewerMaxLines`）。超過時は切り捨て、ドロップ数を warning 色で通知する。
- level → 文字色: error=ARGB(198,40,40) / warning=ARGB(180,95,6) / success=ARGB(46,125,50) / skip=ARGB(120,120,120) / info=`$fgText`。ログが空のときは「No log captured for this entry.」を表示。
- フッタに「Values are redacted; see the raw transcript for unmasked detail.」の注記 + `[Close]`。

kernel 公開 API は不変で、テレメトリはこのビューワが read-only 消費するのみ（KERNEL_API.md に telemetry セクションは無い）。

## 実行ツールバー (`execution_toolbar.ps1`, 3.4.0+)

3.4.0 で retire した `kernel/ps1/status_monitor.ps1`（`Start/Stop-StatusMonitor`）の **in-process 後継**。旧モニタの視覚的アイデンティティ（ダークテーマ / Consolas / シアン GroupBox タイトル / `[OK]` `[!!]` `[--]` マーカー / Surkitinisme アートパネル）をそのまま再現しつつ、以下を改善している:

- kernel の powershell.exe 内の専用 STA Runspace 上で動作し、Defender / ASR の子プロセス制限を回避する。
- PC Info を `$FabriqToolbarShared.TargetHostInfo`（`Set-SelectedHostEnvironment` が `Update-ExecutionToolbar` 経由で push）+ ライブ OS クエリ（`Get-NetIPAddress` / `Get-Printer` 等）から構成し、書き出し済み `status.json` への依存を排除した。
- 旧モニタ同様、アートパネルは `art_pulse.txt` / `art_sentences.txt` / `silence.flag` / `status.json` をディスクから直接読む。

### 公開関数（kernel main.ps1 / common.ps1 から呼ばれる）

- `Show-ExecutionToolbar` — ツールバーを起動（main.ps1 が旧モニタの代わりに呼ぶ）。
- `Hide-ExecutionToolbar` — ツールバーを閉じる。
- `Update-ExecutionToolbar [-ExecutionState 'Idle'|'Running'] [-ModuleName <s>] [-TargetHostInfo <hashtable>]` — 状態反映。`Idle` で Skip / Gyotaq ボタン無効、`Running` で有効。main.ps1 はバッチ開始/各モジュール直前で `Running`、完走で `Idle` を渡す。
- 補助: `Get-FabriqHostInfoFromEnv` — `SELECTED_*` 環境変数から `TargetHostInfo` ハッシュテーブルを再構成（host 選択変更時 / 起動時に使用）。

### ステータスバーのアクションボタン

- **`Skip`**（オレンジ ARGB(255,170,60)）: async モジュールの skip を要求する flag を書く。tooltip「Request async module skip. Only effective for modules running after `__ASYNC__` marker.」。`__ASYNC__` マーカー以降に走るモジュールにのみ有効。
- **`Gyotaq`**（シアン）: クリックでフォームを一時的に Hide してスクリーンショットを取得（`Save-Screenshot`、保存先は `<EvidenceBasePath>\gyotaku`）し、フォームを復帰させる。
- いずれも既定は `Enabled=false`（Idle）で、`Update-ExecutionToolbar -ExecutionState 'Running'` 時のみ有効化される。

## テーマシステム (`theme.ps1`)

CentreCOM 風ライトテーマ。fabriq_evidence_manager と同じデザイン言語で揃えてある。

### カラーパレット (主要)

| 名前 | 値 (ARGB) | 用途 |
|---|---|---|
| `bgForm` | #BABEC2 | フォーム背景 |
| `bgPanel` | #4A4A4A | ヘッダーバー |
| `bgGrid` | #E1E4E7 | DataGridView 背景 |
| `bgCellAlt` | #EDEEEF | 交互行 |
| `bgHeader` | #A0A6AB | グリッドヘッダー |
| `bgButton` | #989DA1 / `bgButtonHov` #AAAEB3 | 通常ボタン |
| `bgAccent` | #4A90D9 | アクセント青 (Execute Profile 等) |
| `bgAdd` | #4CAF50 | 成功緑 (Execute Flex / Complete) |
| `bgDelete` | #C62828 | エラー赤 |
| `bgSelection` | #4CAF50 | 行選択時 |
| `bgTabPage` | #C4C8CC | タブ本体 |
| `stripeBlue/Yellow/Red` | #4A90D9 / #F2C94C / #EB5757 | ヘッダー 3 色ストライプ |

### フォント

- `fontNormal` Segoe UI 9pt
- `fontBold` Segoe UI 9pt Bold
- `fontSemiBold` Segoe UI Semibold 9pt
- `fontLarge` Segoe UI 11pt Bold
- `fontTitle` Segoe UI 14pt Bold
- `fontMono` Consolas 8.5pt

### 提供ヘルパー

- `New-StyledButton` — 角無し Flat、border = `borderColor`、accent 色のときは白文字
- `Set-GridStyle` — DoubleBuffered ON、SelectionMode=FullRowSelect、MultiSelect=false、ReadOnly=true
- `Set-TextBoxStyle`, `New-StyledLabel`, `New-StyledPanel`, `New-StyledCheckBox`, `New-StyledComboBox`, `Set-FormStyle`

これらヘルパーが Operator GUI の見た目を一手に管理している。CentreCOM 風 (キャリアグレード機器ライク) の青/黄/赤ストライプは fabriq の視覚的アイデンティティとして session_form / dashboard_form の両方に踏襲される。
