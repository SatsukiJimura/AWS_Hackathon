# Technology Stack

## Programming Languages

- Python 3 — 変換ツール本体、CDK、テスト。バージョンは未指定(`os.replace` などから 3.3 以上が必要。EC2 は Amazon Linux 2023 の `python3`)。

## Frameworks

- なし(本体は標準ライブラリのみ)。
- AWS CDK v2(Python) — `infra/`。`app` は `python3 app.py`(`infra/cdk.json`)。

## Infrastructure

- AWS(東京 `ap-northeast-1`): VPC、EC2(`t3.micro`、Amazon Linux 2023)、S3、IAM、CloudWatch アラーム、SSM(セッションマネージャー接続)。詳細は `component-inventory.md` の `infra.NoimConverterStack`。
- EC2 上では `aws s3 sync`(AWS CLI)を使う。インストールはスタックでは行わず、Amazon Linux 2023 の標準に頼る。
- デプロイは CloudShell から `npx aws-cdk@2`(Node.js が必要。グローバルインストール不要)。

## Build Tools

- 本体にビルドなし。
- CDK: `aws-cdk-lib>=2.140.0,<3.0.0`、`constructs>=10.0.0,<11.0.0`(`infra/requirements.txt`)。

## Testing Tools

- `unittest`(標準ライブラリ)。`tests/` に 3 ファイル。カバレッジ計測の設定なし。

## 標準ライブラリの使用箇所

`argparse`、`difflib`、`os`、`sys`(`noim_converter.py`)/ `csv`、`io`、`os`、`re`(`noim/input_csv.py`)/ `json`、`os`(`noim/output_csv.py`)。

## 可視化の追加で関係する現状の制約

- 本体は標準ライブラリのみという方針で作られている(外部ライブラリ無し)。可視化側が別のプログラムになるかは後続で決める。
- 入出力は CSV。出力の `data` 列は JSON 文字列なので、読む側は CSV 解析のあとに JSON 解析が要る。
