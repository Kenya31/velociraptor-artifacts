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

## 保存ログの解析例

Windows PowerShellでリポジトリのルートから実行する例。`$velo`を使用する実行ファイルへ、CSVのパスを取得済みログの作業コピーへ合わせる。A/Bは採取元端末の区別。

**出力先に新しい結果ZIPと処理用の一時ファイルを作成する。** 入力ファイルの内容は変更しない。例は既存ZIPがある場合に停止する。

```powershell
$velo = 'C:\Tools\Velociraptor.exe'
$definitions = Join-Path $PWD 'artifacts\Windows\Applications\AnyDesk'
$outputZip = 'C:\Lab\Analysis\AnyDesk-001.zip'

if (Test-Path -LiteralPath $outputZip) {
    throw 'Output ZIP already exists.'
}
if (-not (Test-Path -LiteralPath (Split-Path -Parent $outputZip))) {
    throw 'Create the analysis output folder first.'
}

$logGlobs = @'
LogKind,Glob
ad,C:/Lab/Working/AnyDesk/A/ad.trace
connection,C:/Lab/Working/AnyDesk/A/connection_trace.txt
transfer,C:/Lab/Working/AnyDesk/A/file_transfer_trace.txt
chat,C:/Lab/Working/AnyDesk/A/chat/*.txt
ad,C:/Lab/Working/AnyDesk/B/ad.trace
connection,C:/Lab/Working/AnyDesk/B/connection_trace.txt
transfer,C:/Lab/Working/AnyDesk/B/file_transfer_trace.txt
chat,C:/Lab/Working/AnyDesk/B/chat/*.txt
'@

& $velo --definitions $definitions `
    artifacts collect Windows.Applications.AnyDesk.Trace.DeadDisk `
    --args "LogGlobs=$logGlobs" `
    --output $outputZip

if ($LASTEXITCODE -ne 0) {
    throw 'Collection failed. Check the collection log.'
}
```

CSVにはヘッダーが必要。Chat未使用などで存在しないファイルは`ConfiguredTargets`の件数に反映される。累積ログの最終コピーを各端末から1組ずつ指定する。

結果ZIPの`results/`配下にSourceごとの表、`log.json`に処理ログが入る。上記の例は対応表を指定していないため、`ConnectionSummary`の開始・終了は空欄になり、`CorrelationStatus`は`NoReviewedLink`となる。

ディスクイメージを読む場合は、対象の露出パスに合わせた`LogGlobs`と`Accessor`、適切なremapが必要。実ディスクイメージからの収集は今回の検証範囲外。

## 接続・転送の対応表

開始・終了・転送を接続サマリーにまとめる場合は、原文と操作記録を照合した対応表を指定する。

| Parameter | CSVヘッダー |
|---|---|
| `ConnectionLinks` | `ConnectionPath,ConnectionSHA256,ConnectionLine,SessionPath,SessionSHA256,StartLine,StopLine` |
| `TransferLinks` | `ConnectionPath,ConnectionSHA256,ConnectionLine,TransferPath,TransferSHA256,StartLine,FinishLine` |

`FinishLine`は`finish`または`cancel`の終端行を指す。参照には元ファイルのパス・SHA256・行番号を使う。Artifactは参照先、イベント種別、順序、重複、同一分の複数要求などを検査する。

対応表を指定する場合は、収集コマンドに次を追加する。

```powershell
$connectionLinks = Get-Content -LiteralPath 'C:\Lab\Analysis\connection-links.csv' -Raw
$transferLinks = Get-Content -LiteralPath 'C:\Lab\Analysis\transfer-links.csv' -Raw
# artifacts collectの追加引数:
# --args "ConnectionLinks=$connectionLinks" --args "TransferLinks=$transferLinks"
```

`AssociationBasis=AnalystSuppliedAssociation`は解析者が指定した対応に基づく関連付け。対応に矛盾があれば`LinkIssues`へ理由を出し、対象イベントを`UnlinkedEvents`へ残す。

## 記録の扱いと対応範囲

- 時刻は元ログの値を保持し、`TimeBasis=OffsetUnknown`で出力する。ログ内にUTC offsetはなく、時刻変換は行わない。
- Chatには見出し時刻があるが、今回のメッセージ行には個別時刻がない。`HeaderRawTime`と`HeaderSourceLine`を別に保持し、`MessageWallTime`は`null`。
- 進捗の`B`表記は整数バイトとして出力する。`MiB`表記は丸め値として保持し、`AmountPrecision=RoundedDisplayAmounts`となる。この場合の`ProgressBytes`と`TotalBytes`は空欄。
- connection / transfer / ChatはUTF-16LEとして読む。`ad.trace`はASCIIの既知イベントを解析し、非ASCIIの診断行は`EncodingReview`として元ログの行番号を示す。
- 未選択の構造化診断行は`ParseCoverage`の件数、未対応書式は`ReviewLines`の元ログ参照で確認できる。
- ログの接続・転送・Chatを近い時刻や同じPIDだけで自動結合する処理はない。

## 検証

2026-10-04、次の範囲で検証した。

| 対象 | 結果 |
|---|---|
| AnyDesk | 正規のポータブル版9.8.0。Windows 10 build 17763 / Windows 11 build 26100で生成した保存ログ |
| Velociraptor | 0.76.3。テキストログの定義検査・収集に使用 |
| `Server.Utils.ArtifactVerifier` | 対象定義にエラー・警告なし |
| A/Bの保存ログ8ファイル | 接続23行、転送22行、Chat 6行、中止2行。原文・参照行と出力を照合 |
| 境界テスト | UTF-16LEの異常、無効日付、size limit、重複、同一分の複数要求、PID再利用、Chat見出しの異常、中止・Clipboard反復記録を確認 |

未検証：実ディスクイメージ / ntfs remap、インストール版・カスタムクライアント・別バージョン、録画、Prefetch内部、非ASCIIの`ad.trace`本文の完全な意味解析。

生ログ・収集ZIP・実際のIDやChat本文は、このリポジトリには含めていない。
