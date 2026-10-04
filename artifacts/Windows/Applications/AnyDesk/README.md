# Windows.Applications.AnyDesk.Trace.DeadDisk

AnyDeskの保存済みログから、接続要求、セッション境界、ファイル転送、Chatの本文を一覧化するArtifact。

[定義ファイル](Trace.DeadDisk.yaml)

## 入力ログ

| LogKind | 対象 | 主な記録 |
|---|---|---|
| `connection` | `connection_trace.txt` | Incoming / Outgoing、ステータス、相手方ID |
| `ad` | `ad.trace`、指定した`ad_svc.trace` | セッションの開始・終了、転送処理、クリップボードの通知や権限更新 |
| `transfer` | `file_transfer_trace.txt` | File Manager / Clipboard、start / finish / cancel、upload / download、ファイル名と進捗 |
| `chat` | `chat/*.txt` | 送信者表示、本文、日時の見出し |

既定の検索先はユーザーの`AppData/Roaming/AnyDesk/`と`ProgramData/AnyDesk/`。保存コピーやマウントしたディスクを読む場合は、対象のパスを`LogGlobs`へ指定する。`ProgramData`のパスは検索候補で、インストール版の動作検証は未実施。

## 出力

| Source | 内容 |
|---|---|
| `ConfiguredTargets` | 指定した検索パターンと検出件数 |
| `FileInventory` | 元ファイルのパス、サイズ、日時、SHA256 |
| `ParseCoverage` / `ReviewLines` | 読み取り状態、解析対象行数、未対応書式・無効日付・文字コードの確認対象 |
| `LogOverview` | フォルダごとの接続・拒否・セッション開始・転送完了・中止の記録数 |
| `Connections` | 接続方向、ステータス、相手方ID、ログ原文 |
| `ConnectionSummary` | 接続記録ごとの概要。対応表の検査に通った場合は開始・終了・転送を集約 |
| `FileTransfers` | 方式、方向、ステータス、ファイル名、原単位付きの進捗 |
| `ChatMessages` | 送信者表示、本文、日時見出しとその行番号 |
| `KeyEvents` | 選択したセッション・転送・クリップボード関連の診断イベント |
| `UnlinkedEvents` / `LinkIssues` | 関連付けられなかったイベントと対応表の問題 |

各イベントの`SourcePath`、`SourceSHA256`、`SourceLine`から元ログを参照できる。結果には実際のAnyDesk ID、パス、Chat本文が含まれる。

## Parameters

| Name | 既定値 | 用途 |
|---|---|---|
| `LogGlobs` | 定義内の検索パターン | `LogKind,Glob`のCSV。ログの種類と対象パスを指定 |
| `Accessor` | `auto` | ファイルの検索・読み取り・hash計算に使うaccessor |
| `MaxLogBytes` | `8388608`（8 MiB） | ファイルごとの読み込み上限。超過は`SizeLimit`として出力 |
| `ConnectionLinks` | CSVヘッダーのみ | 確認済みの接続記録とセッション開始・終了の対応表 |
| `TransferLinks` | CSVヘッダーのみ | 確認済みの接続記録と転送の開始・終端の対応表 |
