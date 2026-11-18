# Component Inventory

## Application Packages

### noim_converter.py
- **種別**: CLI(入口)
- **責務**: 引数解析、`run()`(読み込み → 変換 → 書き出し → 任意で期待結果との比較)、`ToolError` を `エラー: ...` にして終了コード 1 を返す。
- **依存**: `noim.converters`、`noim.errors`、`noim.input_csv`、`noim.output_csv`。

### noim.input_csv
- **種別**: 入力の読み込み
- **責務**: 入力ファイルの選択(種類ごとに最新)、CSV の読み込み、PostgreSQL 配列表記の解析、入力列名の定義(`NODE_COLUMNS`、`ENTITY_COLUMNS`、`SERVICE_COLUMNS`)。入力列の一覧は `api-documentation.md` に書いた。
- **依存**: `noim.errors`。

### noim.converters
- **種別**: 変換ロジック(中心)
- **責務**: 3 入力から 8 種のリソース(辞書)を作る。`convert_all` が入口。リソースの構造は `api-documentation.md`。
- **依存**: `noim.object_id`、`noim.errors`、`noim.input_csv`(列名の定義のみ)。

### noim.object_id
- **種別**: 採番
- **責務**: `objectId` の採番(`ObjectIdManager`)と、spec ID、CE 用 PStruct の固定 ID の定義。
- **依存**: なし。

### noim.output_csv
- **種別**: 出力の書き出し
- **責務**: `resourceInfo_<種類>.csv` の生成(`to_csv_bytes`)と、一時ファイル経由の書き込み(`write_outputs`)。
- **依存**: `noim.converters`(`RESOURCE_KINDS`)、`noim.errors`。

### noim.errors
- **種別**: 共通
- **責務**: `ToolError`(利用者に表示するエラー)。
- **依存**: なし。

## Infrastructure Packages

### infra.NoimConverterStack
- **種別**: CDK スタック(`infra/noim_stack.py`。入口は `infra/app.py`)
- **責務**: 変換ツールを動かす環境を作る。作成するものは次のとおり(論理 ID は CDK のコード上の名前)。
- **リージョン / タグ**: `ap-northeast-1` 固定。アカウントは `CDK_DEFAULT_ACCOUNT`。タグ `Project=noim-converter`、`Environment=dev`。スタック名は `NoimConverterStack`。

| 論理 ID | AWS リソース | 設定の要点 |
|---|---|---|
| `Vpc` | VPC | AZ 1 つ、公開サブネット 1 つ(/24)、NAT なし |
| `InstanceSg` | セキュリティグループ | インバウンドなし。アウトバウンドは TCP 443 の任意の IPv4 のみ |
| `DataBucket` | S3 バケット | SSE-S3、パブリックアクセスを全面ブロック、SSL 必須、削除時に中身ごと消す。使うプレフィックスは `input/` と `output/` |
| `ToolAsset` | CDK アセット(S3) | リポジトリ全体を zip にして上げる。`infra`、`tests`、`入力ファイル`、`*.md`、`*.xlsx` などを除外 |
| `InstanceRole` | IAM ロール | `AmazonSSMManagedInstanceCore`、`DataBucket` の `input/*` の読み取り(`grant_read`)、`output/*` への書き込み(`grant_put`)、アセットの読み取り |
| `Instance` | EC2 | `t3.micro`、Amazon Linux 2023、IMDSv2 必須、EBS gp3 8 GB 暗号化、UserData が変わると作り直す |
| (UserData) | 起動スクリプト | アセットを `/opt/noim` に展開、`/usr/local/bin/noim-run` を作成、`/var/noim/{input,output}` を作成 |
| `StatusCheckFailedAlarm` | CloudWatch アラーム | `AWS/EC2` の `StatusCheckFailed` が 5 分連続で 0 超。通知先なし |
| (出力) | CfnOutput | `BucketName`、`InstanceId` |

- **無いもの**: Lambda、API Gateway、CloudFront、Cognito、DynamoDB、ALB、ドメイン、アラーム用の S3 プレフィックス。可視化のためのリソースは既存スタックに無い。
- **操作**: デプロイは人が CloudShell から `npx aws-cdk@2 deploy`(手順は `infra/README.md`)。

## Shared Packages

なし(`noim.errors` と `noim.object_id` が共通部品)。

## Test Packages

### tests
- **種別**: unittest
- **責務**: `test_converters.py`(変換)、`test_input.py`(入力)、`test_output.py`(出力)。`def test_` は合計 42。今回のスキャンでは実行していない。

## Total Count

- **Total Packages**: 4(`noim` パッケージ、`noim_converter.py`、`infra`、`tests`)
- **Application**: 6 モジュール(`noim_converter.py` と `noim/` の 5 ファイル)
- **Infrastructure**: 1 スタック(`NoimConverterStack`)
- **Shared**: 0
- **Test**: 1
